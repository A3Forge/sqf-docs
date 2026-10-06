# UI: диалоги, контролы, HTML (ctrlWebBrowser)

## Классический UI

- Диалог: `createDialog "TAG_MyDialog"` (курсор, блокирует управление);
  оверлей без курсора: `("TAG_layer" call BIS_fnc_rscLayer) cutRsc ["TAG_Hud", "PLAIN"]`.
- Ссылку на дисплей сохраняй в `onLoad`:
  `onLoad = "uiNamespace setVariable ['TAG_MyDialog', _this select 0]";`
  и бери оттуда: `uiNamespace getVariable ["TAG_MyDialog", displayNull]`.
- `findDisplay 46` — основной игровой дисплей (для KeyDown), `12` — карта
  (`displayCtrl 51` — контрол карты), `49` — меню Esc, `54` — диалог вставки маркера.
- Динамические контролы: `_display ctrlCreate ["RscText", _idc]`, затем
  `ctrlSetPosition` + `ctrlCommit 0`. Удаление — `ctrlDelete`.
- Координаты — через `safezoneX/Y/W/H` и `pixelW/H`, чтобы не зависеть от разрешения и UI scale.
- Значения display/control нельзя сериализовать: в scheduled-скриптах, которые их хранят,
  пиши `disableSerialization;`.
- Structured text: `ctrlSetStructuredText parseText "<t size='1.2'>...</t>"`; текст
  игрока экранируй (`&`, `<`, `>`). Размер под текст: `ctrlTextWidth`, `ctrlTextHeight`
  (сначала задай заведомо широкую ширину и сделай commit, иначе текст переносится по старой ширине).
- Listbox: `lbAdd`, `lbSetData`, `lbData [_idc, lbCurSel _idc]`, `lbSetValue`.

## HTML-интерфейсы (ctrlWebBrowser, CEF)

Контрол класса `RscWebBrowser` / `CT_WEBBROWSER` в диалоге.

```sqf
createDialog "TAG_Settings";
private _web = uiNamespace getVariable ["TAG_Settings_WEB", controlNull];

_web ctrlWebBrowserAction ["LoadFile", "h\settings.html"];      // файл из миссии

_web ctrlAddEventHandler ["PageLoaded", {
    // страница загрузилась заново — отправить состояние целиком
}];

// JS → SQF: A3API.SendConfirm(JSON.stringify({mode: "...", data: [...]})) в странице
_web ctrlAddEventHandler ["JSDialog", {
    params ["_control", "_isConfirmDialog", "_message"];
    private _map  = fromJSON _message;
    private _mode = _map getOrDefault ["mode", ""];
    private _data = _map getOrDefault ["data", []];
    switch (_mode) do {
        case "apply": {
            _data params [["_key", "", [""]], ["_value", 0, [0]]];   // данные из JS тоже валидируем
            [_key, _value] remoteExec ["TAG_srv_apply", 2];
        };
        case "close": { (ctrlParent _control) closeDisplay 0; };
    };
    false
}];

// SQF → JS. Конкатенация, не format: JSON может быть длинным, а format до 2.18
// молча обрезает результат до 8191 символа
_web ctrlWebBrowserAction ["ExecJS", "updateSettings(" + toJSON _patch + ")"];
```

```js
// В странице: заглушка для отладки в обычном браузере
if (typeof A3API === "undefined") {
    window.A3API = { SendConfirm: function (s) { console.log("SQF <-", s); } };
    updateSettings({ /* тестовые данные */ });
}
```

Нюансы игрового CEF:
- Картинки вставляй в HTML как base64 (`data:image/...`). `<img src="../...">` блокируется
  как cross-origin, а `A3API.RequestTexture` на обычных JPEG ненадёжен.
- Нативная прокрутка (`overflow-y: auto`) глючит на длинных списках — добавь свой обработчик колеса:
  ```js
  document.addEventListener('wheel', function (e) {
      var t = e.target.closest('.scroll-area');   // перечисли все прокручиваемые контейнеры
      if (t) { e.preventDefault(); t.scrollTop += e.deltaY; }
  }, { passive: false });
  ```
- `ExecJS` с полным состоянием каждый тик перерисовывает всё (сбивает фокус в полях ввода) —
  отправляй только изменения (см. patterns.md, раздел 5).
- Функции, которые SQF вызывает через `ExecJS`, должны быть глобальными; при минификации
  или обфускации их имена переименовывать нельзя.
- Данные из JS (через `JSDialog`) такие же недоверенные, как и любой ввод клиента; права
  проверяются на сервере, окно только отправляет запросы.
- Обновление по таймеру — через CBA PFH (если CBA есть в сборке), ванильный EachFrame
  с интервалом (scheduler.md) или цикл со `sleep`; в любом случае он должен останавливаться,
  когда контрол/дисплей стал null.
