# Строим AI-кодинг агент

## Каркас (harness) для AI-агента с нуля

Перевод-адаптация курса [Build Your Own AI Coding Agent Harness](https://vercel.com/academy/build-ai-agent-harness) от Vercel Academy (автор — Joel Hooks).

> **⚠️ Примечание о правах:** Перевод выполнен в образовательных целях. Оригинальный контент принадлежит [Vercel](https://vercel.com). Если вы являетесь правообладателем и возражаете против перевода — свяжитесь с нами через [Issues](https://github.com/galaersh-ai/ai-agent-harness-guide/issues).

---

### О чём этот курс

Три инструмента в цикле — это демо. Проблемы начинаются, когда пытаешься использовать агента для настоящей работы:

- Прочитал файл на 5000 строк — он **навсегда** остаётся в контексте
- Даёшь `bash` — он выполняет `rm -rf`
- Просишь рефакторинг — агент **объясняет**, как рефакторить, но не делает
- Одна длинная задача заполняет контекст — агент забывает инструкции

**Harness** (обвязка, каркас) — это система вокруг агента, которая решает все эти проблемы. Курс строит `TeensyCode` — рабочий каркас с нуля на TypeScript + AI SDK v6.

---

### Что вы построите

- **Агент-лупа** — `ToolLoopAgent` с инструментами `read`, `grep`, `bash`, `write`, `edit`, `task`, `askUser`
- **Безопасность** — allowlist команд, configurable approval (interactive / background / delegated)
- **Системный промпт** — структурированные инструкции с секциями Agency, Guardrails, Verification
- **Абстракция песочницы** — один интерфейс `Sandbox`, три реализации (local / in-memory / cloud)
- **Управление контекстом** — обрезка старых результатов, ограничение вывода, cache control
- **Субагенты** — Explorer (только чтение) и Executor (полные инструменты), изолированный контекст
- **Жизненный цикл** — state machine, snapshot/restore, durable workflows
- **Human-in-the-loop** — `askUser` с multiple choice, протокол «ищи → спрашивай → действуй»
- **Планирование** — todo tool, grep-first стратегия, контракт верификации
- **Поверхности** — CLI, стриминг, веб — один агент, разные рендереры
- **Расширяемость** — event bus, skills с progressive disclosure, custom tool registration

---

### Стек

| Компонент | Назначение |
|-----------|-----------|
| **AI SDK** (Vercel) | `ToolLoopAgent`, `tool()`, `stepCountIs`, `pruneMessages`, streaming |
| **AI Gateway** | Роутинг моделей строкой `"anthropic/claude-haiku-4-5"` |
| **Vercel Sandbox** | Удалённая ВМ с изолированной ФС, git, npm |
| **just-bash** | In-memory виртуальная ФС с copy-on-write |
| **Vercel Workflow** | Долговечные процессы для жизненного цикла |
| **Zod v3** | Схемы входа инструментов |
| **TypeScript** | Язык, async/await |

---

### Предварительные требования

- TypeScript, async/await, базовый опыт терминала
- Переменная окружения `AI_GATEWAY_API_KEY`
- Node.js 20+ или Bun

---

### Как устроен курс

**Причинная последовательность.** Каждый шаг существует потому, что предыдущий что-то сломал. Шаг 1 добавляет `read`, потому что чатбот не видит файлы. Шаг 2 добавляет `grep`, потому что агент не может искать. Шаг 3 добавляет `bash`, потому что не может запускать команды — но теперь может `rm -rf`. Каждый шаг выделяет одну концепцию, пока остальное остаётся рабочим.

**Модули 1–6** — сборка: пишешь код, запускаешь, проверяешь. **Модуль 7** — концепции и анализ. **Модули 8–11** — смешанный формат.

---

### Модули курса

#### [Модуль 1: Агент-лупа](01-agent-loop/1a-from-chat-to-agent.md)

Построить `ToolLoopAgent` от нуля инструментов (чатбот) к `read` и `grep` (агент) к `bash` с шлюзами безопасности.

- [От чата к агенту](01-agent-loop/1a-from-chat-to-agent.md) — один инструмент превращает чатбота в агента
- [Твои первые инструменты](01-agent-loop/1b-your-first-tools.md) — описания инструментов как API выбора для модели
- [Завершаем набор инструментов](01-agent-loop/1c-completing-the-toolbox.md) — опасные инструменты получают шлюзы

#### [Модуль 2: Дизайн инструментов](02-tool-design/2a-descriptions-that-work.md)

Эволюция описаний в 5-секционный контракт, паттерн «фабрика», configurable approval.

- [Описания, которые работают](02-tool-design/2a-descriptions-that-work.md) — WHEN TO USE, WHEN NOT TO USE, DO NOT USE FOR, USAGE, EXAMPLES
- [Shell с безопасностью](02-tool-design/2b-shell-execution-with-safety.md) — фабрика + operations отделяют контракт от выполнения
- [Шлюзы одобрения](02-tool-design/2c-approval-gates.md) — boolean → функция → discriminated union

#### [Модуль 3: Системный промпт](03-system-prompt/3a-structuring-instructions.md)

Формирование поведения через структурированные инструкции, динамическая композиция, верификация, AGENTS.md.

- [Структурирование инструкций](03-system-prompt/3a-structuring-instructions.md) — Agency + Guardrails, «действуй, не объясняй»
- [Динамический промпт](03-system-prompt/3b-dynamic-prompt.md) — `buildSystemPrompt()` адаптируется к runtime-контексту
- [Верификационные шлюзы](03-system-prompt/3c-verification-gates.md) — typecheck, lint, test, build контракт
- [Контекст проекта](03-system-prompt/3d-project-context.md) — положи `AGENTS.md`, измени агента

#### [Модуль 4: Абстракция песочницы](04-sandbox/4a-designing-interface.md)

Один интерфейс `Sandbox`, три реализации. Инструменты вызывают `sandbox.exec()`, а не `child_process.exec()`.

- [Проектирование интерфейса](04-sandbox/4a-designing-interface.md) — тип `Sandbox` с `readFile`, `exec`, `stop`
- [Локальная реализация](04-sandbox/4b-local-implementation.md) — Node fs + child_process обёртка
- [In-memory реализация](04-sandbox/4c-in-memory.md) — just-bash с copy-on-write
- [Облачная реализация](04-sandbox/4d-cloud.md) — удалённая ВМ, компромиссы
- [Хуки жизненного цикла](04-sandbox/4e-lifecycle-hooks.md) — `afterStart`, `beforeStop`, `onTimeout`

#### [Модуль 5: Управление контекстом](05-context/5a-the-problem.md)

Каждый вызов инструмента остаётся в контексте навсегда. Исправляем обрезкой, ограничением вывода и cache control.

- [Проблема](05-context/5a-the-problem.md) — телеметрия токенов показывает линейный рост
- [Обрезка старых результатов](05-context/5b-pruning.md) — `pruneMessages` не даёт накапливаться
- [Дизайн вывода инструментов](05-context/5c-tool-output-design.md) — профилактика лучше уборки
- [Управление кешем](05-context/5d-cache-control.md) — заголовки провайдера снижают стоимость

#### [Модуль 6: Субагенты](06-subagents/6a-why-delegate.md)

Родитель планирует, субагенты выполняют. Изолированный контекст, ограниченные инструменты, модель на роль.

- [Почему делегировать](06-subagents/6a-why-delegate.md) — режимы отказа одиночного агента
- [Explorer-подагент](06-subagents/6b-explorer.md) — только чтение, дешёвая модель, ограниченное исследование
- [Executor-подагент](06-subagents/6c-executor.md) — полные инструменты, сильная модель, делегированное доверие
- [Инструмент Task](06-subagents/6d-task-tool.md) — маршрутизация, разрешения, модель на роль

#### [Модуль 7: Жизненный цикл песочницы](07-lifecycle/7a-state-machine.md)

Облачные песочницы стоят денег и таймаутятся. Концептуальный модуль.

- [Конечный автомат](07-lifecycle/7a-state-machine.md) — переходы состояний, таймауты, трекинг активности
- [Снимок и восстановление](07-lifecycle/7b-snapshot-restore.md) — freeze, restore, идемпотентность
- [Долговечные процессы](07-lifecycle/7c-durable-workflows.md) — Vercel Workflow с `sleep()`
- [Добытые трудом уроки](07-lifecycle/7d-hard-won-lessons.md) — продакшн-готчи

#### [Модуль 8: Human-in-the-Loop](08-human-in-loop/8a-structured-questions.md)

Агенты, которые угадывают неправильно, тратят больше времени, чем агенты, которые спрашивают.

- [Структурированные вопросы](08-human-in-loop/8a-structured-questions.md) — `askUser` с multiple choice и протокол неоднозначности
- [Конфигурация одобрения](08-human-in-loop/8b-approval-config.md) — конфиг для режимов, события для политик

#### [Модуль 9: Планирование и верификация](09-planning/9a-todo-tool.md)

Планируй перед действием, проверяй после.

- [Инструмент Todo](09-planning/9a-todo-tool.md) — декомпозиция задач с трекингом состояния
- [Быстрое понимание контекста](09-planning/9b-fast-context.md) — сначала grep, потом читай только то, что будешь менять
- [Контракт верификации](09-planning/9c-verification-contract.md) — последовательность шлюзов с ограниченными заявлениями

#### [Модуль 10: Поверхности](10-surfaces/10a-cli-entry.md)

Агент headless. CLI, TUI и веб — стратегии рендеринга.

- [Точка входа CLI](10-surfaces/10a-cli-entry.md) — аргументы, фабрика песочницы, чистое завершение
- [Стриминг и рендеринг](10-surfaces/10b-streaming.md) — текст в реальном времени + отображение вызовов инструментов
- [Веб-поверхность](10-surfaces/10c-web-surface.md) — тот же агент, другой рендерер

#### [Модуль 11: Расширяемость](11-extensibility/11a-skills.md)

События, а не наследование. Навыки как progressive disclosure. Инструменты как регистрации.

- [Система навыков](11-extensibility/11a-skills.md) — имена в промпте, полное содержимое по запросу
- [Пользовательские инструменты](11-extensibility/11b-custom-tools.md) — регистрация без форка, композиция существующих
- [Точки расширения](11-extensibility/11c-extension-points.md) — события жизненного цикла для подписки, блокировки, модификации

---

### Ссылки

- [Оригинальный курс Vercel Academy](https://vercel.com/academy/build-ai-agent-harness)
- [AI SDK документация](https://sdk.vercel.ai/docs)
- [GitHub репозиторий](https://github.com/galaersh-ai/ai-agent-harness-guide)

---

*Перевод и адаптация: [galaersh-ai](https://github.com/galaersh-ai). Оригинальный контент — Vercel Academy, Joel Hooks.*
