# Моды интерфейса: главное меню и ванильные окна

Как из мода менять ванильные окна (главное меню, дебрифинг, браузер серверов, окна
сообщений движка) и делать в меню своё поведение: обновление по кадрам, подключение к
серверу, отслеживание выхода с сервера, окно переподключения.

Проверено на Arma 3 2.22 (мод главного меню с расширением и тестами через sqf-mcp).

## Точка входа: невидимый контрол с onLoad

Чтобы выполнить код при каждом открытии ванильного окна, добавь в `controls` его класса
пустой контрол с `onLoad`. Сам `onLoad` окна не трогай: его уже задают ванильные скрипты
и другие моды.

```cpp
class RscText;
class RscStandardDisplay;
class RscDisplayMain: RscStandardDisplay {
    class controls {
        class TAG_MainMenuHook: RscText {
            idc = -1; x = 0; y = 0; w = 0; h = 0; text = "";
            onLoad = "_this call (uiNamespace getVariable 'TAG_fnc_mainMenuHook')";
        };
    };
};
```

- `onLoad` контрола выполняется unscheduled до первого кадра окна. Если строить свой
  интерфейс прямо здесь, ванильный не успеет мелькнуть. `_this select 0` — контрол,
  окно — `ctrlParent`.
- Так же подключаются `RscDisplayDebriefing` (IDD 50), `RscDisplayMultiplayer`
  (браузер серверов, IDD 8), `RscDisplayLogin` (профиль).
- **`requiredAddons[]`** — перечисли все моды, которые меняют те же классы (добавляют
  кнопки в меню, переопределяют `onLoad`). Иначе порядок загрузки случайный, и их правки
  перекроют твои. Кроме ванильных `A3_Ui_F` и `A3_Data_F_Decade_Loadorder`, ищи такие моды
  в сборке по имени класса.
- Удалить чужой контрол из окна — `delete ClassName;` внутри `class controls`.
- Если всё-таки нужно переопределить `onLoad` окна целиком (так сделано для
  `RscMsgBox`, см. ниже), повтори в нём ванильный вызов, иначе сломаются скрипты BIS:
  ```cpp
  class RscMsgBox {
      onLoad = "[""onLoad"", _this, 'RscMsgBox'] call (uiNamespace getVariable 'BIS_fnc_initDisplay'); [_this select 0] call (uiNamespace getVariable 'TAG_fnc_msgBox')";
  };
  ```

## Функции интерфейса: preStart и final

В меню нет миссии, поэтому функции интерфейса компилируются в `uiNamespace` при запуске
игры через функцию с `preStart = 1` в CfgFunctions:

```sqf
// fn_preStart.sqf
{
    private _path = format ["\TAG_ui\functions\fn_%1.sqf", _x];
    if (fileExists _path) then {
        uiNamespace setVariable ["TAG_fnc_" + _x, compileScript [_path, true]];   // true — final
    } else {
        // пустая final-заглушка: иначе свободное имя функции можно занять своим кодом
        uiNamespace setVariable ["TAG_fnc_" + _x, compileFinal ""];
        diag_log format ["[TAG] preStart: нет файла %1", _path];
    };
} forEach ["mainMenuHook", "mainMenuInit", "mainMenuTick"];
```

- Вызывай функции как `call (uiNamespace getVariable "TAG_fnc_x")`, в том числе из
  `onLoad` в конфиге и из обработчиков событий.
- `preStart` выполняется раньше, чем игрок может запустить свой код, а final-значение
  нельзя перезаписать. Проверено: после `uiNamespace setVariable ["TAG_fnc_x", {...}]`
  функция остаётся прежней, без ошибки. Проверка — `isFinal (uiNamespace getVariable "TAG_fnc_x")`.
- **Из-за final горячая подгрузка не работает**: заменить функцию через debug console или
  MCP нельзя. Правка проверяется только пересборкой PBO и перезапуском игры.
- Код в переменных окон и контролов не храни (`_display setVariable ["TAG_fnc_close", {...}]`):
  пиши его прямо в обработчике. Данные (флаги, контролы, числа) хранить можно.
- Функции с `postInit = 1` из того же CfgFunctions выполняются в каждой миссии, как обычно.

## Обновление по кадрам в меню

- **`spawn` в главном меню не годится**: при загрузке мира главного меню и при завершении
  любой миссии движок снимает scheduled-скрипты.
- Вместо цикла повесь на окно `MouseMoving` и `MouseHolding`. Активное окно получает одно
  из них каждый кадр: `MouseMoving` при движении мыши, `MouseHolding` когда мышь стоит.
  ```sqf
  {
      _display displayAddEventHandler [_x, {
          params ["_d"];
          [_d] call (uiNamespace getVariable "TAG_fnc_mainMenuTick");
      }];
  } forEach ["MouseMoving", "MouseHolding"];
  ```
- События получает только окно, которое сверху. Пока открыт дочерний диалог или модальное
  окно, «тик» главного меню стоит.
- Тяжёлую работу внутри тика ограничивай по времени (`diag_tickTime` и интервал в
  переменной окна).

## Закрыть окно сразу после открытия

`closeDisplay` внутри `onLoad` не работает. Закрывай на первом кадре:

```sqf
// onLoad контрола-хука в RscDisplayDebriefing: в MP итоги не показываем
params ["_hook"];
private _display = ctrlParent _hook;
if (isNull _display || {!isMultiplayer}) exitWith {};
{_x ctrlShow false} forEach (allControls _display);    // спрятать до закрытия
{
    _display displayAddEventHandler [_x, {
        params ["_display"];
        if (_display getVariable ["TAG_closing", false]) exitWith {};
        _display setVariable ["TAG_closing", true];
        _display closeDisplay 1;                       // как «Далее» (IDC 1)
    }];
} forEach ["MouseMoving", "MouseHolding"];
```

## Кнопки из нескольких контролов

- Составную кнопку (фон, текст, иконка) рисуй отдельными контролами, а поверх клади один
  прозрачный `RscButton` для кликов (`text = ""`, все `color*[] = {0,0,0,0}`,
  `offsetPressedX/Y = 0`).
- **`RscActiveText` с пустым текстом не ловит клики мышью**: у него нет области нажатия.
  Нужен именно `RscButton`.
- **Кликабельные контролы не должны перекрываться**: реальный клик мышью может уйти не
  верхнему контролу. `ctrlActivate` из скрипта этого не показывает, проверяй мышью.
- Подсветка — `MouseEnter`/`MouseExit` на кликабельном контроле, действие — `ButtonClick`.
- Кнопка-ссылка: свойство `url = "https://..."` у класса кнопки. Домен должен быть в
  `class CfgCommands { allowedHTMLLoadURIs[] += {"https://t.me/*"}; };`.

## Свои окна из контролов (без классов диалогов в конфиге)

Окна меню удобно строить в скрипте через `ctrlCreate` в группах. Тогда не нужны классы
диалогов в конфиге, а окно живёт внутри существующего дисплея и получает его события.

### Единицы: макет 1920×1080 в любом разрешении

```sqf
// «Пиксель макета»: высота экрана = 1080 единиц, ширина в тех же единицах, без растяжения
private _py = safeZoneH / 1080;
private _px = _py * pixelW / pixelH;
_display setVariable ["TAG_px", _px];      // обработчикам событий нужны те же единицы
_display setVariable ["TAG_py", _py];

private _fnc_p = {
    params ["_x", "_y", "_w", "_h"];
    [_x * _px, _y * _py, _w * _px, _h * _py]
};
```

Позиции контролов внутри группы отсчитываются от левого верхнего угла группы, поэтому
внутри окна пиши координаты макета: `[[32, 100, 200, 20] call _fnc_p, ...]`.

### Помощники

```sqf
private _fnc_ctrl = {
    params ["_cls", "_pos", ["_grp", controlNull], ["_idc", -1]];
    private _c = _display ctrlCreate [_cls, _idc, _grp];   // _grp = controlNull — прямо в дисплей
    _c ctrlSetPosition _pos;
    _c ctrlCommit 0;
    _c
};
private _fnc_rect = {
    params ["_pos", "_color", ["_grp", controlNull]];
    private _c = ["RscText", _pos, _grp] call _fnc_ctrl;
    _c ctrlSetBackgroundColor _color;
    _c
};
private _fnc_text = {
    params ["_pos", "_text", "_grp", "_font", "_size", "_color", ["_cls", "RscText"]];
    private _c = [_cls, _pos, _grp] call _fnc_ctrl;
    _c ctrlSetFont _font;
    _c ctrlSetFontHeight (_size * _py);          // размер шрифта — тоже в единицах макета
    _c ctrlSetTextColor _color;
    _c ctrlSetText _text;
    _c
};
```

### Модальное окно поверх меню

```sqf
// Слой на весь экран: затемнение + клик мимо панели = «Отмена»
private _dlg = ["RscControlsGroupNoScrollbars", [safeZoneX, safeZoneY, safeZoneW, safeZoneH]] call _fnc_ctrl;
_display setVariable ["TAG_connectDlg", _dlg];
[[0, 0, safeZoneW, safeZoneH], [0.03, 0.025, 0.02, 0.75], _dlg] call _fnc_rect;

private _pw = 560 * _px;
private _ph = 300 * _py;
private _x0 = (safeZoneW - _pw) / 2;
private _y0 = (safeZoneH - _ph) / 2;

// Четыре кликабельные области вокруг панели, а не одна под ней: кликабельные контролы
// не должны перекрываться (см. «Кнопки из нескольких контролов»)
{
    private _out = ["TAG_HitArea", _x, _dlg] call _fnc_ctrl;
    _out ctrlAddEventHandler ["ButtonClick", {
        [ctrlParent (_this select 0), "close"] call (uiNamespace getVariable "TAG_fnc_connectDlg");
    }];
} forEach [
    [0, 0, safeZoneW, _y0],
    [0, _y0 + _ph, safeZoneW, safeZoneH - _y0 - _ph],
    [0, _y0, _x0, _ph],
    [_x0 + _pw, _y0, safeZoneW - _x0 - _pw, _ph]
];

private _panel = ["RscControlsGroupNoScrollbars", [_x0, _y0, _pw, _ph], _dlg] call _fnc_ctrl;
[[0, 0, 560, 300] call _fnc_p, [0.075, 0.065, 0.055, 0.96], _panel] call _fnc_rect;

// Поле ввода: фон и цвет рамки при фокусе
private _ipEdit = [[32, 122, 380, 44] call _fnc_p, "127.0.0.1", _panel, "RobotoCondensed", 22, [1, 1, 1, 1], "RscEdit"] call _fnc_text;
_ipEdit ctrlSetBackgroundColor [0.12, 0.105, 0.09, 1];
_ipEdit ctrlSetActiveColor [0.72, 0.33, 0.23, 1];
_display setVariable ["TAG_connectEdits", [_ipEdit]];
ctrlSetFocus _ipEdit;

// Плавное появление
_dlg ctrlSetFade 1;
_dlg ctrlCommit 0;
_dlg ctrlSetFade 0;
_dlg ctrlCommit 0.15;
```

- Закрыть окно — `ctrlDelete` группы. Если закрытие вызвано из обработчика кнопки внутри
  этой группы, удаляй **последним действием**, после всех остальных строк.
- Ссылку на окно храни в переменной дисплея: так её найдут тик, KeyDown и кнопки.
  Открыто ли окно — `!isNull (_display getVariable ["TAG_connectDlg", controlNull])`.
- Поля читай при подтверждении: `trim ctrlText _ipEdit`. Числа — `parseNumber` с проверкой
  диапазона и `_n == floor _n`.
- Структурированный текст с цветами: контрол `RscStructuredText`,
  `ctrlSetStructuredText parseText format ["<t font='%1' color='%2'>%3</t>", ...]`.
- `RscEdit` не умеет прятать ввод (точки вместо символов) — пароль в нём виден.

### Наведение и клик по составной кнопке

```sqf
private _bg  = [[32, 212, 240, 52] call _fnc_p, _color, _panel] call _fnc_rect;
[[52, 212, 200, 52] call _fnc_p, "ПОДКЛЮЧИТЬСЯ", _panel, "PuristaMedium", 22, [1, 1, 1, 1]] call _fnc_text;
private _hit = ["TAG_HitArea", [32, 212, 240, 52] call _fnc_p, _panel] call _fnc_ctrl;
_hit setVariable ["TAG_parts", [_bg, _color, _hoverColor]];   // данные — в переменных контрола
_hit ctrlAddEventHandler ["MouseEnter", {
    ((_this select 0) getVariable "TAG_parts") params ["_bg", "", "_hover"];
    _bg ctrlSetBackgroundColor _hover;
}];
_hit ctrlAddEventHandler ["MouseExit", {
    ((_this select 0) getVariable "TAG_parts") params ["_bg", "_color"];
    _bg ctrlSetBackgroundColor _color;
}];
_hit ctrlAddEventHandler ["ButtonClick", {
    params ["_c"];
    [ctrlParent _c, "confirm"] call (uiNamespace getVariable "TAG_fnc_connectDlg");
}];
```

### Клавиши: одно KeyDown на дисплей

```sqf
_display displayAddEventHandler ["KeyDown", {
    params ["_d", "_key", "_shift", "_ctrl", "_alt"];
    // открыто своё окно: ESC — закрыть, Enter — подтвердить, остальное — в поля ввода
    if (!isNull (_d getVariable ["TAG_connectDlg", controlNull])) exitWith {
        switch (true) do {
            case (_key == 1): {[_d, "close"] call (uiNamespace getVariable "TAG_fnc_connectDlg"); true};       // DIK_ESCAPE
            case (_key in [28, 156]): {[_d, "confirm"] call (uiNamespace getVariable "TAG_fnc_connectDlg"); true}; // Enter, Num Enter
            default {false};
        }
    };
    // ... горячие клавиши самого меню
    false
}];
```

- `true` — клавиша обработана, движок её не получит. Без этого ESC в главном меню уйдёт
  ванильному меню, а в окне без полей ввода «лишние» клавиши лучше тоже глушить (`true`).
- Коды DIK вынеси в `#define`.

## Подключение к серверу из меню

```sqf
connectToServer [_ip, _port, _password];
```

- Работает **только из UI-события** (клик, клавиша, `MouseMoving`) и **только когда
  окно игры в фокусе**. Из debug console, MCP или кода без UI-события молча ничего не
  делает. Для теста через MCP: активировать окно игры (WinAPI `AppActivate`) и выполнить
  команду из одноразового `MouseMoving` на дисплее 0.
- Сама открывает браузер серверов (`RscDisplayMultiplayer`) и подключается из него. Если
  мод прячет браузер, первые секунды после `connectToServer` его закрывать нельзя.
- Адрес, порт и пароль, с которыми подключались, запоминай сам: у клиента нет команды,
  которая вернёт адрес текущего сервера.
- Статус подключения — `getClientStateNumber`. Наблюдалось: 0 — меню, 1 — подключение,
  5–6 — загрузка и брифинг, 8 — миссия загружается, 10 — в игре.

## Окна сообщений движка (RscMsgBox)

Кик, потеря связи, подтверждение выхода, ошибки подключения показываются окном
`RscMsgBox`. Как найти его из скрипта и нажать кнопку — см.
[game-navigation.md](game-navigation.md).

- **`onLoad` класса `RscMsgBox` в конфиге срабатывает всегда**, даже когда окно
  показывается после выгрузки миссии (кик, обрыв связи). Скриптовый
  `OnDisplayRegistered` и обработчики в `missionNamespace` в этот момент уже не работают.
- **Текст может появиться уже после `onLoad`** (при обрыве связи), а может быть сразу (кик).
  Проверяй текст в трёх местах: при открытии, на кадрах окна (`MouseMoving`/`MouseHolding`)
  и при закрытии (`Unload`).
- **Обработчики ставь первыми, до проверок текста.** Если ветка «текст уже есть» выходит
  через `exitWith` раньше, обработчики кадров и `Unload` не появятся.
- Код закрытия — второй параметр `Unload`: 1 — «Да» / «OK», 2 — «Нет» / «Отмена».
- Закрыть окно само (игроку не нужно нажимать «OK») можно `_display closeDisplay 1`
  из обработчика кадра окна.
- Пока окно открыто, игра стоит, и задания MCP не выполняются («Игра не забрала задание»).

Опознавай окно по тексту через ключи локализации, а не по русскому или английскому тексту:

| Ключ | Текст (ru) | Когда |
|---|---|---|
| `str_mp_kicked_client` | «Вас изгнали из игры.» | кик; причина кика дописывается в скобках: «Вас изгнали из игры. (Причина)» |
| `STR_MP_KICKED_SLOW_NETWORK` | | кик за медленную сеть |
| `STR_MP_TIMEOUT_CLIENT` | «Соединение с сервером разорвано.» | обрыв связи (сервер упал или пропала сеть) |
| `STR_MP_SESSION_LOST` | | потеря сессии |
| `STR_MSG_CONFIRM_DISCONNECT_CLIENT` | «Вы уверены, что хотите отключиться от игры?» | ESC → «Выйти с сервера» |

```sqf
// Есть ли в окне один из текстов (ключи локализации); точку в конце не учитываем
private _fnc_hasText = {
    params ["_display", "_keys"];
    forceUnicode 0;
    private _texts = _keys apply {
        private _t = localize _x;
        if (_t select [count _t - 1] == ".") then {_t = _t select [0, count _t - 1]};
        _t
    };
    (allControls _display) findIf {
        private _text = ctrlText _x;
        _text != "" && {_texts findIf {_x != "" && {_x in _text}} >= 0}
    } >= 0
};
```

Причину кика с сервера (`#kick <uid> <причина>`) можно опознать по подстроке. Так плановый
рестарт отличается от обычного кика.

## Как клиент уходит с сервера

| Что произошло | Что видит клиент | Mission EH `Ended` |
|---|---|---|
| `endMission` / `BIS_fnc_endMission` на клиенте (скрипт, проверка версии) | дебрифинг → меню | да |
| ESC → «Выйти с сервера» → «Да» | подтверждение `RscMsgBox` → меню | — |
| Кик (`#kick`) | `RscMsgBox` «Вас изгнали из игры» | нет: миссия уже выгружена |
| Сервер упал или пропала сеть | через 1,5–2 мин `RscMsgBox` «Соединение с сервером разорвано» | нет |
| Смена миссии на сервере (`#mission`) | загрузка следующей миссии, клиент остаётся на сервере | — |

После кика игрок какое-то время (наблюдалось ~100 с) не может зайти снова. Рестарт
процесса сервера это сбрасывает.

## Пример: окно переподключения

Задача: если связь оборвалась не по воле игрока, в меню показать окно, которое ждёт
сервер и подключается само.

1. **Отметка сессии.** Функция с `postInit = 1` (только `hasInterface && isMultiplayer &&
   !isServer`) ставит `uiNamespace setVariable ["TAG_sessionActive", true]` и вешает
   mission EH `Ended`, который снимает отметку. `uiNamespace` переживает выгрузку миссии.
2. **Намеренный выход снимает отметку**: `Ended`, `RscMsgBox` с текстом кика,
   подтверждение выхода, закрытое с кодом 1.
3. **Тик главного меню**: `getClientStateNumber == 0` и отметка стоит → снять отметку,
   открыть окно переподключения.
4. **Свежие данные о сервере.** Окно опрашивает сервер (A2S_INFO через расширение) и
   учитывает только ответы, полученные после открытия окна, а не кэш. В A2S-ответе Arma
   поле keywords содержит токен `s<N>` — состояние сервера: 7 — идёт игра, 3 и 6 — выбор
   ролей и брифинг. Подключаться можно, когда сервер отвечает и состояние в
   `[3, 6, 7]`.
5. **Ограничь автопопытки** (например, 3 подряд) и сбрасывай счётчик, когда игрок дошёл до
   миссии: иначе бан после кика превратится в бесконечный цикл подключений.
6. **Сообщение о потере связи закрывай само**, если ждёшь переподключения (см. раздел
   про `RscMsgBox`).

Флаги из `uiNamespace` читай с проверкой типа: `(uiNamespace getVariable ["TAG_flag", false]) isEqualTo true`,
числа — через `param` (см. syntax-pitfalls.md).

## Расширение (DLL) для меню

- Сетевые запросы (A2S, HTTP) делай в фоновом потоке расширения, а `callExtension` пусть
  только забирает готовый результат. Иначе меню подвисает на время запроса.
- Результат отдавай в формате, который читает `parseSimpleArray`: он не создаёт код
  (`{...}`), поэтому ответ расширения нельзя превратить в выполнение.
- `callExtension [функция, [аргументы]]` возвращает `[результат, код возврата, код ошибки]`.
- Пока игра запущена, DLL заблокирована: пересобрать её можно только после выхода из игры.

## Цвета в стиле меню

- Задавай палитру макросами в общем `.hpp` (`#define TAG_COL_ACCENT [r,g,b,a]`), а в
  конфигах — отдельными макросами `{...}`. Запятые внутри `{}` препроцессор считает
  разделителями аргументов, поэтому в параметризованные макросы передавай только имена
  макросов цветов.
- В `#define` нельзя ставить `//`: препроцессор вырежет остаток строки как комментарий.
- Для structured text держи те же цвета в hex (`#define TAG_HEX_DIM "#a89e8f"`), чтобы
  текст и контролы не расходились.
