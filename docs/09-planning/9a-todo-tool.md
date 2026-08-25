# Урок 9a: Todo tool — план в контексте агента

Агент, которому поручили задачу на несколько шагов, держит план «в голове». Но «в голове» — значит в неструктурированном контексте, который размывается с каждым вызовом инструмента. Todo tool даёт агенту **явный, структурированный** список задач, который он сам ведёт и по которому можно отслеживать прогресс.

---

## Проблема

Задача «реализуй авторизацию» — это на самом деле 5–7 шагов: изучить текущий код, добавить модель, написать middleware, обновить тесты, прогнать проверки. Без явного списка агент:

- **Забывает шаги** по мере роста контекста (тот же режим отказа №2 из урока 6a).
- **Не может показать прогресс** — ни пользователю, ни себе.
- **Прыгает между подзадачами** хаотично, вместо того чтобы доводить одну до конца.

Для задач из 1–2 шагов список не нужен. Но для задач из **3+ шагов** явный план — это разница между «агент довёл до конца» и «агент потерялся на середине».

---

## Решение с кодом

Инструмент `todo` с четырьмя операциями: `add`, `start`, `complete`, `list`.

```typescript
import { tool } from "ai";
import { z } from "zod";
import { randomUUID } from "node:crypto";

type TodoStatus = "pending" | "in_progress" | "completed";

interface TodoItem {
  id: string;
  content: string;
  status: TodoStatus;
}

// Состояние живёт в замыкании — один список на сессию
const todos: TodoItem[] = [];

const todo = tool({
  description: `Manage a task list for the current session.
WHEN TO USE: tasks with 3 or more steps. Break the work into items,
  mark one in_progress, complete items as you finish them.
WHEN NOT TO USE: trivial single-step tasks. Do not use for notes or commentary.`,
  inputSchema: z.object({
    action: z.enum(["add", "start", "complete", "list"]),
    content: z.string().optional().describe("Task text (for add)"),
    id: z.string().optional().describe("Task id (for start/complete)"),
  }),
  execute: async ({ action, content, id }) => {
    switch (action) {
      case "add":
        return addTodo(content ?? "");
      case "start":
        return startTodo(id ?? "");
      case "complete":
        return completeTodo(id ?? "");
      case "list":
        return listTodos();
    }
  },
});
```

> **📝 Перевод промта (описание todo):**
>
> *«Управляй списком задач текущей сессии.*
> *КОГДА ИСПОЛЬЗОВАТЬ: задачи из 3 и более шагов. Разбей работу на пункты, отметь один как in_progress, завершай пункты по мере выполнения.*
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: тривиальные одношаговые задачи. Не используй для заметок или комментариев.»*

### Операции

```typescript
function addTodo(content: string): string {
  // ID: первые 8 символов UUID — короткий, уникальный, читаемый
  const id = randomUUID().slice(0, 8);
  todos.push({ id, content, status: "pending" });
  return `Added todo ${id}: ${content}`;
}

function startTodo(id: string): string {
  const item = todos.find((t) => t.id === id);
  if (!item) return `Todo ${id} not found.`;

  // ПРАВИЛО: только один in_progress одновременно.
  // Прежде чем начать новый, завершаем или останавливаем текущий.
  for (const t of todos) {
    if (t.status === "in_progress") {
      t.status = "pending";
    }
  }
  item.status = "in_progress";
  return `Started todo ${id}: ${item.content}`;
}

function completeTodo(id: string): string {
  const item = todos.find((t) => t.id === id);
  if (!item) return `Todo ${id} not found.`;
  item.status = "completed";
  return `Completed todo ${id}: ${item.content}`;
}

function listTodos(): string {
  if (todos.length === 0) return "(no todos)";
  const marks = { pending: "[ ]", in_progress: "[>]", completed: "[x]" } as const;
  return todos
    .map((t) => `${marks[t.status]} ${t.id} ${t.content}`)
    .join("\n");
}
```

### Почему `crypto.randomUUID().slice(0, 8)`

Полный UUID — это `9f8c2e1a-4b3d-...` — 36 символов, которые модель должна копировать в последующие вызовы без ошибок. Восемь первых символов (`9f8c2e1a`) дают **достаточно** уникальности для списка из десятков задач и **легко** читаются/копируются моделью. Короткий ID — это меньше шансов на опечатку при обращении к задаче.

### Правило: только один in_progress

Самое важное правило — **один `in_progress` одновременно**. Оно заставляет агента доводить одну подзадачу до конца, прежде чем браться за следующую, и убивает хаотичное перепрыгивание:

```
[x] 1a2b3c4d Изучить текущий код авторизации
[>] 5e6f7a8b Добавить модель User              ← ровно один in_progress
[ ] 9b0c1d2e Написать auth middleware
[ ] 3e4f5a6b Обновить тесты
```

---

## Ключевые выводы

- **Todo tool = `add` / `start` / `complete` / `list`.** Четыре операции, одна структура.
- **Для задач 3+ шагов.** Одношаговые задачи не требуют списка — не переусложняй.
- **Правило «только один in_progress»** заставляет агента доводить подзадачу до конца, а не прыгать хаотично.
- **`randomUUID().slice(0, 8)`** — короткий, уникальный, легко копируемый ID; меньше шансов на опечатку модели.
- **Состояние в замыкании** — список живёт в течение сессии, и агент может к нему обращаться на любом шаге.

---

## Попробуй сам

1. Дай агенту задачу из 5 шагов и попроси сначала составить todo-список через `add`. Проверь, что он разбил задачу на пункты.
2. Проследи, что при `start` нового пункта предыдущий `in_progress` автоматически возвращается в `pending` (правило одного активного).
3. Проверь `list` на каждом шаге — виден ли прогресс в формате `[ ]` / `[>]` / `[x]`.
4. Попробуй ID на коллизии: сгенерируй 10 000 ID через `randomUUID().slice(0, 8)` и убедись, что коллизий практически нет (для списка задач этого хватает).
5. Добавь операцию `remove` (удалить пункт) и подумай, нужна ли она агенту — или трёх операций достаточно.
