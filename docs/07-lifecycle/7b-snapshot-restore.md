# Урок 7b: Snapshot и restore — заморозка и восстановление

Гибернация была бы бесполезна, если бы после неё терялось состояние. Механизм, который сохраняет состояние песочницы, — **snapshot** (снимок файловой системы). А восстановление — **restore**, которое создаёт новую ВМ из снимка. Этот урок — о том, как устроены снимки и какие проблемы идемпотентности они порождают.

---

## Проблема

Песочница уходит в гибернацию, ВМ останавливается. Но пользователь через час возвращается и ждёт **то же самое** окружение: те же файлы, те же установленные зависимости, тот же прогресс работы. Значит, перед остановкой нужно «заморозить» файловую систему, а при возврате — развернуть её заново.

Наивная реализация «вызвал `snapshot()` → получил id → при возврате вызвал `restore(id)`» ломается в трёх местах, и все три — про **идемпотентность**: что будет, если операцию вызвали дважды, или в неправильный момент, или когда она уже в процессе.

---

## Решение с кодом

### snapshot() замораживает ФС, restore() создаёт новую ВМ

```typescript
interface Sandbox {
  // ...readFile, exec, stop
  snapshot?(): Promise<{ snapshotId: string }>;
}

// snapshot: заморозить текущее состояние файловой системы
async function snapshot(sandbox: CloudSandbox): Promise<string> {
  const snap = await sandbox.vm.snapshot();
  return snap.id;
}

// restore: из снимка создать НОВУЮ ВМ (не ту же самую!)
async function restore(snapshotId: string, config: SandboxConfig): Promise<CloudSandbox> {
  const vm = await VercelSandbox.create({ ...config, snapshotId });
  return wrapCloudSandbox(vm);
}
```

Ключевой момент: **`restore()` не «размораживает» старую ВМ — он создаёт новую** из сохранённого снимка. Старая ВМ уже уничтожена; снимок — это образ, из которого поднимается свежий экземпляр.

### Что сохраняется и не сохраняется в снимке

| Сохраняется | Не сохраняется |
|---|---|
| Файловая система (все файлы проекта) | Работающие процессы (снимок — это не пауза ВМ) |
| Установленные зависимости (`node_modules`) | Состояние сетевых соединений |
| Конфигурация окружения | In-memory состояние запущенных программ |
| git-репозиторий и рабочее дерево | PID-ы, хэндлы, дескрипторы |

Снимок — это **слепок файловой системы**, а не «замороженный процесс». Поэтому всё, что живёт только в памяти (запущенный dev-сервер, незакоммиченный ход программы), после restore потеряется. Зато файлы — на месте.

### Три проблемы идемпотентности

#### Проблема 1: снимок в процессе → кэшировать Promise

`snapshot()` — долгая операция. Если гибернация запущена дважды (например, два параллельных триггера), обе вызовут `snapshot()` и получат два разных снимка — или сломают друг друга. Решение: **кэшировать Promise** — первый вызов создаёт его, все последующие получают тот же.

```typescript
let snapshotPromise: Promise<string> | null = null;

async function getSnapshot(sandbox: CloudSandbox): Promise<string> {
  if (!snapshotPromise) {
    // Создаём Promise один раз. Пока он не разрешился,
    // все повторные вызовы получают тот же самый.
    snapshotPromise = sandbox.vm.snapshot().then((s) => s.id);
  }
  return snapshotPromise;
}
```

#### Проблема 2: песочница уже запущена → проверять активную

Пользователь вернулся, `restore` поднял новую ВМ. Но если пришёл **ещё один** `restore` (повторный запрос, ретрай), он поднимет **вторую** ВМ из того же снимка — и теперь две ВМ борются за внимание, а одна из них «утекла». Решение: перед `restore` проверять, нет ли уже активной песочницы.

```typescript
let activeSandbox: CloudSandbox | null = null;

async function ensureSandbox(snapshotId: string, config: SandboxConfig) {
  // Уже есть активная песочница? Не поднимаем вторую.
  if (activeSandbox && await isAlive(activeSandbox)) {
    return activeSandbox;
  }
  activeSandbox = await restore(snapshotId, config);
  return activeSandbox;
}
```

#### Проблема 3: двойная остановка → булевый флаг

`stop()` тоже небезопасно вызывать дважды: вторая остановка уже остановленной ВМ может бросить ошибку «not found» или, хуже, остановить **новую** ВМ, поднятую между вызовами. Решение: булевый флаг, который делает `stop()` идемпотентным.

```typescript
let stopped = false;

async function stopOnce(sandbox: CloudSandbox): Promise<void> {
  if (stopped) return;   // уже останавливали — выходим
  stopped = true;
  await sandbox.vm.close();
}
```

### Полная картина: жизненный цикл с защитой от гонок

```typescript
class SandboxLifecycle {
  private snapshotPromise: Promise<string> | null = null;
  private activeSandbox: CloudSandbox | null = null;
  private stopped = false;

  constructor(private config: SandboxConfig) {}

  // Гибернация: ровно один снимок, сколько бы раз ни вызвали
  async hibernate(): Promise<string> {
    if (!this.snapshotPromise) {
      this.snapshotPromise = this.activeSandbox!.vm.snapshot().then((s) => s.id);
    }
    const id = await this.snapshotPromise;
    await this.stopOnce();
    return id;
  }

  // Восстановление: ровно одна активная ВМ
  async ensureActive(snapshotId: string): Promise<CloudSandbox> {
    if (this.activeSandbox && await this.isAlive(this.activeSandbox)) {
      return this.activeSandbox;
    }
    this.activeSandbox = await restore(snapshotId, this.config);
    this.snapshotPromise = null;   // сброс — следующая гибернация сделает новый снимок
    this.stopped = false;
    return this.activeSandbox;
  }

  private async stopOnce(): Promise<void> {
    if (this.stopped) return;
    this.stopped = true;
    await this.activeSandbox?.vm.close();
  }

  private async isAlive(s: CloudSandbox): Promise<boolean> {
    // echo probe — см. урок 7d про устаревшие хэндлы
    const { exitCode } = await s.exec("echo probe");
    return exitCode === 0;
  }
}
```

---

## Ключевые выводы

- **`snapshot()` замораживает ФС, `restore()` создаёт новую ВМ.** Снимок — это образ, а не пауза; из него поднимается свежий экземпляр.
- **Снимок сохраняет файлы и зависимости, но не процессы.** Всё in-memory (запущенные серверы, PID-ы) теряется.
- **Идемпотентность — три ловушки:**
  1. Снимок в процессе → кэшировать Promise.
  2. Песочница уже запущена → проверять активную перед `restore`.
  3. Двойная остановка → булевый флаг в `stopOnce`.
- **Сброс флагов при восстановлении** (`snapshotPromise = null`, `stopped = false`) — иначе следующий цикл гибернации/restore сломается.

---

## Попробуй сам

1. Вызови `hibernate()` дважды параллельно и убедись, что `snapshot()` выполнился один раз (Promise закэширован).
2. Вызови `ensureActive()` дважды подряд и проверь, что создалась только одна ВМ.
3. Вызови `stopOnce()` дважды и убедись, что `close()` сработал один раз.
4. Сделай снимок, создай файл, затем `restore` — и проверь, что файла нет (снимок — это слепок на момент снятия).
5. Напиши юнит-тесты на все три идемпотентные ловушки, мокая `vm.snapshot` / `vm.close` счётчиками вызовов.
