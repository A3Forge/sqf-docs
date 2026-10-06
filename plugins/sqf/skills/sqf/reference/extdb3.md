# extDB3 (MySQL) и SQL_CUSTOM

Нюансы взяты из рабочего сервера (протокол `SQL_CUSTOM`, файл запросов `sql_custom/<имя>.ini`).
Работа с БД — **только на сервере**.

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
- Проверяй, что расширение загружено (`"extDB3" callExtension "9:VERSION"` не пустой),
  и не запускай логику, зависящую от БД, если подключение не удалось.

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

// большой результат: читаем по ключу, пока не вернётся пустая строка
private _out = "";
while {true} do {
    private _part = "extDB3" callExtension ("5:" + _key);
    if (_part isEqualTo "") exitWith {};
    _out = _out + _part;               // конкатенация, не format: строка может быть огромной
};
parseSimpleArray _out
```

- Ответ всегда разбирай `parseSimpleArray`, никогда `call compile`.
- Синхронный режим блокирует кадр сервера на время запроса: тяжёлое и не нужное
  сразу — асинхронно (режим 1).
- Строку запроса собирай через `joinString` / `+`, не `format` (лимит длины `format`,
  см. syntax-pitfalls.md).

### Входные параметры

- Входы разделяются `:` — **двоеточие внутри значения ломает разбор** (входов
  становится больше, запрос падает). Значения, введённые игроком (имена, тексты),
  очищай от `:` до отправки, либо передавай их не строкой, а числом/идентификатором.
- `Strip Chars` в `[Default]` — символы, которые extDB3 вырезает из входов
  (например `"/\|;{}<>'\`"`), `Strip Chars Mode` — что делать при их нахождении (вырезать /
  логировать / ошибка — см. документацию extDB3). Это защита, а не замена валидации на сервере.
- Опция входа `N-mysql_escape` (`SQL1_INPUTS = 1-mysql_escape,2-mysql_escape`) экранирует
  значение средствами MySQL — для произвольного текста.
- Типы входов проверяй в SQF до запроса (`params` с типами): в строку уйдёт что угодно.

## Описание запросов в SQL_CUSTOM .ini

```ini
[Default]
Version = 1
Strip Chars = "/\|;{}<>'`"
Strip Chars Mode = 1
Input SQF Parser = false

; Комментарий над каждым запросом: кто вызывает, что возвращает, неочевидные решения
[getWarehouses]
SQL1_1 = SELECT id, var_name, REPLACE(name, '"', '""'), side FROM cfg_warehouses ORDER BY id
OUTPUT = 1,2-STRING,3-STRING,4-STRING

[addKillLog]
SQL1_1 = INSERT INTO kill_logs (session_id, killer, victim) VALUES (?, ?, ?)
SQL1_INPUTS = 1,2,3

[createSession]
SQL1_1 = INSERT INTO game_sessions (`mission`, `map`) VALUES (?, ?)
SQL1_INPUTS = 1-mysql_escape,2-mysql_escape
Return InsertID = true
```

- `?` — плейсхолдеры prepared statement; `SQL1_INPUTS` — какие входы в каком порядке.
- Длинный запрос можно разбить на строки `SQL1_1`, `SQL1_2`, ... (склеиваются).
- `Return InsertID = true` — вернуть id вставленной строки. Заодно позволяет вызвать
  INSERT синхронно и **дождаться коммита** перед следующим запросом, который на него опирается.
- UPSERT: `INSERT ... ON DUPLICATE KEY UPDATE col = VALUES(col)` / `minutes = minutes + 1`.
- `Prepared Statement = false` + `$CUSTOM_1$` вставляет вход в SQL как есть — это
  SQL-инъекция, если туда попадёт что-то от клиента. Только для строк, целиком собранных
  сервером, а лучше не использовать вовсе.

### Вывод: кавычки, переводы строк, JSON

- `N-STRING` в `OUTPUT` оборачивает значение в кавычки для SQF, но **кавычки внутри
  значения extDB3 не экранирует**. Текст, который вводят люди (имена отрядов, ролей,
  названия, описания с сайта), с `"` внутри ломает `parseSimpleArray` всего ответа.
  Удваивай кавычки в самом SELECT:
  ```sql
  SELECT id, REPLACE(name, '"', '""') FROM `groups`
  ```
- Переводы строк в тексте: `CHAR(13)` убирай, `CHAR(10)` заменяй на то, что понимает
  получатель (`<br>` для structured text / HTML, пробел для обычного текста):
  ```sql
  REPLACE(REPLACE(REPLACE(description, '"', '""'), CHAR(13), ''), CHAR(10), '<br>')
  ```
- Колонки, где хранится готовый SQF/JSON-массив (TEXT с `[...]` внутри), отдавай **без**
  `-STRING` — они придут уже массивами. Пустое значение делай через
  `NOT NULL DEFAULT '[]'` в схеме, а не через функции в SELECT (см. ниже).
- Числа и булевы — без `-STRING`.
- NULL в коротких колонках приводи явно: `IFNULL(rank_slug, '')`.

### LONGTEXT и prepared statements

extDB3 не умеет читать LONGTEXT из подготовленного запроса — падает **весь** запрос:
`MYSQL_TYPE_LONG_BLOB type not supported when using Prepared Statements`.

- Не храни такие данные в LONGTEXT.
- Не применяй `IFNULL`, `CONCAT`, `CAST`, `CASE` и подобные функции к MEDIUMTEXT/TEXT-колонкам
  в SELECT: MySQL расширяет тип результата до LONGTEXT. Читай такие колонки «голыми»,
  а NULL исключай схемой (`NOT NULL DEFAULT ...`).

## Прочее

- Лог запросов и ошибок — в папке `logs` extDB3; при ошибке разбора смотри туда и в RPT.
- Миграции схемы держи в репозитории сервера (одна папка, нумерованные файлы), а запрос
  в .ini сопровождай комментарием со ссылкой на миграцию, если он зависит от её решений.
- Конфиг подключения (`extdb3-conf.ini`) содержит логин и пароль к БД — не коммить его
  в публичные репозитории.
