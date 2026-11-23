# Практика №4. Миграция Mattermost: MySQL 8.0 → PostgreSQL 15 (v9.5 → v12.0.1)

## Структура репозитория

```
.
├── .github/workflows/migration.yml     # идемпотентный пайплайн
├── docker-compose.prod.yml             # Mattermost 12.0.1 + PostgreSQL 15
├── legacy/
│   ├── docker-compose.yml              # Mattermost 9.5 + MySQL 8.0
│   └── .env.example                    # шаблон паролей для legacy (сам .env не коммитится)
├── migration/migration.load.template   # конфиг pgloader (include drop)
└── .gitignore
```

---

## Порядок выполнения

### Шаг 1. Legacy-стек

```bash
cd legacy
cp .env.example .env        # впишите пароли (см. комментарий в файле)
docker compose up -d
```

Откройте http://localhost:8065, создайте администратора, команду, приватный канал
и напишите 3–5 сообщений. Загрузите аватар, чтобы потом показать, что файлы на месте.

> Если legacy-стек у вас **уже запущен** со старым файлом (пароли прямо в YAML), ничего
> не пересоздавайте. Просто впишите в `legacy/.env` те же пароли, что были в YAML.
> Имена томов и сети я оставил прежними, данные никуда не денутся.

Проверьте, что имя compose-проекта — `legacy`:

```bash
docker inspect mattermost-mysql --format '{{ index .Config.Labels "com.docker.compose.project" }}'
# должно вывести: legacy
```

Если выводится другое имя, замените `name: legacy` в начале `docker-compose.prod.yml` на него.

### Шаг 2. GitHub Secrets

Settings → Secrets and variables → Actions → New repository secret:

| Секрет | Значение |
|---|---|
| `DB_MYSQL_ROOT_PASSWORD` | root-пароль MySQL (как `MYSQL_ROOT_PASSWORD` в `legacy/.env`) |
| `DB_POSTGRES_PASSWORD` | новый пароль для PostgreSQL (придумайте) |

Пароли — **только латинские буквы и цифры**: они подставляются в URL подключения.

### Шаг 3. Self-hosted runner

Раннер должен стоять **на той же машине**, где запущен legacy-стек
(Settings → Actions → Runners → New self-hosted runner, выполнить команды оттуда).
Пользователь раннера должен иметь доступ к Docker:

```bash
sudo usermod -aG docker $USER   # затем перезапустить раннер / перелогиниться
```

### Шаг 4. Запуск

```bash
git checkout -b migration
git add .
git commit -m "Practice 4: idempotent Mattermost migration"
git push -u origin migration
```

Пуш в ветку `migration` запускает пайплайн. Затем запустите его **второй раз**
(Actions → workflow → Run workflow): шаги бэкапа и pgloader будут пропущены (`skipped`),
и прогон пройдёт заметно быстрее.

### Шаг 5. Проверка (чек-лист)

```bash
docker ps                                      # mattermost + mattermost-postgres, mysql нет
ls ~/mattermost-backups/                       # бэкап MySQL
ls -la ~/.mattermost_migration_completed       # флаг-файл
```

Откройте http://localhost:8065: сообщения и аватары на месте, в «О программе» указана версия 12.0.1.

---

## Что пайплайн делает и чем отличается от шаблона из методички

| # | Шаг | Комментарий |
|---|---|---|
| 1 | Check Migration Status | Флаг лежит в `$HOME`, а не в `/home/runner` (пользователь раннера может называться иначе). Если флаг потерян, но MySQL уже удалён и в PostgreSQL есть данные, флаг восстанавливается и пайплайн не падает. |
| 2 | Backup Legacy MySQL | Старый Mattermost останавливается, чтобы во время переноса никто не писал в БД. Дамп делается с `--single-transaction`, сжимается и проверяется, что он не пустой. Пароль передаётся через `MYSQL_PWD`, а не через `-p`. Бэкапы хранятся в `~/mattermost-backups`, потому что рабочую папку `actions/checkout` очищает при каждом запуске. |
| 3 | Prepare MySQL auth | Официальный образ pgloader **не поддерживает `caching_sha2_password`** (плагин по умолчанию в MySQL 8.0, это написано в документации Mattermost). Шаг переключает root на `mysql_native_password`. Команду можно повторять сколько угодно раз. |
| 4 | Start Target PostgreSQL | Вместо `sleep 10` используется `--wait`: ждём healthcheck (`pg_isready`). |
| 5 | Run pgloader | Конфиг лежит в файле, потому что флагами командной строки не задать перенос в схему `public`: по умолчанию pgloader кладёт таблицы в схему `mattermost`, и Mattermost их не находит. Внутри **`include drop`** (идемпотентность) и `CAST json → jsonb`. Файл с паролями создаётся во временной папке и удаляется через `trap`. |
| 6 | Verify migrated data | Число строк в Users / Teams / Channels / Posts в MySQL и PostgreSQL должно совпасть, иначе пайплайн падает. |
| 7 | Mark Migration as Completed | Флаг ставится только после того, как проверка данных прошла. |
| 8 | Deploy Upgraded Stack | `up -d --remove-orphans` удаляет `mattermost-mysql`. Для этого у prod-файла **то же имя проекта** (`name: legacy`), иначе MySQL не считается «сиротой», а имя контейнера `mattermost` конфликтует со старым. |
| 9 | Health Checks | До 5 минут опрашивается `/api/v4/system/ping` (ответ 200 и `"status":"OK"`), выводятся версия и число сообщений, проверяется, что MySQL удалён. |

Ещё исправлено:
- в шаблоне pgloader подключался к `mattermost-postgres` как пользователь `postgres`, а в compose его нет; везде используется `mmuser`;
- `MM_TEAMSSETTINGS_…` → `MM_TEAMSETTINGS_…` (лишняя S, настройка не применялась);
- в compose-файлах больше нет паролей, они приходят из `.env` / GitHub Secrets;
- `concurrency` в workflow не даёт запустить два прогона одновременно.

### Соответствие критериям оценки

- **Docker (3):** новые контейнеры работают, MySQL удалён через `--remove-orphans`; общий том `legacy_mattermost_data` (`external: true`); всё в bridge-сети `mattermost_network`, pgloader запускается с `--network=mattermost_network`.
- **Секреты (2):** только `${{ secrets.* }}`; пароли не попадают в аргументы команд, а GitHub маскирует их в логах как `***`.
- **Идемпотентность (3):** флаг-файл; `include drop`; `--remove-orphans`.
- **Тесты (2):** автоматическая проверка ping API и сверка числа строк; демонстрация сообщений в UI.

---

## Если что-то пошло не так

**pgloader: ошибка аутентификации MySQL.** Проверьте, что `DB_MYSQL_ROOT_PASSWORD` совпадает с реальным root-паролем.

**Mattermost 12 не стартует после миграции** (в логах ошибки миграций схемы). Шаблон
методички создаёт схему PostgreSQL прямо из MySQL, и в редких случаях Mattermost её не принимает.
Откат без потерь:

```bash
docker compose -f docker-compose.prod.yml down          # без -v
docker volume rm legacy_postgres_data
rm ~/.mattermost_migration_completed
cd legacy && docker compose up -d                       # MySQL-том не удалялся, данные целы
```

После этого можно перезапустить пайплайн или пройти официальный путь из методички
(`migration-assist mysql …` → `migration-assist postgres … --run-migrations --mattermost-version=v9.5.0`
→ `migration-assist pgloader … > migration.load` → `pgloader migration.load`
→ `migration-assist postgres post-migrate --create-indexes …`).
В крайнем случае есть бэкап в `~/mattermost-backups`.

**Полный сброс, чтобы прогнать всё с нуля:**

```bash
docker compose -f docker-compose.prod.yml down
docker volume rm legacy_postgres_data
rm -f ~/.mattermost_migration_completed
cd legacy && docker compose up -d
```
