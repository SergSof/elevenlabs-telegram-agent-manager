# ElevenLabs Telegram Agent Manager — краткий отчет

## 1. Цель проекта

Реализован Telegram-бот на n8n для управления ElevenLabs voice agents.

Бот позволяет пользователю:

* просматривать список своих ElevenLabs voice agents;
* выбирать агента для управления;
* обновлять prompt выбранного агента;
* обновлять welcome message / first message;
* обновлять knowledge base/content выбранного агента.

Решение соответствует тестовому заданию: используется n8n, MySQL, Telegram Bot API и ElevenLabs REST API. В ТЗ также требуется, чтобы пользователь мог видеть и менять только своих агентов, и эта проверка реализована через связку `telegram_users → voice_agents → bot_sessions`. 

---

## 2. Основная логика workflow

Workflow начинается с Telegram Trigger.

Дальше входящий Telegram update нормализуется в единую структуру:

```text
telegram_user_id
username
chat_id
text
callback_data
update_type
```

После этого workflow:

1. регистрирует или обновляет Telegram-пользователя в MySQL;
2. создает bot session, если её ещё нет;
3. получает текущую session пользователя;
4. показывает список доступных пользователю voice agents;
5. сохраняет выбранного агента в `bot_sessions.selected_voice_agent_id`;
6. переводит session в нужное состояние:

   * `waiting_prompt`
   * `waiting_welcome_message`
   * `waiting_knowledge_base`
7. принимает следующий текст от пользователя;
8. отправляет соответствующий PATCH/POST-запрос в ElevenLabs API;
9. сбрасывает состояние обратно в `idle`;
10. отправляет пользователю сообщение об успешном обновлении.

В workflow реализованы отдельные ветки для prompt, welcome message и knowledge base/content. Ветка Knowledge Base создает text document в ElevenLabs, сохраняет `document_id` в MySQL и привязывает документ к выбранному агенту. 

---

## 3. Реализованные функции

### Agent selection

Пользователь отправляет `/start`, после чего бот показывает список доступных ему агентов.

Список агентов берется из MySQL только для текущего Telegram user.

### Update Prompt

Пользователь выбирает агента, нажимает `Update Prompt` и отправляет новый prompt.

Workflow отправляет PATCH-запрос в ElevenLabs:

```text
PATCH /v1/convai/agents/{agent_id}
```

Обновляется:

```json
conversation_config.agent.prompt.prompt
```

### Update Welcome Message

Пользователь нажимает `Update Welcome Message` и отправляет новый welcome message.

Workflow отправляет PATCH-запрос в ElevenLabs:

```text
PATCH /v1/convai/agents/{agent_id}
```

Обновляется:

```json
conversation_config.agent.first_message
```

### Update Knowledge Base / Content

Пользователь нажимает `Update Knowledge Base` и отправляет текст.

Workflow:

1. создает text document в ElevenLabs Knowledge Base;
2. сохраняет `elevenlabs_document_id` в MySQL;
3. обновляет выбранного агента, привязывая к нему новый KB document.

---

## 4. Security / ownership

В workflow реализована проверка ownership.

Пользователь не может получить список всех агентов. Он видит только записи из `voice_agents`, где:

```sql
voice_agents.user_id = telegram_users.id
```

При выборе агента используется проверка:

```sql
JOIN telegram_users tu ON tu.id = va.user_id
WHERE tu.telegram_user_id = current_telegram_user_id
  AND va.id = selected_agent_id
  AND va.is_active = TRUE
```

При обновлении prompt, welcome message и knowledge base агент также выбирается через текущую session и текущего Telegram user. Это предотвращает доступ к чужим агентам.

---

## 5. Token security

Telegram Bot Token и ElevenLabs API Key не хранятся напрямую в workflow JSON.

В workflow используются environment variables:

```text
TELEGRAM_BOT_TOKEN
ELEVENLABS_API_KEY
```

Примеры использования:

```js
$env["TELEGRAM_BOT_TOKEN"]
$env["ELEVENLABS_API_KEY"]
```

Это позволяет экспортировать workflow без раскрытия реальных токенов.

---

## 6. MySQL schema

```sql
CREATE TABLE telegram_users (
  id BIGINT NOT NULL AUTO_INCREMENT,
  telegram_user_id BIGINT NOT NULL,
  username VARCHAR(255) NULL,
  first_name VARCHAR(255) NULL,
  last_name VARCHAR(255) NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_telegram_user_id (telegram_user_id)
);
```

```sql
CREATE TABLE voice_agents (
  id BIGINT NOT NULL AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  elevenlabs_agent_id VARCHAR(255) NOT NULL,
  agent_name VARCHAR(255) NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_voice_agents_user_id (user_id),
  CONSTRAINT fk_voice_agents_user
    FOREIGN KEY (user_id) REFERENCES telegram_users(id)
    ON DELETE CASCADE
);
```

```sql
CREATE TABLE bot_sessions (
  id BIGINT NOT NULL AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  selected_voice_agent_id BIGINT NULL,
  state VARCHAR(100) NOT NULL DEFAULT 'idle',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_bot_sessions_user_id (user_id),
  KEY idx_bot_sessions_selected_agent (selected_voice_agent_id),
  CONSTRAINT fk_bot_sessions_user
    FOREIGN KEY (user_id) REFERENCES telegram_users(id)
    ON DELETE CASCADE,
  CONSTRAINT fk_bot_sessions_voice_agent
    FOREIGN KEY (selected_voice_agent_id) REFERENCES voice_agents(id)
    ON DELETE SET NULL
);
```

```sql
CREATE TABLE knowledge_base_documents (
  id BIGINT NOT NULL AUTO_INCREMENT,
  voice_agent_id BIGINT NOT NULL,
  elevenlabs_document_id VARCHAR(255) NOT NULL,
  document_name VARCHAR(255) DEFAULT NULL,
  source_type VARCHAR(50) NOT NULL DEFAULT 'text',
  created_at TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_kb_voice_agent_id (voice_agent_id),
  CONSTRAINT fk_kb_voice_agent
    FOREIGN KEY (voice_agent_id) REFERENCES voice_agents(id)
    ON DELETE CASCADE
);
```

```sql
CREATE TABLE agent_change_log (
  id BIGINT NOT NULL AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  voice_agent_id BIGINT NULL,
  change_type VARCHAR(100) NOT NULL,
  old_value TEXT NULL,
  new_value TEXT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_agent_change_log_user_id (user_id),
  KEY idx_agent_change_log_voice_agent_id (voice_agent_id),
  CONSTRAINT fk_agent_change_log_user
    FOREIGN KEY (user_id) REFERENCES telegram_users(id)
    ON DELETE CASCADE,
  CONSTRAINT fk_agent_change_log_voice_agent
    FOREIGN KEY (voice_agent_id) REFERENCES voice_agents(id)
    ON DELETE SET NULL
);
```

---

## 7. Known limitation

В сообщениях бота есть текст:

```text
Send /cancel to cancel this action.
```

На текущем этапе отдельная ветка `/cancel` не реализована.

В production-версии нужно сделать одно из двух:

1. убрать текст про `/cancel`;
2. или добавить отдельную обработку `/cancel`, которая сбрасывает `bot_sessions.state` в `idle`.

---

## 8. Что реализовано

1. exported n8n workflow JSON;
2. MySQL schema;
3. Telegram bot workflow, работающий через Telegram Bot API;
4. интеграция с ElevenLabs REST API;
5. логика ограничения доступа к агентам по Telegram user.
