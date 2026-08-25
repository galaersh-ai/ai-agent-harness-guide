# Урок 10a: CLI-вход — точка входа в агента

Агент — это функция. Но пользователю нужен способ её запустить: с флагами, с обработкой прерываний, с аккуратным завершением. CLI-вход — это тонкий слой, который превращает «библиотеку агента» в «программу, которую можно запустить из терминала». Тонкий, но с парой подводных камней, о которых этот урок.

---

## Проблема

Наивный вход: жёстко зашитые параметры и отсутствие обработки завершения.

```typescript
// НАИВНО: параметры зашиты, Ctrl+C не обрабатывается, утечка песочницы
const sandbox = createLocalSandbox(process.cwd());
const agent = new ToolLoopAgent({ /* ... */ });
await agent.generate({ prompt: "..." });
// если пользователь нажмёт Ctrl+C, песочница останется висеть
```

Проблемы:

1. **Параметры зашиты** — нельзя выбрать модель или песочницу без правки кода.
2. **Ctrl+C не обрабатывается** — песочница (особенно облачная, поминутная) продолжает жечь деньги после выхода.
3. **Нет гарантии очистки** — при любом выходе (нормальном или аварийном) ресурсы должны освобождаться.

---

## Решение с кодом

### parseArgs из node:util

Вместо ручного разбора `process.argv` — встроенный `parseArgs`:

```typescript
import { parseArgs } from "node:util";

const { values } = parseArgs({
  options: {
    sandbox: { type: "string", default: "local" },   // --sandbox local|cloud
    model:   { type: "string", default: "anthropic/claude-haiku-4-5" },
    prompt:  { type: "string", default: "" },
  },
});

const sandboxType = values.sandbox as string;
const model = values.model as string;
const prompt = values.prompt as string;
```

`parseArgs` даёт типизированные опции с дефолтами и валидацией — не нужно писать свой парсер и разбирать `--flag=value` / `--flag value`.

### Флаги --sandbox и --model

```typescript
// Переключатель песочницы: local | cloud (из env SANDBOX или флага)
const sandboxType = process.env.SANDBOX || values.sandbox;

const sandbox =
  sandboxType === "cloud"
    ? await createCloudSandbox({ /* ... */ })
    : createLocalSandbox(process.cwd());

// Модель — тоже настраивается, не зашивается
const agent = new ToolLoopAgent({
  model,
  instructions: buildSystemPrompt({
    workingDirectory: sandbox.workingDirectory,
    sandboxType,
    toolNames: Object.keys(tools),
  }),
  tools,
  stopWhen: stepCountIs(20),
});
```

### SIGINT → sandbox.stop() → process.exit(0)

Обработка прерывания — обязательная часть CLI. При `Ctrl+C` нужно остановить песочницу и выйти чисто:

```typescript
let shuttingDown = false;

process.on("SIGINT", async () => {
  if (shuttingDown) return;   // защита от повторного Ctrl+C
  shuttingDown = true;
  console.error("\nInterrupted — stopping sandbox...");
  await sandbox.stop();       // освобождаем ресурсы (важно для облака!)
  process.exit(0);
});
```

### finally блок

Независимо от того, как завершился прогон — успехом, ошибкой или прерыванием — песочница должна остановиться. `finally` гарантирует это:

```typescript
async function main() {
  const sandbox = sandboxType === "cloud"
    ? await createCloudSandbox({})
    : createLocalSandbox(process.cwd());

  try {
    const agent = buildAgent(sandbox, model);
    const result = await agent.generate({ prompt });
    console.log(result.text);
  } catch (error) {
    console.error("Agent failed:", error);
    process.exitCode = 1;
  } finally {
    await sandbox.stop();    // сработает и при успехе, и при ошибке, и при исключении
  }
}

main();
```

### Полный CLI-вход

```typescript
import { parseArgs } from "node:util";
import { ToolLoopAgent, stepCountIs } from "ai";

const { values } = parseArgs({
  options: {
    sandbox: { type: "string", default: "local" },
    model:   { type: "string", default: "anthropic/claude-haiku-4-5" },
    prompt:  { type: "string", default: "" },
  },
});

async function main() {
  const sandboxType = process.env.SANDBOX || (values.sandbox as string);
  const model = values.model as string;
  const prompt = values.prompt as string;

  const sandbox = sandboxType === "cloud"
    ? await createCloudSandbox({})
    : createLocalSandbox(process.cwd());

  let shuttingDown = false;
  process.on("SIGINT", async () => {
    if (shuttingDown) return;
    shuttingDown = true;
    console.error("\nInterrupted — stopping sandbox...");
    await sandbox.stop();
    process.exit(0);
  });

  try {
    const agent = new ToolLoopAgent({
      model,
      instructions: buildSystemPrompt({
        workingDirectory: sandbox.workingDirectory,
        sandboxType,
        toolNames: Object.keys(tools),
      }),
      tools,
      stopWhen: stepCountIs(20),
    });
    const result = await agent.generate({ prompt });
    console.log(result.text);
  } catch (error) {
    console.error("Agent failed:", error);
    process.exitCode = 1;
  } finally {
    await sandbox.stop();
  }
}

main();
```

---

## Ключевые выводы

- **`parseArgs` из `node:util`** — встроенный парсер флагов с типами и дефолтами; не пиши свой.
- **`--sandbox` и `--model`** — настройка через флаги, а не через правку кода. Модель и песочница не зашиваются.
- **SIGINT → `sandbox.stop()` → `process.exit(0)`** — чистое завершение по Ctrl+C; без этого облачная песочница продолжит жечь деньги.
- **`finally` блок** — гарантия очистки при любом исходе (успех, ошибка, исключение).
- **Флаг `shuttingDown`** — защита от повторного Ctrl+C, чтобы `stop()` не вызвался дважды (см. урок 7b про идемпотентность).

---

## Попробуй сам

1. Запусти CLI с `--model anthropic/claude-sonnet-4-6 --prompt "..."` и убедись, что модель и промпт подхватились из флагов.
2. Переключи песочницу через `--sandbox cloud` (или env `SANDBOX=cloud`) и проверь, что используется облачный бэкенд.
3. Нажми Ctrl+C во время работы и убедись, что `sandbox.stop()` вызвался и процесс вышел с кодом 0.
4. Нажми Ctrl+C **дважды** подряд и проверь, что `shuttingDown`-флаг предотвратил двойной `stop()`.
5. Намеренно вызови ошибку в `agent.generate` и убедись, что `finally` всё равно остановил песочницу, а `exitCode = 1`.
