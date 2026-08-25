# Все уроки курса

> Перевод-адаптация курса [Build Your Own AI Coding Agent Harness](https://vercel.com/academy/build-ai-agent-harness) (Vercel Academy, Joel Hooks)

---

## Модуль 1: Агент-лупа (The Agent Loop)

1. [От чата к агенту](01-agent-loop/1a-from-chat-to-agent.md) — один инструмент `read` превращает чатбота в агента
2. [Твои первые инструменты](01-agent-loop/1b-your-first-tools.md) — добавляем `grep`, проблема выбора, bash gravity
3. [Завершаем набор инструментов](01-agent-loop/1c-completing-the-toolbox.md) — добавляем `bash` с allowlist безопасности

## Модуль 2: Дизайн инструментов (Tool Design)

4. [Описания, которые работают](02-tool-design/2a-descriptions-that-work.md) — 5-секционный контракт описания
5. [Выполнение shell с безопасностью](02-tool-design/2b-shell-execution-with-safety.md) — паттерн «фабрика» с внедрением бэкенда
6. [Шлюзы одобрения](02-tool-design/2c-approval-gates.md) — эволюция: boolean → функция → discriminated union

## Модуль 3: Системный промпт (The System Prompt)

7. [Структурирование инструкций](03-system-prompt/3a-structuring-instructions.md) — секции Agency и Guardrails
8. [Динамическое построение промпта](03-system-prompt/3b-dynamic-prompt.md) — `buildSystemPrompt()` как чистая функция
9. [Верификационные шлюзы](03-system-prompt/3c-verification-gates.md) — честность, а не покрытие
10. [Контекст проекта / AGENTS.md](03-system-prompt/3d-project-context.md) — конвенция вместо конфигурации

## Модуль 4: Абстракция песочницы (The Sandbox Abstraction)

11. [Проектирование интерфейса](04-sandbox/4a-designing-interface.md) — минимальный async `Sandbox`
12. [Локальная реализация](04-sandbox/4b-local-implementation.md) — `createLocalSandbox()`, ~15 строк
13. [In-memory реализация](04-sandbox/4c-in-memory.md) — just-bash, copy-on-write
14. [Облачная реализация](04-sandbox/4d-cloud.md) — концептуальный урок, таблица компромиссов
15. [Хуки жизненного цикла](04-sandbox/4e-lifecycle-hooks.md) — `afterStart`, `beforeStop`, `onTimeout`

## Модуль 5: Управление контекстом (Context Management)

16. [Проблема: контекст](05-context/5a-the-problem.md) — телеметрия токенов, линейный рост
17. [Обрезка старых результатов](05-context/5b-pruning.md) — `pruneMessages`, 4 строки кода
18. [Дизайн вывода инструментов](05-context/5c-tool-output-design.md) — bounded output, хранение хвоста
19. [Управление кешем](05-context/5d-cache-control.md) — `addCacheControl()`, ~$30 → ~$6/сессия

## Модуль 6: Делегирование субагентам (Subagent Delegation)

20. [Почему делегировать](06-subagents/6a-why-delegate.md) — три режима отказа одиночного агента
21. [Explorer-подагент](06-subagents/6b-explorer.md) — только чтение, Haiku, 5 шагов
22. [Executor-подагент](06-subagents/6c-executor.md) — полные инструменты, Sonnet, 15 шагов
23. [Инструмент Task](06-subagents/6d-task-tool.md) — маршрутизация, модель на роль

## Модуль 7: Жизненный цикл песочницы (Sandbox Lifecycle)

24. [Конечный автомат](07-lifecycle/7a-state-machine.md) — 4 состояния, два таймера
25. [Снимок и восстановление](07-lifecycle/7b-snapshot-restore.md) — snapshot/restore, идемпотентность
26. [Долговечные рабочие процессы](07-lifecycle/7c-durable-workflows.md) — Vercel Workflow, математика стоимости
27. [Добытые трудом уроки](07-lifecycle/7d-hard-won-lessons.md) — 5 продакшн-проблем

## Модуль 8: Human-in-the-Loop

28. [Структурированные вопросы](08-human-in-loop/8a-structured-questions.md) — `askUser`, протокол «ищи→спрашивай→действуй»
29. [Конфигурация одобрения](08-human-in-loop/8b-approval-config.md) — два уровня: config + events

## Модуль 9: Планирование и верификация (Planning and Verification)

30. [Инструмент Todo](09-planning/9a-todo-tool.md) — один `in_progress` одновременно
31. [Быстрое понимание контекста](09-planning/9b-fast-context.md) — grep-first стратегия
32. [Контракт верификации](09-planning/9c-verification-contract.md) — обнаружение ворот из `package.json`

## Модуль 10: Поверхности (Surfaces)

33. [Точка входа CLI](10-surfaces/10a-cli-entry.md) — `parseArgs`, `--sandbox`, SIGINT
34. [Стриминг и рендеринг](10-surfaces/10b-streaming.md) — `agent.stream()`, рендеринг по инструментам
35. [Веб-поверхность](10-surfaces/10c-web-surface.md) — агент headless, SSE, персистентность

## Модуль 11: Расширяемость (Extensibility)

36. [Система навыков](11-extensibility/11a-skills.md) — progressive disclosure, SKILL.md
37. [Пользовательские инструменты](11-extensibility/11b-custom-tools.md) — реестр, wrapTool
38. [Точки расширения](11-extensibility/11c-extension-points.md) — event bus, 5 событий, правило цепочки

---

[← Назад к обзору](../README.md)
