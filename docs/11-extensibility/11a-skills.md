# Урок 11a: Skills — прогрессивное раскрытие знаний

Специализированные знания — «как деплоить через Docker», «как настроить CI», «как писать API-тесты» — хочется вставить в системный промпт. Но каждый навык — это тысячи токенов, за которые ты платишь при **каждом** вызове модели. Skills решают это через **прогрессивное раскрытие**: в промпте живут только имена и описания, а полное содержимое загружается по запросу.

---

## Проблема

Захардкодить все навыки в системный промпт — наивно и дорого:

- **1–2 навыка** — это ещё нормально, промпт остаётся компактным.
- **5 навыков** — промпт уже 15 000+ токенов.
- Платишь за все 15 000 токенов **при каждом** вызове модели, даже если агенту нужен **один** навык.

Масштаб не тот: специализированных знаний много (десятки), а нужных в конкретной сессии — одно-два. Держать всё в промпте — значит платить за то, что не используется.

---

## Решение с кодом

**Прогрессивное раскрытие (progressive disclosure):** в промпте — имена и однострочные описания (дёшево), полный текст — на диске, загружается через инструмент `loadSkill` только когда нужен.

### Структура на диске: skills/name/SKILL.md с YAML-фронтматтером

```
skills/
  docker-deploy/
    SKILL.md          ← YAML-фронтматтер (name + description), затем полный текст
  ci-setup/
    SKILL.md
  api-testing/
    SKILL.md
```

Каждый `SKILL.md` начинается с YAML-фронтматтера:

```markdown
---
name: docker-deploy
description: Deploy containers with Docker Compose
---

# Docker Deploy

Полный текст навыка: команды, шаги, ловушки, примеры…
```

### discoverSkills() — парсит фронтматтер, дедупликация по имени

```typescript
import { existsSync, readdirSync, readFileSync } from "node:fs";
import { join } from "node:path";

interface Skill {
  name: string;
  description: string;
  path: string;
}

function parseFrontmatter(content: string): { description?: string } {
  // Наивный парсер: берём блок между первой и второй строкой "---"
  const match = content.match(/^---\n([\s\S]*?)\n---/);
  if (!match) return {};
  const description = match[1]
    .split("\n")
    .find((line) => line.startsWith("description:"))
    ?.replace(/^description:\s*/, "")
    .trim();
  return { description };
}

function discoverSkills(dirs: string[]): Skill[] {
  const skills: Skill[] = [];
  const seen = new Set<string>();

  for (const dir of dirs) {
    if (!existsSync(dir)) continue;
    for (const entry of readdirSync(dir)) {
      const path = join(dir, entry, "SKILL.md");
      if (existsSync(path) && !seen.has(entry)) {
        seen.add(entry);                          // дедупликация: первый каталог побеждает
        const content = readFileSync(path, "utf-8");
        const { description } = parseFrontmatter(content);
        skills.push({ name: entry, description: description ?? "", path });
      }
    }
  }

  return skills;
}
```

Дедупликация через `seen` важна: если навык `docker-deploy` есть и в глобальном каталоге, и в проектном, **первый** (проектный, если он идёт раньше в `dirs`) побеждает. Проектные навыки переопределяют глобальные.

### В системном промпте — только имена

```typescript
function buildSkillsSection(skills: Skill[]): string {
  return [
    "# Skills",
    ...skills.map((s) => `- ${s.name}: ${s.description}`),
  ].join("\n");
}
```

Результат в промпте — компактный список:

```
# Skills
- docker-deploy: Deploy containers with Docker Compose
- ci-setup: Configure GitHub Actions CI pipeline
- api-testing: Write and run API integration tests
```

### loadSkill — полное содержимое по запросу

Когда агенту нужен полный текст — он вызывает `loadSkill`:

```typescript
import { tool } from "ai";
import { z } from "zod";

function createLoadSkillTool(skills: Skill[]) {
  return tool({
    description: `Load the full content of a skill by name.
WHEN TO USE: when a task matches a listed skill and you need its detailed steps.
WHEN NOT TO USE: you only need the one-line summary already in the prompt.`,
    inputSchema: z.object({
      name: z.string().describe("Skill name from the # Skills list"),
    }),
    execute: async ({ name }) => {
      const skill = skills.find((s) => s.name === name);
      if (!skill) return `Skill "${name}" not found. Available: ${skills.map((s) => s.name).join(", ")}`;
      return readFileSync(skill.path, "utf-8");
    },
  });
}
```

> **📝 Перевод промта (описание loadSkill):**
>
> *«Загрузи полное содержимое навыка по имени.*
> *КОГДА ИСПОЛЬЗОВАТЬ: когда задача совпадает с навыком из списка и нужны его подробные шаги.*
> *КОГДА НЕ ИСПОЛЬЗОВАТЬ: тебе достаточно однострочного описания, уже данного в промпте.»*

### Экономика прогрессивного раскрытия

| Подход | Токены в промпте | Когда платишь |
|---|---|---|
| Всё в промпте | 15 000+ | при каждом вызове |
| Skills (имена + описания) | ~200 | при каждом вызове |
| Полный навык | 2 000–5 000 | **только** когда вызван `loadSkill` |

Имена и описания — это «оглавление», полный текст — «глава». Платишь за оглавление всегда (оно дёшево), за главу — только когда реально открываешь.

---

## Ключевые выводы

- **Прогрессивное раскрытие:** имена + описания в промпте (дёшево), полный текст на диске (по запросу).
- **Структура `skills/name/SKILL.md`** с YAML-фронтматтером (`name` + `description`), затем полный текст.
- **`discoverSkills()`** парсит фронтматтер и **дедуплицирует по имени**: первый каталог побеждает (проектные навыки переопределяют глобальные).
- **`loadSkill`** возвращает полное содержимое по запросу — платишь за главу, только когда открываешь.
- **Экономика:** оглавление (~200 токенов) вместо всего текста (15 000+) при каждом вызове.

---

## Попробуй сам

1. Создай три навыка (`docker-deploy`, `ci-setup`, `api-testing`) с фронтматтером и полным текстом. Прогони `discoverSkills` и убедись, что вернулись имена, описания и пути.
2. Проверь дедупликацию: помести `docker-deploy` в два каталога и убедись, что в результате он один (первый каталог).
3. Проверь, что в системном промпте — только `# Skills` с именами, а не полный текст.
4. Вызови `loadSkill("docker-deploy")` и убедись, что вернулся полный текст; вызови с несуществующим именем — получи список доступных.
5. Замерь: сравни количество токенов промпта с навыками в виде списка и с навыками, зашитыми целиком.
