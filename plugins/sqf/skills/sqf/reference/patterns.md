# Проверенные паттерны (из боевого MP-сервера)

Паттерны взяты из реального мода сервера с сотнями функций. Имена обезличены (`TAG_`).

## 1. Запрос клиента → проверка на сервере → ответ

Клиент ничего не решает, только просит. Сервер сам определяет, кто спросил, и перепроверяет всё.

```sqf
// Клиент
[_warehouseVar, _itemId] remoteExecCall ["TAG_srv_warehouse_take", 2];
```

```sqf
// Сервер: TAG_srv_warehouse_take.sqf
/*
    Клиент зовёт через remoteExecCall — без планировщика, поэтому проверка остатка
    и его списание не перемежаются с запросом другого игрока.
    Окну не верим ни в чём — всё проверяется заново.
*/
#include "\TAG_server\script_macros.hpp"     // WAREHOUSE_TAKE_COOLDOWN, NOTIFY_TIME

params [["_var", "", [""]], ["_itemId", 0, [0]]];

private _owner  = remoteExecutedOwner;
private _player = _owner call TAG_srv_getRemotePlayer;
if (isNull _player) exitWith {};

private _fail = {
    ["Склад", _this, "warning", NOTIFY_TIME] remoteExec ["TAG_cl_notify", _owner];
};

if (diag_tickTime < (_player getVariable ["TAG_nextTake", 0])) exitWith {
    "Подождите немного" call _fail;
};
if ((TAG_stock getOrDefault [_itemId, 0]) <= 0) exitWith {"Нет на складе" call _fail};

// ... выдача ...
_player setVariable ["TAG_nextTake", diag_tickTime + WAREHOUSE_TAKE_COOLDOWN];   // локально на сервере, без рассылки
TAG_stock set [_itemId, (TAG_stock get _itemId) - 1];
```

```sqf
// TAG_srv_getRemotePlayer.sqf — игрок по owner ID отправителя
params [["_owner", 0, [0]]];
if (_owner in [0, -2, 2]) exitWith {objNull};      // не remoteExec или сам сервер
private _players = allPlayers select {owner _x isEqualTo _owner};
[objNull, _players select 0] select (count _players > 0)
```

Ключевое:
- Игрока берём из `remoteExecutedOwner`, **а не из аргументов**: объект игрока в
  параметрах клиент может подменить.
- `remoteExecCall` на сервер = unscheduled = атомарная проверка и изменение общего состояния.
- Кулдауны на сервере храним в локальном `setVariable` объекта (без флага public).
- Ответ отправляем только отправителю: `remoteExec [..., _owner]`.
- Цепочка проверок одним выражением с ленивыми `{}`:
  ```sqf
  if !(
      alive _player &&
      {isNull objectParent _player} &&
      {_side in [west, east]} &&
      {(getPosASL _player) distance _posASL <= RETRANSLATOR_MAX_PLACE_DISTANCE} &&
      {!surfaceIsWater _posASL}
  ) exitWith {};
  ```

## 2. Мод только на сервере, клиентский код рассылается сервером

Если клиенты не качают мод (server-side only), клиентские функции компилируются на
сервере и рассылаются как публичные переменные:

Проблема: клиент (особенно JIP) может начать инициализацию раньше, чем дошли все функции,
а часть рассылки может потеряться. Решение — сервер вместе с функциями рассылает **их количество**. Клиент считает, сколько функций
дошло до него, и если число не совпадает (или самого счётчика ещё нет) — просит сервер
прислать функции заново. Список имён клиенту не нужен.

```cpp
// script_macros.hpp
//Префикс клиентских функций, которые рассылает сервер. Только для функций: переменные
//с этим префиксом собьют счётчик (allVariables возвращает имена в нижнем регистре)
#define CLIENT_FNC_PREFIX "tag_cl_"
//Публичная переменная с количеством клиентских функций
#define CLIENT_FNC_COUNT_VAR "TAG_clientFunctionsCount"
//Как часто клиент повторяет запрос, пока функции не дошли, с
#define CLIENT_FNC_RETRY_INTERVAL 5
//Не чаще какого интервала сервер отвечает одному клиенту, с
#define CLIENT_FNC_RESEND_COOLDOWN 3
```

```sqf
// preInit мода на сервере
#include "\TAG_server\script_macros.hpp"

TAG_srv_clientFunctions = [];          // имена — только на сервере, клиентам не рассылаются
{
    _x params ["_name", "_path"];
    private _code = compileFinal preprocessFileLineNumbers format ["\TAG_client\code\%1.sqf", _path];
    missionNamespace setVariable [_name, _code, true];         // всем + JIP
    TAG_srv_clientFunctions pushBack _name;
} forEach [
    ["TAG_cl_notify", "ui\TAG_cl_notify"],
    ...
];

// Счётчик — после функций: клиент ждёт, пока число дошедших функций с ним совпадёт
missionNamespace setVariable [CLIENT_FNC_COUNT_VAR, count TAG_srv_clientFunctions, true];
TAG_srv_resendCooldown = createHashMap;
```

```sqf
// postInit миссии на клиенте (миссия есть у клиента, мод — нет)
#include "script_macros.hpp"

[] spawn {
    private _localCount = {
        {
            _x find CLIENT_FNC_PREFIX == 0 &&
            {(missionNamespace getVariable _x) isEqualType {}}
        } count (allVariables missionNamespace)
    };

    private _nextRequest = 0;
    waitUntil {
        private _expected = missionNamespace getVariable [CLIENT_FNC_COUNT_VAR, -1];
        // Счётчика нет или функций не столько, сколько разослал сервер
        private _ready = _expected >= 0 && {(call _localCount) == _expected};

        if (!_ready && {diag_tickTime >= _nextRequest}) then {
            [] remoteExecCall ["TAG_srv_resendFunctions", 2];
            _nextRequest = diag_tickTime + CLIENT_FNC_RETRY_INTERVAL;   // повтор, если ответ потерялся
        };
        if (!_ready) then {sleep 0.5};
        _ready
    };

    [] call TAG_cl_init;
};
```

```sqf
// TAG_srv_resendFunctions.sqf — сервер: шлём функции и счётчик только запросившему
#include "\TAG_server\script_macros.hpp"

private _owner = remoteExecutedOwner;
if (_owner <= 2) exitWith {};          // 0 — не remoteExec, 2 — сам сервер (у хоста функции и так есть)

// Защита от спама запросами: каждый ответ — пачка кода по сети
if (diag_tickTime < (TAG_srv_resendCooldown getOrDefault [_owner, 0])) exitWith {};
TAG_srv_resendCooldown set [_owner, diag_tickTime + CLIENT_FNC_RESEND_COOLDOWN];

{
    _owner publicVariableClient _x;
} forEach TAG_srv_clientFunctions;     // только из серверного списка, не из аргументов клиента
_owner publicVariableClient CLIENT_FNC_COUNT_VAR;   // счётчик последним
```

Нюансы:
- `allVariables` возвращает имена **в нижнем регистре** — префикс сравнивай в нижнем регистре.
- Префикс клиентских функций не должен использоваться для обычных переменных, иначе
  счётчик на клиенте не совпадёт никогда. Фильтр `isEqualType {}` частично защищает от этого.
- Клиент ничего не передаёт в запросе: сервер отправляет только то, что сам скомпилировал.
  Никогда не рассылай переменные по именам, присланным клиентом.
- Серверные функции не рассылаются (`setVariable [_name, _code]` без `true`): клиенту незачем
  видеть серверную логику.
- Ключи hashmap кулдауна — owner ID; при отключении игрока запись можно удалить
  (`HandleDisconnect` / `PlayerDisconnected`), иначе она просто останется маленьким мусором.

## 3. HandleDamage

```sqf
private _handleDamage = {
    params ["_unit", "_selection", "_damage", "_source", "_projectile", "_hitIndex", "_instigator", "_hitPoint"];

    // EH срабатывает много раз за одно попадание — кешируем дорогую проверку на кадр
    private _cache = _unit getVariable ["TAG_szCache", [-1, false]];
    private _inSafezone = if ((_cache # 0) == diag_frameNo) then {_cache # 1} else {
        private _r = (TAG_safezones findIf {_unit inArea _x}) != -1;
        _unit setVariable ["TAG_szCache", [diag_frameNo, _r]];
        _r
    };

    if (_inSafezone) then {
        [_unit getHitIndex _hitIndex, damage _unit] select (_hitIndex < 0)   // урон не меняется
    } else {
        nil     // ничего не возвращаем → остаётся урон, посчитанный другими обработчиками/движком
    };
};
_veh setVariable ["TAG_hdId", _veh addEventHandler ["HandleDamage", _handleDamage]];
```

- Движок использует возвращаемое значение **только последнего добавленного** `HandleDamage`.
  Моды (медицина, ACE и т.п.) могут добавить свой позже — тогда твой возврат игнорируется.
  Если твой должен быть последним: некоторое время после спавна проверяй через
  `getEventHandlerInfo` и переподвешивай.
- `getEventHandlerInfo` по несуществующему id возвращает `[]` — проверяй до `select`.
- Возврат `nil` = «не вмешиваюсь». Возврат числа — новый урон этой части.
- `_hitIndex < 0` — урон по всему объекту (`damage`), иначе по hit point (`getHitIndex`).

## 4. Event handlers UI: не копить и переподвешивать

```sqf
addMissionEventHandler ["Map", {
    params ["_opened"];
    if (!_opened) exitWith {};
    private _display = findDisplay 12;

    // EH на display 12 вешается при каждом открытии карты — снимаем предыдущий, иначе копятся
    private _old = _display getVariable ["TAG_mouseMovingId", -1];
    if (_old != -1) then {_display displayRemoveEventHandler ["MouseMoving", _old]};
    _display setVariable ["TAG_mouseMovingId", _display displayAddEventHandler ["MouseMoving", {...}]];
}];
```

- Id обработчика храни в `getVariable` самого дисплея/контрола или в `uiNamespace`.
- Обработчики на контроле карты (`findDisplay 12 displayCtrl 51`) бывает слетают
  (другие моды пересоздают контролы) — периодически проверяй `getEventHandlerInfo` и вешай заново.
- В `KeyDown` возврат `true` перехватывает клавишу (например, Esc не закроет диалог) —
  возвращай `true` только когда действительно обработал.
- `MarkerCreated` с префиксом `_USER_DEFINED` — пользовательские маркеры; так можно
  заменить стандартную систему маркеров своей серверной (с проверкой видимости).
- В UI-коде, который хранит контролы/дисплеи в переменных scheduled-скрипта, — `disableSerialization;`.

## 5. Обновление UI по разнице

Если UI перерисовывается по таймеру, отправляй только изменившиеся поля:

```sqf
private _last  = missionNamespace getVariable ["TAG_ui_lastSent", createHashMap];
private _patch = createHashMap;
{
    // заглушка "@none" вместо nil: значения — числа/булевы, совпасть не могут
    if ((_last getOrDefault [_x, "@none"]) isNotEqualTo _y) then {_patch set [_x, _y]};
} forEach _data;
if (count _patch == 0) exitWith {};
missionNamespace setVariable ["TAG_ui_lastSent", +_data];     // копия, а не ссылка!
```

После перезагрузки страницы/диалога сбрасывай `lastSent`, чтобы следующий пакет ушёл целиком.

## 6. Никаких «магических» данных — константы в #define

Числа и строки с особым смыслом (задержки, радиусы, лимиты, цены, роли, классы, имена
переменных и маркеров, тексты сообщений, IDD/IDC) не пишутся прямо в коде. Они
выносятся в `#define` в общий `.hpp` (например `script_macros.hpp`) с комментарием:
что значит и в каких единицах.

```cpp
// script_macros.hpp

//Склады (TAG_srv_warehouse_*, TAG_cl_warehouse_*)
//Радиус действия «Склад» на объекте, м
#define WAREHOUSE_ACTION_RADIUS 5
//Насколько далеко от склада игрок может получить элемент (проверка на сервере), м
#define WAREHOUSE_USE_DISTANCE 15
//Пауза между двумя выдачами одному игроку, с
#define WAREHOUSE_TAKE_COOLDOWN 5

//Ретрансляторы FPV
//На каком расстоянии перед игроком ставится, м (клиент)
#define RETRANSLATOR_PLACE_DISTANCE 1.5
//Допуск при проверке на сервере, м — шире клиентского из-за рассинхрона позиций
#define RETRANSLATOR_MAX_PLACE_DISTANCE 5
//Сколько живых ретрансляторов может стоять у одной стороны
#define RETRANSLATOR_SIDE_LIMIT 2
//Роли, которым доступна установка
#define RETRANSLATOR_ROLES ["KomandirBPLA", "SpecBPLA"]
//Имя публичной переменной со списком ретрансляторов стороны
#define RETRANSLATOR_LIST_VAR(SIDE) (format ["TAG_retranslators_%1", SIDE])
#define RETRANSLATOR_LIMIT_MESSAGE (format ["Ваша сторона уже установила максимум: %1", RETRANSLATOR_SIDE_LIMIT])

//Время показа уведомления, с
#define NOTIFY_TIME 5
```

```sqf
#include "\TAG_server\script_macros.hpp"

// ПЛОХО: что такое 15? в скольких местах ещё написано 15?
if (_player distance _box > 15) exitWith {};

// ХОРОШО
if (_player distance _box > WAREHOUSE_USE_DISTANCE) exitWith {};
```

Правила:
- Один `.hpp` подключают и клиент (подсказка, UI), и сервер (проверка) — значения не расходятся.
- Если в коде, который правишь, встречается магическое число/строка, используемая в
  нескольких местах или имеющая смысл настройки, — вынеси в `#define` и замени все вхождения.
  Очевидные значения (`0`, `1`, `-1` как «не найдено», `[0,0,0]`, `count _arr - 1`) выносить не нужно.
- Макросы — текстовая подстановка. Выражения оборачивай в скобки:
  `#define TOTAL_TIME (WARMUP_TIME + GAME_TIME)`, иначе `TOTAL_TIME * 2` посчитается неверно.
- Строки — в кавычках внутри макроса: `#define DEFAULT_ROLE "Rifleman"`.
- Внутри строковых литералов макросы не раскрываются: в условии `addAction`, записанном
  строкой, `"_target distance _this < WAREHOUSE_ACTION_RADIUS"` останется как есть.
  Собирай такую строку через `format ["_target distance _this < %1", WAREHOUSE_ACTION_RADIUS]`.
- `#define` работает только в файлах, которые проходят через препроцессор (CfgFunctions,
  `execVM`, `preprocessFileLineNumbers`, `compileScript`), и в конфигах — не в `compile "строка"`.
- `#define` фиксируется при сборке PBO. Если значение должно меняться без перепаковки
  мода — в конфиг миссии (`missionConfigFile`), CBA settings (если CBA есть в сборке) или БД.
- Имена макросов — `UPPER_SNAKE_CASE` с префиксом системы (`WAREHOUSE_`, `RETRANSLATOR_`);
  константы группируй по системам с заголовком-комментарием. Если в проекте уже есть
  `.hpp` с макросами — добавляй туда и следуй его стилю.

## 7. Мелочи

- Имена классов для сравнения: `toLowerANSI typeOf _veh` (быстрее `toLower`, для ASCII достаточно).
- `isNotEqualTo` вместо `!(a isEqualTo b)`.
- Текст игрока в structured text экранируй:
  `_s regexReplace ["&/g", "&amp;"] regexReplace ["</g", "&lt;"] regexReplace [">/g", "&gt;"]`.
- Объект на точной позиции: `createVehicle [_type, [0,0,0], [], 0, "CAN_COLLIDE"]`,
  потом `setPosASL` и `setVectorDirAndUp [[sin _dir, cos _dir, 0], [0,0,1]]`.
  Для статики — `enableSimulationGlobal false`.
- Список объектов в публичной переменной: перед добавлением фильтруй мёртвых
  (`select {alive _x}`), рассылай одним `setVariable [.., true]`.
- Логи важных действий игроков пиши на сервере с именем и `getPlayerUID`.

## 8. extDB3 (MySQL)

Вынесено в отдельный файл: [extdb3.md](extdb3.md).
