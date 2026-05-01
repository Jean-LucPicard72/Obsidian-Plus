# Claude Best Practices

> Полное руководство по продуктивной работе с Claude Code и Claude Agents.
> Собрано из официальной документации Anthropic, Хабра и опыта сообщества (2026).

---

## Оглавление

1. [[#Главное ограничение — контекстное окно]]
2. [[#CLAUDE.md — фундамент всего]]
3. [[#Как правильно ставить задачи]]
4. [[#Workflow: план → реализация → проверка]]
5. [[#Управление сессией]]
6. [[#Skills — повторяемые инструкции]]
7. [[#Subagents — изолированные помощники]]
8. [[#Hooks — автоматические действия]]
9. [[#MCP — подключение внешних инструментов]]
10. [[#Параллельные сессии и автоматизация]]
11. [[#Claude Agent SDK]]
12. [[#Частые ошибки]]
13. [[#Шпаргалка по командам]]

---

## Главное ограничение — контекстное окно

Всё, что нужно знать о Claude Code, строится вокруг одного факта: **контекстное окно конечно, и качество работы падает по мере его заполнения**.

В контекст попадает всё: каждое сообщение, каждый прочитанный файл, каждый вывод команды. Одна сессия отладки может сжечь десятки тысяч токенов.

```
Заполненность контекста → Claude «забывает» ранние инструкции → больше ошибок
```

**Как с этим работать:**

- Используй `/clear` между несвязанными задачами
- Делегируй исследование файлов subagents — они работают в отдельном окне
- Следи за заполненностью через statusline (`/statusline`)
- Компактизируй вручную: `/compact Focus on API changes`
- Не копи ошибки — если Claude дважды не понял задачу, запусти `/clear` и переформулируй

---

## CLAUDE.md — фундамент всего

`CLAUDE.md` — это файл, который Claude читает в начале **каждой** сессии. Это постоянная память проекта.

### Как создать

```bash
# Claude сам проанализирует проект и создаст базовый файл
/init
```

### Где размещать

| Место | Область действия | Нужно в git |
|---|---|---|
| `~/.claude/CLAUDE.md` | Все проекты на машине | Нет |
| `./CLAUDE.md` | Текущий проект (для команды) | Да |
| `./CLAUDE.local.md` | Текущий проект (личное) | Нет, в .gitignore |
| `./src/CLAUDE.md` | Конкретная папка | Да |

CLAUDE.md в родительских и дочерних директориях подгружаются автоматически — удобно для монорепозиториев.

### Что включать, а что нет

| ✅ Включать | ❌ Не включать |
|---|---|
| Команды сборки/тестирования | То, что Claude видит из кода |
| Правила стиля кода | Стандартные конвенции языка |
| Архитектурные решения | Подробная документация API (дай ссылку) |
| Особенности окружения (env vars) | Описание каждого файла |
| Нюансы репозитория (ветки, PR) | Часто меняющаяся информация |
| Нестандартные паттерны проекта | «Пиши чистый код» и очевидное |

### Правило одной строки

Для каждой строки спроси: *«Если убрать это — Claude сделает ошибку?»*
Если нет — удали.

**Переполненный CLAUDE.md хуже, чем пустой** — важные правила теряются в шуме.

### Пример хорошего CLAUDE.md

```markdown
# Build & Test
- Build: `npm run build`
- Tests: `npm test -- --testPathPattern=<file>` (не весь suite)
- Typecheck: `npm run typecheck`

# Code style
- ES modules (import/export), не CommonJS
- Деструктуризация импортов где возможно
- Без лишних комментариев

# Workflow
- Ветки: `feature/`, `fix/`, `chore/`
- PR через `gh pr create`

# Важно
- Auth — только через src/auth/session.ts
- Env vars документированы в .env.example
```

### Импорты внутри CLAUDE.md

```markdown
# CLAUDE.md
See @README.md for project overview.

# Git workflow
@docs/git-guide.md

# Personal overrides
@~/.claude/my-preferences.md
```

### Усиление инструкций

Если Claude игнорирует правило — добавь `IMPORTANT:` или `YOU MUST`:

```markdown
IMPORTANT: Никогда не изменяй файлы в папке /migrations без явного разрешения.
```

---

## Как правильно ставить задачи

### Принцип точности

Расплывчатый запрос → расплывчатый результат. Чем конкретнее условие — тем меньше правок.

| Плохо | Хорошо |
|---|---|
| `добавь тесты для foo.py` | `напиши тест для foo.py: случай когда юзер не авторизован, без моков` |
| `почини баг логина` | `пользователи говорят, что логин падает после истечения сессии. смотри src/auth/, особенно token refresh. напиши падающий тест, потом починй` |
| `сделай дашборд красивее` | `[вставь скриншот] реализуй этот дизайн. сделай скриншот результата и сравни с оригиналом. перечисли отличия и исправь` |
| `добавь calendar widget` | `посмотри HotDogWidget.php как пример паттерна. реализуй calendar widget по тому же паттерну без новых библиотек` |

### Давай Claude способ проверить себя

Это самый важный совет. Без критерия успеха Claude выдаёт что-то «похожее на правильное».

```
❌ "реализуй валидацию email"
✅ "напиши validateEmail. тест-кейсы: user@example.com → true, invalid → false, 
    user@.com → false. запусти тесты после реализации"
```

### Богатый контекст

- **`@filename`** — ссылка на файл (Claude прочитает перед ответом)
- **Вставляй скриншоты** — просто Ctrl+V в промпт
- **Пайп данных**: `cat error.log | claude`
- **Давай URL** документации — Claude умеет их читать
- **Разреши Claude самому** найти нужное: `"используй git log, чтобы понять историю этого файла"`

### Интервью вместо ТЗ

Для крупных задач попроси Claude взять у тебя интервью:

```
Я хочу сделать [краткое описание]. Возьми у меня подробное интервью 
через AskUserQuestion. Спрашивай про техническую реализацию, UX, 
edge cases и трейдоффы. Не задавай очевидное — копай в сложные части.
После — запиши спецификацию в SPEC.md.
```

Затем **начни новую сессию** для реализации с чистым контекстом.

---

## Workflow: план → реализация → проверка

### Четыре фазы

#### 1. Исследование (Plan Mode)

Нажми `Ctrl+Shift+H` или введи `/plan` для входа в Plan Mode.
Claude читает файлы и отвечает на вопросы **без каких-либо изменений**.

```
[Plan Mode] прочитай /src/auth и разберись, как мы работаем 
с сессиями и refresh токенами
```

#### 2. Планирование

```
[Plan Mode] хочу добавить Google OAuth. Какие файлы нужно изменить? 
Какой flow сессии? Создай подробный план.
```

`Ctrl+G` — открыть план в редакторе для правки перед реализацией.

#### 3. Реализация

```
[Normal Mode] реализуй OAuth flow по плану. напиши тесты для callback 
handler, запусти suite и исправь падения
```

#### 4. Коммит

```
[Normal Mode] сделай commit с описательным сообщением и открой PR
```

### Когда план нужен, а когда нет

| Нужен план | Без плана |
|---|---|
| Меняешь несколько файлов | Поправить опечатку |
| Незнакомый код | Добавить лог-строку |
| Архитектурное изменение | Переименовать переменную |
| Неясен подход | Очевидный однострочный фикс |

---

## Управление сессией

### Ключевые команды

| Команда | Что делает |
|---|---|
| `/clear` | Сбросить контекст (новая сессия) |
| `/compact` | Автоматически сжать историю |
| `/compact <инструкции>` | Сжать с акцентом: `/compact Focus on API changes` |
| `/rewind` | Открыть меню восстановления состояния |
| `Esc` | Остановить Claude (контекст сохраняется) |
| `Esc + Esc` | Открыть меню rewind |
| `/rename` | Дать сессии имя для поиска |
| `/btw` | Боковой вопрос (ответ не попадает в контекст) |

### Возобновление сессий

```bash
claude --continue   # продолжить последнюю сессию
claude --resume     # выбрать из недавних
```

Называй сессии осмысленно через `/rename oauth-migration` — они как ветки.

### Правило двух правок

Если Claude дважды не понял одно и то же → `/clear` и переформулируй задачу с учётом того, что узнал.
Длинная сессия с накопленными ошибками хуже, чем чистая сессия с лучшим промптом.

### Компактизация

Claude автоматически сжимает историю при приближении к лимиту.
Управляй этим в CLAUDE.md:

```markdown
When compacting: always preserve the full list of modified files 
and any test commands that were used.
```

### Checkpoints

Claude автоматически делает checkpoint перед каждым изменением.
Через `/rewind` можно восстановить:
- только разговор
- только код
- и то, и другое

> ⚠️ Checkpoints не заменяют git — они не отслеживают внешние изменения.

---

## Skills — повторяемые инструкции

Skills — это Markdown-файлы, которые учат Claude выполнять конкретные задачи. В отличие от CLAUDE.md (всегда загружен), Skills загружаются по требованию.

### Создание

```
.claude/skills/fix-issue/SKILL.md
```

```markdown
---
name: fix-issue
description: Fix a GitHub issue end-to-end
disable-model-invocation: true
---
Исправь GitHub issue: $ARGUMENTS

1. `gh issue view $ARGUMENTS` — получи детали
2. Изучи описание проблемы
3. Найди релевантные файлы
4. Реализуй исправление
5. Напиши и запусти тесты
6. Проверь lint и typecheck
7. Создай коммит с описанием
8. Запуши и открой PR
```

Вызов: `/fix-issue 1234`

`disable-model-invocation: true` — для задач с побочными эффектами (запускать только вручную).

### Skills vs CLAUDE.md

| | CLAUDE.md | Skills |
|---|---|---|
| Когда грузится | Всегда | По требованию |
| Содержимое | Правила и контекст | Инструкции и workflow |
| Вызов | Автоматически | `/skill-name` или по контексту |
| Для чего | Постоянные правила | Повторяемые задачи |

### Примеры полезных Skills

```markdown
# .claude/skills/pr-review/SKILL.md
---
name: pr-review
description: Review a PR for quality, security and conventions
---
Проведи code review PR #$ARGUMENTS:
1. `gh pr view $ARGUMENTS --json files,body` — получи изменения
2. Проверь соответствие code style
3. Найди потенциальные security issues
4. Проверь покрытие тестами
5. Оставь конкретные комментарии с примерами исправлений
```

---

## Subagents — изолированные помощники

Subagents решают главную проблему: **исследование кода засоряет контекст**.
Subagent работает в отдельном окне и возвращает только summary.

### Встроенные subagents

| Агент | Модель | Для чего |
|---|---|---|
| **Explore** | Haiku (быстрый) | Поиск и анализ кода (read-only) |
| **Plan** | Наследует | Исследование перед планированием |
| **general-purpose** | Наследует | Сложные многошаговые задачи |

### Создать свой subagent

```bash
# Через интерфейс
/agents

# Или вручную создать файл
.claude/agents/security-reviewer.md
```

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: opus
---
Ты — senior security engineer. При review ищи:
- Injection (SQL, XSS, command injection)
- Auth и authorization flaws
- Секреты в коде
- Небезопасная работа с данными

Давай конкретные ссылки на строки и варианты исправления.
```

### Приоритет загрузки

```
Managed settings (org) > --agents CLI > .claude/agents/ > ~/.claude/agents/ > Plugins
```

### Когда использовать

```
"используй subagent, чтобы исследовать как работает наш auth"
"используй subagent для проверки кода на edge cases"
```

Subagent читает сотни файлов — всё это не попадает в твой основной контекст.

### Subagents vs Agent Teams

| | Subagents | Agent Teams |
|---|---|---|
| Работа | В рамках одной сессии | Параллельно в отдельных сессиях |
| Общение | Через главного агента | Через shared tasks и сообщения |
| Для чего | Изолировать боковые задачи | Параллельная разработка |

### Полезный паттерн: Writer + Reviewer

```
Сессия A (Writer):               Сессия B (Reviewer):
"реализуй rate limiter"    →
                           →     "проревью @src/middleware/rateLimiter.ts.
                                  ищи race conditions и edge cases"
"исправь по feedback: ..."  ←
```

---

## Hooks — автоматические действия

Hooks — это shell-команды, HTTP-запросы или LLM-промпты, которые запускаются **автоматически** в определённые моменты. В отличие от инструкций в CLAUDE.md, hooks — детерминированы: они **всегда** выполнятся.

### Конфигурация

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/lint.sh"
          }
        ]
      }
    ]
  }
}
```

### Все события

| Событие | Когда срабатывает |
|---|---|
| `SessionStart` | Начало/возобновление сессии |
| `SessionEnd` | Конец сессии |
| `PreToolUse` | **До** выполнения инструмента (может заблокировать) |
| `PostToolUse` | После успешного выполнения |
| `UserPromptSubmit` | Когда ты отправляешь промпт |
| `Stop` | Когда Claude закончил отвечать |
| `PreCompact` / `PostCompact` | До/после компактизации |
| `SubagentStart` / `SubagentStop` | Запуск/остановка subagent |
| `FileChanged` | Изменился файл на диске |

### Типы hooks

| Тип | Когда использовать |
|---|---|
| `command` | Гибкость, shell-скрипты |
| `http` | Внешние сервисы валидации |
| `mcp_tool` | Инструменты подключённых MCP-серверов |
| `prompt` | Нужна оценка Claude для решения |
| `agent` | Сложная многошаговая проверка |

### Практические примеры

#### Автолинтинг после редактирования

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "cd \"$CLAUDE_PROJECT_DIR\" && npm run lint --fix" }]
      }
    ]
  }
}
```

#### Блокировка опасных команд

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "./.claude/hooks/block-dangerous.sh" }]
      }
    ]
  }
}
```

```bash
# block-dangerous.sh
COMMAND=$(jq -r '.tool_input.command')
if echo "$COMMAND" | grep -qE 'rm -rf|DROP TABLE|git push --force'; then
  jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "Опасная команда заблокирована"}}'
fi
```

#### Контекст при старте сессии

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [{ "type": "command", "command": "./.claude/hooks/load-context.sh" }]
      }
    ]
  }
}
```

```bash
# load-context.sh  
echo "Branch: $(git branch --show-current)"
echo "Changes: $(git status --short | head -10)"
```

### Правило: hooks vs инструкции

```
Hooks  → то, что должно выполняться ВСЕГДА без исключений
CLAUDE.md → рекомендации и контекст
```

Если Claude регулярно «забывает» что-то делать — это hook, не инструкция.

---

## MCP — подключение внешних инструментов

MCP (Model Context Protocol) позволяет Claude работать с внешними системами: базами данных, браузерами, API и т.д.

### Подключение

```bash
# Добавить MCP сервер
claude mcp add

# Или вручную в settings.json
```

```json
// .claude/settings.json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    },
    "postgres": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

### С чего начать

| MCP-сервер | Для чего |
|---|---|
| Playwright | Тестирование UI, проверка вёрстки |
| PostgreSQL/MySQL | Прямые запросы к БД, анализ схемы |
| GitHub | Issues, PR без `gh` CLI |
| Figma | Дизайн → код |
| Slack | Читать баг-репорты из тредов |

### MCP vs Skills vs CLI

```
MCP    → внешние сервисы и данные
Skills → повторяемые внутренние workflows
CLI    → инструменты командной строки (gh, aws, gcloud)
```

Для CLI: просто скажи Claude использовать их напрямую.

```
"используй gh issue list, чтобы найти задачи с меткой bug"
"используй aws cloudwatch, чтобы посмотреть логи последних ошибок"
```

---

## Параллельные сессии и автоматизация

### Параллельные сессии

Запуск 10–15 сессий одновременно (5 в терминале + остальные в веб) — главный способ ускориться.

Каждая сессия — свой git worktree, изменения не конфликтуют.

```bash
# Десктопное приложение
# Manage > New Session — каждая в изолированном worktree

# В терминале — несколько вкладок с разными задачами
```

### Non-interactive режим

```bash
# Одиночный запрос
claude -p "объясни что делает этот проект"

# Структурированный вывод для скриптов
claude -p "перечисли все API endpoints" --output-format json

# Потоковый вывод
claude -p "проанализируй этот лог" --output-format stream-json
```

### Массовые операции (Fan-out)

```bash
# Миграция 2000 файлов параллельно
for file in $(cat files.txt); do
  claude -p "мигрируй $file с React на Vue. верни OK или FAIL." \
    --allowedTools "Edit,Bash(git commit *)" &
done
wait
```

`--allowedTools` ограничивает права при автономной работе.

### Auto mode

```bash
# Classifier-модель проверяет команды в фоне
# Блокирует: расширение scope, неизвестную инфраструктуру, hostile-content
claude --permission-mode auto -p "исправь все lint ошибки"
```

### Message queuing

Можно отправить несколько задач подряд — Claude выполнит их по очереди.
Уходишь заниматься своими делами, возвращаешься к готовому результату.

---

## Claude Agent SDK

Agent SDK — это библиотека (Python/TypeScript) для создания AI-агентов, которые используют те же инструменты, что и Claude Code.

### Когда использовать

| | Claude Code CLI | Agent SDK |
|---|---|---|
| Интерактивная разработка | ✅ | — |
| CI/CD пайплайны | — | ✅ |
| Кастомные приложения | — | ✅ |
| Разовые задачи | ✅ | — |
| Production автоматизация | — | ✅ |

### Установка

```bash
# Python
pip install claude-agent-sdk

# TypeScript
npm install @anthropic-ai/claude-agent-sdk
```

```bash
export ANTHROPIC_API_KEY=your-api-key
```

### Базовый агент

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="найди и исправь баг в auth.py",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"]),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "найди и исправь баг в auth.ts",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

### Встроенные инструменты

| Tool | Что делает |
|---|---|
| `Read` | Читать файлы |
| `Write` | Создавать файлы |
| `Edit` | Точечно редактировать |
| `Bash` | Выполнять команды |
| `Glob` | Искать файлы по паттерну |
| `Grep` | Поиск по содержимому |
| `WebSearch` | Поиск в вебе |
| `WebFetch` | Загрузить страницу |
| `AskUserQuestion` | Задать уточняющий вопрос |
| `Monitor` | Следить за фоновым процессом |

### Hooks в SDK

```python
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher
from datetime import datetime

async def log_file_change(input_data, tool_use_id, context):
    file_path = input_data.get("tool_input", {}).get("file_path", "?")
    with open("./audit.log", "a") as f:
        f.write(f"{datetime.now()}: изменён {file_path}\n")
    return {}

async def main():
    async for message in query(
        prompt="отрефактори utils.py",
        options=ClaudeAgentOptions(
            permission_mode="acceptEdits",
            hooks={
                "PostToolUse": [
                    HookMatcher(matcher="Edit|Write", hooks=[log_file_change])
                ]
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)
```

### Subagents в SDK

```python
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition

async def main():
    async for message in query(
        prompt="используй code-reviewer для проверки кодовой базы",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    description="Эксперт по code review: качество и безопасность.",
                    prompt="Анализируй качество кода и предлагай улучшения.",
                    tools=["Read", "Glob", "Grep"],
                )
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)
```

### Сессии и возобновление

```python
session_id = None

# Первый запрос — сохраняем session_id
async for message in query(
    prompt="прочитай auth модуль",
    options=ClaudeAgentOptions(allowed_tools=["Read", "Glob"]),
):
    if isinstance(message, SystemMessage) and message.subtype == "init":
        session_id = message.data["session_id"]

# Второй запрос — продолжаем с полным контекстом
async for message in query(
    prompt="теперь найди все места где он вызывается",
    options=ClaudeAgentOptions(resume=session_id),
):
    if isinstance(message, ResultMessage):
        print(message.result)
```

### Agent SDK vs Managed Agents vs Client SDK

| | Client SDK | Agent SDK | Managed Agents |
|---|---|---|---|
| Что это | API напрямую | Библиотека с агент-лупом | Hosted REST API |
| Tool loop | Ты пишешь сам | Claude делает сам | Claude делает сам |
| Инфраструктура | Твоя | Твоя | Anthropic |
| Для чего | Кастомный tool loop | Агенты на своей инфре | Production без своего сервера |

---

## Частые ошибки

### «Кухонный таз» (kitchen sink session)

Начал с одной задачи, переключился на другую, вернулся к первой.
Контекст забит нерелевантным мусором.

> **Решение**: `/clear` между несвязанными задачами

---

### Бесконечные правки

Claude ошибся → ты поправил → он снова ошибся → ты снова поправил.
Контекст полон неудачных попыток, что только мешает.

> **Решение**: После двух безуспешных правок — `/clear` и новый, более точный промпт

---

### Раздутый CLAUDE.md

500+ строк правил — Claude игнорирует половину.
Важное тонет в очевидном.

> **Решение**: Безжалостно чисти. Если правило не предотвращает реальную ошибку — удали. Рутинные действия — в hooks.

---

### Отсутствие верификации

Claude написал код, который «выглядит правильно», но не проверен.
Ты не узнаешь об ошибке до прода.

> **Решение**: Всегда давай способ проверить — тесты, скрипты, скриншоты.

---

### Бесконечное исследование

«Исследуй кодовую базу» без ограничений → Claude читает 300 файлов → контекст переполнен.

> **Решение**: Ограничивай scope или используй subagent для исследования.

---

### Доверие без проверки

Claude помечает задачу как выполненную, не протестировав end-to-end.

> **Решение**: Явно проси тестировать: `"протестируй как реальный пользователь через браузер"`. Или используй Playwright MCP.

---

## Шпаргалка по командам

### Основные команды

```bash
# Запуск
claude                          # интерактивный режим
claude -p "задача"              # одиночный запрос
claude --continue               # продолжить последнюю сессию
claude --resume                 # выбрать из последних сессий
claude --permission-mode auto   # авто-режим без прерываний
```

### В интерактивном режиме

```
/init          → создать CLAUDE.md
/clear         → сбросить контекст
/compact       → сжать историю
/plan          → Plan Mode (только чтение, без изменений)
/agents        → управление subagents
/hooks         → просмотр настроенных hooks
/skills        → список доступных skills
/rewind        → восстановить предыдущее состояние
/rename        → дать имя сессии
/btw           → боковой вопрос без загрязнения контекста
/permissions   → управление правами
/statusline    → настройка статус-строки
Esc            → остановить Claude
Esc + Esc      → меню rewind
Ctrl+Shift+H   → Plan Mode
```

### Структура проекта

```
project/
├── CLAUDE.md                  # правила для команды
├── CLAUDE.local.md            # личные правила (в .gitignore)
└── .claude/
    ├── settings.json          # hooks, MCP, permissions
    ├── settings.local.json    # личные настройки
    ├── agents/
    │   └── security-reviewer.md
    └── skills/
        ├── fix-issue/
        │   └── SKILL.md
        └── pr-review/
            └── SKILL.md
```

### Что куда

```
Всегда нужно     → CLAUDE.md
Периодично       → Skills (/skill-name)
Автоматически    → Hooks
Внешние сервисы  → MCP
Изолированно     → Subagents
В CI/CD          → Agent SDK
```

---

## Источники

- [Best Practices — официальная документация](https://code.claude.com/docs/en/best-practices)
- [Agent SDK Overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Hooks Reference](https://code.claude.com/docs/en/hooks)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code: маршрут обучения — Хабр](https://habr.com/ru/articles/983214/)
- [Claude Code в 2026: гайд для тех, кто пишет руками — Хабр](https://habr.com/ru/articles/987382/)
- [3000+ часов в Claude Code: три плагина — Хабр](https://habr.com/ru/articles/1017110/)
- [Claude Code лучшие практики агентного программирования — Хабр](https://habr.com/ru/companies/surfstudio/articles/943108/)
- [50 Claude Code Tips — builder.io](https://www.builder.io/blog/claude-code-tips-best-practices)
- [7 Best Practices from Real Projects — eesel.ai](https://www.eesel.ai/blog/claude-code-best-practices)
- [Building agents with Claude Agent SDK — Anthropic Engineering](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
