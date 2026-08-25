# Урок 6d: Task tool — слой маршрутизации

Родитель не должен помнить, какие субагенты существуют и как их строить. Ему нужен **один** инструмент, через который он делегирует работу, а выбор конкретного типа — за каркасом. Этот единый инструмент — `task`, и он же — место, где инкапсулируется вся логика маршрутизации.

---

## Проблема

Если у родителя будет два отдельных инструмента — `explore` и `execute` — родительская логика расползается: модель должна сама помнить, чем они отличаются, когда какой вызывать, и писать отдельные описания для каждого. Это дублирование и путаница.

Хуже того: сборка субагентов (выбор модели, навешивание trust-листа, лимит шагов) — это **код каркаса**, а не забота родителя. Если эта логика размазана по вызовам, её нельзя ни протестировать, ни переиспользовать.

---

## Решение с кодом

Один инструмент `task` с полем `subagentType`, которое выбирает ветку. Родитель думает категориями «исследовать» / «выполнить», а каркас решает, **как** это устроено.

```typescript
import { tool } from "ai";
import { z } from "zod";

function createTaskTool(sandbox, parentTools) {
  return tool({
    description: `Delegate work to a subagent.
      Explorer (default): read-only research with a fast model.
      Executor: implementation with a stronger model and delegated trust on bash.
      WHEN TO USE: research across many files (explorer),
        bulk implementation (executor).
      WHEN NOT TO USE: ambiguous requirements (use askUser),
        architectural decisions (the parent decides).`,
    inputSchema: z.object({
      description: z.string().describe("What the subagent should do"),
      subagentType: z.enum(["explorer", "executor"]).default("explorer"),
    }),
    execute: async ({ description, subagentType }) => {
      const agent = subagentType === "executor"
        ? buildExecutor(sandbox, parentTools)
        : buildExplorer(sandbox, parentTools);
      return runSubagent(subagentType, agent, description);
    },
  });
}
```

> **📝 Перевод промта (описание инструмента task):**
>
> *«Делегируй работу подагенту.*
> *Explorer (по умолчанию): исследование только на чтение с быстрой моделью.*
> *Executor: реализация с более сильной моделью и делегированным доверием на bash.*
>
> *КОГДА ИСПОЛЬЗОВАТЬ: исследование множества файлов (explorer), массовая реализация (executor).*
>
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: неоднозначные требования (используй askUser), архитектурные решения (родитель решает сам).»*

### Рефакторинг: buildExplorer() / buildExecutor() / runSubagent()

Три функции с чёткими обязанностями. Каждая делает одну вещь:

```typescript
// 1. Сборка Explorer-а — инкапсулирует ВСЕ решения о «как исследовать»
function buildExplorer(sandbox, parentTools): ToolLoopAgent {
  return new ToolLoopAgent({
    model: "anthropic/claude-haiku-4-5",       // модель на роль
    instructions: `You are an explorer agent. Investigate and report back concisely.
    Working directory: ${sandbox.workingDirectory}`,
    tools: { read: parentTools.read, grep: parentTools.grep },
    stopWhen: stepCountIs(5),
  });
}

// 2. Сборка Executor-а — инкапсулирует решения о «как исполнять»
function buildExecutor(sandbox, parentTools): ToolLoopAgent {
  const executorBash = createBashTool(sandbox, createApproval({
    mode: "delegated",
    trust: ["npm test", "npm run build", "npx tsc"],
  }));
  return new ToolLoopAgent({
    model: "anthropic/claude-sonnet-4-6",      // модель на роль
    instructions: `You are an executor agent. Follow instructions precisely.
    Working directory: ${sandbox.workingDirectory}
    Do NOT ask questions. Do NOT explore beyond what's needed. Execute the task.`,
    tools: { read: parentTools.read, grep: parentTools.grep, bash: executorBash },
    stopWhen: stepCountIs(15),
  });
}

// 3. Запуск — единый путь для любого субагента
async function runSubagent(type: string, agent: ToolLoopAgent, description: string) {
  try {
    const result = await agent.generate({ prompt: description });
    return result.text;
  } catch (error) {
    return `${type} subagent failed: ${error instanceof Error ? error.message : String(error)}`;
  }
}
```

### Модель на роль

Заметьте, что модель выбирается **внутри** `buildExplorer` / `buildExecutor`, а не в `createTaskTool`. Это и есть «модель на роль»:

- **Explorer → Haiku:** исследование — это поиск и чтение, дёшево и быстро.
- **Executor → Sonnet:** написание кода — это рассуждение, нужна сильная модель.

Родитель (или пользователь) **не выбирает** модель — он выбирает **роль**. Каркас сам знает, какая модель соответствует каждой роли. Захочешь обновить модель — поменяешь одну строку в одном месте, а не во всех вызовах.

### Почему две роли — правильная отправная точка

Легко увлечься и наплодить десяток ролей: `reviewer`, `debugger`, `refactorer`, `doc-writer`… Сопротивляйся. Две роли — Explorer (только чтение) и Executor (чтение + ограниченная запись) — покрывают **почти всё**, потому что ловят главное различие: *читает агент или пишет*.

| Роль | Читает | Пишет | Запускает команды |
|---|---|---|---|
| Explorer | ✅ | ❌ | ❌ |
| Executor | ✅ | ✅ | ✅ (trust-лист) |

Всё остальное — это вариации на тему этих двух. `debugger` — это Executor с упором на `npm test`. `reviewer` — это Explorer с упором на чтение diff-ов. Начни с двух, добавляй третью роль **только когда** реально упрёшься в ограничение, которое две не покрывают.

### Вся картина

```
Родитель вызывает task({ description, subagentType })
        │
        ▼
createTaskTool.execute
        │  subagentType === "executor" ?
        ├── да ──► buildExecutor(sandbox, parentTools)  → Sonnet, 15 шагов, trust-лист
        │
        └── нет ─► buildExplorer(sandbox, parentTools)  → Haiku, 5 шагов, только чтение
                        │
                        ▼
                runSubagent(type, agent, description)  → текст отчёта или строка-ошибка
```

---

## Ключевые выводы

- **`task` — единая точка делегирования.** Родитель видит один инструмент, а не коллекцию субагентов.
- **`subagentType: explorer | executor`** — enum с дефолтом `explorer`: если модель не указала тип, получает безопасный read-only вариант.
- **Рефакторинг на три функции:** `buildExplorer()` / `buildExecutor()` / `runSubagent()`. Сборка отделена от запуска, запуск общий для всех.
- **Модель на роль:** выбор модели живёт внутри билдеров, а не в вызывающем коде. Пользователь выбирает роль, каркас — модель.
- **Две роли — правильная отправная точка.** Различие «читает vs пишет» покрывает почти всё; новые роли добавляй только по реальной необходимости.

---

## Попробуй сам

1. Вызови `task` без указания `subagentType` и убедись, что срабатывает дефолт `explorer` (read-only).
2. Вынеси выбор модели в константу-словарь `ROLE_MODELS = { explorer: "...", executor: "..." }` и перепиши билдеры на неё. Убедись, что смена модели — это правка одной строки.
3. Добавь третий тип `reviewer` (только чтение + Sonnet + акцент на анализ diff-ов) через новую ветку. Заметь, сколько кода потребовалось — это цена третьей роли.
4. Проверь обработку ошибок: передай несуществующий тип (если ослабить `z.enum` до строки) и увидишь, как `runSubagent` вернёт строку-ошибку вместо падения.
5. Напиши юнит-тест на `createTaskTool.execute`: передай `subagentType: "executor"` и проверь, что вернулся текст (а не исключение) и что внутри был использован trust-лист.
