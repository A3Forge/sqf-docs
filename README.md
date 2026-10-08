# sqf-docs

База знаний по SQF (Arma 3) для ИИ-агентов в виде плагина Claude Code (Agent Skill).
Когда плагин подключён, Claude при написании и ревью кода Arma 3 автоматически
учитывает нюансы SQF: приоритет операторов, scheduled/unscheduled окружение,
локальность в мультиплеере, безопасность `remoteExec`, производительность,
CfgFunctions, синтаксис конфигов, HTML-интерфейсы и проверенные паттерны.

## Подключение

В Claude Code:

```
/plugin marketplace add A3Forge/sqf-docs
/plugin install sqf@sqf-docs
```

Или для конкретного проекта в `.claude/settings.json` (подхватится у всех, кто работает
с репозиторием):

```json
{
  "extraKnownMarketplaces": {
    "sqf-docs": { "source": { "source": "github", "repo": "A3Forge/sqf-docs" } }
  },
  "enabledPlugins": { "sqf@sqf-docs": true }
}
```

Скилл загружается сам, когда вы работаете с `.sqf`, `.hpp`, `.ext`, `.cpp` или
спрашиваете про SQF. Вызвать явно: `/sqf:sqf`.

Обновить: `/plugin marketplace update sqf-docs`.

## Структура

```
.claude-plugin/marketplace.json          манифест маркетплейса
plugins/sqf/.claude-plugin/plugin.json   манифест плагина
plugins/sqf/skills/sqf/SKILL.md          основные правила (загружаются вместе со скиллом)
plugins/sqf/skills/sqf/reference/        подробности по темам, читаются по необходимости
    syntax-pitfalls.md                   синтаксис, приоритеты, типы, массивы, hashmap, строки
    scheduler.md                         scheduled / unscheduled, циклы, тайминги, ошибки
    multiplayer.md                       локальность, remoteExec, JIP, init order, безопасность
    performance.md                       производительность
    project-structure.md                 CfgFunctions, препроцессор, конфиги, локализация
    positions-and-time.md                форматы позиций, векторы, время, случайность
    ui.md                                диалоги, контролы, ctrlWebBrowser / A3API
    patterns.md                          проверенные паттерны с боевого MP-сервера
    extdb3.md                            extDB3 / MySQL: SQL_CUSTOM, экранирование, LONGTEXT
    testing.md                           тесты в превью Eden: перезапуск, экран смерти, дебрифинг
    game-navigation.md                   управление игрой из SQF: меню, редактор, миссии, выход; MCP (sqf-mcp)
```

## Как дополнять

- `SKILL.md` держать коротким: только правила, нужные почти в каждой задаче по SQF.
- Подробности — в `reference/*.md`, со ссылкой из таблицы в `SKILL.md`.
- Факты сверять с [BIKI](https://community.bistudio.com/wiki/)
  (или с [acemod/arma3-wiki](https://github.com/acemod/arma3-wiki/tree/dist)).
- При изменениях повышать `version` в `plugins/sqf/.claude-plugin/plugin.json`,
  чтобы установленные копии обновились.
