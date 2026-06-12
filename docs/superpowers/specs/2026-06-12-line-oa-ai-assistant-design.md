# Design Document: LINE OA AI Assistant with In-Chat Training

Date: 2026-06-12
Based on: PRD.md (same directory root)

---

## 1. Architecture Overview

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Language | PHP 8.4 | Per PRD, stable for production. Server runs PHP 8.4 via DirectAdmin |
| Framework | Vanilla PHP (no framework) | Single webhook endpoint, ~30 files total, no framework overhead |
| DI Container | Pimple (`pimple/pimple`) | Clean dependency wiring, auto-inject, lazy loading |
| Config Management | `vlucas/phpdotenv` + direct `$_ENV` in factory closures | No Config object wrapper needed — env vars are few and simple |
| HTTP Client | Guzzle (single instance for LINE SDK + Zen API) | LINE SDK v12 requires Guzzle. Zen API uses same Guzzle instance |
| Logger | Monolog (`monolog/monolog`) | Standard PHP logger, stream to `storage/logs/app.log` |
| LLM Provider | Zen API via `LLMProvider` interface | Strategy pattern. Zen implements `LLMProvider`. Model: `deepseek-v4-flash-free` |
| LLM Call | Text-in, text-out. No streaming, no tool calling, no vision | System only needs chat completion |
| Database | MySQL 8 via PDO (prepared statements only) | Per PRD — SQL injection prevention |
| Database Migration | Raw SQL files + PHP `migrate.php` runner + `migrations` tracking table | Schema changes are frequent; version-controlled SQL with tracking |
| LINE SDK | `linecorp/line-bot-sdk` v12.5 | Latest version. Uses Guzzle natively. Provides `EventRequestParser` for webhook signature verification + event deserialization |
| Admin Onboarding | First admin via `init <ADMIN_SETUP_CODE>` in LINE Chat | No DB insert needed. `ADMIN_SETUP_CODE` in `.env`. Remove from `.env` to disable |
| Admin Management | `/invite` generates 1-time code → recipient sends `accept <code>` | No need to know LINE User ID. 24h expiration |
| API Key Management | Separate `api_keys` table — primary + backups for auto failover | If primary key fails, try backup automatically |

### Dependencies

```json
{
  "require": {
    "php": ">=8.2",
    "vlucas/phpdotenv": "^5.6",
    "pimple/pimple": "^3.5",
    "monolog/monolog": "^3.0",
    "linecorp/line-bot-sdk": "^12.5"
  },
  "require-dev": {
    "phpunit/phpunit": "^11.0"
  },
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

---

## 2. Project Structure

```
botz.know-tify.com/
├── webhook.php              ← Entry point (LINE Webhook) — อยู่ที่ root
├── .htaccess                ← Block direct access to src/, vendor/, .env etc.
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_faq.sql
│   ├── 003_create_rules.sql
│   ├── 004_create_chat_logs.sql
│   ├── 005_create_bot_settings.sql
│   ├── 006_create_training_logs.sql
│   ├── 007_create_invite_codes.sql
│   ├── 008_create_api_keys.sql
│   └── 009_create_eval_criteria.sql
├── migrate.php              ← Migration runner
├── src/
│   ├── Container.php        ← Pimple wrapper / service registration
│   ├── Contracts/
│   │   └── LLMProvider.php
│   ├── Controllers/
│   │   └── WebhookController.php
│   ├── Services/
│   │   ├── LineService.php       ← LINE Messaging API via Guzzle
│   │   ├── LLMService.php        ← Prompt builder + LLM call + conversation summary
│   │   ├── UserService.php       ← User findOrCreate + admin management
│   │   ├── FaqService.php        ← FAQ CRUD
│   │   ├── RuleService.php       ← Rule matching
│   │   ├── ChatService.php       ← Chat logging
│   │   ├── CommandService.php    ← Admin command parser
│   │   ├── ReviewService.php     ← /review evaluation engine
│   │   └── ApiKeyService.php     ← API key management + failover
│   ├── Repositories/
│   │   ├── UserRepository.php
│   │   ├── FaqRepository.php
│   │   ├── RuleRepository.php
│   │   ├── ChatRepository.php
│   │   ├── ApiKeyRepository.php
│   │   ├── InviteCodeRepository.php
│   │   └── EvalCriteriaRepository.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── Faq.php
│   │   ├── Rule.php
│   │   ├── ChatLog.php
│   │   ├── ApiKey.php
│   │   ├── InviteCode.php
│   │   └── EvalCriterion.php
│   └── Providers/
│       └── ZenProvider.php
├── config/
│   └── database.php         ← DB connection config
├── storage/
│   └── logs/
│       └── app.log
├── tests/
│   ├── Unit/
│   │   ├── CommandServiceTest.php
│   │   ├── RuleServiceTest.php
│   │   ├── UserServiceTest.php
│   │   ├── LLMServiceTest.php
│   │   └── ...
│   └── Integration/
│       └── WebhookTest.php
├── composer.json
├── .env
├── .env.example
├── CONTEXT.md
└── phpunit.xml
```

### No `public/` directory

- `webhook.php` lives at project root
- Dev (Herd nginx): document root = project root → `https://botz.test/webhook.php`
- Prod (Apache): document root = project root → `https://botz.know-tify.com/webhook.php`
- Same path, no rewrite, no difference between environments

---

## 3. Processing Flow

```
LINE POST → webhook.php
    │
    ▼
EventRequestParser::parseEventRequest(body, secret, signature)
    │ (fail → catch exception → 200 OK + exit)
    ▼
Loop each event
    │
    ├── NOT a MessageEvent → ignore
    │
    ├── MessageEvent BUT NOT TextMessageContent
    │   → reply "ขออภัย ระบบรองรับเฉพาะข้อความเท่านั้นค่ะ"
    │   → do NOT log
    │
    └── TextMessageContent → process:
            │
            ▼
        UserService::findOrCreate($userId)
            │
            ▼
        ChatService::log('user', $messageText)
            │
            ▼
        User IS Admin? ──YES──→ CommandService::parse($messageText)
            │                        │
            NO                       ▼
            ▼              ┌─── Has Command? ──YES──→ Execute → Reply
        RuleService::match($message)    │
            │                           NO
            ▼                           │
        Found Rule? ──YES──→ Reply      ▼
            │                    (fall through to AI
            NO                    if not a command)
            │
            ▼
        LLMService::ask($message)
          → build prompt with:
              - Settings (from bot_settings)
              - FAQ (all active, < 100)
              - Conversation summary (from previous turn)
          → try primary API key, failover to backup if needed
          → ZenProvider::ask() via Guzzle
            │
            ▼
        ChatService::log('assistant', $response)
            │
            ▼
        LineService::reply($replyToken, $response)
```

> **AI Clarification**: The LLM is prompted to ask clarifying questions when the user's message is ambiguous (e.g., vague intent, multiple possible interpretations), but answer directly when intent is clear. No separate state machine — the AI decides naturally based on the prompt. The conversation summary captures both answers and follow-up questions, so context carries across turns.

---

## 4. Database Tables

### 4.1 `users`
Per PRD. Stores LINE users with role.

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(255) DEFAULT NULL,
    role ENUM('admin','user') NOT NULL DEFAULT 'user',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_line_user_id (line_user_id)
);
```

### 4.2 `faq`
Per PRD. FAQ knowledge base.

```sql
CREATE TABLE faq (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question TEXT NOT NULL,
    answer TEXT NOT NULL,
    active TINYINT NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_active (active)
);
```

### 4.3 `rules`
Per PRD. Keyword matching rules.

```sql
CREATE TABLE rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    trigger_text VARCHAR(255) NOT NULL,
    response_text TEXT NOT NULL,
    active TINYINT NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_active (active)
);
```

### 4.4 `chat_logs`
Per PRD. All chat messages. Only text messages are logged.

```sql
CREATE TABLE chat_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100) NOT NULL,
    role ENUM('user','assistant') NOT NULL,
    message TEXT NOT NULL,
    is_reviewed TINYINT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_line_user (line_user_id),
    INDEX idx_created (created_at),
    INDEX idx_reviewed (is_reviewed)
);
```

### 4.5 `bot_settings`
Per PRD. Key-value store. Includes `company_name`, `company_description`, and eval criteria guidance.

```sql
CREATE TABLE bot_settings (
    setting_key VARCHAR(100) PRIMARY KEY,
    setting_value TEXT
);
INSERT INTO bot_settings (setting_key, setting_value) VALUES
('company_name', ''),
('company_description', '');
```

### 4.6 `training_logs`
Per PRD. Log of all admin commands executed.

```sql
CREATE TABLE training_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    admin_id INT NOT NULL,
    command_type VARCHAR(50) NOT NULL,
    content TEXT,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (admin_id) REFERENCES users(id)
);
```

### 4.7 `migrations`
Tracks which migration files have been executed.

```sql
CREATE TABLE migrations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    filename VARCHAR(255) UNIQUE NOT NULL,
    executed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 4.8 `invite_codes`
For admin invitation system.

```sql
CREATE TABLE invite_codes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(64) UNIQUE NOT NULL,
    created_by INT NOT NULL,
    used_by VARCHAR(100) DEFAULT NULL,
    expires_at DATETIME NOT NULL,
    is_used TINYINT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id),
    INDEX idx_code (code),
    INDEX idx_expires (expires_at)
);
```

### 4.9 `api_keys`
LLM provider API keys with primary/backup support.

```sql
CREATE TABLE api_keys (
    id INT AUTO_INCREMENT PRIMARY KEY,
    provider VARCHAR(50) NOT NULL DEFAULT 'zen',
    key_value TEXT NOT NULL,
    label VARCHAR(100) DEFAULT NULL,
    is_primary TINYINT NOT NULL DEFAULT 0,
    is_active TINYINT NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_provider (provider),
    INDEX idx_primary (is_primary)
);
```

### 4.10 `eval_criteria`
Admin-defined evaluation criteria for the `/review` system.

```sql
CREATE TABLE eval_criteria (
    id INT AUTO_INCREMENT PRIMARY KEY,
    criteria TEXT NOT NULL,
    is_active TINYINT NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. LLM Provider (Zen API)

### Interface

```php
interface LLMProvider
{
    public function ask(
        string $systemPrompt,
        string $userMessage,
        string $apiKey
    ): string;
}
```

### Using Guzzle (shared with LINE SDK)

```php
class ZenProvider implements LLMProvider
{
    public function __construct(
        private Client $guzzle,
        private readonly string $model,
        private readonly string $baseUrl
    ) {}

    public function ask(string $systemPrompt, string $userMessage, string $apiKey): string
    {
        $response = $this->guzzle->post("{$this->baseUrl}/chat/completions", [
            'headers' => [
                'Authorization' => "Bearer {$apiKey}",
                'Content-Type' => 'application/json',
            ],
            'json' => [
                'model' => $this->model,
                'messages' => [
                    ['role' => 'system', 'content' => $systemPrompt],
                    ['role' => 'user', 'content' => $userMessage],
                ],
                'max_tokens' => 500,
                'temperature' => 0.3,
            ],
        ]);

        $body = json_decode($response->getBody(), true);
        return $body['choices'][0]['message']['content']
            ?? 'ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้';
    }
}
```

### API Key Failover

```php
// LLMService.php
public function ask(string $message): string
{
    $prompt = $this->buildPrompt($message);

    try {
        // Try primary key first
        $primaryKey = $this->apiKeyRepo->getPrimaryKey('zen');
        return $this->provider->ask($prompt, $message, $primaryKey->keyValue);
    } catch (\Exception $e) {
        $this->logger->error('Primary LLM key failed', ['error' => $e->getMessage()]);
    }

    try {
        // Failover to backup
        $backupKey = $this->apiKeyRepo->getNextBackup('zen');
        return $this->provider->ask($prompt, $message, $backupKey->keyValue);
    } catch (\Exception $e) {
        $this->logger->error('All LLM keys failed');
        return 'ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้';
    }
}
```

---

## 6. Conversation Summary

Instead of sending raw chat history, the system maintains a concise **conversation summary** generated by the LLM:

```
Turn 1:
  User: ร้านเปิดกี่โมง
  → Send prompt: (FAQ + summary="" + userMessage)
  → LLM responds + generates summary:
     Summary: "ลูกค้าถามเวลาทำการร้าน"
     Answer: "เปิดทุกวัน 09:00-18:00 ค่ะ"

Turn 2:
  User: แล้วปิดกี่โมง
  → Send prompt: (FAQ + summary="ลูกค้าถามเวลาทำการร้าน" + userMessage)
  → LLM knows context → answers correctly
  → Updates summary: "ลูกค้าถามเวลาทำการ (เปิด 09:00 ปิด 18:00)"
```

### Prompt Template

```text
คุณคือเจ้าหน้าที่บริการลูกค้า

ข้อมูลบริษัท:
{SETTINGS}

คำถามที่พบบ่อย:
{FAQ_DATA}

ประวัติการสนทนาล่าสุด:
{CONVERSATION_SUMMARY}

ข้อความล่าสุดจากลูกค้า:
{USER_MESSAGE}

กฎ:
1. ใช้ข้อมูล FAQ เป็นหลัก
2. ห้ามเดา ข้อมูลที่ไม่มี ห้ามสร้าง
3. อย่าเดาราคา ห้ามสร้างโปรโมชั่น
4. ตอบเป็นภาษาไทย
5. กระชับ ได้ใจความ
6. ถ้าไม่รู้ ให้ตอบ: ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้
7. ต่อท้ายคำตอบด้วย:
   ---
   Summary: <สรุปประเด็นที่คุยกันมาทั้งหมด 1-2 ประโยคสั้นๆ>
   ---
```

The `LLMService` parses the `Summary:` section from the response and stores it for the next turn.

---

## 7. Admin System

### 7.1 First Admin Setup

1. Deploy system with `ADMIN_SETUP_CODE=your-secret` in `.env`
2. First person sends `init your-secret` via LINE Chat
3. System finds user, sets `role = 'admin'`
4. Remove `ADMIN_SETUP_CODE` from `.env` to prevent future registrations

### 7.2 Admin Commands (Thai)

All commands use Thai language. Both English aliases and Thai commands work.

| Command | Description | Format |
|---------|-------------|--------|
| `/เพิ่มคำถาม` | Add FAQ | Q: ... A: ... |
| `/แก้คำถาม` | Edit FAQ | id: N, answer: ... |
| `/ลบคำถาม` | Delete FAQ | id: N |
| `/เพิ่มกฎ` | Add Rule | keyword: ..., response: ... |
| `/ลบกฎ` | Delete Rule | id: N |
| `/สถิติ` | Statistics | (no args) |
| `/เพิ่มแอดมิน` | Add admin by LINE User ID | LINE_USER_ID |
| `/ลบแอดมิน` | Remove admin by LINE User ID | LINE_USER_ID |
| `/เชิญ` | Generate invite code | (no args) → outputs code |
| `รับเชิญ <code>` | Accept invitation | code |
| `/ตั้งค่าคีย์` | Set API key | คีย์: ..., ชื่อ: ..., หลัก: 1/0 |
| `/ลบคีย์` | Delete API key | id: N |
| `/ดูคีย์` | List API keys | (no args) |
| `/ประเมิน` | Start chat log evaluation | (no args) |
| `/สอนประเมิน` | Teach evaluation criteria | free text in Thai |
| `/รีเซ็ตประเมิน` | Reset all reviewed flags | (no args) or `N` for last N days |
| `/ยกเลิก` | Cancel pending invite code | (no args) |

### 7.3 Interactive Command Flow

Commands support two modes: **direct input** (power users) and **AI-guided** (recommended UX).

#### Direct Input Mode
Admin provides all data in one message:

```
/เพิ่มคำถาม

Q: ร้านเปิดกี่โมง
A: เปิดทุกวัน 09:00-18:00 ค่ะ
```
→ Bot executes immediately: ✅ เพิ่ม FAQ สำเร็จ!

#### AI-Guided Mode
If admin sends only the command without data, AI asks step by step:

```
Admin: /เพิ่มคำถาม

Bot: ได้ค่า! ต้องการเพิ่ม FAQ
     คำถามคืออะไรคะ?

Admin: ร้านเปิดกี่โมง

Bot: แล้วคำตอบคืออะไรคะ?

Admin: เปิดทุกวัน 09:00-18:00 ค่ะ

Bot: ✅ เพิ่ม FAQ แล้ว!
     คำถาม: ร้านเปิดกี่โมง
     คำตอบ: เปิดทุกวัน 09:00-18:00 ค่ะ
     ID: 8
     ต้องการเพิ่มอีกไหมคะ? (พิมพ์ /เพิ่มคำถาม หรือพิมพ์อย่างอื่นเพื่อจบ)
```

This applies to all commands that require arguments:
- `/เพิ่มคำถาม` → AI asks for Q: and A:
- `/แก้คำถาม` → AI asks for id: and field to edit
- `/ลบคำถาม` → AI asks for id:
- `/เพิ่มกฎ` → AI asks for keyword: and response:
- `/ลบกฎ` → AI asks for id:
- `/ตั้งค่าคีย์` → AI asks for คีย์:, ชื่อ:, หลัก:

### 7.4 Invite Code Flow

### 7.5 Invite Code Flow

```
Admin sends: /invite
Bot: รหัสเชิญของคุณ: INVITE-a1b2c3d4
     (ใช้ได้ 1 ครั้ง | หมดอายุ 24 ชม.)

Admin shares code with recipient → recipient sends:
accept INVITE-a1b2c3d4
Bot: คุณเป็น Admin แล้ว!
```

---

## 8. Review & Evaluation System

### 8.1 Flow

```
Admin sends: /review
    │
    ▼
Bot fetches unreviewed chat logs (is_reviewed = 0)
    │
    ▼
Batches logs into groups of 20-50
    │
    ▼
For each batch, sends to LLM for evaluation:
  - "Should we add a FAQ for this?"
  - "Rate this response 1-5"
  - Uses stored eval_criteria as guidelines
    │
    ▼
Presents results to Admin one question at a time:
  "Chat #42: 'ร้านเปิดกี่โมง' → 'เปิดทุกวัน 09:00-18:00'
   → ควรเพิ่ม FAQ นี้? (ใช่/ไม่ใช่/แก้ไข)"
    │
    ▼
Admin answers → Bot moves to next question
Shows progress: "กำลังตรวจสอบ 45/120 (37%)"
    │
    ▼
After batch complete, ask: "ต้องการทำต่ออีกไหม?"
If yes → next batch, else → stop
```

### 8.2 Eval Criteria

Admin teaches the LLM how to evaluate conversations:

```
Admin: /evalcriteria "เวลาลูกค้าถามเรื่องราคาแต่เราไม่มีข้อมูล ให้ถือว่าตอบไม่ดี"
    │
    ▼
LLM extracts structured criteria from natural language
    │
    ▼
Stored in `eval_criteria` table
    │
    ▼
On next /review, criteria are sent to LLM as context
```

### 8.3 Reset Review

```
/resetreview       → reset all chat_logs.is_reviewed = 0
/resetreview 7     → reset only logs from last 7 days
```

---

## 9. LINE Integration (SDK v12)

### 9.1 Webhook Handling

```php
use LINE\Webhook\EventRequestParser;
use LINE\Webhook\Model\MessageEvent;
use LINE\Webhook\Model\TextMessageContent;

$parser = new EventRequestParser();

try {
    $events = $parser->parseEventRequest(
        body: $requestBody,
        channelSecret: $channelSecret,
        signature: $lineSignatureHeader
    );
} catch (\Exception $e) {
    // LINE verification or malformed request — just 200 OK
    http_response_code(200);
    exit;
}

foreach ($events as $event) {
    if (!$event instanceof MessageEvent) {
        continue;
    }

    $message = $event->getMessage();

    if (!$message instanceof TextMessageContent) {
        // Non-text message → reply and skip
        $lineService->reply($event->getReplyToken(), 'ขออภัย ระบบรองรับเฉพาะข้อความเท่านั้นค่ะ');
        continue;
    }

    $userId = $event->getSource()->getUserId();
    $text = $message->getText();

    // ... process text message
}
```

### 9.2 Reply

```php
use LINE\Clients\MessagingApi\Api\MessagingApiApi;
use LINE\Clients\MessagingApi\Model\TextMessage;
use LINE\Clients\MessagingApi\Model\ReplyMessageRequest;

$messagingApi->replyMessage(
    new ReplyMessageRequest([
        'replyToken' => $replyToken,
        'messages' => [new TextMessage(['text' => $responseText])],
    ])
);
```

### 9.3 Same Guzzle Client used for Zen API

```php
$guzzle = new \GuzzleHttp\Client();

// LINE SDK uses this Guzzle
$messagingApi = new MessagingApiApi(client: $guzzle, config: $lineConfig);

// Zen API uses the same Guzzle
$zenProvider = new ZenProvider(guzzle: $guzzle, model: $model, baseUrl: $baseUrl);
```

---

## 10. Error Handling

| Scenario | Response |
|----------|----------|
| Invalid LINE signature | Catch, HTTP 200, no processing |
| Non-text message | Reply "ขออภัย ระบบรองรับเฉพาะข้อความเท่านั้น", no log |
| LLM timeout (primary key) | Tries backup key automatically |
| LLM timeout (all keys failed) | Reply "ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้" |
| DB connection failure | Log error, reply fallback |
| Unknown admin command | Reply "ไม่พบคำสั่ง..." |
| Malformed command | Reply "รูปแบบคำสั่งไม่ถูกต้อง..." |

---

## 11. PHP Extensions Required

| Extension | Reason |
|-----------|--------|
| curl | Guzzle requires it for HTTP calls |
| pdo_mysql | Database connection |
| mbstring | Thai string handling, rule matching |
| json | Webhook/API serialization |
| openssl | HTTPS (required by Guzzle/cURL) |
| opcache | Performance (recommended) |

---

## 12. Testing Strategy

| Type | Coverage | Tools |
|------|----------|-------|
| **Unit tests** | ~90% — Service logic, command parser, rule matching, prompt building | PHPUnit + Mockery for repositories |
| **Integration tests** | ~10% — LINE webhook payload → full flow with mocked Guzzle | PHPUnit + fake LINE payloads |

### Unit test targets:
- `CommandServiceTest` — parse all command formats, handle malformed input
- `RuleServiceTest` — match keywords, prioritization, no match case
- `UserServiceTest` — findOrCreate, role checks
- `LLMServiceTest` — prompt building with FAQ + settings + summary
- `ApiKeyServiceTest` — primary/backup failover logic
- `FaqServiceTest` — CRUD with validation

### Integration test target:
- `WebhookTest` — send real LINE webhook JSON, verify flow from parse to reply

---

## 13. Timezone & Language

| Aspect | Setting |
|--------|---------|
| Timezone | Asia/Bangkok (UTC+7) |
| DB time | DATETIME with Bangkok time |
| Log time | Bangkok time via Monolog timezone config |
| LINE timestamp | Convert from UTC+9 to UTC+7 on receipt |
| Display language | Thai only — all prompts, responses, commands, errors |
| User | Thai customers |
| Admin | Thai administrators |

---

## 14. Security

| Requirement | Implementation |
|-------------|---------------|
| Admin role check | Every command checks `$user->isAdmin()` before execution |
| SQL injection | Prepared statements via PDO only |
| API keys in .env | All secrets in `.env`, never hardcoded |
| API keys in DB | `api_keys` table values encrypted / access via repository only |
| Directory access | `.htaccess` blocks `src/`, `vendor/`, `.env`, `composer.json` |
| LINE webhook verification | `EventRequestParser` validates HMAC-SHA256 signature |
| Invite codes | 24h expiration, single use, random generation |

---

## 15. Key Design Decisions

1. **LINE SDK v12** — Not v9. Uses Guzzle natively. Provides `EventRequestParser` + typed models for webhook events.
2. **Guzzle for both LINE + Zen** — Single HTTP client instance, shared via container.
3. **No `public/` directory** — `webhook.php` at project root. Same entry point for dev (Herd) and prod (Apache). No .htaccess rewrite.
4. **Conversation summary instead of raw history** — LLM generates 1-2 sentence summary each turn. Reduces token usage and prevents LLM fixating on old messages.
5. **LLMProvider interface** — Even though only Zen is planned, the interface ensures loose coupling and testability.
6. **Separate `api_keys` table** — Primary + backup keys with auto-failover. Admin manages via LINE chat without touching `.env`.
7. **API key failover** — If primary LLM key fails, attempt backup keys in order. All fail → fallback message.
8. **Admin via `init <CODE>`** — First admin setup through LINE Chat. No DB insert needed. Code removable from `.env`.
9. **Invite system** — `/invite` generates code. `accept <code>` activates admin. No need to share LINE User IDs.
10. **`/review` with batch processing** — LLM evaluates unreviewed chats in batches, presents to admin one at a time with progress %, asks to continue after each batch.
11. **`/evalcriteria`** — Admin writes in natural Thai, LLM extracts structured criteria. Stored in separate table for per-item management.
12. **Repository pattern** — Data access isolated in repositories. Services call repositories, never write SQL directly.
13. **Readonly models** — Models are immutable value objects with `readonly` properties. Repositories are responsible for creation.
14. **Database migration runner** — `migrate.php` reads raw `.sql` files, tracks execution in `migrations` table. Prevents double execution.
15. **AI clarification instead of structured command flow** — When customer message is ambiguous (not an admin command), the LLM asks clarifying questions naturally via prompt instructions. No separate state machine, no mode toggling. The system prompt tells the AI: "If the user's intent is unclear, ask for clarification. If clear, answer directly." Conversation summaries carry both answers and follow-ups across turns.
