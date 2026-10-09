# FEATURES — что реализовано в `websoft-env-promotion`

Справочный перечень реализованных возможностей: что инструмент умеет, через что этим
управлять и где его границы. Форматы, сигнатуры и коды выхода — не здесь, а в
[`CONTRACTS.md`](CONTRACTS.md); обоснования решений — в [`ARCHITECTURE.md`](ARCHITECTURE.md);
работа руками и схемы — в [`PROCESS.md`](PROCESS.md) и [`ADMIN-START.md`](ADMIN-START.md).
Здесь перечислено только то, что уже реализовано и подтверждено тестами.

## 1. Что это и где его границы

- Переносит **конфигурацию** контура WebSoft HCM из прода в preprod и dev: значения адресов,
  строк подключения, портов, хостов, включения компонентов. не «копию прода», а результат,
  описанный правилами. Каждое изменение — декларативное правило (`action` + ссылка на цель) в
  `config/overlays/<env>/rules.json`; незакрытый обязательный ключ реестра — провал пайплайна,
  а не предупреждение.
- Один общий слой команд (`core/lib/WscCfg.ps1`) и **две реализации хранилища** — Vault KV v2
  (`impl/vault`) и S3/MinIO (`impl/minio`). Различается только `impl/<x>/store.ps1`; логика
  план/apply/verify одна на обе (сравнение реализаций — [`ARCHITECTURE.md`](ARCHITECTURE.md)).
- Только Windows PowerShell 5.1, без внешних модулей и без `pwsh`-only синтаксиса; для
  тренировки и тестов живые Vault/MinIO не нужны — есть файловый фейк-магазин (раздел 6).
- Вне инструмента и сознательно не реализовано: перенос данных (копия ФС + дамп БД), создание
  маркера `.wscfg-snapshot.ok`, запуск сервера источника. Маркер только проверяется на
  preflight, без него — exit 2 (`WSC_PREFLIGHT`). Подробный список границ — раздел 12.

## 2. Точки входа

| вход | назначение |
|---|---|
| `impl/vault/wscfg.ps1` | тонкий CLI Vault: `-Command` + общие параметры, `-StoreConfig <json>` (живое хранилище) или `-StoreRoot <каталог>` (фейк) |
| `impl/minio/wscfg.ps1` | тонкий CLI MinIO/S3: те же общие параметры без `-StoreConfig`, плюс `-Transport rest|fake`, `-Endpoint -Bucket -Region -Prefix -AccessKey -SecretKey -SecretKeyFile` |
| `impl/vault/bootstrap.ps1`, `bootstrap.cmd`, `bootstrap.sh` | оркестрация прогона: 8 параметров — `-ServerDir -Env -StoreConfig -StoreRoot -DryRun -NoStart -OutDir -HealthTimeoutSec` (таймаут по умолчанию 120 с) |
| `impl/minio/bootstrap.ps1` | то же на S3: параметров ~26 — к vault-набору добавлены `-Transport -Endpoint -Bucket -Region -Prefix -AccessKey -SecretKey -SecretKeyFile -RegistryPath -OverlayDir -BaselineFilesPath -SourceServer -Launcher -HealthUrl -ServerProcessNames -ProdValues -Manifest -RollbackOnFailure -Snapshot -SkipPreflight -SkipValidate -StartOnly -Json` (`-HealthTimeoutSec` по умолчанию 60 с) |
| `tests/run-tests.ps1` | единственная точка входа тестов: `-Filter <подстрока> -Detailed -OutDir`; exit = число провалов |
| `tests/integration/Run-FullScenario.ps1` | сквозной прогон `export -> diff -> plan -> apply -> повторный apply -> rollback` на фикстурном контуре, `-Impl vault|minio|both` |
| `tools/normalize-encoding.ps1` | навешивание UTF-8 BOM на все `.ps1`/`.psm1`: `-Repo <путь> -Check` |

Общие параметры тонких входов (одинаковы на обоих): `-Command -Options -Env -ServerDir
-SourceServer -Item -Value -SourceFile -Manifest -Against -Files -RegistryPath -OverlayDir
-BaselineFilesPath -OutDir -ExpectedVersion -DryRun -NoStart -Get -Set -Json -Verbose`;
`-Options <hashtable>` — сквозной проход для ключей вне сигнатуры, прямые параметры перекрывают
его одноимённые ключи. Разница флагов bootstrap осознанна: vault-вход держит только путь к
`StoreConfig`, а minio-вход принимает параметры S3 и ключи напрямую, поэтому у него есть и
управляющие переключатели (`-SkipPreflight`, `-SkipValidate`, `-StartOnly`, `-Snapshot`,
`-RollbackOnFailure`, `-Json`). Тренировка без живого хранилища: `-Transport fake -StoreRoot
<каталог>` (у Vault достаточно `-StoreRoot`).

## 3. Команды

Общий слой знает 8 команд; обе тонкие реализации вызывают одну и ту же
`Invoke-WscCfg -Command <имя>` (описание по командам — [`CONTRACTS.md`](CONTRACTS.md)):

| команда | назначение | нужен ли контур | нужно ли хранилище | артефакт |
|---|---|---|---|---|
| `export` | снять снимок конфига источника-контура, замаскировав секреты в `**REDACTED**` | `-SourceServer` (у Vault допускается `-ServerDir`), читается деревом | да (запись) | `baseline/**` в хранилище + локальный кэш `config/baseline/` |
| `diff` | показать расхождения `file:`-целей снимка с текущим контуром | `-ServerDir` | да, либо локальный каталог в `-Against` | список расхождений |
| `edit` | прочитать/опубликовать `rules`, `profile` или `secret:<key>` в хранилище | нет | да (CAS по `-ExpectedVersion`) | версия записи |
| `validate` | схема правил, покрытие реестра, отсутствие `**REDACTED**`/прод-значений в `config/**` | нет | **нет** — правила читаются из `config/overlays` | отчёт покрытия |
| `plan` | скомпилировать правила × реестр × профиль × baseline в план, ничего не записывая | `-Env -ServerDir` | да (нужен экспортированный baseline) | `plan.json` + `plan.md` |
| `apply` | backup -> файлы -> runtime -> БД -> verify-гейт, авто-откат при провале | `-Env -ServerDir` | да | манифест в `<ServerDir>\.wscfg-backup\<id>\` + `apply-report.json|md` |
| `rollback` | вернуть контур по манифесту | `-Manifest`, `-ServerDir` опционален (берётся из манифеста) | **нет** — на обоих входах | восстановленные файлы, `status=rolledBack` |
| `scan` | предложить кандидатов в реестр по чувствительным шаблонам | `-ServerDir` | **нет** | список (в реестр не пишет) |

Три команды — `validate`, `scan`, `rollback` — работают без хранилища; это осознанно: проверку
правил и разбор отказа можно делать на машине, куда хранилище не доступается.

## 4. Модель конфигурации

Семь `action` (полная таблица поведения — [`CONTRACTS.md`](CONTRACTS.md)):

| action | поле значения | что делает |
|---|---|---|
| `set` | `value` / `fromProfile` | установить значение |
| `secret` | `secret` (имя) | взять значение из `secrets/<env>/<key>` |
| `append` | `value` / `fromProfile` | добавить элемент (INI-ключ, JSON-массив), не затирая |
| `remove` | — | удалить ключ/узел/вхождение |
| `keep` | — | явно «не менять», но покрыть обязательный ключ |
| `runtime_override` | `value` / `fromProfile` | запись в `*.runtime/<dir>/package.runtime.json` |
| `manual` | — | пункт в `plan.manualActions[]`, сервер не трогается |

Приоритет источника значения: `secret` > `fromProfile` > `value`.

Шесть каналов цели — одна строка-ссылка на файл и на БД
([`CONTRACTS.md`](CONTRACTS.md)):

| канал | вид | пример |
|---|---|---|
| `json` | `file:<rel>::json:<dot-path>` | `file:xhttp_config.json::json:Kestrel.EndPoints.Http.Url` |
| `xpath` | `file:<rel>::xpath:<xpath>` | `file:spxml_unibridge_config.xml::xpath:/configuration/appSettings/add[@key='ConnectionString']/@value` |
| `ini` | `file:<rel>::ini:<KEY>` | `file:xHttp.ini::ini:DESIGN-DB` (дубли — полем `occurrence`) |
| `regex` | `file:<rel>::regex:<pattern>` | последний резерв, лимит — `max_matches` |
| `db` | `db:<компонент>.<тип>::<field-path>` | `db:websoft.aihub.ai_provider::endpoints[*].url` |
| `pkg` | `pkg:<каталог-компонента>::<packageKey>` | `pkg:platform/aihub_mng@websoft.aihub::enabled` |

Поля правила, влияющие на строгость: `mandatory`, `waiver`, `acked` (только для `manual`),
`optional_file` (нет файла — WARNING, не ошибка), `optional_key` (нет ключа/цели — WARNING),
`max_matches`, `reason`. Неизвестное поле правила — `WSC_PLAN` (exit 3), а не «игнорировать».

`config/overlays/<env>/profile.json` — только нечувствительные значения по `dot-path`: хост,
порт, `allowedHosts`, блок `db` (тип, строка подключения, пользователь, база, схема), `redis`,
`cache`, `aihub.baseUrl`, диапазон RTC-портов `mediasoup`, идентификаторы узла и кластера.
Секретных полей и прод-маркеров в профиле нет — это проверяется тестами (раздел 11).

`config/mandatory.registry.json` — **31 запись** и единственный источник истины verify-гейта:
`key` (уникальный), `target`, `check`, `file_optional`, `secret`, `optional_key`, `title`,
`notes`. Словарь `check` — 7 вариантов: `not_redacted`, `not_prod_value`, `not_empty`,
`equals_profile:<dot-path>`, `regex:<pat>`, `port_match`, `no_cross_contour`. `validate`
требует закрыть каждый `key` правилом с `mandatory: true` или явным `waiver`, иначе exit 3.

Версионируется в git: `config/overlays/<env>/{rules.json,profile.json}`. Не версионируется:
`config/baseline/` (кэш экспорта), `tests/out/`, `.wscfg-backup/` — политику фиксирует
`.gitignore`, а тест `ConfigCoverage` проверяет её саму.

## 5. План: предсказуемость и наблюдаемость

- `planHash` = `sha256` канонического JSON от `items + runtime + manualActions`. Поле
  `generatedAt` и служебный `ruleIndex` в хеш **не** входят: один и тот же набор правил,
  реестра, профиля и baseline даёт тот же хеш (формат — [`CONTRACTS.md`](CONTRACTS.md)).
- Состав пункта: `ruleId`, канонический `target`, `kind`, `channel`, `file`, `path`,
  `occurrence`, `match`, `action`, `oldValue` (из baseline), `newValue`, `secretName`,
  `mandatory`, `optionalFile`, `runtimeRedirect`, `source` (`baseline|profile|secret|literal`).
- `newValue == oldValue` — пункт остаётся в плане с `already = true` и в запись не
  превращается; это механизм идемпотентности, а не ошибка.
- Секции плана: `runtime` (переопределения компонентов), `manualActions` (что должен сделать
  человек), `coverage{registryTotal, covered, waived, missing}`, `warnings`, `unresolved`.
- В dry-run секреты показаны как `<secret:имя>` — план не содержит значений секретов даже
  когда хранилище доступно.
- `plan.md` — человекочитаемая версия плана для апрува в CI; читается вместо `plan.json`.
- Правило, давшее 0 фактических изменений, — `WSC_PLAN` (кроме `keep`, `manual` и «уже
  применено»): правило, которое «ни на что не влияет», не проедет незаметно.

## 6. Хранилище

Интерфейс — **9 обязательных функций** в `impl/<x>/store.ps1`: `New-WscStore`,
`Test-WscStoreAccess`, `Get-WscStoreText`, `Set-WscStoreText`, `List-WscStore`,
`Get-WscStoreVersions`, `Get-WscStoreSecret`, `Set-WscStoreSecret`, `Remove-WscStorePath`.
Общая логика работает с любой реализацией, поэтому обе проходят один и тот же набор тестов.
Логический layout един: `wscfg/v1/{baseline,overlays,secrets}`. Что строже и что проще в двух
реализациях — [`ARCHITECTURE.md`](ARCHITECTURE.md).

| аспект | Vault KV v2 | MinIO / S3 |
|---|---|---|
| версии и аудит | целочисленная версия из коробки + штатный `audit device` | Versioning бакета обязателен; версия непрозрачна, порядок — листингом `?versions` |
| CAS против параллельной записи | `options.cas = <n>`, один запрос | порядковая сверка `ExpectedVersion` + `x-amz-if-match` по ETag (сервер отвечает `412`) |
| политики на раннера | Vault-policies по путям, `deny` на секреты прода, AppRole на каждый контур | IAM по префиксам + явный `Deny` на `secrets/prod/**` |
| шифрование секретов | сервером, ключ не покидает Vault | обязательное клиентское envelope: `age`, иначе AES-256-CBC с префиксом `WSCEF1:`; открытый текст в бакете — fatal на `validate` |
| неизменяемость baseline | только история версий | Object Lock `COMPLIANCE` 30 дней |
| доступность без сервера | нужен Vault | нужен MinIO |

`Transport=fake` — не заглушка «для галочки», а песочница с той же CAS-семантикой: при
`Transport='fake'` (у Vault — при `-StoreRoot`) тонкий вход и bootstrap подключают
`core/lib/WscStoreFake.ps1` **на уровне входа**, вместо `store.ps1`, поэтому сам `store.ps1`
остаётся тонким реальным клиентом. Реальный клиент проверяется подменой единственной границы I/O
(`Invoke-WscVaultApi`, `Invoke-WscS3Api`) в области видимости теста.

## 7. Секреты

- Правило хранит **имя** секрета (`secret: <имя>`), значение читается из хранилища только на
  `apply`. В `plan`, `plan.md`, `apply-report.*`, манифесте, логах и в git значения нет: в
  dry-run подставляется `<secret:имя>`, в baseline значения секретных целей — `**REDACTED**`.
- Публикация секрета — только `wscfg edit -Item secret:<key> -Set` (значение из `-Value` или
  `-SourceFile`), с CAS по прочитанной версии.
- Клиент не читает прод-секреты: при `Env='prod'` на чтение `Test-WscStoreAccess` и
  `Get-WscStoreSecret` бросают `WSC_STORE` с текстом `prod secrets are not readable by runner`
  — даже если токен ошибочно дали широкий. Плюс слой политик хранилища (`deny`/`Deny`, таблица
  раздела 6) и гейт `validate` на открытый текст в бакете MinIO.
- Гейт `secrets_resolved` проверяет, что в контуре не осталось ни `<secret:…>`, ни `**REDACTED**`.

## 8. Работа с БД

- Три драйвера за единственной границей `core/db/Db.psm1`: `filexml`, `mssql`, `postgres`
  (сигнатуры — [`CONTRACTS.md`](CONTRACTS.md)). Выбор
  драйвера — `Get-WscDbDriverKind` по `CATALOG-STORAGE`, `DBType` и фактическому `wt_data`;
  `CatalogStorage='xdb'` или наличие `wt_data/objects/**` дают `filexml` независимо от `DBType`.
- XMD — источник типов и полей: `Read-WscXmd`/`Find-WscXmdForForm`/`Resolve-WscXmdFieldPath`
  дают имена таблиц и скалярность/массивность полей, `Get-WscXmdSensitiveFields` — кандидатов
  на маскировку в снимке.
- Изменение только под транзакцией: перед `UPDATE` — SELECT ожидаемого `OldValue`, после —
  SELECT нового; `WHERE <idColumn> = <idValue>`, где `id` взят из XMD-объекта, никогда «по
  имени». Ошибка — откат транзакции, частичных правок не бывает.
- Для каждого изменённого значения в манифест пишется `db[].undo` (драйвер, таблица, `id`,
  колонка, индекс массива или путь к XML-файлу) — из него откат делает обратную операцию.
- Контекст подключения (`Get-WscDbContext -ServerDir`) собирается из самого контура:
  `spxml_unibridge_config.xml` + `xHttp.ini`; значения секретов приходят правилами, не из CLI.
- Из переменных окружения БД-слой реально читает две: `WSC_PSQL_EXE` и `WSC_NPGSQL_DLL` (обе —
  только для postgres); `PGPASSWORD` код выставляет сам из контекста. Для MSSQL и `filexml`
  переменные не нужны вообще.

## 9. Прогон применения и откат

`impl/<x>/bootstrap.ps1` — оркестрация вокруг общего слоя; порядок шагов фиксирован
(таблица и детали — [`ARCHITECTURE.md`](ARCHITECTURE.md)):

| # | шаг | код при провале |
|---|---|---|
| 1 | preflight: PS >= 5.1, каталог, остановленные процессы, запись пробником, маркер `.wscfg-snapshot.ok`, доступ к хранилищу | 2 `WSC_PREFLIGHT` (доступ к хранилищу — 7) |
| 2 | fetch baseline/оверлеев из хранилища, версии как основа CAS | 7 `WSC_STORE` |
| 3 | компиляция правил в план + гейт `Assert-WscPlan` | 3 `WSC_PLAN` |
| 4 | backup: байтовые копии затрагиваемых файлов + манифест с хешами | 4 `WSC_APPLY` |
| 5 | правки файлов (json/xpath/ini/regex) с ре-парсингом после правки | 4 `WSC_APPLY`, авто-откат |
| 6 | runtime-переопределения (`pkg:`) | 4 `WSC_APPLY`, авто-откат |
| 7 | БД-транзакция по `id`, SELECT до и после | 4 `WSC_APPLY`, откат транзакции |
| 8 | verify-гейт `Test-Contour` | 5 `WSC_VERIFY`, авто-откат |
| 9 | запуск `xhttp_launcher` + health-проба в таймауте `-HealthTimeoutSec` | 6 `WSC_HEALTH`, откат |
| 10 | отчёт: `apply-report.json`/`.md`, `bootstrap.log` | — |

Путь прогона одним блоком (ASCII-дубль, как в [`ADMIN-START.md`](ADMIN-START.md), раздел 7):

```
prod --export--> store(wscfg/v1) --plan--> plan.md --bootstrap--> target contour
   rules/profile/secrets --> store           |                      |
                                             |-- повтор -----------> changes 0
   target contour --rollback -Manifest-----> состояние до изменений
```

- Идемпотентность подтверждена числами: первый apply — `changes: 17`, повторный — `changes: 0`
  и exit 0 (`tests/out/integration-vault.md`, `tests/out/integration-minio.md`).
- Авто-откат — часть модели, а не опция: ошибка записи, провал verify-гейта и провал health-пробы
  одинаково ведут `Rollback-WscApply` (побайтовое восстановление файлов, включая удаление ранее
  не существовавших, плюс обратные БД-операции из `db[].undo`); хранилище для отката не нужен.
- Ручной откат — `wscfg.ps1 -Command rollback -Manifest <ServerDir>\.wscfg-backup\<id>\manifest.json`;
  `-ServerDir` опционален. На фикстуре откат возвращает 18 файлов байтово-идентично.
- Управляющие флаги bootstrap: `-DryRun`, `-NoStart`, `-HealthTimeoutSec`, `-OutDir`; у minio-входа
  дополнительно `-Snapshot`, `-SkipPreflight`, `-SkipValidate`, `-StartOnly`, `-RollbackOnFailure`,
  `-Json`, `-Launcher`, `-HealthUrl`, `-ServerProcessNames`, `-ProdValues`, `-Manifest`. В общем
  слое `Invoke-WscCfg -Command apply` запуск и health **не** выполняются — для реального контура
  нужен именно `bootstrap.ps1`.
- Коды возврата пригодны для мониторинга: `0` успех (включая 0 изменений), `2` preflight, `3` план,
  `4` apply, `5` verify-гейт, `6` health, `7` хранилище/CAS, `8` подписанный путь, `9` аргументы
  CLI; `bootstrap.cmd` отдаёт код как есть (таблица — [`CONTRACTS.md`](CONTRACTS.md)).

## 10. Контроль правильности

- verify-гейт `Test-Contour` — единственный общий гейт на `apply`, и он не «догадывается», что
  важно: проверяет ровно записанное в реестре плюс 11 обязательных `Id` (`no_redacted`,
  `no_prod_value_leak`, `db_not_prod`, `mandatory_check`, `ports_match`,
  `allowed_hosts_match_profile`, `hosts_match_profile`, `cache_not_prod`, `rtc_no_collision`,
  `no_signed_files_touched`, `secrets_resolved`; сигнатура — [`CONTRACTS.md`](CONTRACTS.md)).
- Поверх обязательных `Id` выполняется машиночитаемая `check` каждой записи реестра; на фикстуре
  это **41 проверка**, все проходят на зелёном прогоне.
- Защита подписанного: `file:`-цель внутрь `platform/`, `components/`, `*_external/` запрещена
  компилятору для **всех** action, включая `keep` и `manual`, — `WSC_SIGNED_PATH` (exit 8); то же
  проверяет гейт `no_signed_files_touched`. Легальная замена — только `pkg:` в
  `*.runtime/<dir>/package.runtime.json` и только ключи `enabled`, `onFailure`, `roles`
  ([`ARCHITECTURE.md`](ARCHITECTURE.md)).
- Redaction при `export`: маскируются поля реестра с `secret: true` и значения под
  `Get-WscSensitivePattern` (пароль/токен/key/secret/сегменты строк подключения); файл пишется
  с исходной кодировкой (BOM и переносы сохранены), `baseline/meta.json` хранит `OrigHash`,
  `RedactedCount`, `Encoding` на каждый файл ([`CONTRACTS.md`](CONTRACTS.md)).
- `validate` ловит проблему до записи в контур: схема правил, покрытие всех 31 ключа,
  `**REDACTED**`, прод-значения и энтропийные секреты в `config/**` и оверлеях.

## 11. Эксплуатация и автоматизация

**CI.** Оба пайплайна Gitea Actions запускаются только вручную (`workflow_dispatch`), матрицей по
контурам `preprod`/`dev`, и оба содержат ровно один опасный шаг:

- Vault — одна задача `promote`; ручной апрув сделан `environment` с required reviewer,
  путь контура берётся из машинной переменной раннера `WSC_SERVER_DIR_<ENV>`, адрес — из
  `VAULT_ADDR` ([`gitea-promote-prod-to-preprod.yml`](../impl/vault/pipeline/gitea-promote-prod-to-preprod.yml)).
- MinIO — две задачи `plan` -> `apply`, между ними явный шаг-гейт: сверяет input `approve` с
  точной строкой `APPROVE-<env>` и при несовпадении даёт `exit=9`; есть `dry_run_only`
  (по умолчанию `true`) и `source_server`; ключи и эндпоинт — из секретов
  ([`gitea-promote-prod-to-preprod.yml`](../impl/minio/pipeline/gitea-promote-prod-to-preprod.yml)).
- Артефакты прогона: `plan.json`, `plan.md`, `apply-report.json`, `apply-report.md`,
  `bootstrap.log`; у MinIO дополнительно `bootstrap-report.json` и манифесты откатов.

**Полностью автоматический сценарий без администратора** описан отдельно —
[`ADMIN-START.md`](ADMIN-START.md), раздел 11: «клон-робот перенёс ФС+БД и создал маркер ->
планировщик или CI дёргает bootstrap -> plan -> apply -> verify-гейт -> health -> отчёт».
Безопасность без человека держится на verify-гейте, health-пробе и авто-откате; код возврата
задачи равен коду выхода из таблицы, поэтому мониторинг реагирует без разбора логов.

**Тестовая инфраструктура.** 479 статических проверок `It` в 25 файлах `*.Tests.ps1`, единая
точка входа `tests/run-tests.ps1` (exit = число провалов, пишет `tests/out/junit.xml`,
`test-results.json`, `run-tests.log`). Pester 5 на машине нет, поэтому используется собственный
`tests/Harness.ps1` — подмножество API Pester 5, которое принимает и настоящий Pester. Фикстурный
контур `tests/fixture/New-FixtureContour.ps1` (с `-Real <контур>` — на реальном образце);
сквозной `tests/integration/Run-FullScenario.ps1 -Impl both` прогоняет реальные тонкие входы и
bootstrap через весь путь, включая негативные сценарии (убранное обязательное правило -> exit 3,
прод-строка подключения -> exit 5 с авто-откатом), и сверяет коды выхода. Политики конфигурации
проверяет `tests/config/ConfigCoverage.Tests.ps1` (24 проверки: схема и уникальность `Id`,
закрытие целей реестром и наоборот, отсутствие секретов и прод-маркеров в профилях, детерминизм
`planHash`, негативы гейта).

**Код-политика репозитория** (полный текст — [`../README.md`](../README.md), раздел
«Код-политика»): только PS 5.1, `.ps1`/`.psm1` в UTF-8 с BOM, `core/lib/*.ps1` — только
`function`, ошибки исключительно `throw (New-WscError -Code ...)`, запрет записи внутрь
`platform/`/`components/`, документация на русском.

## 12. Чего решение сознательно не делает

- Не переносит данные (копия ФС + дамп БД) и не создаёт маркер `.wscfg-snapshot.ok` — только
  требует их на preflight.
- Общий `apply` (`Invoke-WscCfg -Command apply`) не подставляет секреты в dry-run, не запускает
  сервер и не делает health-пробу: `-NoStart` там принимается, но ни на что не влияет.
- Advisory-lock («один прогон на контур») не реализован: нет ни `sp_getapplock`, ни аналога.
  Взаимная исключительность держится CAS (exit 7), транзакцией БД и ручным апрувом окружения.
- `diff` сравнивает только `file:`-цели и только с `-ServerDir`; `db:`-цели в diff не входят.
- Живые Vault, MinIO, MSSQL и Postgres не проверялись: тесты подменяют единственные границы I/O.
  Не подтверждены HTTP-коды Vault при CAS/404, разбор stdout `vault.exe`, живой AppRole-логин,
  подпись SigV4, поведение `x-amz-if-match`, Object Lock compliance.
- Имена хостов контуров в `config/overlays/*/profile.json` — заглушки (`hcm-dev.local`,
  `dev-db.local` и т. п.), перед боем их нужно заменить.
- Схемы mermaid в этом документе не используются намеренно: при конвертации в Word они
  превращаются в листинг, поэтому путь прогона дан ASCII-блоком (раздел 9).
- Открытые вопросы и неподтверждённые допущения (порядок элементов массивов в SQL как
  `ORDER BY id`, имена таблиц из XMD) — в [`ARCHITECTURE.md`](ARCHITECTURE.md).
- Известное расхождение документации: [`PROCESS.md`](PROCESS.md) (раздел 1) приписывает
  постановку маркера администратору, а [`ADMIN-START.md`](ADMIN-START.md) (разделы 6, 7, 10, 13)
  — клону-роботу; здесь основным считается вариант клона, ручной — исключение для песочницы.

## 13. Документация и подтверждённые числа

Карта документов и кто за что отвечает:

| документ | источник истины в |
|---|---|
| [`../README.md`](../README.md) | быстрые старты, параметры входов, коды выхода, код-политика |
| [`CONTRACTS.md`](CONTRACTS.md) | форматы, сигнатуры, exit code — единственный источник истины |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | модель, потоки, сравнение Vault/MinIO, ограничения |
| [`PROCESS.md`](PROCESS.md) | процесс целиком, схемы, FAQ |
| [`ADMIN-START.md`](ADMIN-START.md) | путь администратора руками, песочница, авто-сценарий |
| [`../impl/vault/layout.md`](../impl/vault/layout.md), [`../impl/minio/layout.md`](../impl/minio/layout.md) | раскладка конкретного хранилища |
| [`FEATURES.md`](FEATURES.md) (этот файл) | что реализовано и где границы — без дублирования форматов |

Артефакты прогона после автоматического запуска: `plan.json`, `plan.md`, `apply-report.json`,
`apply-report.md`, `bootstrap.log`, `bootstrap-report.json` (MinIO), манифест отката
`<ServerDir>\.wscfg-backup\<id>\manifest.json`, `tests/out/junit.xml`,
`tests/out/integration-<impl>.md`.

Числа-доказательства (из закоммиченных прогонов и конфигурации):

| число | что это | откуда |
|---|---|---|
| 31 | обязательных ключей в реестре | `config/mandatory.registry.json`, [`ARCHITECTURE.md`](ARCHITECTURE.md) |
| 60 | пунктов в плане фикстуры | `tests/out/integration-vault.md`, `integration-minio.md` |
| `sha256:ef639670d86dad44f2cc0cb5b4ff526405bd926bdfbf2f20a7a8b9333477b406` | `planHash` фикстуры, одинаковый у обеих реализаций | те же отчёты |
| 17 | изменений при первом apply | те же отчёты |
| 0 | изменений при повторном apply | те же отчёты |
| 41 | проверок verify-гейта на фикстуре | те же отчёты |
| 18 | файлов, возвращённых откатом байтово-идентично | те же отчёты |
| 479 | статических проверок `It` в тестовом наборе | [`PROCESS.md`](PROCESS.md) (раздел 8) и подсчёт по `tests/**/*.Tests.ps1` |
| 8 / 9 / 11 | команд общего слоя / функций интерфейса хранилища / обязательных `Id` гейта | [`CONTRACTS.md`](CONTRACTS.md) |
| 8, 9 | отдельные сигналы «подписанный путь» и «неверные аргументы CLI» для мониторинга | [`CONTRACTS.md`](CONTRACTS.md) |
