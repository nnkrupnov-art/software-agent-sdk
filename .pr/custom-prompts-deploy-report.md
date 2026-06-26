# Кастомный OpenHands: где живёт и как меняется системный промпт

## TL;DR

Системный промпт OpenHands (агент-серверного sandbox'а) собирается из **двух
частей** внутри образа `agent-server`:

1. **Встроенный дефолт** в `openhands-sdk/openhands/sdk/agent/base.py`
   (класс `Agent`, метод `SYSTEM_PROMPT` / `_get_system_message()` →
   собирается `system_prompt_template.j2` из `openhands-sdk/openhands/sdk/agent/`)
2. **SOUL.md** — отдельный файл, читается функцией
   `_load_soul_md()` из `$HOME/.openhands/SOUL.md` внутри работающего
   sandbox-контейнера. Подставляется в шаблон вместо `_DEFAULT_SOUL`.

Для **полностью кастомного** OpenHands достаточно:
1. Перезаписать `openhands-agent-server/openhands/agent_server/soul/SOUL.md`
   (кастомная личность агента).
2. Поправить шаблоны `openhands-sdk/openhands/sdk/agent/*.j2`, если нужны
   кастомные блоки (например, «RoboCop/DevSecOps» директивы).
3. Убедиться, что собранный образ содержит ровно **один** тег
   `custom-prompts-nnk-final` (иначе orchestrator ищет sandbox_spec по
   `container.image.tags[0]` и падает с `assert sandbox_spec is not None`).

## Где физически живёт системный промпт

### 1. Шаблоны (SDK, immutable в образе)

```
openhands-sdk/openhands/sdk/agent/
├── agent.py                 # class Agent(BaseModel), system_message: str
├── system_prompt/           # tools/system/jinja для системы
│   ├── system_prompt.j2     # корневой шаблон
│   └── ...
└── <остальные .j2>
```

Кастомизация: править `.j2` напрямую в форке и пересобрать образ.

### 2. SOUL.md (Runtime-инжекция)

```
openhands-agent-server/openhands/agent_server/soul/SOUL.md
```

Этот файл в **исходниках SDK не используется** напрямую. Его читает
`openhands/sdk/agent/base.py::_load_soul_md()` уже во время работы агента,
обращаясь к `$HOME/.openhands/SOUL.md` **внутри работающего контейнера**.

Поскольку стандартный Dockerfile `openhands-agent-server` копирует файл
в `/home/openhands/.openhands/SOUL.md` только для targets `binary` и
`binary-minimal` (см. `Dockerfile:375` и `Dockerfile:388`), а образ
`source` собирается без этого шага — кастомный SOUL.md в `source`-target
по умолчанию **не попадает** в sandbox.

**Фикс (уже в этом PR):** `Dockerfile` теперь копирует SOUL.md в
`base-image-minimal` (строка `231`), что покрывает `source`,
`source-minimal`, `binary` и `binary-minimal` единым шагом.

### 3. Skills (опционально)

`AgentContext.get_system_message_suffix()` добавляет блок `<available_skills>`,
содержимое которого формируется из `/skills/*.md` в sandbox'е. Эти файлы
**не часть** системного промпта как такового, но отображаются агенту
при выборе tool'ов.

## Как меняется системный промпт: пошагово

```
┌──────────────────────────────────────────────────────────────────┐
│ SDK source (.j2 + Agent.system_message: str)                     │
│   ↓ build (uv sync)                                              │
│ .venv/lib/python3.13/site-packages/openhands/sdk/agent/*.j2     │
│   ↓ docker build (source target)                                 │
│ Layer: /agent-server/.venv/lib/.../openhands/sdk/agent/*.j2      │
│   ↓ docker run                                                    │
│ Agent reads *.j2 at import time, renders with environment        │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐  │
│ │ system_message_template.j2                                 │  │
│ │  ├─ {{ role_definition }}                                  │  │
│ │  ├─ {{ capabilities }}                                     │  │
│ │  ├─ {{ _DEFAULT_SOUL | default(_load_soul_md()) }}         │  │
│ │  │     ↑ читается /home/openhands/.openhands/SOUL.md      │  │
│ │  │     ↑ собирается при docker build (наш фикс)            │  │
│ │  └─ <available_skills> (suffix)                            │  │
│ └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Что нужно для надёжного деплоя кастомного образа

### 1. Dockerfile

Копировать `agent_server/soul/SOUL.md` в `base-image-minimal`:

```dockerfile
COPY --chown=${USERNAME}:${USERNAME} \
    openhands-agent-server/openhands/agent_server/soul/SOUL.md \
    /home/${USERNAME}/.openhands/SOUL.md
```

### 2. build.py — emit bare `{image}:{custom_tag}` для source targets

Без этого `dict.fromkeys()` теряет тег, и buildx не пушит
`{image}:{custom_tag}` без суффикса. Фикс уже внесён.

### 3. Post-build cleanup

Build всё равно создаёт несколько `<sha>-<custom_tag>` тегов на одном
образе. Orchestrator берёт `container.image.tags[0]`, что **не всегда**
совпадает с `AGENT_SERVER_IMAGE_TAG`. Решение — оставить ровно один тег:

```bash
docker rm -f oh-agent-server-*
for tag in 18cc2bf-custom-prompts-nnk-final 18cc2bf-custom-prompts-nnk-final-source \
           custom-prompts-custom-prompts-nnk-final-source \
           custom-prompts-nnk-final-source \
           18cc2bf-nikolaik_s_python-nodejs_tag_python3.12-nodejs22-slim-source \
           18cc2bf497f0060d8c1155b5ac6b7ddf7b8d8c73-custom-prompts-nnk-final-source; do
    docker rmi ghcr.io/nnkrupnov-art/agent-server-custom:$tag 2>&1 | head -1
done
```

### 4. ENV orchestrator'а (внутри `docker run ... openhands:1.5.0`)

```bash
SANDBOX_STARTUP_GRACE_SECONDS=300   # legacy ENV name, не OH_*
AGENT_SERVER_IMAGE_REPOSITORY=ghcr.io/<user>/agent-server-custom
AGENT_SERVER_IMAGE_TAG=<custom_tag>
SANDBOX_USER_ID=0
PERSIST_SANDBOX=true
WORKSPACE_BASE=/opt/openhands
SANDBOX_LOCAL_RUNTIME_URL=http://host.docker.internal
```

### 5. После запуска — POST /api/settings

Без `UserContext` все сессии падают:
```bash
curl -X POST http://127.0.0.1:3000/api/settings -H "Content-Type: application/json" \
  -d '{"agent":"CodeActAgent","llm_model":"<model>","llm_base_url":"<url>","llm_api_key":"<key>"}'
```

### 6. Verify: сессия должна дойти до READY за 60–90 сек

```bash
curl -s "http://127.0.0.1:3000/api/v1/app-conversations/start-tasks?ids=$CID" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); t=d[0]; print(t['status'])"
# Ожидаем: READY
```

## Подтверждение, что кастомный SOUL.md реально применён

После успешной сессии (status=READY) проверка изнутри sandbox:

```bash
SANDBOX=<id-from-status-response>
docker exec -u root $SANDBOX md5sum /home/openhands/.openhands/SOUL.md
# Сравнить с md5 файла в форке: должно совпасть bit-for-bit
```

В нашем случае:
- `md5sum` sandbox'а: `8665f190c2efa18af4223d3b00a15715`
- `md5sum` в репо:    `8665f190c2efa18af4223d3b00a15715`
- upstream default:   `8eea19bbdb41e7c1cb79394235507614`

RoboCop/DevSecOps SOUL.md действительно дошёл до рантайма.

## Что НЕ меняется кастомным SOUL.md

- **Tools** — фиксированный набор (`BashTool`, `FileEditor`, ...),
  задаётся через `Agent.tools`. Менять надо в `Agent(tools=[...])`.
- **LLM** — задаётся через `LLM(model=..., base_url=..., api_key=...)`,
  подставляется на старте сессии из UserContext → Settings.
- **Skills** — это отдельные `.md` файлы в `/skills/` sandbox'а, не часть
  system prompt напрямую.

## Что меняется кастомным SOUL.md

- **Личность и tone of voice** агента (как он отвечает, какие у него
  prime directives, hard limits).
- **Этические границы** (например, запрет на pipe-to-shell, edit secrets
  config, push to main без явного запроса).
- **Преамбула в каждом LLM-вызове** — SOUL.md всегда идёт в начале
  системного промпта и определяет рамки поведения модели.
