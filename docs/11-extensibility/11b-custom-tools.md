# Урок 11b: Custom tools — реестр инструментов

По мере роста каркаса инструменты множатся: `read`, `grep`, `bash`, `task`, `todo`, `askUser`, `loadSkill`… Нужен способ их **регистрировать, искать и оборачивать** единообразно. Реестр инструментов (Tool Registry) — это `Map`, вокруг которого строится вся расширяемость: свои инструменты добавляются через тот же механизм, что и встроенные.

---

## Проблема

Без реестра инструменты разбросаны по коду: где-то передаются объектом в конструктор агента, где-то хранятся в отдельной переменной, где-то захардкожены в промпт. Нет единого места, чтобы:

- **Узнать, какие инструменты существуют** (для промпта, для UI, для дебага).
- **Добавить свой инструмент** так же, как встроенный.
- **Обернуть инструмент** хуками (логирование, метрики, блокировка) без правки его кода.

---

## Решение с кодом

### Tool Registry: Map<string, Tool>

```typescript
type Tool = {
  description: string;
  inputSchema: unknown;
  execute: (input: any) => Promise<string> | string;
};

class ToolRegistry {
  private tools = new Map<string, Tool>();

  // Регистрация: имя → инструмент
  register(name: string, tool: Tool): void {
    this.tools.set(name, tool);
  }

  // Получение одного
  get(name: string): Tool | undefined {
    return this.tools.get(name);
  }

  // Список имён (для промпта, UI, дебага)
  list(): string[] {
    return [...this.tools.keys()];
  }

  // Все пары [имя, инструмент] (для передачи в агента)
  entries(): [string, Tool][] {
    return [...this.tools.entries()];
  }
}
```

`Map` — правильная структура: уникальные ключи (имя инструмента), порядок вставки, O(1) доступ.

### registerBuiltins хелпер

Встроенные инструменты регистрируются через тот же механизм, что и пользовательские:

```typescript
function registerBuiltins(registry: ToolRegistry, sandbox: Sandbox) {
  registry.register("read",  createReadTool(sandbox));
  registry.register("grep",  createGrepTool(sandbox));
  registry.register("bash",  createBashTool(sandbox, createApproval({ mode: "interactive" })));
  registry.register("todo",  createTodoTool());
  registry.register("task",  createTaskTool(sandbox, registry));
  registry.register("askUser", createAskUserTool());
}

const registry = new ToolRegistry();
registerBuiltins(registry, sandbox);

// Передаём агенту все зарегистрированные инструменты
const agent = new ToolLoopAgent({
  model,
  instructions: buildSystemPrompt({ toolNames: registry.list() }),
  tools: Object.fromEntries(registry.entries()),
  stopWhen: stepCountIs(20),
});
```

Заметь: `toolNames: registry.list()` — список инструментов для промпта берётся из реестра, а не хардкодится. Добавил инструмент в реестр → он сам попал и в агента, и в промпт.

### Добавление своего инструмента

Пользовательский инструмент регистрируется так же, как встроенный:

```typescript
// Свой инструмент — тот же интерфейс, что и встроенные
const myTool: Tool = {
  description: `Summarize a file into bullet points.
WHEN TO USE: quickly understanding a file's structure without reading it fully.`,
  inputSchema: z.object({ path: z.string() }),
  execute: async ({ path }) => {
    const content = await sandbox.readFile(path);
    // ... своя логика
    return summarize(content);
  },
};

registry.register("summarize", myTool);   // ← просто добавили в реестр
```

### wrapTool(base, hooks) — beforeExecute / afterExecute

Оборачивание — это способ добавить сквозную логику (логирование, метрики, блокировку) к любому инструменту **без правки его кода**:

```typescript
interface ToolHooks {
  beforeExecute?: (name: string, input: any) => Promise<void> | void;
  afterExecute?: (name: string, input: any, result: string) => Promise<void> | void;
}

function wrapTool(name: string, base: Tool, hooks: ToolHooks): Tool {
  return {
    description: base.description,
    inputSchema: base.inputSchema,
    execute: async (input) => {
      await hooks.beforeExecute?.(name, input);      // до выполнения
      const result = await base.execute(input);
      await hooks.afterExecute?.(name, input, result); // после выполнения
      return result;
    },
  };
}

// Пример: логируем каждый вызов read
const loggedRead = wrapTool("read", registry.get("read")!, {
  beforeExecute: (name, input) => {
    console.error(`[tool] ${name}(${JSON.stringify(input)})`);
  },
  afterExecute: (name, input, result) => {
    console.error(`[tool] ${name} → ${result.length} chars`);
  },
});
registry.register("read", loggedRead);   // заменили на обёрнутую версию
```

Тот же паттерн, что и событийная шина из урока 11c, но **локальный для одного инструмента**. Событийная шина — для глобальных политик, `wrapTool` — для точечной обёртки.

---

## Ключевые выводы

- **Tool Registry = `Map<string, Tool>`** — единое место регистрации, поиска и перечисления инструментов.
- **`register` / `get` / `list` / `entries`** — четыре операции, покрывающие все нужды (добавить, найти, перечислить имена, передать агенту).
- **`registerBuiltins`** — встроенные инструменты регистрируются через тот же механизм, что и пользовательские; нет «особого» пути.
- **`toolNames` для промпта берётся из реестра** — добавил инструмент → он сам попал в агента и в промпт.
- **`wrapTool(base, hooks)`** — `beforeExecute`/`afterExecute` для сквозной логики (логирование, метрики) без правки кода инструмента.

---

## Попробуй сам

1. Собери реестр с тремя встроенными инструментами и проверь `list()` — должны вернуться все имена.
2. Добавь свой инструмент `summarize` через `register` и убедись, что он попал и в агента, и в `registry.list()` (а значит, в промпт).
3. Оберни `read` через `wrapTool` с `beforeExecute`/`afterExecute` (логирование) и убедись, что логи пишутся, а поведение инструмента не изменилось.
4. Проверь `get("несуществующий")` — должен вернуть `undefined`, а не упасть.
5. Напиши юнит-тест: после `register` переопределение инструмента с тем же именем должно заменять старый (а не добавлять дубль).
