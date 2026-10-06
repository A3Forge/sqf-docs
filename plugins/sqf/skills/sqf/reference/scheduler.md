# Scheduled и unscheduled окружение

## Два окружения

| | Scheduled | Unscheduled |
|---|---|---|
| Кто запускает | `spawn`, `execVM`, `init.sqf`, `initServer.sqf`, `initPlayerLocal.sqf`, функции `postInit`, `call` из scheduled-кода, функции через `remoteExec` | Event handlers, функции `preInit`, init-поля объектов, UI EH, `call` из unscheduled-кода, `isNil {код}`, цели `remoteExecCall` и команды через `remoteExec`, условия/активация триггеров |
| `sleep` / `uiSleep` / `waitUntil` | можно | **ошибка**: "Suspending not allowed in this context" |
| Выполнение | планировщик даёт всем скриптам ~3 мс за кадр, затем приостанавливает их до следующего кадра | выполняется до конца, прежде чем кадр продолжится |
| Атомарность | может быть прерван между **любыми** двумя инструкциями | атомарно |
| Лимит цикла `while` | нет | 10 000 итераций, потом молча выходит |

- Проверка во время выполнения — `canSuspend`.
- `call` наследует окружение вызывающего; `spawn` всегда создаёт новый scheduled-скрипт.

## Следствия

- Под нагрузкой (много spawn-скриптов) scheduled-код может задерживаться на секунды.
  Не клади в scheduled-циклы логику, критичную ко времени или к согласованности состояния.
- Гонки: в scheduled-коде другой скрипт может изменить глобальную переменную между твоей
  проверкой и записью. Критические секции оборачивай в `isNil { ... }` — они выполнятся unscheduled:
  ```sqf
  isNil {
      private _n = TAG_counter + 1;
      TAG_counter = _n;
  };
  ```
- Event handlers должны быть короткими и не приостанавливаться. Если EH нужно ждать — делай `spawn` из него.
- Тяжёлая работа в unscheduled (EH, EachFrame) роняет FPS; разбивай её на несколько
  кадров или переноси в scheduled-цикл со `sleep`.

## Циклы и тайминги

```sqf
// Периодический серверный цикл
[] spawn {
    while {true} do {
        // работа
        sleep 5;
    };
};

// Каждый кадр (unscheduled, держи минимальным)
TAG_eh = addMissionEventHandler ["EachFrame", { ... }];
removeMissionEventHandler ["EachFrame", TAG_eh];

// Задержка без отдельного цикла — ТОЛЬКО если CBA есть в сборке (см. SKILL.md, правило 18)
[{ ... }, _args, 2] call CBA_fnc_waitAndExecute;
[{ условие }, { ... }, _args] call CBA_fnc_waitUntilAndExecute;
```

Ванильные замены функций CBA:

| CBA | Ваниль |
|---|---|
| `CBA_fnc_waitAndExecute` | `_args spawn { sleep 2; ... };` (scheduled — может опоздать под нагрузкой) |
| `CBA_fnc_waitUntilAndExecute` | `_args spawn { waitUntil {sleep 0.1; условие}; ... };` |
| `CBA_fnc_addPerFrameHandler` (с интервалом) | `[] spawn { while {условие} do { ...; sleep _interval; }; };` или EachFrame с проверкой времени (ниже) |
| `CBA_fnc_removePerFrameHandler` | `removeMissionEventHandler [_thisEvent, _thisEventHandler]` внутри EH |
| `CBA_missionTime` | `serverTime` (MP) / `time` |
| `CBA_fnc_addEventHandler` / `CBA_fnc_*Event` | свой `remoteExec` функций из CfgFunctions |

```sqf
// Покадровый обработчик с интервалом без CBA (unscheduled, точный)
TAG_nextTick = 0;
addMissionEventHandler ["EachFrame", {
    if (isNull (uiNamespace getVariable ["TAG_dialog", displayNull])) exitWith {
        removeMissionEventHandler [_thisEvent, _thisEventHandler];   // окно закрыто — снять
    };
    if (diag_tickTime < TAG_nextTick) exitWith {};
    TAG_nextTick = diag_tickTime + 1;
    [] call TAG_fnc_update;
}];
```

- `waitUntil {...}` проверяет условие каждый кадр; если не срочно — добавь `sleep`:
  `waitUntil {sleep 1; _cond}`.
- `sleep` использует игровое время (зависит от `accTime`, стоит на паузе в SP);
  `uiSleep` — реальное время.
- Сохраняй handle от `spawn`, проверяй через `scriptDone`, останавливай через `terminate`.
- Один управляющий цикл лучше, чем отдельный цикл на каждый юнит/объект.
- Не используй `onEachFrame` (он один на миссию и перезаписывается) — бери
  `addMissionEventHandler ["EachFrame", ...]` или per-frame handlers CBA (если CBA есть в сборке).

## Ошибки скриптов

- Запускай игру с `-showScriptErrors`, чтобы видеть ошибки на экране.
- RPT-лог: `%LOCALAPPDATA%\Arma 3\*.rpt` (клиент), папка профиля для выделенного сервера.
- Для трассировки — `diag_log`; для замеров — `diag_codePerformance`.
- Ошибка в scheduled-скрипте обычно убивает только этот скрипт; в unscheduled пропускается
  остаток блока. Ошибки — не исключения, `try/catch` их не поймает.
