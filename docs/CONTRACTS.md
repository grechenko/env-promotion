# CONTRACTS — общий контракт `websoft-env-promotion`

Единый источник истины по форматам и сигнатурам. Реализация не имеет права менять формат
без правки этого файла. Все комментарии/документация — на русском.

Корень репозитория: `D:\AI\cicdconf\websoft-env-promotion` (далее `ROOT`).

## 0. Соглашения

- Только PowerShell 5.1 (`powershell.exe`). Никакого `pwsh`-only синтаксиса (`??`, `?.`, ternary).
- Исходники `.ps1/.psm1` — UTF-8 **с BOM**, чтобы кириллица в PS 5.1 не искажалась.
- Относительные пути всегда с `/` (`wt_data/objects/...`). В системные конвертировать внутри функций.
- Токен маскировки: `**REDACTED**` (`Get-WscRedactedToken`).
- `core/lib/*.ps1` — dot-source-библиотеки: только `function`, никакого кода верхнего уровня,
  кроме `$script:`-инициализации в начале файла.
- Каждая библиотека принимает необязательный `-Log` (объект из `New-WscLog`) и пишет через `Write-WscLog`.
- Ошибки только через `throw (New-WscError -Code ... -Message ... [-Target ...] [-Detail ...])`.
  Текст исключения всегда начинается с `[WSC_*]`. Код читается `Get-WscErrorCode -ErrorRecord $_`.
- Запрещено: писать что-либо внутрь `platform/`, `components/`, а также в `D:\WS\**`.

### 0.1 Коды ошибок и exit code

| exit | код | смысл |
|---|---|---|
| 0 | — | успех (в т.ч. apply с 0 изменений — идемпотентность) |
| 2 | `WSC_PREFLIGHT` | PS/пути/диск/процессы сервера/маркер снимка |
| 3 | `WSC_PLAN` | правило не разрешилось, покрытие реестра не закрыто, неверная схема правила |
| 4 | `WSC_APPLY` | ошибка записи файла/БД (выполнен откат) |
| 5 | `WSC_VERIFY` | verify-гейт не пройден (выполнен откат) |
| 6 | `WSC_HEALTH` | сервер не поднялся (выполнен откат) |
| 7 | `WSC_STORE` | хранилище недоступно/аутентификация/`cas`-конфликт/NOTFOUND |
| 8 | `WSC_SIGNED_PATH` | попытка таргета внутрь подписанного компонента |
| 9 | `WSC_USAGE` | неверные аргументы CLI |

`Assert-WscExitCode` нет; CLI обязан `exit <код>` из таблицы. Внутренний код `WSC_IO`
(ошибки чтения/атомарной записи в `WscUtil`) внешним слоем мапится в exit 4.

### 0.2 Обязательные правила реализации (выявлено на шагах 01–03)

- PS 5.1 разворачивает возврат функции: `return @($x)` с одним элементом отдаёт голый объект,
  у hashtable `.Count` = число свойств. ВСЕ вызовы, возвращающие списки, оборачивать в `@(...)`
  на месте вызова и фильтровать `$null`.
- `Get-*` при 0 совпадений возвращают `@()` и НЕ бросают; `Set-*`/`Remove-*` при 0 совпадений
  бросают `WSC_PLAN` (кроме правил с `optional_file`).
- Справочник `Get-Wsc*TextValue`: `Start`/`End` — полуинтервал `[Start,End)`, `End` — индекс после
  value-токена; `Path` в совпадении — конкретизация шаблона (`Kestrel.EndPoints.*.Url` ->
  `Kestrel.EndPoints.Http.Url`). Сопоставление ключей — `Ordinal`, регистрозависимое.
- Текст-сканеры: `String.IndexOf(<char>,int,[StringComparison])` молча возвращает -1 (биндинг на
  (char,int,int)), `String.StartsWith(<string>,int,[StringComparison])` бросает Invalid cast —
  передавать только `[string]`/литералы строки или `Substring(...) -ceq ...`.
- `Remove-Item -Recurse -Force` на этой машине периодично даёт "directory is not empty" —
  для tmp использовать `Remove-WscTempDir` (внутри ретраи).
- Файл-образец `xHttp.ini`: `DESIGN-DB` = 14 вхождений, `DATA-DB` = 12, BOM нет, завершающего
  перевода строки нет. Не хардкодить другие числа.

## 1. Ссылка на цель (target reference)

Единая строка для файлов и БД. Канонический вид (регистр ключей/channel'ов — нижний):

```
file:<rel-path>::json:<dot-path>
file:<rel-path>::xpath:<xpath>
file:<rel-path>::ini:<KEY>                 # дубли: KEY#occurrence (1-based) задаётся полем правила occurrence
file:<rel-path>::regex:<pattern>
db:<component>.<type>::<field-path>
pkg:<component-dir>::<packageKey>          # только action=runtime_override -> *.runtime/<dir>/package.runtime.json
```

- `<rel-path>` — относительно корня сервера (`WebSoftServer\`), через `/`.
- `<dot-path>` JSON: `.` — спуск, `*` — любой ключ объекта, `[N]` — индекс массива, `[*]` — все элементы.
  Примеры: `Kestrel.EndPoints.Http.Url`, `Kestrel.EndPoints.*.Url`, `Authentication[*].Host`.
- `<field-path>` БД: те же `.`/`[*]`. Примеры: `base_url`, `endpoints[*].url`.
- `regex:` — последний резерв; 0 совпадений → `WSC_PLAN`; больше `max_matches` (по умолчанию 1) → `WSC_PLAN`.
- Разбор: `Split-WscTargetRef -Ref` → `@{ Kind; Rel; Channel; Path; Key; Occurrence; ComponentDir; PackageKey }`
  (поля заполняются только для своего kind/channel). Собранная обратно строка обязана быть
  канонической: `Format-WscTargetRef -Ref` идемпотентен.

## 2. Правило (`overlays/<env>/rules.json`)

```json
{
  "version": 1,
  "env": "dev",
  "rules": [ {
    "id": "unibridge-connection-string",
    "target": "file:spxml_unibridge_config.xml::xpath:/configuration/appSettings/add[@key='ConnectionString']/@value",
    "action": "set",
    "value": null,
    "fromProfile": "db.connectionString",
    "secret": null,
    "match": null,
    "occurrence": null,
    "mandatory": true,
    "optional_file": false,
    "max_matches": 1,
    "acked": false,
    "waiver": null,
    "reason": "иначе прод-БД затирается из dev"
  } ]
}
```

`action`:

| action | поле значения | поведение |
|---|---|---|
| `set` | `value` \| `fromProfile` | установить значение |
| `secret` | `secret` (имя секрета) | значение берётся из `secrets/<env>/<key>` |
| `append` | `value` \| `fromProfile` | добавить элемент (INI-ключ, JSON-массив) не затирая |
| `remove` | — | удалить ключ/узл/вхождение |
| `keep` | — | явно «не менять», но покрыть обязательный ключ |
| `runtime_override` | `value` \| `fromProfile` | запись в `platform.runtime|components.runtime/<dir>/package.runtime.json` (только `enabled`,`onFailure`,`roles`) |
| `manual` | — | в `plan.manualActions[]`, сервер не трогается |

- Приоритет источника значения: `secret` > `fromProfile` > `value`.
- `fromProfile` — dot-path в `overlays/<env>/profile.json` (см. §5).
- `mandatory: true` → пункт обязан закрыть запись из `config/mandatory.registry.json`.
- `optional_file: true` → если файла нет, правило даёт WARNING, а не fatal (`node_config.json`).
- `waiver` — строка-причина, разрешающая незакрытый обязательный ключ (`validate` печатает WARNING, не fail).
- `acked: true` — только для `manual`: подтверждение админа; иначе обязательный ключ не считается закрытым.
- Схема проверяется `Test-WscRuleSchema`; неизвестное поле → `WSC_PLAN`.

## 3. План (`plan.json`) — детерминированный

```json
{
  "version": 1, "env": "dev", "serverDir": "…", "planHash": "sha256:…",
  "items": [ {
     "ruleId": "…", "target": "…", "kind": "file|db|pkg",
     "channel": "json|xpath|ini|regex|db|pkg", "file": "rel-path", "path": "dot-path|xpath|KEY",
     "occurrence": null, "match": null, "action": "set",
     "oldValue": "из baseline", "newValue": "результирующее значение",
     "secretName": null, "mandatory": true, "optionalFile": false,
     "runtimeRedirect": false, "source": "baseline|profile|secret|literal"
  } ],
  "runtime": [ { "ruleId": "…", "file": "platform.runtime/<dir>/package.runtime.json", "path": "enabled", "newValue": "true" } ],
  "manualActions": [ { "ruleId": "…", "reason": "…", "acked": false } ],
  "coverage": { "registryTotal": 0, "covered": 0, "waived": 0, "missing": ["registry-key…"] },
  "warnings": [ "…" ], "unresolved": [ { "ruleId": "…", "code": "WSC_PLAN", "message": "…" } ]
}
```

- Сортировка `items` — по `(kind, file, path, ruleId)`; `planHash` = sha256 канонического JSON
  `items + runtime + manualActions` (`ConvertTo-WscCanonicalJson`). `generatedAt` в хеш не входит.
- Правило с 0 фактических изменений в `items` → `WSC_PLAN` (кроме `keep`, `manual`, и случая
  «уже применено», см. §7 идемпотентность).
- Компиляция: `New-WscPlan -Rules <hashtable> -Registry <hashtable> -Profile <hashtable> -Baseline <hashtable> -ServerDir <path> [-SecretResolver <scriptblock>] [-Log]`.
  `SecretResolver` — scriptblock `{ param($key) "…" }`; в dry-run/plan (`$null -eq -SecretResolver`)
  даёт `"<secret:name>"` и `source='secret'`.
  `-Baseline` (снимок прод-контра, собирается из хранилища; в тестах — литерал):

  ```powershell
  @{
    Files    = @{ '<rel-path>' = '<текст прод-файла с **REDACTED**>' }
    FileMeta = @{ '<rel-path>' = @{ Encoding = 'utf8'; HasBom = $false; NewLine = 'CRLF' } }
    Db       = @{ '<component>.<type>' = @{ Objects = @( @{ Id = '0x…'; Values = <psobject>; Source = <hashtable> } ) } }
    Missing  = @('<rel-path>')   # файлы, которых в контуре нет (node_config.json)
  }
  ```

  `oldValue` пункта плана: для файлов — чтение из `Baseline.Files` функциями `Get-Wsc*TextValue`
  (0 совпадений → `unresolved`, кроме `optional_file`); для БД — выборка объектов по `rule.match`
  из `Baseline.Db` и обход пути `Get-WscPathValues` (он поддерживает `*`, `[*]`, `[N]`).
  `newValue == oldValue` → пункт остаётся в `items` с полем `already = $true` и не считается
  нерезолвённым (идемпотентный повторный apply).

## 4. Манифест (`manifest.json`)

```json
{
  "version": 1, "id": "guid", "env": "dev", "createdAt": "ISO8601Z", "serverDir": "…",
  "planHash": "sha256:…", "status": "pending|applied|rolledBack|failed",
  "backupDir": "<serverDir>/.wscfg-backup/<id>",
  "files": [ { "relPath": "…", "backupPath": ".wscfg-backup/<id>/files/…",
               "origHash": "sha256", "newHash": "sha256", "hasBom": true, "newLine": "CRLF",
               "runtimeRedirect": false } ],
  "db": [ { "objectId": "0x…", "type": "websoft.aihub.ai_provider", "fieldPath": "endpoints[*].url",
            "oldValue": "…", "newValue": "…",
            "undo": { "driver": "filexml|mssql|postgres", "table": "…", "idColumn": "id",
                      "idValue": "…", "column": "…", "arrayIndex": null, "filePath": "wt_data/objects/…/A9.xml" } } ],
  "verify": { "passed": false, "checks": [] }
}
```

- Бэкап — байтовые копии в `backupDir/files/<relPath>` (структура сохраняется). Откат = побайтовое
  восстановление + обратные БД-операции из `db[].undo`.
- `WscBackup`: `New-WscBackupPlan`, `Save-WscFileBackup`, `Complete-WscManifest`,
  `Restore-WscManifest -ManifestPath [-ServerDir]`, `Update-WscManifestStatus`.

## 5. Профиль контура (`overlays/<env>/profile.json`) и реестр

`profile.json` — только нечувствительные значения, `dot-path` → строка:

```json
{ "version": 1, "env": "dev",
  "host": "hcm-dev.local", "port": "8080", "allowedHosts": ["localhost","hcm-dev.local"],
  "db": { "type": "MSSQL", "connectionString": "Server=dev-db;Database=wt_data;…", "username": "wt", "database": "wt_data", "schema": "dbo" },
  "redis": { "connection": "dev-redis:6379" }, "cache": { "type": "Redis" },
  "aihub": { "baseUrl": "https://ai-dev.local/v1" },
  "mediasoup": { "listenIp": "10.0.0.21", "announcedIp": "10.0.0.21", "rtcMinPort": "30000", "rtcMaxPort": "39999" },
  "cluster": { "masterNodeId": "dev-1" }, "node": { "id": "dev-1", "internalPorts": [] } }
```

`config/mandatory.registry.json`:

```json
{ "version": 1,
  "entries": [ {
    "key": "unibridge.ConnectionString",
    "target": "file:spxml_unibridge_config.xml::xpath:/configuration/appSettings/add[@key='ConnectionString']/@value",
    "check": "not_redacted|not_prod_value|not_empty|equals_profile:<dot-path>|regex:<pat>|port_match|no_cross_contour",
    "file_optional": false, "secret": false, "title": "строка подключения к БД", "notes": "" } ] }
```

`check` — машиночитаемая проверка verify-гейта (§8). Один `key` уникален; `wscfg validate` требует,
чтобы каждый `key` был закрыт правилом (`mandatory`) или `waiver`.

## 6. Интерфейс хранилища

Логические пути (едины для Vault и Minio):

```
wscfg/v1/baseline/meta.json
wscfg/v1/baseline/files/<rel-path>            # текст прод-файла, секреты => **REDACTED**
wscfg/v1/baseline/db/<component>.<type>.json  # снимок значений БД-настроек (без секретов)
wscfg/v1/overlays/<env>/rules.json
wscfg/v1/overlays/<env>/profile.json
wscfg/v1/secrets/<env>/<key>
```

Функции, которые **обязана** определить каждая реализация `impl/<x>/store.ps1`
(одинаковые имена = общая логика `core/lib/WscCfg.ps1` работает с любой):

```
New-WscStore -Config <hashtable> -Log        -> $store (@{ Kind; Config; State })
Test-WscStoreAccess -Store -Env -As @('read'|'write')
Get-WscStoreText    -Store -Path             -> @{ Path; Text; Version }        # нет -> throw WSC_STORE/NOTFOUND
Set-WscStoreText    -Store -Path -Text [-ExpectedVersion]  -> @{ Path; Version }
List-WscStore       -Store -Prefix           -> @('wscfg/v1/…')
Get-WscStoreVersions-Store -Path             -> @(@{ Version; Timestamp })
Get-WscStoreSecret  -Store -Env -Key         -> 'значение'                      # отсутствие -> WSC_STORE
Set-WscStoreSecret  -Store -Env -Key -Value [-ExpectedVersion]
Remove-WscStorePath -Store -Path
```

Шов для тестов (единственное место I/O, мокается переопределением в scope теста):

- Vault: `Invoke-WscVaultApi -Store -Method -ApiPath [-Body] [-Token]` → объект ответа (JSON уже разобран).
  Реальный транспорт: `vault.exe` если есть, иначе REST `Invoke-RestMethod` на `Config.Endpoint`.
  KV v2: `secret/data/<...>`, запись с `options: { cas: <n> }`. `Config = @{ Endpoint; Mount; Token|AppRolePath+RoleId+SecretId; Transport('rest'|'vaultcli') }`.
- Minio: `Invoke-WscS3Api -Store -Method -Key [-Body] [-Query] [-Headers]` → `@{ Status; Headers; Body }`
  (SigV4-подпись внутри; `Config = @{ Endpoint; Bucket; Region; AccessKey; SecretKey; Transport('rest'|'awstools') }`).
  Секреты — envelope-шифрование `age` при наличии бинаря, иначе `Protect-WscSecret` (AES + ключ из
  `Config.SecretKeyFile`) — plaintext-секрет в бакете = fatal на `validate`.
- `transport: 'fake'` обрабатывается **на уровне входа** (`impl/<x>/wscfg.ps1`, `bootstrap.ps1`): при
  `Transport='fake'` они dot-source'ят `core/lib/WscStoreFake.ps1` ВМЕСТО своего `store.ps1`, поэтому сам
  `store.ps1` fake не реализует и остаётся тонким реальным клиентом. Реальный клиент тестируется
  переопределением шлюза (`Invoke-WscVaultApi` / `Invoke-WscS3Api`) в области видимости теста ПОСЛЕ
  dot-source файла клиента. Так `impl/vault` и `impl/minio` проходят один и тот же набор тестов, а
  различие остаётся ровно в `store.ps1` (D1).

Общее: `core/lib/WscStoreFake.ps1` → `New-WscFakeStore -Root -Kind` + те же 9 функций
(десpatch по `$store.Kind -in 'vault-fake','minio-fake'`), чтобы `WscCfg`-тесты не зависели от реализации.

## 7. Интерфейс драйвера БД (`core/db/Db.psm1`)

Потребитель импортирует **только** `Db.psm1`; внутри — дисpatch по `$session.Kind`.

```
Open-WscDb   -Context $ctx -Log                -> $session = @{ Kind('filexml'|'mssql'|'postgres'); Context; State }
Find-WscDbObject -Session $s -Type 'ns.type' [-Match @{ name='openai' }]  -> @($object)
   $object = @{ Id; Type; Values(<nested PSObject>); Source(@{ Driver; FilePath|Table; IdColumn; IdValue }) }
Get-WscDbValue  -Session $s -Object $o -Path 'endpoints[*].url'  -> @($value)   # всегда массив
Set-WscDbValue  -Session $s -Object $o -Path … -Value …          -> @(@{ ObjectId; Type; FieldPath; OldValue; NewValue; Undo })
Submit-WscDb    -Session $s                    -> @{ Applied; Records }         # commit; ошибка -> откат транзакции
Undo-WscDb      -Session $s                    -> @{ Restored }                 # откат незакоммиченного
Close-WscDb     -Session $s
Get-WscDbDriverKind -Context $ctx              -> 'filexml'|'mssql'|'postgres'  # по CATALOG-STORAGE + DBType + фактическому wt_data
```

`$ctx = @{ ServerDir; ConfigXml(path); DbType; CatalogStorage; ConnectionString; Username; Password; Database; Schema; WtDataDir }`
— собирается `Get-WscDbContext -ServerDir` (читает `spxml_unibridge_config.xml` + `xHttp.ini`).

Правила: только транзакция; перед `UPDATE` — SELECT ожидаемого `OldValue`, после — SELECT нового;
`WHERE <idColumn> = <idValue>` (id из XMD-объекта), никогда «по имени»; `CatalogStorage='xdb'`
или наличие `wt_data/objects/**` ⇒ `filexml`, независимо от `DBType`.

Шов для SQL-тестов: `Invoke-WscSql -Driver $kind -Context $ctx -Sql <text> [-AsQuery]`
(единственное место исполнения SQL в `Db.psm1`: `Invoke-Sqlcmd` для mssql, `psql.exe`/Npgsql для
postgres). Тесты подменяют не её, а `$ctx.SqlExecutor` — scriptblock
`{ param($Driver, $Ctx, $Sql, $AsQuery) ... }`: если он задан в `Open-WscDb -Context`, `Invoke-WscSql`
обязан вызвать его и не лезть в сеть/процессы. Возврат: массив строк (или скаляр) для `AsQuery`,
число затронутых строк для DML.

## 8. Подписи модулей `core/lib`

`WscUtil.ps1` (готово, менять нельзя): `New-WscError`, `Get-WscErrorCode`, `Get-WscRedactedToken`,
`Test-WscRedacted`, `Read-WscRaw`, `Write-WscRawAtomic`, `Get-WscSha256Text`, `Get-WscSha256File`,
`ConvertTo-WscCanonicalJson`, `Get-WscNow`, `New-WscGuid`, `Resolve-WscServerPath`,
`Test-WscSignedPath`, `Split-WscTargetRef`, `Format-WscTargetRef`, `Get-WscNestedValue`,
`Set-WscNestedValue`, `Test-WscPathUnder`, `Get-WscFileHashes`, `New-WscTempRoot`, `Get-WscRepoRoot`.

- `WscJson.ps1`: `Read-WscJsonC -Path` → `@{ Text; Data; HasBom; NewLine; Bytes }`;
  `Test-WscJsonC -Text`; `Get-WscJsonTextValue -Text -Path` → `@(@{ Path; Value; Start; End })`;
  `Set-WscJsonTextValue -Text -Path -Value [-Remove]` → новый текст (правка узловая, байты-фидельны:
  комментарии/висячие запятые/табы сохранены); после правки обязателен `Test-WscJsonC`.
  Значение в JSON пишется по типу узла (строка в кавычках, числа/bool — без).
- `WscIni.ps1`: `Read-WscIni -Path` → `@{ Text; Entries; HasBom; NewLine }`, `Entries` = список
  `@{ Key; Value; Line; Occurrence }` (дубли сохраняются, порядок сохраняется);
  `Get-WscIniTextValue -Text -Key [-Occurrence]`; `Set-WscIniTextValue -Text -Key -Value [-Occurrence] [-Add]`;
  `Add-WscIniTextValue -Text -Key -Value`; `Remove-WscIniTextValue -Text -Key [-Occurrence]`;
  `Get-WscIniKeyList -Text -Key` → все значения ключа по вхождениям. Комментарии `#` и порядок строк — неизменны.
- `WscXml.ps1`: `Read-WscXml -Path` → `@{ Text; Xml; HasBom; NewLine }`;
  `Get-WscXmlAppSetting -Text -Key`; `Set-WscXmlAppSetting -Text -Key -Value [-Add]`;
  `Get-WscXmlValue -Text -XPath`; `Set-WscXmlValue -Text -XPath -Value [-Remove]`;
  `Set-WscXmlText -Xml` (сериализация с сохранением BOM/новой строки/отступов источника).
  XML-атрибуты c `&amp;`/`&quot;`Round-trip побайтово для неизменённых узлов.
- `WscXmd.ps1`: `Read-WscXmd -Path` → `@{ File; Namespace; TypeName; QualifiedName; Root; Title; DispName; Fields }`,
  `Fields` = список `@{ Name; Type; Multiple; PrimaryKey; ForeignArray; Title; Path; Children[] }`;
  `Find-WscXmdForForm -ServerDir -FormUri` (по `SPXML-FORM="x-local://platform/<comp>/spxml/object_types/<t>.xmd"`);
  `Resolve-WscXmdFieldPath -Schema -Path` → `@{ Path; Kind('scalar'|'array'|'arrayField'); Field; ElementPath }`;
  `Get-WscXmdSensitiveFields -Schema` → поля-кандидаты на REDACTED (token/key/secret/password/url/base_url/value).
- `WscRules.ps1`: `Test-WscRuleSchema -Rules`, `New-WscPlan` (§3), `Get-WscRuleValue -Rule -Profile -SecretResolver`,
  `Get-WscMandatoryCoverage -Plan -Registry`, `Assert-WscPlan -Plan`.
- `WscApply.ps1`: `Apply-WscPlan -Plan -ServerDir [-DbSession] [-DryRun] [-ProdValues @()] [-Log]` →
  `@{ Manifest; ManifestPath; Changes=@(...); Skipped=@(...); RuntimeRedirects=@(...); Warnings=@() }`.
  Порядок — §4 (backup -> файлы -> runtime -> БД c `Submit-WscDb`), ре-парсинг после правки,
  идемпотентность (`newValue == oldValue` -> в `Skipped`, не в `Changes`), авто-откат при любой ошибке.
  `Rollback-WscApply -ManifestPath [-ServerDir] [-DbSession] [-Log]` -> `@{ FilesRestored; FilesRemoved; DbRestored }`:
  файлы из манифеста (в т.ч. удаление ранее несуществовавших) + обратные БД-операции из `db[].undo`
  (filexml — возврат `OldXml`, SQL — reverse UPDATE через `Invoke-WscSql`); в конце `status=rolledBack`.
- `WscVerify.ps1`: `Test-Contour -ServerDir -Baseline -Registry -Profile -Manifest -ProdValues @() [-DbSession] [-Log]` →
  `@{ Passed; Checks = @(@{ Id; Key; Passed; Detail }) }`; обязательные `Id`: `no_redacted`,
  `no_prod_value_leak`, `db_not_prod`, `mandatory_check`, `ports_match`, `allowed_hosts_match_profile`,
  `hosts_match_profile`, `cache_not_prod`, `rtc_no_collision`, `no_signed_files_touched`,
  `secrets_resolved`. `check`-словарь реестра: `not_redacted|not_prod_value|not_empty|equals_profile:<dot-path>|regex:<pat>|port_match|no_cross_contour`.
- `WscReport.ps1`: `New-WscLog -VerbosePreference`, `Write-WscLog -Log -Level -Message`,
  `Write-WscApplyReport -Result -OutDir` (`apply-report.json` + `apply-report.md`),
  `Write-WscJUnitXml -Results -Path`, `Write-WscJsonReport -Object -Path`.
- `WscCfg.ps1`: общий слой команд `Invoke-WscCfg -Command <export|diff|edit|validate|plan|apply|rollback|scan> -Options <hashtable> [-Log]`
  -> `@{ ExitCode; Result; Artifacts=@(); Warnings=@() }`. Общие ключи `-Options`:
  `ServerDir, Env, OutDir, DryRun, RegistryPath, OverlayDir, BaselineFilesPath, Verbose`.
  По командам: `export` — `SourceServer` (контур-источник, обычно прод) читается ДРЕЙВОМ, redaction по
  `RegistryPath`+`baseline.files.json`, результат пишется в хранилище (`baseline/**`) и в локальный кэш
  `config/baseline/`; `diff` — `Env`, `Against` (версия оверлея или `$null`) -> список изменений
  baseline+overlay против baseline; `edit` — `Env`, `Item` (`rules|profile|secret:<key>`), `Get`/`Set`
  (`Set` требует `Value`|`SourceFile`), CAS по прочитанной версии; `validate` — `Env`: схема правил,
  покрытие реестра, отсутствие `**REDACTED**`/прод-значений/энтропийных секретов в `config/**` и overlays;
  `plan` — `Env` -> `plan.json`+`plan.md` в `OutDir`, без записи; `apply` — `Env`, `SecretResolver`,
  `NoStart` -> план, apply, `Test-Contour`, авто-откат при провале, `apply-report.json|md`;
  `rollback` — `Manifest`; `scan` — `ServerDir` -> кандидаты в реестр (только предложение админу).
  `Get-WscBaseline -ServerDir -Files <@()> -Registry <hashtable> [-DbSession] [-Redact @()]` ->
  `-Baseline` вида §3 (тексты файлов, метаданные кодировок, снимок БД со REDACTED по секретным целям).
  `impl/<x>/wscfg.ps1` — тонкий вход (~60 строк): аргументы -> свой `store.ps1` (`New-WscStore`) ->
  `Invoke-WscCfg`; вся логика ОДНА на обе реализации (D1).

### 8.1 Redaction при `export`

`Export-WscBaseline`: для каждого файла из множества `mandatory.registry.json` и белого списка
`config/baseline.files.json` читает прод-файл и маскирует значения: поля из реестра с `secret: true`,
а также значения, попадающие под `Get-WscSensitivePattern` (пароль/токен/key/secret/connection-string-сегменты).
Файл пишется в `baseline/files/<rel>` с исходной кодировкой (BOM/UTF-8/CRLF сохраняются).
`baseline/meta.json`: `@{ ExportedAt; SourceServer; Files = @(@{ Rel; OrigHash; RedactedCount; Encoding }) }`.

## 9. Fixture-контур (`tests/fixture/New-FixtureConture.ps1`)

`New-FixtureContour -OutDir [-Real <D:\WS\pl1260924\WebSoftServer>]` → мини-копия:
корневые конфиги (реальные образцы, прод-значения подставлены маркерными `prod-*.local`),
`platform/core@websoft/package.json` + `content.json` (подписанный, неизменяемый),
`platform/aihub_mng@websoft.aihub/{package.json,spxml/object_types/*.xmd}`,
`wt_data/catalogs/websoft.aihub.ai_providers.xdb|ai_credentials.xdb`,
2 объекта `ai_provider` и 1 `ai_credential` в `wt_data/objects/<ID14>/<HEX2>.xml`,
`xhttp_launcher.cmd`-заглушка (пишет `health-ok` и ждёт `GET /verify.html` — эмулируется файлом).
Прод-значения-маркеры (только они ищутся verify-гейтом): `prod-db.internal`, `prod-redis.internal`,
`prod-portal.internal`, `https://ai.prod.local/v1`, `ProdPass0rd`, `sk-prod-TOKEN`. Точный список
возвращает `New-FixtureContour` в `ProdValues` — читать его, не хардкодить. Типы объектов фикстуры:
`websoft.aihub.ai_provider` (2 шт.) и `websoft.aihub.ai_credentials` (1 шт.).

## 10. Тестовый harness (`tests/Harness.ps1`)

`Pester 5` на машине нет (есть только 3.4.0 без `Should -Be`), поэтому тесты пишутся на
**подмножестве API Pester 5**, которое реализует harness и которое принимает настоящий Pester 5:

```powershell
Describe 'WscJson' {
  BeforeAll { $script:x = 1 }
  Context 'read' {
    It 'parses jsonc' { $v | Should -Be 1 }
    It 'throws' { { throw } | Should -Throw }
    It 'skipped example' -Skip { }
  }
}
```

Разрешено: `Describe`, `Context`, `It` (с `-Skip`), `BeforeAll`, `BeforeEach`, `AfterEach`, `AfterAll`,
`Should` с операторами: `-Be`, `-BeExactly`, `-Not`, `-BeTrue`, `-BeFalse`, `-BeNullOrEmpty`,
`-Match`, `-Throw`, `-Contain`, `-Exist`, `-HaveCount`, `-BeGreaterThan`, `-BeLessThan`, `-ErrorMessage`.
Библиотеки подключаются dot-source'ом **на верхнем уровне файла теста, до `Describe`**
(`. (Join-Path $script:repo 'core\lib\WscUtil.ps1')`, где `$script:repo = Split-Path -Parent (Split-Path -Parent $PSScriptRoot)`);
в `BeforeAll` так делать нельзя — функции переопределяются следующим тестовым файлом.
Запрещено: `Mock`, `-TestDrive`, `Set-TestInconclusive`, in-It-определения функций,
`$TestDrive`. Временные каталоги — `Join-Path (New-WscTempRoot) (New-WscGuid)` (создаёт `tests/out/tmp`),
удаляются в `AfterAll` через `Remove-WscTempDir`.
`tests/run-tests.ps1` — единственная точка входа: `powershell -File tests/run-tests.ps1 [-Filter <sub>] [-Detailed]`;
сам находит `**/*.Tests.ps1`, пишет `tests/out/junit.xml`, exit = число провалов (0 = успех).

## 11. Формы и решения, зафиксированные по итогам шагов 01–07

- `pkg:`-цель обязана содержать префикс каталога компонента: `pkg:platform/<dir>::roles` ->
  `platform.runtime/<dir>/package.runtime.json`; `pkg:components/<dir>::k` -> `components.runtime/...`
  (префиксы `platform_external/`, `components_external/` тоже поддерживаются); голый `<dir>` без
  префикса -> `components.runtime`. Разрешённые packageKey: `enabled`, `onFailure`, `roles`.
- Любая `file:`-цель внутри `platform/`, `components/`, `*_external/` запрещена компилятором для
  ВСЕХ action (включая `keep`/`manual`) -> `WSC_SIGNED_PATH`. Значит в `mandatory.registry.json`
  и в оверлеях допустимы только корневые файлы сервера, `wt_data/**`-исключения и `db:`-цели.
- План содержит поле сверх схемы §3: `ruleIndex = @(@{ ruleId; target; action; mandatory; acked;
  waiver; secret; optionalFile })` — нужно `Get-WscMandatoryCoverage -Plan`; в `planHash` не входит.
- `Get-WscMandatoryCoverage` сопоставляет цели без `#occurrence`, поэтому INI-цели в реестре пишутся
  как `file:xHttp.ini::ini:DESIGN-DB`.
- `action='keep'` -> пункт с `already=$true` (не изменение); `action='remove'` -> `newValue=$null`.
- Поле правила `optional_key: true` (и одноимённое поле записи реестра): если ключ/цель отсутствует в
  контуре, правило даёт WARNING вместо `unresolved` (файл при этом может существовать). Отличие от
  `optional_file`: тот же флаг решает судьбу ОТСУТСТВУЮЩЕГО ФАЙЛА. Используется для ключей, которых
  нет на свежей установке (`DistributedCacheType`, `DistributedCacheServer`).
- `-Rules/-Registry/-Profile/-Baseline` принимаются и hashtable'ами, и PSCustomObject из
  `ConvertFrom-Json` — `WscCfg` вправе не конвертировать.
- Версионируется в git: `config/overlays/<env>/{rules.json,profile.json}` (редактируемый админом
  источник, синхронизируется с хранилищем командами `wscfg edit`/`export`). НЕ версионируется:
  `config/baseline/` (локальный кэш экспорта), `tests/out/`, `.wscfg-backup/`.
- Реальные XMD содержат дубли атрибутов (`PARAM= PARAM=`) — `Read-WscXmd` их чистит; в своём
  XML-коде всегда использовать `.LocalName` (у XML-адаптера PS `.Name` перекрывается дочерним
  элементом `<name>`), имена фильтровать `-cmatch`.
- `Convert-WscCanonicalValue` сохраняет тип массива при 0/1 элементе — план-хеши и манифесты
  корректны; не обходить этим центральным кодом.
- PS 5.1: в аргументе команды приведение индексируемого выражения требует скобок —
  `-Message ([string]$h['k'])`; локальная `$matches` затирается любым `-match`; в тесте для ОДНОГО
  элемента — `Should -HaveCount 1` + `Should -Contain`, а не `Should -Be @('один')`.
