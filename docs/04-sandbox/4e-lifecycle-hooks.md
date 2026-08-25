# Урок 4e: Хуки жизненного цикла песочницы

Песочница — это не только «выполнить команду». Вокруг неё есть **жизненный цикл**: что сделать сразу после старта, что — перед остановкой, что — при таймауте. Без этих хуков каждый разработчик изобретает их заново: настраивает `git config` вручную, забывает сделать снимок перед завершением, теряет несохранённые изменения при таймауте. В этом уроке мы оформим жизненный цикл в явный интерфейс `SandboxLifecycle`.

## Проблема

Представь типичную сессию облачного агента:

1. Песочница стартовала — но `git` не настроен (нет `user.name`/`user.email`), зависимости не установлены, `.env` не скопирован.
2. Агент поработал — но перед остановкой никто не проверил, остались ли незакоммиченные изменения.
3. Песочница получила таймаут — и всё несохранённое состояние просто исчезло.

Каждый из этих шагов — **повторяющийся сценарий**, который без структуры реализуется как попало: часть логики зашита в инструменты, часть — в код запуска, часть вообще забыта. Нужен один явный контракт, который описывает «что происходит на границах жизни песочницы».

## Решение с кодом

Интерфейс жизненного цикла с тремя опциональными хуками:

```typescript
interface SandboxLifecycle {
  afterStart?(sandbox: Sandbox): Promise<void>;
  beforeStop?(sandbox: Sandbox): Promise<void>;
  onTimeout?(sandbox: Sandbox): Promise<void>;
}
```

Все три хука **опциональны** (`?`) — реализация подключает только те, что ей нужны. Это честный контракт: песочница без таймаутов не обязана притворяться, что обрабатывает `onTimeout`.

### `afterStart` — подготовка окружения

Запускается сразу после создания песочницы. Здесь — всё, что нужно сделать один раз перед работой:

```typescript
const lifecycle: SandboxLifecycle = {
  afterStart: async (sandbox) => {
    await sandbox.exec('git config user.email "agent@example.com"');
    await sandbox.exec('git config user.name "Coding Agent"');
    await sandbox.exec("npm install");
    await sandbox.exec("cp .env.example .env");
  },
};
```

> **📝 Комментарий:**
>
> *После старта песочницы: настраиваем `git` (иначе коммиты падают с «Please tell me who you are»), устанавливаем зависимости (`npm install`), копируем `.env.example` в `.env` (секреты не храним в песочнице, а подкладываем шаблон).*

### `beforeStop` — зафиксировать состояние

Запускается перед остановкой. Здесь — всё, что нужно сохранить, чтобы не потерять работу:

```typescript
const lifecycle: SandboxLifecycle = {
  beforeStop: async (sandbox) => {
    const { stdout } = await sandbox.exec("git status --porcelain");
    if (stdout.trim()) {
      await sandbox.exec('git add -A && git commit -m "WIP: auto-save"');
    }
    if (sandbox.snapshot) {
      const { snapshotId } = await sandbox.snapshot();
      console.log(`Snapshot saved: ${snapshotId}`);
    }
  },
};
```

Три действия:

1. **`git status --porcelain`** — проверяем, есть ли незакоммиченные изменения (машиночитаемый формат: пусто = чисто).
2. **Автокоммит** — если изменения есть, сохраняем их как `WIP: auto-save`, чтобы не потерять.
3. **Снимок** — если песочница поддерживает `snapshot`, делаем снимок для последующего восстановления.

### `onTimeout` — реакция на истечение срока

Запускается, когда песочница исчерпала своё время (напомню: у облачной песочницы есть `expiresAt`). Здесь — аварийное сохранение:

```typescript
const lifecycle: SandboxLifecycle = {
  onTimeout: async (sandbox) => {
    await sandbox.exec('git add -A && git commit -m "WIP: timeout auto-save"');
    if (sandbox.snapshot) {
      await sandbox.snapshot();
    }
  },
};
```

Разница с `beforeStop`: `onTimeout` — **аварийный** сценарий. Времени мало, поэтому только самое критичное — закоммитить и сделать снимок, без лишних проверок.

### try/finally — гарантия, что `beforeStop` выполнится

Хуки бесполезны, если их не вызывать. Ключевой паттерн — `try/finally`: `afterStart` перед работой, `beforeStop` **всегда** после, даже если работа упала:

```typescript
async function runWithLifecycle(
  sandbox: Sandbox,
  lifecycle: SandboxLifecycle,
  task: () => Promise<void>,
) {
  if (lifecycle.afterStart) await lifecycle.afterStart(sandbox);

  try {
    await task();
  } finally {
    if (lifecycle.beforeStop) await lifecycle.beforeStop(sandbox);
    await sandbox.stop();
  }
}
```

Почему `finally` важен: если задача агента бросила исключение, без `finally` блок `beforeStop` не выполнится — и все изменения пропадут. `finally` гарантирует очистку **при любом исходе**: успех, ошибка, даже если `task` сама бросила.

А `onTimeout` вызывается отдельно — по событию истечения срока, а не в основном потоке:

```typescript
if (sandbox.expiresAt && Date.now() > sandbox.expiresAt) {
  if (lifecycle.onTimeout) await lifecycle.onTimeout(sandbox);
}
```

### Полная картина

```
[afterStart]  ── git config, npm install, cp .env
     │
     ▼
[работа агента]  (внутри try)
     │
     ▼
[beforeStop]  ── git status, автокоммит, snapshot   (в finally, всегда)
     │
     ▼
[sandbox.stop()]
     │
     └── или по таймауту: [onTimeout] ── аварийный коммит + snapshot
```

## Ключевые выводы

- **`SandboxLifecycle` = три опциональных хука:** `afterStart`, `beforeStop`, `onTimeout` — реализация подключает только нужные.
- **`afterStart`** — подготовка: `git config`, `npm install`, `cp .env.example .env`.
- **`beforeStop`** — фиксация: `git status` → автокоммит → `snapshot` (если поддерживается).
- **`onTimeout`** — аварийное сохранение при истечении срока, только критичное.
- **`try/finally` обязателен** — иначе `beforeStop` не выполнится при ошибке задачи, и работа пропадёт.
- **`snapshot` проверяется через `if (sandbox.snapshot)`** — опциональная возможность есть не у всех песочниц.

## Попробуй сам

1. Реализуй `afterStart`, который настраивает `git config` и делает `npm install`. Проверь, что после старта коммиты больше не падают.
2. Напиши `beforeStop` с автокоммитом и проверь: заверши сессию с несохранёнными изменениями — появился ли коммит `WIP: auto-save`.
3. Сломай `task` (брось исключение) и убедись, что `beforeStop` всё равно выполняется благодаря `finally`.
4. Добавь `onTimeout` и симулируй истечение срока — проверь, что аварийный коммит создаётся даже без нормального `beforeStop`.
