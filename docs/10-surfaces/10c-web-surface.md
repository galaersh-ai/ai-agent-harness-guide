# Урок 10c: Web-поверхность — агент не знает о вебе

CLI — это одна поверхность. Web — другая. Но агент **не должен знать** ни о той, ни о другой. Агент — headless (безголовый): он принимает промпт и отдаёт поток событий. CLI и web — это просто **разные стратегии рендеринга** одного и того же потока. Этот урок — о том, как держать агента чистым и подключать к нему web-интерфейс.

---

## Проблема

Соблазн — написать web-специфичный агент: зашить в него `res.write`, знание об HTTP, о том, что «пользователь в браузере». Это превращает агента в монолит, привязанный к одному транспорту. Захочешь второй интерфейс (CLI, Telegram, API) — придётся переписывать.

Хуже того: web добавляет две проблемы, которых нет в CLI — **персистентность** (сообщения должны переживать перезагрузку страницы) и **стриминг через сеть** (события нужно доставлять в браузер в реальном времени).

---

## Решение с кодом

### Агент headless: принимает промпт, отдаёт поток

Агент ничего не знает о транспорте. Он работает с промптом и событиями:

```typescript
// Агент НЕ знает о вебе. Никаких res.write, никакого HTTP.
// Только промпт на вход и поток событий на выход.
async function* runAgent(agent: ToolLoopAgent, prompt: string) {
  const stream = await agent.stream({ prompt });
  for await (const part of stream.fullStream) {
    yield part;   // отдаём события наружу — кому, решает вызывающий
  }
}
```

CLI и web получают **один и тот же** `runAgent`:

```typescript
// CLI-рендеринг: события → stdout/stderr
for await (const part of runAgent(agent, prompt)) {
  if (part.type === "text-delta") process.stdout.write(part.text);
  if (part.type === "tool-call") console.error(`🔧 ${part.toolName}`);
}

// Web-рендеринг: события → SSE-поток в браузер
for await (const part of runAgent(agent, prompt)) {
  res.write(`data: ${JSON.stringify(part)}\n\n`);
}
```

### Таблица сравнения поверхностей

| | CLI | Web |
|---|---|---|
| Рендеринг текста | `process.stdout.write` | `res.write` (SSE) |
| Рендеринг инструментов | stderr + эмодзи | JSON-события → UI-компоненты |
| Персистентность | Не нужна (терминал не перезагружается) | Нужна (страница перезагружается) |
| Стриминг | Прямой в терминал | Через сеть (SSE/WebSocket) |
| Знание агента о транспорте | Нет | Нет |

### Персистентность: db.saveMessages / loadMessages

В CLI сообщения живут в памяти — терминал закрылся, разговор пропал, и это ок. В web страница перезагружается, и пользователь ждёт, что история **останется**. Нужен слой хранения:

```typescript
interface MessageStore {
  saveMessages(sessionId: string, messages: Message[]): Promise<void>;
  loadMessages(sessionId: string): Promise<Message[]>;
}

// Простейшая реализация на любой БД (здесь — концептуально)
const db: MessageStore = {
  async saveMessages(sessionId, messages) {
    await database.upsert("messages", { sessionId, messages });
  },
  async loadMessages(sessionId) {
    return (await database.get("messages", { sessionId }))?.messages ?? [];
  },
};

// При старте web-сессии — восстанавливаем историю
async function handleWebSession(sessionId: string, prompt: string) {
  const history = await db.loadMessages(sessionId);
  const agent = buildAgent({ history });   // агент стартует с прошлым контекстом
  // ... стримим ответ, и в конце:
  await db.saveMessages(sessionId, await agent.getMessages());
}
```

### SSE-стриминг

Доставка событий в браузер в реальном времени — через **Server-Sent Events** (SSE): однонаправленный поток `data: {...}\n\n`:

```typescript
import { createServer } from "node:http";

const server = createServer(async (req, res) => {
  if (req.url !== "/stream") return;

  // Заголовки SSE: поток текста, без буферизации
  res.writeHead(200, {
    "Content-Type": "text/event-stream",
    "Cache-Control": "no-cache",
    Connection: "keep-alive",
  });

  const prompt = await readRequestBody(req);
  const agent = buildAgent({ history: await db.loadMessages("default") });

  for await (const part of runAgent(agent, prompt)) {
    // Каждое событие — отдельный SSE-фрейм
    res.write(`data: ${JSON.stringify(part)}\n\n`);
  }

  res.write("data: [DONE]\n\n");
  res.end();
  await db.saveMessages("default", await agent.getMessages());
});
```

Браузер читает поток через `EventSource` и рендерит каждый фрейм как UI-событие (текст — в чат, tool-call — в панель активности).

### Правило: агент НЕ знает о вебе

Вся web-специфика (SSE-заголовки, `res.write`, `db`) живёт **вне** агента — в слое рендеринга. Сам агент оперирует только промптом, историей и потоком событий:

```
┌─────────────────────────────────────────────┐
│              Агент (headless)               │
│   prompt → runAgent → поток событий         │
│   НЕ знает: HTTP, SSE, stdout, БД           │
└──────────────────┬──────────────────────────┘
                   │  поток событий (общий для всех)
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────┐     ┌───────────────┐
│  CLI-рендерер │     │ Web-рендерер  │
│  → stdout     │     │  → SSE/UI     │
└───────────────┘     └───────────────┘
```

Если нарушить это правило и вшить `res.write` в агента, CLI-поверхность сломается (нет `res`), а агент станет непереиспользуемым. Разделение «агент vs рендерер» — это и есть архитектурная чистота, ради которой всё затевалось.

---

## Ключевые выводы

- **Агент headless** — принимает промпт, отдаёт поток событий. Ничего не знает о транспорте.
- **CLI и web — это стратегии рендеринга** одного потока, а не разные агенты.
- **Web добавляет две проблемы:** персистентность (`db.saveMessages`/`loadMessages`) и сетевой стриминг (SSE).
- **SSE** — однонаправленный поток `data: {...}\n\n` для доставки событий в браузер.
- **Правило: агент НЕ знает о вебе.** Вся web-специфика — в слое рендеринга, иначе CLI сломается, а агент станет монолитом.

---

## Попробуй сам

1. Вынеси логику агента в `runAgent` (генератор событий) и подключи **два** рендерера — CLI и web — к одному агенту. Убедись, что оба работают без правки агента.
2. Реализуй `db.saveMessages`/`loadMessages` на любой БД (или на файле) и проверь, что история переживает перезагрузку страницы.
3. Напиши SSE-эндпоинт и проверь его `curl -N`: события должны приходить по мере работы агента, а не одним куском.
4. Попробуй вшить `res.write` прямо в агента — и сразу увидишь, как CLI-поверхность ломается (нет `res`). Верни разделение.
5. Добавь третью поверхность (например, Telegram-бота или JSON API) — и убедись, что агент снова не потребовал изменений.
