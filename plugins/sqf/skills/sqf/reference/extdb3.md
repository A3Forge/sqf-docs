# extDB3 (MySQL) и SQL_CUSTOM

Источники: вики [SteezCram/extDB3](https://github.com/SteezCram/extDB3/wiki), исходный код
extDB3 (там, где вики устарела или неточна — отмечено) и опыт рабочего сервера
(протокол `SQL_CUSTOM`, файл запросов `sql_custom/<имя>.ini`). Работа с БД — **только на сервере**.

## Установка

- Мод `@extDB3` подключается только на сервере: `-serverMod=@extDB3`
  (`@extDB3_Debug` — отладочная сборка с подробным логом).
- Windows: `tbbmalloc.dll` и `tbbmalloc_x64.dll` скопировать в корень сервера (рядом с
  `arma3server.exe`); нужен Visual C++ Redistributable 2019 **x86 и x64**.
  (Файлы tbbmalloc переименованы автором, чтобы 32- и 64-битные версии лежали рядом.)
- Linux: glibc 2.17+ (CentOS 7+, Debian 8+, Ubuntu 14.10+). Deb: `apt-get install libtbb2:i386`
  и `apt-get install libtbbmalloc2`; RPM: `yum install tbb.i686` и `yum install tbb`.
- Настройки подключения — `@extDB3/extdb3-conf.ini` (см. ниже).
- Сборка из исходников (Windows): проект Visual Studio в `build/msvc`, Visual C++ 14.2x,
  распаковать `libs.7z` и поправить пути к библиотекам и include в свойствах проекта.
- Аддон `@extDB3` сам вызывает только `extDB3_fnc_preInit`: пишет в RPT «extDB3 Loaded» или
  ошибку загрузки. Остальные SQF-обёртки — свои.

## extdb3-conf.ini

```ini
[Main]
Version = 1
Randomize Config File = false   ; переименовать конфиг после загрузки — включай при -filePatching
Allow Reset = false             ; разрешить 9:RESET (для разработки)
Threads = 0                     ; число рабочих потоков, 0 = авто (2–6 по числу ядер)

[Log]
Flush = true                    ; сбрасывать лог после каждой записи (нужно в основном для Debug)

[Database]                      ; имя секции — это <DATABASE_CONFIG_NAME> в 9:ADD_DATABASE
IP = 127.0.0.1
Port = 3306
Username = changeme
Password = changeme
Database = changeme
```

- Секций баз может быть сколько угодно (`[Database2]`, `[A3W]`, ...).
- В вики и в поставляемом конфиге ключ потоков написан как `Thread`, а код читает `Threads` —
  с `Thread` настройка молча игнорируется (остаётся авто).
- Файл содержит логин и пароль к БД — не коммить его в публичные репозитории.

## Формат вызова

Запрос: `<тип>:<имяПротокола>:<данные>`. Ответ: `[тип]` или `[тип, данные]`.

| Тип вызова | Что делает |
|---|---|
| `0` | Синхронно: ждёт результат, блокирует кадр сервера |
| `1` | Асинхронно без результата (INSERT/UPDATE, логи) |
| `2` | Асинхронно с сохранением результата, возвращает `[2,"id"]` для `4:`/`5:` |
| `4:<id>` | Забрать результат одним сообщением |
| `5:<id>` | Забрать результат по частям |
| `9:...` | Системные команды (всегда синхронные) |

| Тип ответа | Значение |
|---|---|
| `[0,"текст"]` | Ошибка (подробности — в логе extDB3) |
| `[1,данные]` | OK |
| `[2,"id"]` | ID: для асинхронного запроса или большого результата (id — **строка**) |
| `[3]` | Результат ещё не готов — подожди и спроси снова |
| `[5]` | Ответ на `4:` — результат многочастный, забирай через `5:` |

- Синхронный `0:` возвращает `[2,"id"]` вместо данных, если результат не влез в буфер
  `callExtension` (размер — `9:OUTPUTSIZE`). Тогда забирай его через `5:`.
- `5:<id>` вызывай, пока не вернётся пустая строка: только тогда id освобождается. Пустая
  строка придёт и при неверном id. Части — это куски одной строки, склеивай их и разбирай целиком.
- После `4:` повторять вызов не нужно (в отличие от `5:`).

## Системные команды (`9:`)

| Команда | Назначение |
|---|---|
| `9:ADD_DATABASE:<секция>[:<имя>]` | Подключиться к БД из секции конфига (имя по умолчанию = имя секции) |
| `9:ADD_DATABASE_PROTOCOL:<имяБД>:<SQL\|SQL_CUSTOM>:<имяПротокола>[:<опции>]` | Протокол поверх БД. Для SQL_CUSTOM опция — имя `.ini` |
| `9:ADD_PROTOCOL:<LOG\|MISC>:<имяПротокола>[:<опции>]` | Протокол без БД |
| `9:LOCK` / `9:LOCK:<код>` | Запретить системные команды; с кодом — можно разблокировать |
| `9:LOCK_STATUS` | `[1]` — заблокировано, `[0]` — нет |
| `9:UNLOCK:<код>` | `[1]` — разблокировано, `[0]` — неверный код |
| `9:RESET` | Закрыть все соединения и протоколы (дождавшись текущих задач). Нужны `Allow Reset = true` и снятая блокировка |
| `9:VERSION` | Версия; пустая строка — расширение не загружено |
| `9:OUTPUTSIZE` | Размер буфера `callExtension` (также пишется в лог) |
| `9:LOCAL_TIME[:смещение]` / `9:UTC_TIME[:смещение]` | Время `[Y,M,D,H,M,S]`; смещение — часы (`:3`) или `[0,0,дни,часы,мин,сек]` (годы/месяцы не поддерживаются) |
| `9:UPTIME:<SECONDS\|MINUTES\|HOURS>` | Время с первого `callExtension` — например, для предупреждений о рестарте |
| `9:DATEADD:[Y,M,D,H,M,S]:[дни,часы,мин,сек]` | Дата плюс/минус интервал |

- Все `ADD_*` — **только до `LOCK`**. После `LOCK` доступны только `VERSION`, `LOCK_STATUS`,
  `UNLOCK` и команды времени.
- `ADD_*` и `RESET` **не потокобезопасны** (так задумано ради скорости): не выполняй их,
  пока идут запросы `0:`/`1:`/`2:`. Порядок настройки: `RESET` → `ADD_DATABASE` →
  `ADD_DATABASE_PROTOCOL` → `ADD_PROTOCOL` → `LOCK`, потом запросы.
- Протоколы добавляй от самого используемого к наименее используемому — поиск чуть быстрее.

## Подключение (один раз за запуск сервера)

Расширение остаётся загруженным между миссиями (и после `9:LOCK` добавить протокол уже
нельзя). Поэтому подключение делается один раз, а имя протокола сохраняется в `uiNamespace`
**сервера** (переживает смену миссии; игроки его не контролируют — см. правило 17 в SKILL.md):

```sqf
if (isNil {uiNamespace getVariable "TAG_dbProtocol"}) then {
    if (("extDB3" callExtension "9:ADD_DATABASE:db1") isNotEqualTo "[1]") exitWith {
        diag_log "extDB3: ошибка подключения к db1";
    };
    private _protocol = str round random 9999;        // случайное имя протокола
    private _res = parseSimpleArray ("extDB3" callExtension
        ("9:ADD_DATABASE_PROTOCOL:db1:SQL_CUSTOM:" + _protocol + ":my_custom.ini"));
    if ((_res # 0) isEqualTo 0) exitWith {diag_log ("extDB3: ошибка протокола " + str _res)};

    "extDB3" callExtension "9:LOCK";                  // больше никто не добавит протоколы
    uiNamespace setVariable ["TAG_dbProtocol", _protocol];
};
TAG_dbProtocol = compileFinal str (uiNamespace getVariable "TAG_dbProtocol");   // call TAG_dbProtocol
```

- `9:LOCK` после настройки — обязательно: иначе любой код, получивший доступ к
  `callExtension`, может добавить свой протокол (например, сырой SQL).
- Случайное имя протокола усложняет прямой вызов extDB3 чужим кодом.
- Проверяй, что расширение загружено (`"extDB3" callExtension "9:VERSION"` не пустой),
  и не запускай логику, зависящую от БД, если подключение не удалось.
- Если в `.ini` есть ошибка или неизвестная опция, протокол **не загрузится**:
  `ADD_DATABASE_PROTOCOL` вернёт `[0,...]`, причина будет в логе extDB3.
- Альтернатива для разработки: `9:LOCK:<код>`, а при смене миссии — `9:UNLOCK:<код>` +
  `9:RESET` (нужен `Allow Reset = true`) и настройка заново.

## Вызов запроса

Строка: `<режим>:<протокол>:<имяЗапроса>:<вход1>:<вход2>...`

```sqf
// синхронно с результатом: режим 0
private _res = parseSimpleArray ("extDB3" callExtension
    (["0", call TAG_dbProtocol, "getPlayer", _uid] joinString ":"));
switch (_res # 0) do {
    case 0: { diag_log ("DB error: " + str (_res # 1)); };     // ошибка
    case 2: { _res = [1, (_res # 1) call TAG_db_fetchBig]; };   // большой результат — по частям
};
_res # 1

// асинхронно без ответа: режим 1 (INSERT/UPDATE, логи, статистика)
"extDB3" callExtension (["1", call TAG_dbProtocol, "addKillLog", _a, _b] joinString ":");

// TAG_db_fetchBig: большой результат читаем по id, пока не вернётся пустая строка
private _out = "";
while {true} do {
    private _part = "extDB3" callExtension ("5:" + _this);
    if (_part isEqualTo "") exitWith {};
    _out = _out + _part;               // конкатенация, не format: строка может быть огромной
};
(parseSimpleArray _out) # 1
```

Асинхронно с результатом (режим 2) — только в scheduled-коде:

```sqf
private _id = (parseSimpleArray ("extDB3" callExtension
    (["2", call TAG_dbProtocol, "getPlayer", _uid] joinString ":"))) # 1;
private _raw = "";
waitUntil {
    _raw = "extDB3" callExtension ("4:" + _id);
    _raw isNotEqualTo "[3]"            // [3] — ещё не готово
};
if (_raw isEqualTo "[5]") then {       // не влез в один ответ — забираем по частям
    _raw = "";
    while {true} do {
        private _part = "extDB3" callExtension ("5:" + _id);
        if (_part isEqualTo "") exitWith {};
        _raw = _raw + _part;
    };
};
private _res = parseSimpleArray _raw;  // [1, [[строка1], [строка2], ...]] или [0, "ошибка"]
```

- Ответ всегда разбирай `parseSimpleArray`, никогда `call compile`.
- Синхронный режим блокирует кадр сервера на время запроса: тяжёлое и не нужное
  сразу — асинхронно (режим 1, или 2, если нужен результат).
- Строку запроса собирай через `joinString` / `+`, не `format` (лимит длины `format`,
  см. syntax-pitfalls.md).
- Формат успешного ответа SQL_CUSTOM: `[1, [[строка], [строка], ...]]`; с `Return InsertID` —
  `[1, [insertId, [[строка], ...]]]`.

### Входные параметры

- Входы разделяются `:` — **двоеточие внутри значения ломает разбор**. Число входов должно
  **точно** совпадать с наибольшим номером в `SQLx_INPUTS`, иначе ошибка
  `Config Invalid Number Number of Inputs`. Значения, введённые игроком (имена, тексты),
  очищай от `:` до отправки, либо передавай их не строкой, а числом/идентификатором.
- Альтернатива — `Input SQF Parser = true` (экспериментально): входы передаются SQF-массивом
  `0:SQL:UpdatePlayer:["Joe",[1,2,0],0.22333,"PlayerBackpack",-3]`, и `:` в значениях
  чистить не нужно.
- `Strip Chars` / `Strip Chars Mode` работают **только для входов и выходов с опцией
  `strip`** (`SQL1_INPUTS = 1-strip,2`). Без неё символы не вырезаются, хотя вики пишет
  иначе (проверено по исходникам). Режимы: `0` — вырезать, `1` — вырезать и записать в лог,
  `2` — вернуть ошибку `[0,"Error Strip Char Found"]`. Это защита, а не замена валидации на сервере.
- `mysql_escape` работает **только при `Prepared Statement = false`**. С prepared statement
  (`?`) опция игнорируется, и экранирование не нужно: значение передаётся отдельно от SQL.
- Типы входов проверяй в SQF до запроса (`params` с типами): в строку уйдёт что угодно.

## Описание запросов в SQL_CUSTOM .ini

Файл лежит в `@extDB3/sql_custom/<имя>.ini`. Вместо прямого SQL от сервера в нём заранее
описаны все разрешённые запросы — это безопаснее протокола `SQL`.

```ini
[Default]
Version = 1
Strip Chars = "/\|;{}<>'`"
Strip Chars Mode = 1
Input SQF Parser = false
Number of Retrys = 1

; Комментарий над каждым запросом: кто вызывает, что возвращает, неочевидные решения
[getWarehouses]
SQL1_1 = SELECT id, var_name, name, side FROM cfg_warehouses ORDER BY id
OUTPUT = 1,2-STRING,3-STRING-add_escape_quotes,4-STRING

[addKillLog]
SQL1_1 = INSERT INTO kill_logs (session_id, killer, victim) VALUES (?, ?, ?)
SQL1_INPUTS = 1,2,3

[createSession]
SQL1_1 = INSERT INTO game_sessions (`mission`, `map`) VALUES (?, ?)
SQL1_INPUTS = 1-strip,2-strip
Return InsertID = true
```

### Настройки `[Default]`

Можно переопределить в секции запроса: `Strip Chars`, `Strip Chars Mode`,
`Input SQF Parser`, `Number of Retrys`.

| Ключ | Значение |
|---|---|
| `Version` | Версия формата (на случай несовместимых изменений) |
| `Strip Chars` | Символы для вырезания (только для значений с опцией `strip`) |
| `Strip Chars Mode` | `0` вырезать, `1` вырезать + лог, `2` ошибка + лог |
| `Input SQF Parser` | Входы — SQF-массивом вместо разделителя `:` (экспериментально) |
| `Number of Retrys` | Повторы при ошибке (по умолчанию 1). Повторяется **весь** запрос, включая все `SQLx` |

### Настройки запроса

| Ключ | Значение |
|---|---|
| `SQLx_y` | SQL: `x` — номер оператора, `y` — часть строки (части склеиваются через пробел) |
| `SQLx_INPUTS` | Входы для оператора `x` и опции к ним |
| `OUTPUT` | Опции выходных колонок. Возвращается результат **только последнего** `SQLx` |
| `Prepared Statement` | `true` (по умолчанию) — prepared statement с `?`; `false` — SQL с подстановкой `$CUSTOM_N$` |
| `Return InsertID` | Вернуть AUTO_INCREMENT id последнего INSERT/UPDATE |
| `Return InsertID String` | То же, но id строкой — для id длиннее 6–7 цифр (точность float в SQF) |

- `?` — плейсхолдеры prepared statement; `SQL1_INPUTS` — какие входы в каком порядке.
  Входы можно переставлять и повторять: `SQL1_INPUTS = 3, 2-strip, 1, 2`.
- Несколько операторов в одной строке SQL запрещены (безопасность): для нескольких —
  `SQL1_1`, `SQL2_1`, ... Они выполняются по порядку, и у каждого свои `SQLx_INPUTS`.
- `Return InsertID = true` заодно позволяет вызвать INSERT синхронно и **дождаться коммита**
  перед следующим запросом, который на него опирается.
- UPSERT: `INSERT ... ON DUPLICATE KEY UPDATE col = VALUES(col)` / `minutes = minutes + 1`.
- `Prepared Statement = false` + `$CUSTOM_1$` вставляет вход в SQL **как есть** — это
  SQL-инъекция, если туда попадёт что-то от клиента. Если без этого никак — `N-mysql_escape`
  и значение в кавычках: `WHERE uid = "$CUSTOM_1$"`, — но лучше не использовать вовсе.
  (В вики про переход с extDB2 упомянут `$INPUT_x$`; текущий код подставляет только `$CUSTOM_x$`.)

### Опции входов и выходов

Пишутся через `-` после номера: `2-STRING-add_escape_quotes`. Регистр не важен. Опечатка
в опции — протокол не загрузится (ошибка в логе).

| Опция | Вход | Выход |
|---|---|---|
| `string` / `string2` | Обернуть в `"` / `'` | Обернуть в `"` / `'` (для текстовых колонок) |
| `add_escape_quotes` | `"` → `""` | `"` → `""` (применяется до `string`) |
| `remove_escape_quotes` | `""` → `"` | `""` → `"` |
| `remove_quotes` | Удалить `"` и `'` | — |
| `strip` | Применить `Strip Chars` | Применить `Strip Chars` |
| `null` | Пустая строка → NULL | NULL → `objNull` (без опции NULL → `""`) |
| `bool` | `true` → 1 | `1` → `true`, остальное → `false` |
| `time` | `[Y,M,D,H,M,S]` → DATETIME (только prepared statement) | — |
| `beguid` | SteamID64 → BattlEye GUID | SteamID64 → BattlEye GUID |
| `mysql_escape` | Экранирование MySQL (только `Prepared Statement = false`) | — |

Устаревшие (до v1.011): `string_escape_quotes`, `string_escape_quotes2` — замени на
`string-add_escape_quotes`.

### Вывод: кавычки, переводы строк, JSON

- `N-STRING` в `OUTPUT` оборачивает значение в кавычки для SQF, но **кавычки внутри
  значения сам не экранирует**. Текст, который вводят люди (имена отрядов, ролей,
  названия, описания с сайта), с `"` внутри ломает `parseSimpleArray` всего ответа.
  Добавляй `add_escape_quotes`: `OUTPUT = 1,2-STRING-add_escape_quotes`. Так колонка
  остаётся «голой» в SELECT (см. LONGTEXT ниже). Второй вариант — удваивать в SQL:
  `REPLACE(name, '"', '""')`.
- Переводы строк в тексте: `CHAR(13)` убирай, `CHAR(10)` заменяй на то, что понимает
  получатель (`<br>` для structured text / HTML, пробел для обычного текста):
  ```sql
  REPLACE(REPLACE(description, CHAR(13), ''), CHAR(10), '<br>')
  ```
- Колонки, где хранится готовый SQF/JSON-массив (TEXT с `[...]` внутри), отдавай **без**
  `-STRING` — они придут уже массивами. Пустое значение делай через
  `NOT NULL DEFAULT '[]'` в схеме, а не через функции в SELECT (см. ниже).
- Числа и булевы — без `-STRING`. Исключение — большие целые (BIGINT id, SteamID64):
  отдавай их с `-STRING`, иначе SQF потеряет точность (правило 14 в SKILL.md).
- NULL по умолчанию приходит как `""` (даже в числовой колонке), с опцией `null` — как
  `objNull`. В коротких колонках можно привести явно: `IFNULL(rank_slug, '')`.

### Дата и время

- DATE, TIME, DATETIME, TIMESTAMP приходят **массивом** `[год,месяц,день,час,минута,секунда]`
  (в протоколах `SQL` и `SQL_CUSTOM`), а не строкой. Так их удобно локализовать для игрока.
- Чтобы записать дату из SQF, передай `[Y,M,D,H,M,S]` во вход с опцией `time`
  (только prepared statement).
- Серверное время можно получить без БД: `9:UTC_TIME`, `9:LOCAL_TIME`, `9:DATEADD`.

### LONGTEXT и prepared statements

extDB3 не умеет читать LONGTEXT из подготовленного запроса — падает **весь** запрос:
`MYSQL_TYPE_LONG_BLOB type not supported when using Prepared Statements`.

- Не храни такие данные в LONGTEXT.
- Не применяй `IFNULL`, `CONCAT`, `CAST`, `CASE`, `REPLACE` и подобные функции к
  MEDIUMTEXT/TEXT-колонкам в SELECT: MySQL может расширить тип результата до LONGTEXT.
  Читай такие колонки «голыми», кавычки экранируй опцией `add_escape_quotes`, а NULL
  исключай схемой (`NOT NULL DEFAULT ...`).
- Протокол `SQL` (без prepared statements) LONGTEXT читает, но для данных от игроков его
  использовать нельзя (см. ниже).

## Другие протоколы

### SQL — сырой SQL

```
9:ADD_DATABASE_PROTOCOL:db1:SQL:<имяПротокола>[:TEXT|TEXT2|NULL|TEXT-NULL]
0:<имяПротокола>:SELECT * FROM PlayerSave
```

- Опции: `TEXT` / `TEXT2` — оборачивать текстовые значения в `"` / `'`; `NULL` — NULL → `objNull`
  (по умолчанию `""`).
- Любой код с доступом к протоколу выполняет любой SQL. На боевом сервере не добавляй его
  вовсе (или только с `LOCK` и для чисто серверного кода), используй SQL_CUSTOM.

### LOG — свои лог-файлы

```
9:ADD_PROTOCOL:LOG:HACKER:hacker.log    // отдельный файл
9:ADD_PROTOCOL:LOG:LOG                  // без имени файла — в основной лог extDB3
1:HACKER:player x detected hack
```

Файлов может быть сколько угодно — по протоколу на каждый.

## Отличия от extDB2 (для миграции)

- Нет SQLite, RCON, Steam, опции Sanitize и `9:SHUTDOWN`.
- Добавлены `LOCK` с кодом, `UNLOCK` и `RESET`.
- Дата и время автоматически приходят как `[Y,M,D,H,M,S]` (несовместимое изменение для `SQL`).
- NULL → `""` (по умолчанию) или `objNull`; NULL и время можно отправлять.
- В `SQL_CUSTOM` можно выбрать SQL или prepared statement; формат `.ini` немного другой, и
  неизвестные опции теперь ошибка. Прямые `$CUSTOM_x$` работают только с
  `Prepared Statement = false`.

## Прочее

- Лог запросов и ошибок — в папке `logs` extDB3; при ошибке разбора смотри туда и в RPT.
- Миграции схемы держи в репозитории сервера (одна папка, нумерованные файлы), а запрос
  в .ini сопровождай комментарием со ссылкой на миграцию, если он зависит от её решений.
