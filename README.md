# Строим AI-кодинг агент: архитектурные паттерны из Vercel Academy

> Разбор курса «[Build Your Own AI Coding Agent Harness](https://vercel.com/academy/build-ai-agent-harness)» от Vercel Academy (автор — Joel Hooks). Перевод и адаптация на русский.

---

## О чём этот текст

Представь: ты написал простого AI-чатбота. Он отвечает на вопросы, объясняет код, даже генерирует примеры. Но стоит попросить его **сделать** что-то реальное — прочитать файл проекта, найти баг, запустить тесты — и он начинает выдумывать. «Я бы прочитал файл...», «Обычно в таких случаях...», «Вот как это выглядит...» — но файл так и не открыл.

Один инструмент (возможность читать файлы) превращает чатбота в **агента** — существо, которое действует, а не рассказывает. Но тут начинаются настоящие проблемы:

- Агент читает файл на 5000 строк — и этот файл **навсегда** остаётся в его памяти, засоряя контекст
- Даёшь ему возможность запускать команды — он выполняет `rm -rf` вместо нужной операции
- Просишь рефакторинг — он **объясняет**, как рефакторить, но не делает
- Длинная задача на 30+ шагов заполняет всю память — агент забывает свои же инструкции

**Harness** (обвязка, каркас) — это система вокруг агента, которая решает все эти проблемы. Курс Vercel Academy строит такой каркас с нуля. Ниже — 10 ключевых архитектурных паттернов, которые там используются.

---

## Паттерн 1: Описание инструмента как API для модели

### Проблема

У каждого инструмента есть поле `description` — текст, который читает AI-модель, чтобы решить, когда этот инструмент использовать. Если описание слишком короткое (например, просто `"Search files."`), модель не может принять осознанное решение и по умолчанию выбирает самый универсальный инструмент — `bash`. Это называется **bash gravity** — «гравитация bash»: все модели (Haiku, Sonnet, Opus) тяготеют к `bash`, когда описания других инструментов слабые.

### Решение

Пяти-секционный контракт описания. Каждый инструмент описывается не одной фразой, а структурированным блоком:

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
  inputSchema: z.object({
    pattern: z.string(),
    path: z.string().optional(),
    glob: z.string().optional(),
  }),
  execute: async ({ pattern, path: searchPath, glob: globFilter }) => {
    // ... выполнение поиска с лимитом 50 совпадений
  },
});
```

> **📝 Комментарий переводчика — что говорит промпт модели:**
>
> *«Ищи содержимое файлов по регулярному выражению. Возвращает совпавшие строки с путями к файлам.*
>
> *КОГДА ИСПОЛЬЗОВАТЬ: поиск паттернов по множеству файлов, нахождение определений функций, поиск импортов, нахождение TODO или сообщений об ошибках.*
>
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: чтение известного файла (используй read). Запуск команд (используй bash).*
>
> *НЕ ИСПОЛЬЗОВАТЬ ДЛЯ: чтения файлов (read), просмотра директорий (bash), изменения файлов (edit).*
>
> *ПРИМЕРЫ: найти все TODO-комментарии: pattern "TODO" glob "*.ts"; найти определения функций: pattern "function \\w+" glob "*.ts"»*

### Зачем каждая секция

| Секция | Что делает | Пример |
|--------|-----------|--------|
| Первая строка | Говорит модели, **что** делает инструмент и что возвращает | «Ищи по regex, возвращай строки с путями» |
| WHEN TO USE | Конкретные сценарии с ключевыми словами из типичных промптов | «Поиск паттернов, определения функций, импорты» |
| WHEN NOT TO USE | **Мягко** перенаправляет к правильному инструменту, называя его по имени | «Читаешь файл? Используй read» |
| DO NOT USE FOR | **Жёстко** запрещает использовать инструмент не по назначению | «Никогда не используй для чтения файлов» |
| USAGE | Ограничения, которые не видны из схемы входа | «Лимит 50 совпадений» |
| EXAMPLES | Конкретные вызовы — модель повторяет паттерн | «pattern "TODO" glob "*.ts"» |

### Почему негатив дублируется

WHEN NOT TO USE и DO NOT USE FOR говорят одно и то же, но разной жёсткостью. Это намеренно. Тесты показали:

- **Haiku** (быстрая, дешёвая) читает WHEN NOT TO USE, но при неоднозначности игнорирует
- **Sonnet** (средняя) уважает WHEN NOT TO USE, но DO NOT USE FOR служит подкреплением
- **Opus** (тяжёлая) хорошо работает с обоими, повторение не вредит

Сказать «не используй для поиска» один раз — недостаточно. Дважды — почти всегда достаточно.

---

## Паттерн 2: Фабрика с внедрением бэкенда

### Проблема

В первом приближении инструмент `bash` выглядит так: описание + схема входа + проверка безопасности + вызов `execSync` — всё в одном замыкании. Это работает, пока команда выполняется на локальной машине. Но через пару модулей появится облачная песочница, и `execSync` нужно будет заменить на сетевой вызов. Переписывать каждый инструмент — мучительно.

### Решение

Выделяем **интерфейс выполнения** и внедряем его в фабрику:

```typescript
// Интерфейс — контракт «как выполнять команды»
interface BashOperations {
  exec(command: string): Promise<{ stdout: string; exitCode: number }>;
}

// Фабрика — создаёт инструмент, не зная, ГДЕ будут выполняться команды
function createBashTool(operations: BashOperations, safePrefixes: string[]) {
  function isSafe(command: string): boolean {
    return safePrefixes.some((p) => command.trim().startsWith(p));
  }

  return tool({
    description: `Execute a shell command in the working directory.
WHEN TO USE: running build commands, installing packages, running tests,
  git operations, directory listings.
WHEN NOT TO USE: reading file contents (use read instead).
  Searching for patterns (use grep instead).
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

> **📝 Комментарий переводчика — что говорит промпт модели:**
>
> *«Выполни shell-команду в рабочей директории.*
>
> *КОГДА ИСПОЛЬЗОВАТЬ: запуск сборки, установка пакетов, тесты, git-операции, просмотр директорий.*
>
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: чтение файлов (используй read). Поиск паттернов (используй grep).*
>
> *НЕ ИСПОЛЬЗОВАТЬ ДЛЯ: чтения файлов (read), поиска по коду (grep).»*

### Локальный бэкенд (выполняет на этой машине)

```typescript
const localOps: BashOperations = {
  exec: async (command) => {
    try {
      const stdout = execSync(command, {
        cwd: dir,
        encoding: "utf-8",
        timeout: 30_000,
      });
      return { stdout, exitCode: 0 };
    } catch (e: any) {
      return {
        stdout: e.stdout || e.stderr || e.message || "",
        exitCode: e.status ?? 1,
      };
    }
  },
};

const bash = createBashTool(localOps, SAFE_PREFIXES);
```

### Замена на песочницу — одна строка

Когда появляется облачная песочница, достаточно создать другой бэкенд:

```typescript
const sandboxOps: BashOperations = {
  exec: (command) => sandbox.exec(command),
};

const bash = createBashTool(sandboxOps, SAFE_PREFIXES);
```

Описание, схема, проверка безопасности — **не изменились**. Модель видит тот же инструмент. Меняется только «сантехника» под капотом.

### Безопасность: почему шлюз внутри execute

В AI SDK есть опция `needsApproval` — она якобы должна блокировать опасные команды и спрашивать пользователя. Но это **сигнал, а не шлюз**. Когда `needsApproval` возвращает `true`, SDK создаёт запрос на одобрение и **пропускает выполнение**. Если обработчик одобрения не подключён (а на этом этапе его нет), вызов инструмента «исчезает» — модель не получает результата и **выдумывает его**: «Готово! Я удалил файлы». Пользователь видит успех, хотя команда не выполнялась.

Поэтому шлюз стоит **внутри `execute`** — при блокировке инструмент возвращает строку: `Blocked: "rm -rf /" requires approval.`. Модель читает эту строку как любой результат и честно передаёт блокировку пользователю. Нет тихого отказа, нет выдуманного успеха.

---

## Паттерн 3: Discriminated union для конфигурации

### Проблема

Allowlist из паттерна 2 имеет один режим: «блокировать всё, чего нет в списке». Для реального каркаса этого мало:

- У **CI-пайплайна** нет человека, чтобы спрашивать — нужно авто-одобрять
- **Субагент** (подчинённый агент) должен получить лишь **часть** доверия родителя
- **Локальный пользователь** хочет одобрить `npm install express` один раз и не отвечать снова через три шага

### Решение — три этапа эволюции конфига

**Этап 1 — Boolean** (тупик):
```typescript
needsApproval: true  // блокирует ВСЁ. Бесполезно
```

**Этап 2 — Функция** (лучше, но не конфигурируемо):
```typescript
needsApproval: ({ command }) => {
  if (SAFE_PREFIXES.some(p => command.startsWith(p))) return false;
  return true;
}
```
Правило «зашито»: CI получает тот же шлюз, что и локальный терминал. Перенастроить без переписывания нельзя.

**Этап 3 — Discriminated union** (правильная форма):
```typescript
type ApprovalConfig =
  | { mode: "interactive" }                              // человек решает
  | { mode: "background" }                               // авто-одобрение (CI)
  | { mode: "delegated"; trust: string[] };              // субагент с ограниченным доверием

function createApproval(config: ApprovalConfig) {
  return ({ command }: { command: string }) => {
    if (config.mode === "background") return false;      // всё разрешено
    if (config.mode === "delegated") {
      return !config.trust.some((p) => command.trim().startsWith(p));
    }
    // interactive: безопасные команды — автоматически, остальное — спросить человека
    return !SAFE_PREFIXES.some((p) => command.trim().startsWith(p));
  };
}
```

### Три режима на месте вызова

```typescript
// Локальная разработка: человек одобряет опасные команды
const bash = createBashTool(localOps, createApproval({ mode: "interactive" }));

// CI/автоматизация: всё выполняется автоматически
const bash = createBashTool(localOps, createApproval({ mode: "background" }));

// Субагент: наследует только часть доверия от родителя
const bash = createBashTool(
  localOps,
  createApproval({
    mode: "delegated",
    trust: ["npm test", "npm run build", "npx tsc"],
  })
);
```

> **📝 Комментарий переводчика:**
>
> *Режим `delegated` с списком `trust: ["npm test", "npm run build", "npx tsc"]` означает: субагент может запускать тесты, сборку и проверку типов, но не может устанавливать пакеты, не может модифицировать файлы напрямую, не может запускать произвольные команды. Список доверия намеренно мал.*

### Почему discriminated union, а не три отдельные функции

Конфиг — это **данные, а не код**. Его можно:
- Загружать из файла `AGENTS.md`
- Валидировать через Zod: `z.discriminatedUnion("mode", [...])`
- Сериализовать и передавать через границу субагента
- Менять режимы без правки кода каркаса

TypeScript сужает `config.trust` до `string[]` только внутри ветки `delegated` — это та ошибка компиляции, которую boolean и обычная функция поймать не могли.

---

## Паттерн 4: Системный промпт как чистая функция

### Проблема

Секционированный промпт (Agency + Guardrails) захардкожен в коде. Это работает, пока один проект, одна песочница, фиксированный набор инструментов. Как только что-то меняется — другая рабочая директория, другой бэкенд, субагент с подмножеством инструментов — промпт должен меняться вместе. Хардкод строки этого не вывозит.

### Решение

Выносим промпт в чистую функцию `buildSystemPrompt`, которая принимает контекст и возвращает строку:

```typescript
interface PromptContext {
  workingDirectory: string;
  sandboxType: string;        // "local" | "just-bash" | "cloud"
  toolNames: string[];        // какие инструменты реально подключены
  gitBranch?: string;         // текущая ветка (если известна)
  projectContext?: string;    // содержимое AGENTS.md (если есть)
}

function buildSystemPrompt(ctx: PromptContext): string {
  const sections: string[] = [];

  sections.push(
    `You are a coding agent working in: ${ctx.workingDirectory}`
  );

  sections.push(`# Agency
- USE your tools. Read files, search code, run commands, then answer.
- Do NOT explain what you WOULD do. Actually do it.
- Available tools: ${ctx.toolNames.join(", ")}`);

  sections.push(`# Guardrails
- Prefer simple, minimal changes
- Search before creating, and reuse existing patterns
- No new dependencies without asking`);

  if (ctx.gitBranch) {
    sections.push(`- Current branch: ${ctx.gitBranch}`);
  }

  if (ctx.projectContext) {
    sections.push(`# Project Instructions (from AGENTS.md)\n${ctx.projectContext}`);
  }

  return sections.join("\n");
}
```

> **📝 Комментарий переводчика — перевод секций промпта:**
>
> *Секция Agency («Действие»):*
> *«Ты — агент по работе с кодом, рабочая директория: ...*
> *ДЕЙСТВУЙ. Читай файлы, ищи по коду, запускай команды, потом отвечай.*
> *НЕ объясняй, что бы ты СДЕЛАЛ. Действуй на самом деле.*
> *Доступные инструменты: ...»*
>
> *Секция Guardrails («Ограничения»):*
> *«Предпочитай простые, минимальные изменения.*
> *Сначала ищи, потом создавай; переиспользуй существующие паттерны.*
> *Никаких новых зависимостей без спроса.»*
>
> *Секция Project Instructions (из AGENTS.md):*
> *Вставляется целиком из файла проекта — команды сборки, архитектура, стиль кода, уроки.*

### Подключение в коде

```typescript
const instructions = buildSystemPrompt({
  workingDirectory: cwd,
  sandboxType: "local",
  toolNames: Object.keys({ read, grep, bash }),
  projectContext,  // содержимое AGENTS.md или undefined
});

const agent = new ToolLoopAgent({
  model: "anthropic/claude-haiku-4-5",
  instructions,
  tools: { read, grep, bash },
  stopWhen: stepCountIs(10),
});
```

### Почему функция, а не строка

- **Тестируемость:** можно написать assert, что промпт содержит нужную секцию при данном контексте
- **Компонуемость:** добавлять секции, не трогая остальные
- **Заменяемость:** пользователь может подсунуть свой билдер промпта
- **Детерминированность:** тот же контекст → тот же промпт

---

## Паттерн 5: Единый интерфейс песочницы

### Проблема

Инструменты `read`, `grep` и `bash` знают слишком много о среде выполнения. `read` импортирует `readFileSync` из Node.js, `bash` вызывает `execSync`. Когда нужно запускать агента не на локальной машине — в облачной песочнице, в in-memory файловой системе — каждый инструмент приходится переписывать.

### Решение

Минимальный async-интерфейс, за которым прячутся все реализации:

```typescript
interface Sandbox {
  type: string;
  workingDirectory: string;
  readFile(path: string): Promise<string>;
  exec(command: string): Promise<{ stdout: string; exitCode: number }>;
  stop(): Promise<void>;
  expiresAt?: number;                                    // только для cloud
  snapshot?(): Promise<{ snapshotId: string }>;          // только для cloud
}
```

Все методы — `async`, даже если реализация под капотом синхронная. Облачный бэкенд реально async; несогласованные сигнатуры между реализациями — бардак.

### Три реализации

**Локальная** (~15 строк, «скучная обёртка»):
```typescript
function createLocalSandbox(dir: string): Sandbox {
  return {
    type: "local",
    workingDirectory: dir,
    readFile: async (p) => readFileSync(resolve(dir, p), "utf-8"),
    exec: async (command) => {
      try {
        const stdout = execSync(command, {
          cwd: dir, encoding: "utf-8", timeout: 30_000,
        });
        return { stdout, exitCode: 0 };
      } catch (e: any) {
        return {
          stdout: e.stdout || e.stderr || e.message || "",
          exitCode: e.status ?? 1,
        };
      }
    },
    stop: async () => {},
  };
}
```

**In-memory** (just-bash, copy-on-write — читает с диска, пишет в память):
```typescript
const MOUNT = "/home/user/project";  // виртуальная точка монтирования

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
    stop: async () => {},  // при остановке виртуальная ФС утилизируется
  };
}
```

> **⚠️ Ловушка:** `just-bash` монтирует `overlayRoot` в `/home/user/project`, а не в корень. Каждый путь должен проходить через константу `MOUNT`. Это сбивает с толку всех.

**Облачная** (концептуально — реальный API зависит от провайдера):
```typescript
async function createCloudSandbox(config): Promise<Sandbox> {
  const vm = await VercelSandbox.create(config);
  return {
    type: "cloud",
    workingDirectory: "/workspace",
    expiresAt: Date.now() + 30 * 60 * 1000,  // 30 минут
    readFile: async (p) => vm.files.read(resolve("/workspace", p)),
    exec: async (command) => {
      const result = await vm.commands.run(command, { cwd: "/workspace" });
      return { stdout: result.stdout + result.stderr, exitCode: result.exitCode };
    },
    stop: async () => vm.close(),
    snapshot: async () => {
      const snap = await vm.snapshot();
      return { snapshotId: snap.id };
    },
  };
}
```

### Таблица компромиссов

| | Local | just-bash | Cloud |
|---|---|---|---|
| **Стоимость** | Бесплатно | Бесплатно | Поминутно |
| **Задержка** | Микросекунды | Микросекунды | Десятки–сотни мс |
| **Изоляция** | Нет | Частичная (CoW) | Полная, отдельная ВМ |
| **git, npm** | Локальная установка | Симулированы | Реальные |
| **Таймаут** | Нет | Нет | Жёсткий лимит 30–60 мин |

### Переключатель

```typescript
const sandboxType = process.env.SANDBOX || "local";
const sandbox =
  sandboxType === "just-bash"
    ? await createJustBashSandbox(cwd)
    : createLocalSandbox(cwd);
```

Всё, что ниже (инструменты, агент, промпт) — **одинаково** для любого бэкенда.

---

## Паттерн 6: Обрезка контекста и ограничение вывода

### Проблема

Каждый вызов инструмента остаётся в контексте **навсегда**. Входные токены растут линейно:

```
Step 0:  1,200 input tokens
Step 1:  2,800 input tokens
Step 2:  4,100 input tokens
Step 3:  8,900 input tokens   ← ускоряется
Step 4:  9,200 input tokens
...
Step 30: контекст переполнен, агент забывает инструкции
```

Контекстное окно — 200 000 токенов. Занятой агент упрётся за 30–50 шагов. Когда это происходит: инструкции выталкиваются из внимания, модель начинает игнорировать system prompt, выбор инструментов деградирует, агент зацикливается или галлюцинирует.

### Решение — pruning (обрезка старых результатов)

Четыре строки, которые меняют форму кривой:

```typescript
import { pruneMessages } from "ai";

const agent = new ToolLoopAgent({
  // ... существующая конфигурация
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

> **📝 Комментарий переводчика:**
>
> *`prepareCall` — хук, который вызывается перед каждым отправлением запроса к модели. Он получает полный запрос, который SDK собирается отправить, и может его модифицировать.*
>
> *`pruneMessages` с параметром `"before-last-3-messages"` удаляет все пары «вызов инструмента → результат», кроме трёх последних. Исходный промпт пользователя всегда сохраняется. Недавние взаимодействия сохраняются. Середина разговора, где копятся результаты инструментов, выбрасывается.*

**До обрезки (на шаге 15):**
```
[user prompt]                              ← сохраняется
[assistant + tool_call] → [tool_result]    ← старое, удаляется
[assistant + tool_call] → [tool_result]    ← старое, удаляется
...ещё 12 пар...
[assistant + tool_call] → [tool_result]    ← недавнее, остаётся
[assistant + tool_call] → [tool_result]    ← недавнее, остаётся
[assistant] → [user]                       ← недавнее, остаётся
```

**После обрезки:**
```
[user prompt]                              ← исходный промпт
[assistant + tool_call] → [tool_result]    ← недавнее
[assistant + tool_call] → [tool_result]    ← недавнее
[assistant] → [user]                       ← недавнее
```

**Результат — кривая выходит на плато:**
```
Step 0:  1,200 input
Step 1:  2,800 input
Step 2:  3,100 input   ← старое отсечено
Step 3:  3,400 input   ← плато
Step 4:  3,200 input   ← остаётся плоским
```

### Две ловушки

1. **Спред `...options` нужно ставить первым.** `prepareCall` получает полные опции, включая `model`, `tools` и `system`. Забытый спред молча их выбрасывает — агент ломается запутанно.
2. **Защитить `messages` от undefined.** На самом первом вызове SDK даёт поле `prompt`, но не массив `messages`. Вызов `pruneMessages({ messages: undefined })` бросает исключение.

### Bounded output — ограничение вывода инструментов (профилактика)

Обрезка убирает **старые** результаты. Но если один результат — 5000 токенов, ущерб уже нанесён. Лучший фикс — **выше по потоку**: инструменты должны выдавать ограниченный вывод по умолчанию.

| Инструмент | Лимит | Почему именно столько |
|-----------|-------|----------------------|
| `read` | 500 строк | Достаточно понять структуру файла; 10 000 строк — загрязнение |
| `grep` | 50 совпадений | 50 результатов ответили на вопрос; 500 — дамп данных |
| `bash` | 5000 символов | Большинство выводов помещается; `npm install` даёт шум |

**Bash хранит хвост, а не голову:**
```typescript
const MAX_BASH_CHARS = 5000;
const stdout = result.stdout || "(no output)";
const cappedStdout =
  stdout.length > MAX_BASH_CHARS
    ? stdout.slice(-MAX_BASH_CHARS) +
      `\n... (truncated, showing last ${MAX_BASH_CHARS} chars)`
    : stdout;
```

Ошибки обычно в конце: упавший тест печатает ошибки последними, упавшая сборка — ошибку последней. Храня хвост, хранишь часть, на которую агенту нужно реагировать.

Каждое усечение выдаёт **видимое сообщение** для модели: `"... (truncated, showing last 5000 chars)"`. Модель видит, что результат неполный, и может пагинировать (запросить следующую порцию). Инструмент, который молча усекает, хуже отсутствия усечения — модель думает, что видит полную картину.

---

## Паттерн 7: Делегирование подагентам (Explorer/Executor)

### Проблема

Одиночный агент на реальной задаче из 30+ шагов:
- **Загрязняет контекст:** читает 20 файлов для исследования, и все они остаются в памяти
- **Теряет фокус:** к 30-му шагу замечает опечатку в CSS, исправляет её, потом рефакторит импорт... исходная задача забыта
- **Имеет слишком широкие возможности:** во время исследования «по доброте душевной» исправляет файл, ломая что-то другое

### Решение — два типа подчинённых агентов

**Explorer** (исследователь) — только чтение:
```typescript
const explorer = new ToolLoopAgent({
  model: "anthropic/claude-haiku-4-5",       // быстрая, дешёвая
  instructions: `You are an explorer agent. Investigate and report back concisely.
    Working directory: ${sandbox.workingDirectory}`,
  tools: { read: parentTools.read, grep: parentTools.grep },  // только чтение!
  stopWhen: stepCountIs(5),                  // лимит 5 шагов
});
```

> **📝 Комментарий переводчика:**
>
> *Инструкция: «Ты — агент-исследователь. Исследуй и дай краткий отчёт. Рабочая директория: ...»*
>
> *Explorer не может модифицировать файлы, не может запускать команды, не может спрашивать пользователя. Это ограничение — фича: он не может отклониться от задачи, случайно что-то изменить или «сжечь» проект.*

**Executor** (исполнитель) — полные инструменты:
```typescript
const executorBash = createBashTool(sandbox, createApproval({
  mode: "delegated",
  trust: ["npm test", "npm run build", "npx tsc"],
}));

const executor = new ToolLoopAgent({
  model: "anthropic/claude-sonnet-4-6",      // более сильная модель
  instructions: `You are an executor agent. Follow instructions precisely.
    Working directory: ${sandbox.workingDirectory}
    Do NOT ask questions. Do NOT explore beyond what's needed. Execute the task.`,
  tools: { read: parentTools.read, grep: parentTools.grep, bash: executorBash },
  stopWhen: stepCountIs(15),                 // лимит 15 шагов
});
```

> **📝 Комментарий переводчика:**
>
> *Инструкция: «Ты — агент-исполнитель. Следуй инструкциям точно. Рабочая директория: ... Не задавай вопросов. Не исследуй сверх необходимого. Выполни задачу.»*
>
> *Executor получает `bash` с режимом `delegated` и доверенным списком `["npm test", "npm run build", "npx tsc"]`. Он может запускать тесты, сборку и проверку типов, но не может устанавливать пакеты или выполнять произвольные команды.*

### Сравнение

| Параметр | Explorer | Executor |
|----------|----------|----------|
| Инструменты | read, grep | read, grep, bash (delegated) |
| Модель | claude-haiku-4-5 (быстрая) | claude-sonnet-4-6 (сильная) |
| Бюджет шагов | 5 | 15 |
| Может модифицировать | **Нет** | Да (в пределах trust-листа) |
| Может спрашивать пользователя | **Нет** | **Нет** |

### Task tool — слой маршрутизации

Родитель вызывает один инструмент `task`, а тот выбирает нужный тип подагента:

```typescript
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

> **📝 Комментарий переводчика — что говорит промпт модели:**
>
> *«Делегируй работу подагенту.*
> *Explorer (по умолчанию): исследование только на чтение с быстрой моделью.*
> *Executor: реализация с более сильной моделью и делегированным доверием на bash.*
>
> *КОГДА ИСПОЛЬЗОВАТЬ: исследование множества файлов (explorer), массовая реализация (executor).*
>
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: неоднозначные требования (используй askUser), архитектурные решения (родитель решает сам).»*

### Когда делегировать, а когда нет

**Делегировать:**
- Исследование по многим файлам («найди все использования этой функции»)
- Параллельные независимые задачи
- Механическая объёмная работа

**Не делегировать:**
- Однофайловые изменения
- Последовательные зависимые изменения
- Архитектурные решения
- Неоднозначные требования (использовать `askUser`)

---

## Паттерн 8: Событийная шина для сквозных concerns

### Проблема

Реестр инструментов обрабатывает «какие инструменты существуют». Но есть задачи, которые не принадлежат ни одному инструменту: логирование каждого вызова, блокировка записи в защищённые файлы, обёртка команд в OS-sandbox, автокоммит при завершении сессии.

### Решение — event bus (событийная шина)

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

### Четыре примера расширений

**1. Логирование каждого вызова инструмента** (handler ничего не возвращает — pass-through):
```typescript
bus.on("tool_call", async ({ toolName, input }) => {
  console.error(`[${new Date().toISOString()}] ${toolName}: ${JSON.stringify(input)}`);
});
```

**2. Блокировка записи в защищённые файлы** (handler возвращает `block: true`):
```typescript
const PROTECTED = [".env", "package-lock.json"];

bus.on("tool_call", async ({ toolName, input }) => {
  if (toolName === "write" && PROTECTED.some((p) => input.path.endsWith(p))) {
    return { block: true, reason: `${input.path} is protected by policy.` };
  }
});
```

**3. Инъекция safety-промпта перед компактизацией** (handler возвращает `modify`):
```typescript
bus.on("session_before_compact", async () => {
  return {
    modify: {
      customInstructions:
        "Preserve all safety constraints and approval rules across compaction.",
    },
  };
});
```

> **📝 Комментарий переводчика:**
>
> *Событие `session_before_compact` срабатывает перед сжатием контекста. Handler возвращает модифицированный промпт: «Сохрани все ограничения безопасности и правила одобрения при сжатии». Это важно: после сжатия контекста агент может «забыть» свои guardrails.*

**4. Автокоммит при завершении** (handler выполняет действия в песочнице):
```typescript
bus.on("session_shutdown", async ({ sandbox }) => {
  const { stdout } = await sandbox.exec("git status --porcelain");
  if (stdout.trim()) {
    await sandbox.exec(`git add -A && git commit -m "WIP: auto-save"`);
  }
});
```

### Правило цепочки

Несколько обработчиков на одно событие выполняются в порядке регистрации:
- Любой handler вернул `block: true` → цепочка прерывается, вызов блокируется
- Handler вернул `modify` → модификация применяется, следующий handler видит изменённые данные
- Handler ничего не вернул → следующий handler продолжает

---

## Паттерн 9: Прогрессивное раскрытие навыков (Skills)

### Проблема

Специализированные знания (как работать с Docker, как настроить CI, как деплоить) хотелось бы вставлять в системный промпт. Но 1–2 пакета знаний — это нормально, а к пяти — промпт уже 15 000 токенов, и ты платишь за них при **каждом** вызове модели, даже если агенту нужен только один навык.

### Решение

Имена и однострочные описания живут в промпте (дёшево). Полное содержимое — в markdown-файлах на диске, загружается **только по запросу** через инструмент `loadSkill`.

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

Структура на диске:
```
skills/
  docker-deploy/
    SKILL.md          ← YAML-фронтматтер с name + description, затем полный текст
  ci-setup/
    SKILL.md
  api-testing/
    SKILL.md
```

В системном промпте — только имена:
```
# Skills
- docker-deploy: Deploy containers with Docker Compose
- ci-setup: Configure GitHub Actions CI pipeline
- api-testing: Write and run API integration tests
```

Когда агенту нужен полный текст — он вызывает `loadSkill("docker-deploy")` и получает содержимое. Дедупликация по имени: первый каталог побеждает (проектные навыки переопределяют глобальные).

---

## Паттерн 10: AGENTS.md — конвенция вместо конфигурации

### Проблема

Каркас агента общий — а проекты разные. Один проект использует `bun test`, другой `vitest`, третий `npm run check`. Архитектура — монорепо с пакетами. Стиль — функциональные компоненты, именованные экспорты. Агент не должен угадывать — проект должен **сам рассказать** о себе.

### Решение

Положить markdown-файл `AGENTS.md` в корень проекта. Каркас читает его при старте и вставляет содержимое в системный промпт:

```typescript
import { existsSync, readFileSync } from "node:fs";
import { join } from "node:path";

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

### Что класть в AGENTS.md

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

> **📝 Комментарий переводчика:**
>
> *Этот файл — аналог `CONTRIBUTING.md` для людей, но для AI-агента. Он содержит информацию, которую каркас не может вывести из кода: какие команды использовать, как устроена архитектура, какой стиль принят, какие ошибки повторяются.*
>
> *Один и тот же каркас ведёт себя как React-агент в React-проекте и CLI-агент в CLI-проекте — потому что проект сам рассказал агенту, какой он.*

### Знакомый паттерн с разными именами

| Инструмент | Имя файла |
|-----------|-----------|
| Cursor | `.cursorrules` |
| Codex | `AGENTS.md` |
| Claude Code | `CLAUDE.md` |
| pi | своя конвенция |

Имя разное, паттерн один: **обнаружить markdown-файл → вставить как инструкции**.

---

## Верификация: честность, а не покрытие

### Проблема

Агент скажет «all tests pass», ни разу не запустив тесты. Это не злой умысел — модели обучены на данных, где «all tests pass» встречается часто, и подгоняют паттерн. Даже если добавить инструмент запуска тестов, модель будет **выдумывать результат**, если промпт не сопротивляется.

### Решение — контракт верификации в промпте

```typescript
sections.push(`# Verification
After making changes, verify your work:
1. Run \`npx tsc --noEmit\` when TypeScript is present
2. Run lint, test, or build commands only if they exist in this project
   and are allowed by the current approval mode
3. Report exactly what you ran, what was blocked, and what was unavailable
4. Do NOT inflate partial verification into a blanket success claim

Do NOT claim "tests pass" without running them.
Scope your claims honestly. "Verification was limited because writes were blocked"
is honest. "All tests pass" when you didn't run them is not.`);
```

> **📝 Комментарий переводчика:**
>
> *«После внесения изменений проверь свою работу:*
> *1. Запусти `npx tsc --noEmit`, если проект использует TypeScript*
> *2. Запускай lint, тесты или сборку только если они существуют в этом проекте и разрешены текущим режимом одобрения*
> *3. Отчитывайся точно: что запускал, что было заблокировано, что недоступно*
> *4. Не раздувай частичную проверку в заявление о полном успехе*
>
> *Не утверждай «тесты прошли», не запуская их.*
> *Ограничивай свои заявления честно. «Проверка была ограничена, потому что запись была заблокирована» — честно. «Все тесты прошли», когда ты их не запускал — нет.»*

### Scoped claims — что агент может сказать vs что нужно

| Может сказать (конфабуляция) | Нужно (честный отчёт) |
|---|---|
| «All tests pass» | «Ran npm test: 47 passed, 3 failed (pre-existing, unrelated to my change)» |
| «I fixed the bug» | «Fixed null check in auth.ts:42. npx tsc --noEmit passes. Tests blocked by approval.» |
| «The build works» | «Ran npm run build: succeeded in 4.2s, no warnings.» |

### Маркер конфабуляции

Следите за мягкими фразами: **«should be fine»**, **«looks good to me»**, **«I expect this to work»**. Это маркеры. Модель, реально запустившая проверку, использует **прошедшее время и конкретный результат**. Не запустившая — уклончивое будущее время.

---

## Human-in-the-loop: ищи → спрашивай → действуй

### Проблема

Агенты по умолчанию не задают вопросов. Модели, обученные на данных разработчиков, предпочитают «просто разобраться сами». Даже если добавить инструмент `askUser`, модель будет игнорировать его без явной инструкции.

### Решение — два компонента

**1. Инструмент `askUser`:**
```typescript
const askUser = tool({
  description: `Ask the user a multiple-choice question.
WHEN TO USE: scoping ambiguous tasks, choosing between approaches,
  resolving a missing detail before acting.
WHEN NOT TO USE: you already have enough context to proceed.
DO NOT USE FOR: rhetorical questions or progress updates.`,
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

> **📝 Комментарий переводчика:**
>
> *«Задай пользователю вопрос с выбором из нескольких вариантов.*
> *КОГДА ИСПОЛЬЗОВАТЬ: уточнение неоднозначных задач, выбор между подходами, восполнение недостающей детали перед действием.*
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: у тебя достаточно контекста для продолжения.*
> *НЕ ИСПОЛЬЗОВАТЬ ДЛЯ: риторических вопросов или обновлений о прогрессе.»*

**2. Протокол в промпте:**
```typescript
sections.push(`# Handling Ambiguity
When the task is ambiguous or has multiple valid approaches:
1. Search the code or docs to gather context first
2. Use askUser to let the user choose. Do NOT guess.
3. Examples: "add auth" -> ask OAuth or JWT; "set up a db" -> ask Postgres or SQLite

Specific tasks (with file paths, line numbers, or precise instructions) do not
need askUser. Act directly.`);
```

> **📝 Комментарий переводчика:**
>
> *«Когда задача неоднозначна или имеет несколько подходов:*
> *1. Сначала поищи в коде или документации, чтобы собрать контекст*
> *2. Используй askUser, чтобы пользователь выбрал. НЕ угадывай.*
> *3. Примеры: «добавь авторизацию» → спроси OAuth или JWT; «настрой базу» → спроси Postgres или SQLite*
>
> *Конкретные задачи (с путями к файлам, номерами строк или точными инструкциями) не требуют askUser. Действуй напрямую.»*

### Проверка

Запустите неоднозначную задачу («Add authentication to this project») — агент должен вызвать `askUser`. Конкретную задачу («Add a null check at line 42 of src/auth.ts») — агент должен действовать напрямую.

---

## Жизненный цикл облачной песочницы

Облачная песочница — это не «работает» или «остановлен». Она перемещается через **4 состояния**:

```
provisioning → active → hibernating → hibernated
                  ↑             |
                  +-------------+  (restore)
```

| Состояние | Что происходит | Стоимость |
|-----------|---------------|-----------|
| **Provisioning** | VM запускается, зависимости устанавливаются | Биллинг начался |
| **Active** | Агент работает, команды выполняются | Полная стоимость в минуту |
| **Hibernating** | Снимок в процессе, песочница завершает работу | Полная стоимость |
| **Hibernated** | VM остановлена, снимок сохранён | Только хранение |

### Математика стоимости

- **Без жизненного цикла:** песочница работает до жёсткого срока (4 часа) = 4ч × $0.02/мин = **$4.80**
- **С гибернизацией по неактивности (5 мин):** 25 мин × $0.02/мин = **$0.50**

### 5 проблем из продакшна

1. **Устаревшие хэндлы после переподключения** — хэндл переживает отключение, но сессия внутри него — нет. Команды уходят в никуда. *Исправление:* проверять хэндл командой `echo probe`, при неудаче — восстанавливать из снимка.

2. **Устаревшие данные о сроке** — песочница сообщает `expiresAt` при создании. Если кэшировать это значение, проверяешь устаревшие данные. *Исправление:* всегда получать свежие данные от провайдера.

3. **Опрос сбрасывает неактивность** — если lifecycle-workflow опрашивает статус каждые 30 секунд и опрос считается активностью, окно неактивности никогда не закрывается. *Исправление:* трекер активности учитывает только пользовательские события (сообщения, вызовы инструментов, изменения файлов).

4. **Петли автоматического возобновления** — пользователь переподключается → песочница возобновляется → lifecycle проверяет → нет активности → гибернизация → следующее переподключение → бесконечный цикл. *Исправление:* автоматическое возобновление только при первом входе.

5. **Расхождение состояния** — состояние живёт в трёх местах: API провайдера, ваша БД, локальный кэш клиента. Они расходятся. *Исправление:* API провайдера — единственный источник истины.

---

## Стэк курса

| Компонент | Назначение |
|-----------|-----------|
| **AI SDK** (Vercel) | `ToolLoopAgent`, `tool()`, `stepCountIs`, `pruneMessages`, streaming |
| **AI Gateway** | Роутинг моделей строкой `"anthropic/claude-haiku-4-5"` |
| **Vercel Sandbox** | Удалённая ВМ с изолированной ФС, git, npm |
| **just-bash** | In-memory виртуальная ФС с copy-on-write |
| **Vercel Workflow** | Долговечные процессы для жизненного цикла песочницы |
| **Zod v3** | Схемы входа инструментов (v4 ломает типы AI SDK v6) |
| **TypeScript** | Язык, async/await, базовый опыт терминала |

---

## Ссылки

- [Оригинальный курс Vercel Academy](https://vercel.com/academy/build-ai-agent-harness)
- [AI SDK документация](https://sdk.vercel.ai/docs)
- [just-bash](https://github.com/nichochar/just-bash)

---

*Перевод и адаптация: [galaersh-ai](https://github.com/galaersh-ai). Оригинальный контент — Vercel Academy, Joel Hooks.*

---

## Все 38 уроков

Полные переводы-адаптации всех уроков курса с кодом и переводами промтов: **[docs/README.md](docs/README.md)**

