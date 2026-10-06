# Структура проекта, CfgFunctions, препроцессор, конфиги

## Структура миссии

```
MyMission.Altis/
├── mission.sqm
├── description.ext
├── stringtable.xml
├── init.sqf / initServer.sqf / initPlayerLocal.sqf
├── functions/
│   ├── CfgFunctions.hpp
│   └── core/
│       ├── fn_init.sqf
│       └── fn_doThing.sqf
└── ui/ (диалоги .hpp, картинки .paa)
```

## CfgFunctions

```cpp
// description.ext (миссия) или config.cpp (мод)
class CfgFunctions {
    class TAG {                       // тег → TAG_fnc_*
        class core {                  // категория; папка по умолчанию: functions\core
            file = "functions\core";  // необязательное переопределение
            class init { preInit = 1; };     // functions\core\fn_init.sqf → TAG_fnc_init
            class doThing {};                // → TAG_fnc_doThing
            class startLoop { postInit = 1; };
        };
    };
};
```

- Имя файла — `fn_<имя>.sqf` (или укажи `file = "path\to\file.sqf";` в классе функции).
- Функции компилируются через `compileFinal` и проходят препроцессор (макросы и комментарии можно).
- `preInit = 1` — до инициализации объектов, unscheduled. `postInit = 1` — после инициализации,
  scheduled (если важно — проверь `canSuspend`).
- `recompile = 1` у функции или `allowFunctionsRecompile = 1;` в description.ext — только для разработки.
- В конфигах пути через обратный слеш; пути мода — `\prefix\addon\...` (префикс PBO).

## Шаблон заголовка функции

```sqf
/*
    Автор: Имя
    Описание: Что делает функция.
    Параметры:
        0: OBJECT - юнит
        1: NUMBER - радиус (по умолчанию: 100)
    Возвращает: ARRAY - ближайшие враги
    Пример: [player, 200] call TAG_fnc_nearEnemies;
*/
params [["_unit", objNull, [objNull]], ["_radius", 100, [0]]];
if (isNull _unit) exitWith {[]};
...
```

## Препроцессор

- `#include "file.hpp"` (путь относительно текущего файла), `#define`, `#ifdef/#ifndef/#else/#endif`,
  `#undef`, `__FILE__`, `__LINE__`, `##` (склейка), `#` (в строку, с причудами).
- Макросы — текстовая подстановка: оборачивай аргументы в скобки
  (`#define SQR(x) ((x) * (x))`).
- Многострочные макросы — `\` в конце каждой строки.
- Работает для `.sqf`, загруженных через CfgFunctions, `execVM`, `preprocessFile(LineNumbers)`,
  `compileScript`, и для конфигов. Не работает для `compile "строка"`.

### Макросы CBA (только если проект уже зависит от CBA / использует script_component.hpp из HEMTT)

Не вводи макросы и функции CBA в проект, где CBA нет в зависимостях.

`GVAR(x)` → `TAG_component_x`, `QGVAR(x)` → `"TAG_component_x"`, `FUNC(x)` →
`TAG_component_fnc_x`, `EFUNC(comp,x)`, `PREP(x)`, `LOG`, `ERROR`. Следуй существующему
`script_component.hpp` проекта; не смешивай именование через макросы и без них.

## Синтаксис конфигов (description.ext, config.cpp, .hpp)

```cpp
class Base {
    value = 1;
    text = "Hello";
    list[] = {1, 2, "three"};       // массивам нужны []
};
class Child: Base {                 // наследование
    value = 2;
    list[] += {4};                  // добавление (работает только в некоторых контекстах)
};                                  // точка с запятой после класса обязательна
class External;                     // предварительное объявление существующего класса перед наследованием
```

- Строки в кавычках; числа без кавычек; `true/false` нет — используй `1/0`.
- Моды: `CfgPatches` с `requiredAddons[]` (обязательно перечислить аддоны, которые
  меняешь или от которых наследуешься), `units[]`, `weapons[]`, `requiredVersion`.
- Наследование от ванильного класса в моде: объявляй цепочку родителей
  (`class All; class AllVehicles: All {...}`) точно как в оригинале, никогда не
  переопределяй существующие классы с другим родителем.
- Чтение конфига из SQF: `getNumber (configFile >> "CfgVehicles" >> _type >> "maxSpeed")`,
  `getText`, `getArray`, `isClass`, `configProperties`; `missionConfigFile` — для description.ext.

## Локализация

Ключи `stringtable.xml` вида `STR_TAG_name`; в SQF — `localize "STR_TAG_name"`,
в конфигах — `$STR_TAG_name`. В крупных проектах не хардкодь текст для игрока.

## Инструменты

- HEMTT (сборка/линт модов), SQFLint / SQF-VM, расширение VS Code "SQF Language",
  Arma 3 Tools (Addon Builder, упаковка PBO). Используй то, что уже применяется в проекте.
