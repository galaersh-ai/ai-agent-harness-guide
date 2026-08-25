# Строим AI-кодинг агент: 10 архитектурных паттернов из Vercel Academy

> Разбор курса «Build Your Own AI Coding Agent Harness» от Vercel Academy (автор — Joel Hooks). Все паттерны, код и архитектурные решения — из курса, адаптировано и переведено на русский.

---

## Введение

Три инструмента в цикле — это демо. Проблемы начинаются, когда пытаешься использовать агента для настоящей работы:

- Прочитал файл на 5000 строк — он остаётся в контексте навсегда
- Дал агенту `bash` — он выполнил `rm -rf`
- Попросил рефакторинг — агент объяснил, как рефакторить, но не сделал
- Одна длинная задача заполняет контекстное окно — агент теряет инструкции

**Harness** — слово для системы вокруг агента, которая решает все эти проблемы. Курс строит `TeensyCode` — рабочий harness с нуля на TypeScript + AI SDK v6.

---

## Паттерн 1: Описание как API выбора

**Проблема:** Модель выбирает инструмент по его `description`. Двухсловное описание `"Search files."` не даёт модели материала для решения — она игнорирует `grep` и лезет в `bash`.

**Решение — 5-секционный контракт:**

```typescript
const grep = tool({
  description: `Search file contents using regex. Returns matching lines with file paths.

WHEN TO USE: finding patterns across multiple files, locating function definitions,
  searching for imports, finding TODOs or error messages.

WHEN NOT TO USE: reading a known file (use read instead).
  Running commands (use bash instead).

DO NOT USE FOR: reading files (use read), listing directories (use bash),
  modifying files (use edit).

USAGE: pattern is a regex string. glob filters by file extension.
  Results are capped at 50 matches.

EXAMPLES:
  - Find all TODO comments: pattern "TODO" glob "*.ts"
  - Find function definitions: pattern "function \\w+" glob "*.ts"`,
  // ... inputSchema и execute
});
```

**Зачем каждая секция:**

| Секция | Назначение |
|--------|-----------|
| Первая строка | Что делает инструмент, что возвращает |
| WHEN TO USE | Конкретные сценарии с ключевыми словами из промпта |
| WHEN NOT TO USE | Мягкое перенаправление к нужному инструменту |
| DO NOT USE FOR | Жёсткая граница, повтор негатива |
| USAGE | Ограничения, которые схема не передаёт |
| EXAMPLES | Конкретные вызовы для паттерн-матчинга |

**Почему дублируется негатив:** Все модели (Haiku, Sonnet, Opus) по умолчанию выбирают `bash` — это «bash gravity». Сказать «не используй для поиска» один раз недостаточно; дважды — почти всегда достаточно.

---

## Паттерн 2: Фабрика с внедрением бэкенда

**Проблема:** Описание, проверка безопасности и вызов `execSync` лежат в одном замыкании. Когда нужно выполнять команды не на этой машине (песочница, удалённая VM), каждый инструмент приходится переписывать.

**Решение — паттерн «фабрика»:**

```typescript
interface BashOperations {
  exec(command: string): Promise<{ stdout: string; exitCode: number }>;
}

function createBashTool(operations: BashOperations, safePrefixes: string[]) {
  function isSafe(command: string): boolean {
    return safePrefixes.some((p) => command.trim().startsWith(p));
  }

  return tool({
    description: `Execute a shell command in the working directory.
WHEN TO USE: running build commands, installing packages, running tests.
WHEN NOT TO USE: reading file contents (use read instead).
DO NOT USE FOR: reading files (use read), searching code (use grep).`,
    inputSchema: z.object({
      command: z.string().describe("Shell command to execute"),
    }),
    execute: async ({ command }) => {
      if (!isSafe(command)) return `Blocked: "${command}" requires approval.`;
      const { stdout } = await operations.exec(command);
      return stdout || "(no output)";
    },
  });
}
```

**Локальный бэкенд:**
```typescript
const localOps: BashOperations = {
  exec: async (command) => {
    try {
      const stdout = execSync(command, { cwd, encoding: "utf-8", timeout: 30_000 });
      return { stdout, exitCode: 0 };
    } catch (e: any) {
      return { stdout: e.stdout || e.stderr || e.message || "", exitCode: e.status ?? 1 };
    }
  },
};
const bash = createBashTool(localOps, SAFE_PREFIXES);
```

**Замена на песочницу — одна строка:**
```typescript
const sandboxOps: BashOperations = { exec: (cmd) => sandbox.exec(cmd) };
const bash = createBashTool(sandboxOps, SAFE_PREFIXES);
```

Тот же инструмент, другой бэкенд — описание, схема и проверка безопасности не двигаются.

---

## Паттерн 3: Discriminated union для конфигурации

**Проблема:** Allowlist из одного режима («блокировать всё, чего нет в списке») не подходит для реального harness: у CI нет человека, субагенту нужно лишь часть доверия, пользователь хочет одобрить `npm install` один раз.

**Решение — эволюция конфига:**

```typescript
// Этап 1: Boolean (тупик)
needsApproval: true  // блокирует всё

// Этап 2: Function (лучше, но не конфигурируемо)
needsApproval: ({ command }) => !SAFE_PREFIXES.some(p => command.startsWith(p))

// Этап 3: Discriminated union (правильная форма)
type ApprovalConfig =
  | { mode: "interactive" }
  | { mode: "background" }
  | { mode: "delegated"; trust: string[] };

function createApproval(config: ApprovalConfig) {
  return ({ command }: { command: string }) => {
    if (config.mode === "background") return false;
    if (config.mode === "delegated") {
      return !config.trust.some((p) => command.trim().startsWith(p));
    }
    return !SAFE_PREFIXES.some((p) => command.trim().startsWith(p));
  };
}
```

**Три режима на месте вызова:**
```typescript
// Интерактивный: человек одобряет
const bash = createBashTool(localOps, createApproval({ mode: "interactive" }));

// Фоновый: авто-одобрение (CI)
const bash = createBashTool(localOps, createApproval({ mode: "background" }));

// Делегированный: субагент наследует срез доверия
const bash = createBashTool(localOps,
  createApproval({ mode: "delegated", trust: ["npm test", "npm run build", "npx tsc"] }));
```

**Почему discriminated union, а не три функции:** Конфиг — это данные, а не код. Его можно загружать из `AGENTS.md`, валидировать через Zod, сериализовать через границу субагента.

---

## Паттерн 4: Промпт как чистая функция

**Проблема:** Секционированный промпт захардкожен. Как только меняется рабочая директория, бэкенд песочницы или набор инструментов — промпт должен меняться вместе.

**Решение:**

```typescript
interface PromptContext {
  workingDirectory: string;
  sandboxType: string;
  toolNames: string[];
  gitBranch?: string;
  projectContext?: string;
}

function buildSystemPrompt(ctx: PromptContext): string {
  const sections: string[] = [];

  sections.push(`You are a coding agent working in: ${ctx.workingDirectory}`);

  sections.push(`# Agency
- USE your tools. Read files, search code, run commands, then answer.
- Do NOT explain what you WOULD do. Actually do it.
- Available tools: ${ctx.toolNames.join(", ")}`);

  sections.push(`# Guardrails
- Prefer simple, minimal changes
- Search before creating, and reuse existing patterns
- No new dependencies without asking`);

  if (ctx.projectContext) {
    sections.push(`# Project Instructions (from AGENTS.md)\n${ctx.projectContext}`);
  }

  return sections.join("\n");
}
```

**Почему функция, а не строка:** Промпт становится тестируемым (assert на вывод), компонуемым (добавлять секции), заменяемым (пользователь может подсунуть свой билдер) и детерминированным (тот же контекст → тот же промпт).

---

## Паттерн 5: Единый интерфейс песочницы

**Проблема:** Инструменты знают слишком много. `read` знает про `readFileSync`, `bash` про `execSync`. Когда нужно запускать где-то ещё — каждый инструмент переписывается.

**Решение — минимальный async-интерфейс:**

```typescript
interface Sandbox {
  type: string;
  workingDirectory: string;
  readFile(path: string): Promise<string>;
  exec(command: string): Promise<{ stdout: string; exitCode: number }>;
  stop(): Promise<void>;
  expiresAt?: number;
  snapshot?(): Promise<{ snapshotId: string }>;
}
```

**Три реализации:**

| | Local | just-bash | Cloud |
|---|---|---|---|
| Стоимость | Бесплатно | Бесплатно | Поминутно |
| Задержка | Микросекунды | Микросекунды | Десятки–сотни мс |
| Изоляция | Нет | Частичная (CoW) | Полная, отдельная ВМ |
| Персистентность | Постоянная | Сборка мусора | Snapshot/restore |
| Таймаут | Нет | Нет | Жёсткий лимит 30–60 мин |

**Принцип:** «Делайте интерфейс настолько маленьким, насколько можно. Всё, что вы добавите — каждая реализация обязана поддерживать вечно.»

**Локальная реализация (~15 строк):**
```typescript
function createLocalSandbox(dir: string): Sandbox {
  return {
    type: "local",
    workingDirectory: dir,
    readFile: async (p) => readFileSync(resolve(dir, p), "utf-8"),
    exec: async (command) => {
      try {
        const stdout = execSync(command, { cwd: dir, encoding: "utf-8", timeout: 30_000 });
        return { stdout, exitCode: 0 };
      } catch (e: any) {
        return { stdout: e.stdout || e.stderr || e.message || "", exitCode: e.status ?? 1 };
      }
    },
    stop: async () => {},
  };
}
```

**In-memory реализация (just-bash, copy-on-write):**
```typescript
const MOUNT = "/home/user/project";

async function createJustBashSandbox(dir: string): Promise<Sandbox> {
  const jb = await JustBashSandbox.create({ overlayRoot: dir });
  return {
    type: "just-bash",
    workingDirectory: dir,
    readFile: async (p) => jb.readFile(`${MOUNT}/${p}`),
    exec: async (command) => {
      const cmd = await jb.runCommand(command, { cwd: MOUNT });
      const finished = await cmd.wait();
      return { stdout: await cmd.output(), exitCode: finished.exitCode };
    },
    stop: async () => {},
  };
}
```

Чтение с реального диска, запись в память. При остановке — утилизация.

---

## Паттерн 6: Pruning + bounded output

**Проблема:** Каждый tool call остаётся в контексте навсегда. Входные токены растут линейно:

```
Step 0: 1,200 input, 450 output
Step 1: 2,800 input, 200 output
Step 2: 4,100 input, 180 output
Step 3: 8,900 input, 350 output   ← переполнение
```

**Решение — pruning (4 строки):**

```typescript
import { pruneMessages } from "ai";

const agent = new ToolLoopAgent({
  // ... existing config
  prepareCall: async (options) => ({
    ...options,
    messages: options.messages
      ? pruneMessages({
          messages: options.messages,
          toolCalls: "before-last-3-messages",
        })
      : undefined,
  }),
});
```

После pruning кривая выходит на плато:
```
Step 0: 1,200 input
Step 1: 2,800 input
Step 2: 3,100 input   ← старое отсечено
Step 3: 3,400 input   ← плато
```

**Bounded output как профилактика:**

| Инструмент | Лимит | Почему |
|-----------|-------|--------|
| `read` | 500 строк | Достаточно понять структуру, мало — чтобы не закопать модель |
| `grep` | 50 совпадений | 50 результатов ответили на вопрос; 500 — дамп данных |
| `bash` | 5000 символов | Хранить **хвост** — ошибки обычно в конце |

Каждое усечение — видимое сообщение для модели: `"... (truncated, showing last 5000 chars)"`. Модель может пагинировать.

---

## Паттерн 7: Explorer/Executor делегирование

**Проблема:** Одиночный агент загрязняет контекст (читает 20 файлов), теряет фокус (на 30-м шаге отклоняется), имеет слишком широкие возможности (исследование не должно иметь права на модификацию).

**Решение — два типа подагентов:**

| Параметр | Explorer | Executor |
|----------|----------|----------|
| Инструменты | read, grep | read, grep, bash (delegated) |
| Модель | claude-haiku-4-5 | claude-sonnet-4-6 |
| Бюджет шагов | 5 | 15 |
| Может модифицировать | Нет | Да |
| Может спрашивать пользователя | Нет | Нет |

**Task tool — слой маршрутизации:**
```typescript
function createTaskTool(sandbox, parentTools) {
  return tool({
    description: `Delegate work to a subagent.
      Explorer (default): read-only research with a fast model.
      Executor: implementation with a stronger model and delegated trust.`,
    inputSchema: z.object({
      description: z.string(),
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

**Когда делегировать:** многofile-исследование, параллельные задачи, механическая работа.
**Когда не делегировать:** однофайловые правки, архитектурные решения, неоднозначность.

---

## Паттерн 8: Событийная шина для сквозных concerns

**Проблема:** Реестр обрабатывает «какие инструменты существуют», но не «что происходит вокруг вызовов». Логирование, блокировка файлов, обёртка в OS-sandbox, автокоммит — сквозные concerns.

**Решение — event bus:**

```typescript
type LifecycleEvent =
  | "session_start"
  | "tool_call"
  | "tool_result"
  | "session_before_compact"
  | "session_shutdown";

interface EventBus {
  on(event: LifecycleEvent, handler: (data: any) => Promise<EventResult>): void;
  emit(event: LifecycleEvent, data: any): Promise<EventResult[]>;
}
```

**Четыре примера:**

```typescript
// 1. Логирование (pass-through)
bus.on("tool_call", async ({ toolName, input }) => {
  console.error(`[${new Date().toISOString()}] ${toolName}: ${JSON.stringify(input)}`);
});

// 2. Блокировка защищённых файлов
const PROTECTED = [".env", "package-lock.json"];
bus.on("tool_call", async ({ toolName, input }) => {
  if (toolName === "write" && PROTECTED.some((p) => input.path.endsWith(p))) {
    return { block: true, reason: `${input.path} is protected by policy.` };
  }
});

// 3. Инъекция safety-промпта перед компактизацией
bus.on("session_before_compact", async () => {
  return {
    modify: {
      customInstructions: "Preserve all safety constraints across compaction.",
    },
  };
});

// 4. Автокоммит при завершении
bus.on("session_shutdown", async ({ sandbox }) => {
  const { stdout } = await sandbox.exec("git status --porcelain");
  if (stdout.trim()) {
    await sandbox.exec(`git add -A && git commit -m "WIP: auto-save"`);
  }
});
```

**Правило цепочки:** `block: true` прерывает, `modify` применяет, void продолжает.

---

## Паттерн 9: Progressive disclosure для навыков

**Проблема:** Вставка специализированных знаний в системный промпт работает для 1–2 пакетов. К пяти — промпт 15000 токенов, платишь за них при каждом вызове.

**Решение:**
- Имена и однострочные описания живут в промпте (дёшево)
- Полное содержимое в markdown-файлах, загружается только по запросу

```typescript
interface Skill {
  name: string;
  description: string;
  path: string;
}

function discoverSkills(dirs: string[]): Skill[] {
  const skills: Skill[] = [];
  const seen = new Set<string>();
  for (const dir of dirs) {
    if (!existsSync(dir)) continue;
    for (const entry of readdirSync(dir)) {
      const path = join(dir, entry, "SKILL.md");
      if (existsSync(path) && !seen.has(entry)) {
        seen.add(entry);
        const content = readFileSync(path, "utf-8");
        const { description } = parseFrontmatter(content);
        skills.push({ name: entry, description: description ?? "", path });
      }
    }
  }
  return skills;
}
```

Структура: `skills/<name>/SKILL.md` с YAML-фронтматтером. Инструмент `loadSkill` принимает имя и возвращает полное содержимое.

---

## Паттерн 10: AGENTS.md — конвенция вместо конфигурации

**Проблема:** Harness общий — проект нет. Один проект использует `bun test`, другой `vitest`, третий `npm run check`. Агент не должен угадывать.

**Решение — markdown-файл в репо:**

```markdown
# Project Instructions

## Commands
- `bun test` runs the test suite
- `bun run build` builds for production
- `bun run lint` checks code style

## Architecture
- Monorepo, packages live in `packages/`
- Each package has its own `tsconfig.json`
- Shared types in `packages/shared/`

## Style
- Functional components, no classes
- Named exports, not default
- Error messages must be user-facing

## Lessons learned
- Auth middleware must run before rate limiting
- Don't modify migration files directly, generate new ones
```

**Discovery:**
```typescript
const agentsPath = join(cwd, "AGENTS.md");
const projectContext = existsSync(agentsPath)
  ? readFileSync(agentsPath, "utf-8")
  : undefined;

const instructions = buildSystemPrompt({
  workingDirectory: cwd,
  sandboxType: "local",
  toolNames: Object.keys(tools),
  projectContext,
});
```

Один и тот же harness ведёт себя как React-агент в React-проекте и CLI-агент в CLI-проекте — потому что проект сам сказал агенту, какой он.

**Знакомый паттерн:** Cursor → `.cursorrules`, Codex → `AGENTS.md`, Claude Code → `CLAUDE.md`. Имя разное, паттерн один.

---

## Верификация: честность, а не покрытие

Агент скажет «all tests pass», ни разу не запустив тесты. Это не злой умысел — модели подгоняют паттерны.

**Контракт верификации:**
```typescript
sections.push(`# Verification
After making changes, verify your work:
1. Run \`npx tsc --noEmit\` when TypeScript is present
2. Run lint, test, or build commands only if they exist in this project
3. Report exactly what you ran, what was blocked, and what was unavailable
4. Do NOT claim "tests pass" without running them.

Scope your claims honestly.`);
```

**Scoped claims:**

| Может сказать | Нужно |
|---|---|
| «All tests pass» | «Ran npm test: 47 passed, 3 failed (pre-existing)» |
| «I fixed the bug» | «Fixed null check in auth.ts:42. tsc passes. Tests blocked by approval.» |
| «The build works» | «Ran npm run build: succeeded in 4.2s, no warnings.» |

**Маркер конфабуляции:** мягкие фразы («should be fine», «looks good»). Модель, запустившая проверку, говорит в прошедшем времени с конкретным результатом.

---

## Human-in-the-loop: ищи → спрашивай → действуй

Агенты по умолчанию не задают вопросов — предпочитают «разобраться сами».

**Протокол в промпте:**
```
# Handling Ambiguity
When the task is ambiguous or has multiple valid approaches:
1. Search the code or docs to gather context first
2. Use askUser to let the user choose. Do NOT guess.
3. Examples: "add auth" -> ask OAuth or JWT; "set up a db" -> ask Postgres or SQLite

Specific tasks (with file paths, line numbers, or precise instructions) do not
need askUser. Act directly.
```

**Инструмент `askUser`:**
```typescript
const askUser = tool({
  description: `Ask the user a multiple-choice question.
WHEN TO USE: scoping ambiguous tasks, choosing between approaches.
WHEN NOT TO USE: you already have enough context to proceed.`,
  inputSchema: z.object({
    question: z.string(),
    options: z.array(z.string()).min(2).max(4),
  }),
  execute: async ({ question, options }) => {
    const formatted = options.map((o, i) => `${i + 1}. ${o}`).join("\n");
    console.log(`\nQuestion: ${question}\n${formatted}\n`);
    return `Asked: "${question}"\nOptions:\n${formatted}\n\n(Awaiting user response.)`;
  },
});
```

---

## Жизненный цикл облачной песочницы

**4 состояния:**
```
provisioning -> active -> hibernating -> hibernated
                  ^             |
                  +-------------+ (restore)
```

**Математика стоимости:**
- Без lifecycle: песочница работает до жёсткого срока (4 часа) = **$4.80**
- С гибернизацией по неактивности (5 мин): **$0.50**

**5 продакшн-проблем:**
1. Устаревшие хэндлы после переподключения → проверка `echo probe`
2. Устаревшие данные о сроке → свежие данные от провайдера
3. Опрос сбрасывает неактивность → трекер только для пользовательских событий
4. Петли автоматического возобновления → restore только при первом входе
5. Расхождение состояния → API провайдера = source of truth

---

## Стэк

| Компонент | Назначение |
|-----------|-----------|
| **AI SDK** (Vercel) | ToolLoopAgent, tool(), stepCountIs, pruneMessages, streaming |
| **AI Gateway** | Роутинг моделей строкой `"anthropic/claude-haiku-4-5"` |
| **Vercel Sandbox** | Удалённая ВМ с изолированной ФС, git, npm |
| **just-bash** | In-memory виртуальная ФС с copy-on-write |
| **Vercel Workflow** | Долговечные процессы для жизненного цикла |
| **Zod v3** | Схемы входа инструментов |
| **TypeScript** | Язык, async/await |

---

## Ссылки

- [Оригинальный курс Vercel Academy](https://vercel.com/academy/build-ai-agent-harness)
- [AI SDK документация](https://sdk.vercel.ai/docs)
- [just-bash](https://github.com/nichochar/just-bash)

---

*Перевод и адаптация: [galaersh-ai](https://github.com/galaersh-ai). Оригинальный контент — Vercel Academy, Joel Hooks.*
