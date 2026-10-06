# Синтаксис SQF и подводные камни

## Приоритет операторов (от высшего к низшему)

1. Нулярные команды (`player`, `time`, `allUnits`) и унарные (`count _a`, `alive _u`, `!_b`, `-_n`)
2. `#` (hash select: `_arr # 0`)
3. `^`
4. `*` `/` `%` `mod` `atan2`
5. `+` `-` `max` `min`
6. `else`
7. Все остальные бинарные команды (`select`, `setPos`, `distance`, `in`, `isEqualTo`, `getVariable`, ...)
8. `==` `!=` `>` `<` `>=` `<=` `>>`
9. `&&` `and`
10. `||` `or`

Следствия:

```sqf
_arr select _i + 1          // = _arr select (_i + 1)
_arr # _i + 1               // = (_arr # _i) + 1   ← # связывает сильнее, чем +
count _arr - 1              // = (count _arr) - 1
_a distance _b < 100        // = (_a distance _b) < 100  ✓
!_a == _b                   // = (!_a) == _b
_u getVariable "hp" * 2     // = _u getVariable ("hp" * 2) → ошибка. Пиши (_u getVariable "hp") * 2
```

Бинарные команды одного приоритета вычисляются слева направо. Ставь скобки всегда,
когда бинарная команда смешана с арифметикой.

## Ленивое вычисление

`&&` / `||` вычисляют обе стороны, если правая сторона — не код:

```sqf
if (!isNull _veh && {alive driver _veh}) then { ... };      // driver не вычисляется, если _veh null
if (isNil "TAG_cfg" || {TAG_cfg get "debug"}) then { ... };
```

## Типы

Number (32-битный float), String, Boolean, Array, Code, Object, Group, Side, Config,
Display, Control, Location, Namespace, HashMap, Team member, Task, Script handle,
Diary record, Nothing/nil.

- `typeName` возвращает `"SCALAR"`, `"STRING"`, `"BOOL"`, `"ARRAY"`, `"CODE"`, `"OBJECT"`, `"HASHMAP"`, ...
- Вместо сравнения строки `typeName` используй `_x isEqualType 0` / `isEqualTypeAll` / типы в `params`.
- `objNull`, `grpNull`, `controlNull`, `displayNull`, `locationNull`, `scriptNull` — проверяй через `isNull`.
- `nil` нельзя хранить в большинстве мест; чтение неопределённой переменной в выражении
  даёт nil и часто тихую ошибку. Проверяй `isNil "_var"` / `isNil "VAR"`.

## Сравнения

| Выражение | Поведение |
|---|---|
| `"abc" == "ABC"` | `true` — без учёта регистра |
| `"abc" isEqualTo "ABC"` | `false` — с учётом регистра |
| `_bool == true` | не используй: исторически ошибка — пиши просто `_bool` или `isEqualTo` |
| `[1,2] == [1,2]` | ошибка — используй `isEqualTo` |
| `"a" in ["A"]` | `false` — `in` для строк регистрозависим |
| `"abc" in "xabcx"` | `true` — проверка подстроки, с учётом регистра |
| `1 == 1.0000001` | сравнение float; не сравнивай вычисленные float на точное равенство |

Булевы значения: пиши `if (_flag) then`, а не `if (_flag == true)`.

## Массивы

```sqf
private _copy = +_arr;            // глубокая копия; _b = _a копирует только ссылку
_arr pushBack _v;                 // возвращает индекс, меняет массив
_arr pushBackUnique _v;           // возвращает -1, если элемент уже есть
_arr append [_a, _b];             // меняет массив, ничего не возвращает
_arr deleteAt 0;                  // возвращает удалённый элемент
_arr deleteRange [1, 2];          // удаляет на месте: начало, количество
_arr = _arr - [_v];               // удаляет ВСЕ вхождения, создаёт новый массив;
                                  //   для массива массивов нужно `- [[...]]`
_arr resize 0;                    // очистить на месте
_arr select [1, 3];               // подмассив: начало, количество
_arr select {_x > 0};             // фильтр
_arr apply {_x * 2};              // map
_arr findIf {_x > 5};             // индекс или -1, останавливается на первом совпадении
{_x > 0} count _arr;              // условие ОБЯЗАНО вернуть Boolean (nil → ошибка)
_arr param [5, "default"];        // безопасное чтение со значением по умолчанию
```

- `select` с индексом, равным `count`, молча возвращает nil; индекс больше — ошибка.
- `sort` меняет массив на месте и работает только с однотипными числами/строками/массивами.
  Для своей сортировки — `[_arr, [], {ключ}, "ASCEND"] call BIS_fnc_sortBy`.
- Менять массив во время `forEach` по нему небезопасно — итерируй копию или собирай новый.
- `forEach` даёт `_x` и `_forEachIndex`. Во вложенном `forEach` `_x` перекрывается;
  сохраняй внешний: `{ private _unit = _x; { ... _unit ... } forEach _list; } forEach _units;`

## HashMap

```sqf
private _m = createHashMap;
private _m2 = createHashMapFromArray [["a", 1], ["b", 2]];
_m set ["key", _value];
_m get "key";                     // nil, если ключа нет
_m getOrDefault ["key", 0];
"key" in _m;                      // ключ существует
_m deleteAt "key";
keys _m; values _m;
{ diag_log [_x, _y] } forEach _m; // _x = ключ, _y = значение
```

Ключи: числа, строки (регистрозависимо), булевы, массивы из них, код, side, config...
Объекты и группы ключами быть не могут (используй `hashValue _obj` или netId).
HashMap — ссылка, как и массив; копия — `+_m`.

## Управляющие конструкции

```sqf
if (_c) then { ... } else { ... };
private _v = [0, 1] select _c;            // false → индекс 0, true → индекс 1
private _v = if (_c) then {1} else {0};   // if возвращает значение выполненной ветки

switch (_s) do {
    case "a";                              // пустой case проваливается в следующий
    case "b": { ... };
    default { ... };
};                                         // switch по строкам регистрозависим

for "_i" from 0 to (count _arr - 1) do { ... };
for "_i" from 10 to 0 step -1 do { ... };
while {_cond} do { ... };                  // условие — КОД, в фигурных скобках
waitUntil {sleep 0.5; _cond};              // последнее выражение — Boolean; только scheduled
```

- `exitWith` выходит из текущей области. В теле цикла — завершает цикл; на верхнем уровне
  функции — выходит из функции. Из вложенных блоков из функции не выходит.
- Именованные области: `scopeName "main"; ... "main" breakOut;` / `breakTo`.
- В новых версиях игры есть `continue`, `continueWith`, `break`, `breakWith` — если
  проект поддерживает старые билды, проверь минимальную версию на вики.
- `try { throw "x" } catch { _exception }` ловит только `throw`, но не ошибки скрипта движка.

## Строки

- Кавычки внутри строки экранируются удвоением: `"Он сказал ""привет"""`. Одинарные
  кавычки `'...'` тоже ограничивают строку.
- Нет escape-последовательностей вроде `\n`; используй `toString [10]` или `endl` (`endl` = CRLF).
- `format ["%1 имеет %2", name _u, _n]` — `%1`… нумеруются с 1.
- **Лимит `format`**: до Arma 3 2.18 результат молча обрезается до 8191 символа
  (с 2.18 — 8 388 608). Ошибки нет — просто теряется хвост, и, например, JSON становится
  невалидным. Если строка в теории может быть длинной — собирай без `format`:
  ```sqf
  // ПЛОХО: при большом _data JSON обрежется
  _web ctrlWebBrowserAction ["ExecJS", format ["update(%1)", toJSON _data]];

  // ХОРОШО: конкатенация — без лимита
  _web ctrlWebBrowserAction ["ExecJS", "update(" + toJSON _data + ")"];

  // ХОРОШО: много кусков — joinString (без лимита и быстрее)
  private _parts = [];
  { _parts pushBack format ["%1: %2", name _x, score _x] } forEach allPlayers;   // короткие куски — можно format
  private _text = _parts joinString endl;
  ```
  `format` годится для коротких строк с известной длиной (сообщения, имена переменных, пути).
- С 2.18 `%%` в шаблоне `format` превращается в один `%`.
- `str _v` преобразует любое значение; `str "a"` даёт `"""a"""` (с кавычками).
- `parseNumber`, `parseSimpleArray` (безопасно; никогда не делай `call compile` недоверенных строк).
- `joinString` / `splitString` (`splitString` режет по любому из символов-разделителей).
- `toLower`/`toUpper`, `trim`, `_s select [start, len]`, `_s find "sub"` (регистрозависимо).
- **Кириллица / UTF-8**: `count`, `select`, `find` и другие строковые команды по умолчанию
  работают с байтами, а не символами (`count "привет"` → 12). Нужен `forceUnicode`:
  ```sqf
  forceUnicode 1;  count _s;   // 6 — Unicode только для ОДНОЙ следующей команды
  forceUnicode 0;              // Unicode до конца текущей области видимости (и внешней)
  forceUnicode -1;             // отменить
  ```
  `toLower`/`toUpper`, `ctrlSetText`, `getTextWidth` поддерживают Unicode сами.
  `toLowerANSI`/`toUpperANSI` — быстрее, но только для ASCII/ISO-8859-1 (имена классов, ключи).
- Многократный `+` в цикле медленный; собирай в массив, потом `joinString`.

## Код и функции

```sqf
TAG_fnc_add = { params ["_a", "_b"]; _a + _b };
private _r = [1, 2] call TAG_fnc_add;      // 3 — значение последнего выражения
[_u] spawn TAG_fnc_loop;                   // возвращает script handle, выполняется scheduled
```

- `_this` содержит аргументы; `call` без левого аргумента передаёт `_this` вызывающего.
- Локальные переменные вызывающего видны внутри вызванной через `call` функции
  (динамическая область видимости). Поэтому каждая локальная переменная должна быть
  `private` — иначе перезапишешь переменную вызывающего.
- `compileFinal` делает код неизменяемым (нельзя потом перезаписать `TAG_fnc_x = {...}`);
  CfgFunctions делает это автоматически.
- `compile` не запускает препроцессор: никаких `#define` и комментариев. Используй
  `compileScript ["file.sqf"]` или `compile preprocessFileLineNumbers "file.sqf"`.

## Переменные и пространства имён

- Глобальные переменные живут в `missionNamespace` (по умолчанию). Другие: `uiNamespace`
  (переживает перезапуск миссии), `profileNamespace` (сохраняется на диск — вызывай
  `saveProfileNamespace` редко), `parsingNamespace`, `serverNamespace` (только сервер),
  `localNamespace`.
- `uiNamespace` и `profileNamespace` игрок может изменить до захода на сервер — не храни
  там код и ничего важного, только простые флаги/настройки (см. multiplayer.md,
  «Данные на стороне клиента»).
- `_obj setVariable ["TAG_name", _v]` — локально; `[..., true]` — рассылка всем (+ JIP).
  `_obj getVariable ["TAG_name", _default]` — всегда передавай значение по умолчанию.
- Имена переменных регистронезависимы (`_Unit` и `_unit` — одна переменная).
