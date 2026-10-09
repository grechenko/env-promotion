# WebSoft HCM: перенос конфигурации prod -> preprod -> dev

Инструмент переносит **конфигурацию** контура WebSoft HCM из прода в preprod и dev.
Данные (копия файловой системы + дамп БД) переносятся вне инструмента — он проверяет лишь
маркер `.wscfg-snapshot.ok`. Конфигурация никогда не переносится «как есть»: каждое
изменение описывается правилом контура, а незакрытые обязательные ключи ломают пайплайн.

- Архитектура и обоснование решений — [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).
- **Форматы, сигнатуры, коды выхода** — [`docs/CONTRACTS.md`](docs/CONTRACTS.md), единственный
  источник истины. Реализация не вправе менять формат без правки этого файла.
- **Процесс целиком, схемы и FAQ для новичка** — [`docs/PROCESS.md`](docs/PROCESS.md).
- **С чего начать администратору: распаковка, первая команда, путь руками по шагам** — [`docs/ADMIN-START.md`](docs/ADMIN-START.md).
- **Что умеет решение: полный перечень возможностей** — [docs/FEATURES.md](docs/FEATURES.md).

## Требования

Windows PowerShell **5.1** (`powershell.exe`), доступ на запись в каталог целевого сервера,
остановленные процессы `xhttp_64` / `xhttp_launcher` / `dotnetcore`, доступ к хранилищу
(Vault или MinIO — на выбор, см. ниже).

## Быстрый старт (Windows)

```powershell
cd D:\AI\cicdconf\websoft-env-promotion

# 1) проверка правил и покрытия реестра — хранилище не нужен
powershell -NoProfile -ExecutionPolicy Bypass -File impl/vault/wscfg.ps1 -Command validate -Env dev

# 2) dry-run: собрать план по целевому контуру, ничего не записывая
# (plan берёт baseline из хранилища — без экспорта он падает с кодом 2)
powershell -NoProfile -ExecutionPolicy Bypass -File impl/vault/wscfg.ps1 -Command plan `
  -Env dev -ServerDir "D:\WS\preprod\WebSoftServer" -OutDir ".\out\dev" `
  -StoreConfig "C:\secrets\wscfg-vault-dev.json"

# 3) полный прогон контура: preflight -> plan -> apply -> start + health -> отчёт
impl\vault\bootstrap.cmd -ServerDir "D:\WS\preprod\WebSoftServer" -Env preprod `
  -StoreConfig "C:\secrets\wscfg-vault-preprod.json" -OutDir ".\out\preprod"
```

`bootstrap.cmd` — обёртка над `bootstrap.ps1`; `bootstrap.sh` — то же для CI-раннеров
(`pwsh`, с фолбэком на `powershell.exe`). Без живого хранилища всё прогоняется на файловом
фейке: `-StoreRoot <каталог>` (у minio-входа — `-Transport fake`) вместо `-StoreConfig`.

## Реализации и точки входа

| вход | что это |
|---|---|
| `impl/vault/wscfg.ps1` | тонкий вход Vault: общие параметры (ниже) плюс `-StoreConfig <json>` (живое хранилище) и `-StoreRoot <каталог>` (файловый фейк) |
| `impl/minio/wscfg.ps1` | тонкий вход MinIO/S3: общие параметры без `-StoreConfig`, плюс `-Transport rest|fake` (по умолчанию `rest`; `fake` или `-StoreRoot` — файловый фейк), `-Endpoint -Bucket -Region -Prefix -AccessKey -SecretKey -SecretKeyFile` |
| `impl/vault/bootstrap.ps1`, `impl/minio/bootstrap.ps1` | оркестрация контура: `-ServerDir -Env -StoreConfig|-StoreRoot -DryRun -NoStart -OutDir -HealthTimeoutSec` |

Различается только `impl/<x>/store.ps1`; логика команд одна — `core/lib/WscCfg.ps1`. Оба
тонких входа принимают `-Command <имя команды>` и один и тот же набор параметров:
`-Env -ServerDir -SourceServer
-Item -Value -SourceFile -Manifest -Against -Files -RegistryPath -OverlayDir
-BaselineFilesPath -OutDir -ExpectedVersion -DryRun -NoStart -Get -Set -Json -Verbose`
(`-Verbose` — общий параметр `CmdletBinding`, `-ExpectedVersion` приводится к int) плюс
`-Options <hashtable>` как сквозной проход для ключей, не вынесенных в сигнатуру, — прямые
параметры перекрывают одноимённые ключи `-Options`. Минимум по командам: `export` —
хранилище + `-SourceServer` (у Vault допускается и `-ServerDir`), `diff` — `-ServerDir` плюс
хранилище либо локальный каталог в `-Against`, `edit` — хранилище + `-Env` + `-Item`,
`validate` — только `-Env` (правила читаются из `config/overlays`, хранилище не нужен),
`plan`/`apply` — `-Env -ServerDir` плюс хранилище с экспортированным baseline, `rollback` —
только `-Manifest`, `scan` — только `-ServerDir`.

## Ручная работа администратора

### export и diff — снять прод-конфиг и посмотреть, что поехало

`export` читает контур-источник деревом, маскирует секретные цели в `**REDACTED**`, пишет
снимок в хранилище и в локальный кэш `config/baseline/`; идемпотентен (неизменённый файл не
получает новую версию). `diff` сравнивает значения `file:`-целей baseline с текущим контуром
(`db:`-цели в diff не входят).

```powershell
$vault = "impl\vault\wscfg.ps1"; $minio = "impl\minio\wscfg.ps1"

# export — Vault: -StoreRoot это файловый фейк, живое хранилище — -StoreConfig <json>
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command export `
  -SourceServer "D:\WS\prod\WebSoftServer" -StoreRoot ".\store"

# export — MinIO живым S3; файловый фейк — заменить -Endpoint/-Bucket/ключи на -Transport fake -StoreRoot
powershell -NoProfile -ExecutionPolicy Bypass -File $minio -Command export `
  -SourceServer "D:\WS\prod\WebSoftServer" -Endpoint "https://minio.local" `
  -Bucket "websoft-configs" -AccessKey $env:WSC_S3_AK -SecretKey $env:WSC_S3_SK

# diff — baseline из хранилища против текущего контура (db: цели сравниваются только через -ServerDir)
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command diff `
  -ServerDir "D:\WS\dev\WebSoftServer" -StoreRoot ".\store"
powershell -NoProfile -ExecutionPolicy Bypass -File $minio -Command diff `
  -ServerDir "D:\WS\dev\WebSoftServer" -Transport fake -StoreRoot ".\store"
```

### edit и `config/overlays/<env>/rules.json`

Правила и профили редактируются в `config/overlays/<env>/rules.json` и
`config/overlays/<env>/profile.json` (git-версионируемый источник) и публикуются в хранилище;
секреты — только через `edit`. Неизвестное поле правила — `WSC_PLAN`; таргеты внутрь
`platform/` и `components/` запрещены для всех action, переопределение компонента — только
`pkg:` в `platform.runtime|components.runtime/<dir>/package.runtime.json` и только ключи
`enabled`, `onFailure`, `roles`.

```powershell
# чтение правил из хранилища: вернёт Text и Version (нужен для CAS -ExpectedVersion)
powershell -NoProfile -ExecutionPolicy Bypass -File impl\vault\wscfg.ps1 -Command edit `
  -Env dev -Item rules -Get -StoreRoot ".\store" -Json

# публикация отредактированных правил (схема проверяется на записи) + CAS по версии
powershell -NoProfile -ExecutionPolicy Bypass -File impl\vault\wscfg.ps1 -Command edit `
  -Env dev -Item rules -Set -SourceFile ".\config\overlays\dev\rules.json" `
  -ExpectedVersion 4 -StoreRoot ".\store"

# запись секрета контура (в git и в baseline он не попадёт никогда); MinIO — те же
# параметры, хранилище задаётся своими ключами (-Transport fake -StoreRoot для фейка)
powershell -NoProfile -ExecutionPolicy Bypass -File impl\minio\wscfg.ps1 -Command edit `
  -Env dev -Item secret:db-password -Set -Value $env:NEW_DB_PASSWORD `
  -Transport fake -StoreRoot ".\store"
```

### validate, plan, apply, rollback, scan

```powershell
$vault = "impl\vault\wscfg.ps1"; $minio = "impl\minio\wscfg.ps1"

# схема правил, покрытие реестра (31 ключ), **REDACTED**/прод-значения в config/**;
# правила читаются из config/overlays — одинаково на обоих входах, хранилище не нужен
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command validate -Env preprod

# dry-run: plan.json + plan.md, хеш плана, coverage.missing; baseline обязателен из
# хранилища (без export — выход 2), остальное различается только параметрами хранилища
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command plan -Env preprod `
  -ServerDir "D:\WS\preprod\WebSoftServer" -OutDir ".\out\preprod" `
  -StoreConfig "C:\secrets\wscfg-vault-preprod.json"
powershell -NoProfile -ExecutionPolicy Bypass -File $minio -Command plan -Env preprod `
  -ServerDir "D:\WS\preprod\WebSoftServer" -OutDir ".\out\preprod" `
  -Endpoint "https://minio.local" -AccessKey $env:WSC_S3_AK -SecretKey $env:WSC_S3_SK

# применение: backup -> файлы -> runtime -> БД -> verify-гейт; при провале авто-откат
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command apply -Env preprod `
  -ServerDir "D:\WS\preprod\WebSoftServer" -OutDir ".\out\preprod" `
  -StoreConfig "C:\secrets\wscfg-vault-preprod.json"   # запуск сервера — на bootstrap

# откат по манифесту из <ServerDir>\.wscfg-backup\<id>\manifest.json: -Manifest обязателен,
# -ServerDir опционален (читается из манифеста), хранилище не нужен — одинаково на обоих входах
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command rollback `
  -Manifest "D:\WS\preprod\WebSoftServer\.wscfg-backup\<id>\manifest.json" `
  -ServerDir "D:\WS\preprod\WebSoftServer"

# кандидаты в реестр по чувствительным шаблонам; НИЧЕГО не пишет — только список админу;
# хранилище не нужен, одинаково на обоих входах
powershell -NoProfile -ExecutionPolicy Bypass -File $vault -Command scan `
  -ServerDir "D:\WS\prod\WebSoftServer"
```

Коды выхода: `0` успех (включая 0 изменений), `2` preflight, `3` plan, `4` apply,
`5` verify-гейт, `6` health, `7` хранилище/CAS-конфликт, `8` попытка тронуть подписанное,
`9` неверные аргументы.

## Что делать при изменении прод-конфига

1. `export` — снять обновлённый снимок прода, затем `diff` по каждому контуру-получателю.
2. Появился новый обязательный ключ — добавить запись в `config/mandatory.registry.json` и
   закрывающее её правило в `config/overlays/<env>/rules.json`.
3. Изменилось значение существующего ключа — править `profile.json` (нечувствительные
   значения) или секрет через `edit`; в `rules.json` правится только таргет/action.
4. `validate` -> `plan`, прочитать `plan.md` и `coverage.missing` -> апрув в CI -> `apply`.
5. Контур поднялся криво — `rollback` по манифесту (побайтовое восстановление файлов +
   обратные БД-операции), затем разбор `apply-report.md`.

## Тесты

```powershell
# юнит + интеграционные модульные, единственная точка входа
powershell -NoProfile -ExecutionPolicy Bypass -File tests/run-tests.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File tests/run-tests.ps1 -Filter WscCfg -Detailed

# сквозной интеграционный прогон на фикстурном контуре, обе тонкие реализации
powershell -NoProfile -ExecutionPolicy Bypass -File tests/integration/Run-FullScenario.ps1 -Impl both
```

В исходниках 24 файла `*.Tests.ps1`, 464 проверки `It`; прогон пишет `tests/out/junit.xml`,
`test-results.json`, `run-tests.log`, exit code равен числу провалов. `Pester 5` на машине
нет, поэтому тесты идут на собственном `tests/Harness.ps1` (подмножество API Pester 5);
`run-tests.ps1` сначала пытается найти настоящий Pester >= 5. Сквозной прогон
`tests/integration/Run-FullScenario.ps1 -Impl both` (`-Impl vault|minio|both`) поднимает
фикстурный контур `tests/fixture/New-FixtureContour.ps1` и гоняет реальные тонкие
`impl/<x>/wscfg.ps1` и `bootstrap.ps1` через export -> diff -> plan -> apply -> повторный
apply -> rollback, проверяя в том числе коды выхода; пишет `tests/out/integration-<impl>.json`
и `tests/out/integration-<impl>.md` (плюс логи шагов `integration-<impl>-<шаг>.log`), каталог
`tests/out/` целиком git-ignored.

## Структура репозитория

```
config/mandatory.registry.json     реестр обязательных ключей (31 запись) — источник verify-гейта
config/baseline.files.json         белый список файлов для export
config/overlays/<env>/             rules.json + profile.json на каждый контур (git)
config/baseline/                   локальный кэш экспорта (в git не входит)
core/lib/                          jsonc/ini/xml/xmd, правила, plan, apply, verify, отчёты, fake-store
core/db/                           Db.psm1 + filexml / mssql / postgres
impl/vault/, impl/minio/           wscfg.ps1, store.ps1, bootstrap.*, layout.md, pipeline/*.yml
tests/                             Harness.ps1, run-tests.ps1, fixture/, unit/, db/, config/, impl/, integration/
tools/normalize-encoding.ps1       навешивание UTF-8 BOM на все .ps1/.psm1
docs/CONTRACTS.md                  форматы и сигнатуры (источник истины)
docs/ARCHITECTURE.md               модель, потоки, сравнение Vault/MinIO
docs/FEATURES.md                   реализованные возможности и границы
```

## Код-политика

- Только PowerShell 5.1: без `??`, `?.`, тернарника и другого `pwsh`-only синтаксиса.
- Все `.ps1`/`.psm1` — **UTF-8 с BOM**, иначе кириллица в PS 5.1 искажается. Проверка и
  исправление: `powershell -NoProfile -File tools/normalize-encoding.ps1 [-Repo <path>] [-Check]`.
- `core/lib/*.ps1` — только `function`, без кода верхнего уровня; ошибки исключительно через
  `throw (New-WscError -Code ...)`, текст начинается с `[WSC_*]`.
- Возвращающие списки вызовы оборачивать в `@(...)` и фильтровать `$null` (особенность PS 5.1).
- Запрещено писать внутрь `platform/`, `components/` и в `D:\WS\**`.
- Комментарии и документация — на русском; исходники `.ps1` держатся в ASCII, кириллица —
  в документации и в значениях конфига.
