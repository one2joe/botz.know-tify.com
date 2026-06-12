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
| Admin Onboarding | First person to message bot after deploy = แอดมินหลัก automatically | No setup code needed. Simple and fast |
| Admin Management | `/เชิญ` generates 1-time code → LINE URL with pre-filled text → recipient taps link → auto-activated | No need to type "accept". 24h expiration, single-use |
| API Key Management | Separate `api_keys` table — system manages primary/backup order automatically | Keys added via `/เพิ่มคีย์`. Latest key = primary, older keys = backups |

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
│   ├── 006_create_invite_codes.sql
│   ├── 007_create_api_keys.sql
│   └── 008_create_eval_criteria.sql
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
            ├── First user ever? ──YES──→ Set role = 'admin' (แอดมินหลัก)
            │                               → Reply "ยินดีต้อนรับ..."
            │                               → Continue processing normally
            │
            ▼
        ChatService::log('user', $messageText)
            │
            ▼
        Starts with "INVITE_"?
            │
            ├── YES → InviteCodeService::activate($code, $userId)
            │           ├── Valid? → Set role = 'helper_admin', mark code used
            │           │            → Reply "ยินดีต้อนรับ..."
            │           ├── Expired? → Reply "ลิงก์เชิญหมดอายุ..."
            │           └── Used? → Reply "ลิงก์เชิญถูกใช้แล้ว..."
            │
            NO
            │
            ▼
        User IS Admin? ──YES──→ CommandService::parse($messageText)
            │                        │
            NO                       ▼
            ▼              ┌─── Has Command? ──YES──→ Execute → Reply
        API keys exist?        │
            │                  NO
            ├── NO (admin)  → Reply "กรุณา /เพิ่มคีย์ ก่อน..."
            │                 │
            ├── NO (customer) → Reply fallback (no key = no service)
            │                  ▼
            │            (fall through to AI
            YES           if not a command)
            │
            ▼
        RuleService::match($message)
            │
            ▼
        Found Rule? ──YES──→ Reply (no AI call)
            │
            NO
            │
            ▼
        LLMService::ask($message)
          → build prompt with:
              - Settings (from bot_settings)
              - FAQ (all active, < 100)
              - Recent conversation context
                (last 2 rounds; LLM decides if more context needed → up to 3 rounds)
          → failover: try keys in order (latest added = primary)
          → ZenProvider::ask() via Guzzle
            │
            ▼
        ChatService::log('assistant', $response)
            │
            ▼
        LineService::reply($replyToken, $response)
```

> **AI Clarification**: The LLM is prompted to ask clarifying questions when the user's message is ambiguous (e.g., vague intent, multiple possible interpretations), but answer directly when intent is clear. No separate state machine — the AI decides naturally based on the prompt instructions.

---

## 4. Database Tables

### 4.1 `users`
Stores LINE users with role. First user ever to message the bot becomes แอดมินหลัก.

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(255) DEFAULT NULL,
    role ENUM('admin','helper_admin','user') NOT NULL DEFAULT 'user',
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

### 4.6 `invite_codes`
For admin invitation. Code embedded in LINE URL (pre-filled text). Auto-activates when recipient sends first message containing `INVITE_<code>`.

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

### 4.7 `migrations`
Tracks which migration files have been executed.

```sql
CREATE TABLE migrations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    filename VARCHAR(255) UNIQUE NOT NULL,
    executed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 4.8 `api_keys`
LLM provider API keys. System manages primary/backup order automatically — latest added key is primary, older keys become backups.

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

### 4.9 `eval_criteria`
Admin-defined evaluation criteria for the `/รีวิว` system.

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

## 6. Conversation Context (C2)

Instead of LLM-generated summaries, the system keeps **recent raw messages** from `chat_logs` and dynamically adjusts context size:

### Algorithm

```
1. Fetch last 2 rounds (user + assistant) from chat_logs for this user
2. If time gap between newest and oldest in context > 30 min → fall back to 1 round
3. If not too old → include 2 rounds in prompt
4. LLM evaluates if more context is needed:
   - If message refers to previous turns → include up to 3 rounds
   - If message is a new topic → keep at 2 rounds (or 1 if timed out)
5. Token limit: if total prompt (FAQ + settings + context) exceeds ~4000 tokens,
   truncate oldest rounds first
```

No separate `conversation_summaries` table. No LLM call for summary generation. Context is built from raw chat_logs at request time.

### Prompt Template

```text
คุณคือเจ้าหน้าที่บริการลูกค้า

ข้อมูลบริษัท:
{SETTINGS}

คำถามที่พบบ่อย:
{FAQ_DATA}

{CONVERSATION_CONTEXT}

ข้อความล่าสุดจากลูกค้า:
{USER_MESSAGE}

กฎ:
1. ใช้ข้อมูล FAQ เป็นหลัก
2. ห้ามเดา ข้อมูลที่ไม่มี ห้ามสร้าง
3. อย่าเดาราคา ห้ามสร้างโปรโมชั่น
4. ตอบเป็นภาษาไทย
5. กระชับ ได้ใจความ
6. ถ้าข้อความของลูกค้าคลุมเครือหรือไม่ชัดเจน ให้ถามกลับเพื่อขอรายละเอียดเพิ่ม
   ถ้าชัดเจนแล้ว ให้ตอบตรงๆ ไม่ต้องถามกลับโดยไม่จำเป็น
7. ถ้าไม่รู้ ให้ตอบ: ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้
```

---

## 7. Admin System

### 7.1 First Admin Setup

1. Deploy system → first person to send any message via LINE Chat
2. System checks: is this the first user in `users` table?
3. If yes → set `role = 'admin'` (แอดมินหลัก) automatically
4. Reply with welcome message + prompt to add API key

No setup code needed. No `init` command. The very first message = แอดมินหลัก.

### 7.2 Admin Roles & Permissions

| Permission | แอดมินหลัก | ผู้ช่วยแอดมิน |
|---|---|---|
| เพิ่ม/แก้/ลบ FAQ | ✅ | ✅ |
| เพิ่ม/ลบ Rule | ✅ | ✅ |
| ดูสถิติ (`/สถิติ`) | ✅ | ✅ |
| จัดการ Settings (`/ตั้งค่า`) | ✅ | ✅ |
| รีวิวบทสนทนา (`/รีวิว`) | ✅ | ✅ |
| จัดการเกณฑ์ประเมิน (`/เกณฑ์ประเมิน`) | ✅ | ✅ |
| เชิญ admin ใหม่ (`/เชิญ`) | ✅ | ❌ |
| ถอดสิทธิ์ admin (`/ถอดสิทธิ์`) | ✅ | ❌ |
| จัดการ API Keys (`/เพิ่มคีย์`) | ✅ | ❌ |

### 7.3 Admin Commands (Thai)

All commands in Thai. No English aliases. Admin types `/คำสั่ง` → bot asks interactively.

| Command | Who can use | Description |
|---------|-------------|-------------|
| `/เพิ่มfaq` | all admin | Add FAQ |
| `/แก้ไขfaq` | all admin | Edit FAQ |
| `/ลบfaq` | all admin | Delete FAQ (requires confirmation) |
| `/รายการfaq` | all admin | List all FAQ (10 per page, Quick Reply pagination) |
| `/เพิ่มrule` | all admin | Add Rule |
| `/ลบrule` | all admin | Delete Rule (requires confirmation) |
| `/รายการrule` | all admin | List all Rules (10 per page, Quick Reply pagination) |
| `/สถิติ` | all admin | View statistics |
| `/รีวิว` | all admin | Review/evaluate conversations (1 topic at a time) |
| `/เกณฑ์ประเมิน` | all admin | View/manage evaluation criteria |
| `/ตั้งค่า` | all admin | Manage bot settings |
| `/เชิญ` | แอดมินหลัก only | Generate invite link |
| `/ถอดสิทธิ์` | แอดมินหลัก only | Remove admin privileges |
| `/เพิ่มคีย์` | แอดมินหลัก only | Add API key |
| `/help` | all admin | Show help |

### 7.4 AI-Guided Interactive Flow

Admin types `/คำสั่ง` → bot asks for missing info step by step (multiple bubbles, max 5). No need to remember format.

**Example: `/เพิ่มfaq`**

```
Admin: /เพิ่มfaq

บอลลูน 1:
┌───────────────────────────────────────┐
│ กรุณาระบุคำถามที่ต้องการเพิ่ม           │
│                                       │
│ เช่น: ร้านเปิดกี่โมง                   │
└───────────────────────────────────────┘

Admin: ร้านเปิดกี่โมง

บอลลูน 1:
┌───────────────────────────────────────┐
│ ✅ ได้คำถาม: "ร้านเปิดกี่โมง"          │
│                                       │
│ กรุณาระบุคำตอบ                        │
│                                       │
│ เช่น: เปิดทุกวัน 09:00-18:00           │
└───────────────────────────────────────┘

Admin: เปิดทุกวัน 09:00-18:00 น.

บอลลูน 1:
┌───────────────────────────────────────┐
│ ✅ บันทึก FAQ เรียบร้อย               │
│                                       │
│ Q: ร้านเปิดกี่โมง                      │
│ A: เปิดทุกวัน 09:00-18:00 น.          │
│ ID: 8                                │
└───────────────────────────────────────┘
```

**Example: `/ลบfaq` (with confirmation)**

```
Admin: /ลบfaq

บอลลูน 1:
┌───────────────────────────────────────┐
│ กรุณาระบุ ID ของ FAQ ที่ต้องการลบ      │
│                                       │
│ พิมพ์ /รายการfaq เพื่อดูทั้งหมด        │
└───────────────────────────────────────┘

Admin: 3

บอลลูน 1:
┌───────────────────────────────────────┐
│ FAQ #3                                │
│ Q: ร้านเปิดกี่โมง                      │
│ A: เปิดทุกวัน 09:00-18:00 น.          │
└───────────────────────────────────────┘

บอลลูน 2:
┌───────────────────────────────────────┐
│ ⚠️ ยืนยันลบ FAQ นี้?                   │
│                                       │
│ พิมพ์ "ใช่" เพื่อยืนยัน หรือ "ไม่" เพื่อยกเลิก │
└───────────────────────────────────────┘

Admin: ใช่

บอลลูน 1:
┌───────────────────────────────────────┐
│ ✅ ลบ FAQ #3 เรียบร้อย               │
└───────────────────────────────────────┘
```

All delete commands require confirmation. All multi-step interactions are split across multiple text bubbles (max 5 per reply).

### 7.5 Invite Code Flow

**Step 1: Admin หลัก พิมพ์ `/เชิญ`**

```
Bot replies — บอลลูน 1 (สำหรับ admin หลักเท่านั้น):
┌───────────────────────────────────────┐
│ ✅ สร้างลิงก์เชิญเรียบร้อย            │
│                                       │
│ ⚠️ ลิงก์นี้ห้ามกดเอง                   │
│    ให้ส่งต่อเฉพาะบอลลูนด้านล่าง         │
│    ให้คนที่ต้องการเชิญเท่านั้น          │
└───────────────────────────────────────┘

Bot replies — บอลลูน 2 (พร้อมส่งต่อ):
┌───────────────────────────────────────┐
│ คุณได้รับเชิญให้เป็นผู้ช่วยแอดมิน       │
│                                       │
│ กดลิงก์ด้านล่างเพื่อยืนยันสิทธิ์        │
│ ลิงก์หมดอายุใน 24 ชั่วโมง            │
│                                       │
│ https://line.me/R/oaMessage/          │
│ @340bzlph/?text=INVITE_a1b2c3         │
└───────────────────────────────────────┘
```

**Step 2: Admin หลัก Forward บอลลูน 2 ให้ผู้รับ**

**Step 3: ผู้รับกดลิงก์**

LINE opens chat with bot. Message `INVITE_a1b2c3` is pre-filled in input box. User taps send.

**Step 4: ระบบรับข้อความ `INVITE_...`**

- Check if code valid (not expired, not used)
- If valid → set `role = 'helper_admin'`, mark code used
- If expired → Reply: "ลิงก์เชิญหมดอายุแล้ว..."
- If used → Reply: "ลิงก์เชิญนี้ถูกใช้ไปแล้ว..."

```
Reply on success:
┌───────────────────────────────────────┐
│ ✅ ยินดีต้อนรับ!                       │
│                                       │
│ คุณได้รับเชิญเป็นผู้ช่วยแอดมินเรียบร้อย  │
│                                       │
│ พิมพ์ /help เพื่อดูคำสั่งที่ใช้งานได้   │
└───────────────────────────────────────┘
```

### 7.6 API Key Management

**Flow when no key exists:**

Any message from admin (except `/เพิ่มคีย์`) → reply:
```
┌───────────────────────────────────────┐
│ ⚠️ ระบบยังไม่มี API Key              │
│                                       │
│ กรุณาพิมพ์ /เพิ่มคีย์ เพื่อตั้งค่า      │
│ ให้บอทซ์ทำงานได้                      │
└───────────────────────────────────────┘
```

**Adding a key:**

```
Admin: /เพิ่มคีย์

บอลลูน 1:
┌───────────────────────────────────────┐
│ กรุณาส่ง API Key ของคุณมาได้เลย        │
│                                       │
│ (เช่น sk-xxxx...)                     │
└───────────────────────────────────────┘

Admin: sk-zen-xxxxxxxxxxxx

บอลลูน 1:
┌───────────────────────────────────────┐
│ ✅ บันทึก API Key เรียบร้อย           │
│                                       │
│ Key นี้ถูกใช้เป็นตัวหลักแล้ว           │
│ พิมพ์ /เพิ่มคีย์ เพื่อเพิ่ม            │
│ Key สำรองเพิ่มเติมได้                 │
└───────────────────────────────────────┘
```

**Backup management:** System handles automatically. Latest added key = primary. Older keys = backups. Failover tries keys in reverse chronological order until one works. If all fail → fallback message.

---

## 8. Review & Evaluation System

### 8.1 Flow — `/รีวิว`

1. Admin sends `/รีวิว`
2. Bot fetches 1 unreviewed conversation topic (newest first, `is_reviewed = 0`)
3. Sends to LLM with current `eval_criteria` for evaluation
4. LLM evaluates the response and explains reasoning

**Case 1 — AI determines response is good:**
```
→ Auto mark as reviewed (is_reviewed = 1)
→ Ask: "ต้องการดูประเด็นต่อไปไหม?"
   [Quick Reply: ดูต่อ / จบ]
```

**Case 2 — AI determines response may need improvement:**
```
บอลลูน 1:
┌───────────────────────────────────────┐
│ ⚠️ บทสนทนาที่อาจต้องปรับปรุง          │
│                                       │
│ ลูกค้า: ราคาเท่าไหร่                  │
│ บอทซ์: ขออภัย ไม่มีข้อมูลค่ะ           │
│                                       │
│ เหตุผล: FAQ #3 มีข้อมูลราคาอยู่แล้ว    │
│         แต่บอทซ์ตอบว่าไม่มีข้อมูล       │
└───────────────────────────────────────┘

บอลลูน 2:
┌───────────────────────────────────────┐
│ คุณคิดว่าบอทซ์ตอบผ่านไหม?             │
│                                       │
│ (พิมพ์ "ผ่าน" หรือ "ไม่ผ่าน")         │
└───────────────────────────────────────┘
```

**If admin says "ผ่าน"** → mark `is_reviewed = 1`, ask "ดูต่อ?"

**If admin says "ไม่ผ่าน"** → enter teach mode:

```
บอลลูน 1:
┌───────────────────────────────────────┐
│ คุณต้องการสอนกฎการรีวิวเพิ่มเติมไหม?    │
│                                       │
│ พิมพ์กฎที่ต้องการเพิ่ม                 │
│ หรือพิมพ์ "ข้าม" เพื่อไปปรับ FAQ       │
└───────────────────────────────────────┘

Admin: เวลาลูกค้าถามเรื่องราคา ต้องตอบจาก FAQ เท่านั้น

บอลลูน 1:
┌───────────────────────────────────────┐
│ ✅ บันทึกกฎการรีวิวแล้ว               │
│                                       │
│ ต้องการเพิ่มอีกไหม?                    │
│ หรือพิมพ์ "พอ" เพื่อไปปรับ FAQ         │
└───────────────────────────────────────┘

Admin: พอ

บอลลูน 1:
┌───────────────────────────────────────┐
│ ถึงเวลาปรับปรุงคลังความรู้ (FAQ)       │
│                                       │
│ จากผลรีวิวที่ไม่ผ่าน                   │
│ ต้องการเพิ่ม/แก้ FAQ นี้ไหม?           │
│                                       │
│ พิมพ์ /เพิ่มfaq หรือ /แก้ไขfaq        │
│ เพื่อจัดการ                           │
│ หรือพิมพ์ "พอ" เพื่อกลับไปรีวิวต่อ     │
└───────────────────────────────────────┘
```

After admin finishes updating FAQ, bot asks "ดูประเด็นต่อไป?" → continue loop.

### 8.2 Default Eval Criteria

On first use, system has 6 default criteria:
1. ตอบตรงกับคำถามของลูกค้า
2. ใช้ข้อมูลที่ถูกต้องจาก FAQ
3. ภาษาสุภาพ อ่านเข้าใจง่าย
4. ตอบสั้น กระชับ ได้ใจความ
5. ไม่มั่วข้อมูลหรือคิดคำตอบเอง
6. แนะนำให้ติดต่อเจ้าหน้าที่เมื่อเกินความสามารถ

Admin can view/edit/add/delete via `/เกณฑ์ประเมิน`.

### 8.3 `/เกณฑ์ประเมิน` — Manage Criteria

```
Admin: /เกณฑ์ประเมิน

บอลลูน 1:
┌───────────────────────────────────────┐
│ 📋 เกณฑ์การประเมินปัจจุบัน (6 ข้อ)    │
│                                       │
│ 1. ตอบตรงกับคำถามของลูกค้า            │
│ 2. ใช้ข้อมูล FAQ                      │
│ 3. ภาษาสุพาอ่านเข้าใจง่าย              │
│ 4. ตอบสั้น กระชับ                     │
│ 5. ไม่มั่วข้อมูล                       │
│ 6. แนะนำเจ้าหน้าที่เมื่อเกินความสามารถ  │
└───────────────────────────────────────┘

บอลลูน 2:
┌───────────────────────────────────────┐
│ คำสั่งจัดการเกณฑ์:                     │
│ /เกณฑ์ประเมิน เพิ่ม: <ข้อความ>         │
│ /เกณฑ์ประเมิน แก้ไข #3: <ข้อความ>      │
│ /เกณฑ์ประเมิน ลบ #5                  │
└───────────────────────────────────────┘
[Quick Reply: เพิ่ม | แก้ไข | ลบ | พอแล้ว]
```

### 8.4 `/รีเซ็ตประเมิน`

```
/รีเซ็ตประเมิน     → reset all chat_logs.is_reviewed = 0
/รีเซ็ตประเมิน 7   → reset only logs from last 7 days
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
| No API key configured (admin) | Block all except `/เพิ่มคีย์`, prompt to add key |
| No API key configured (customer) | Reply fallback "ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้" |
| LLM timeout (primary key) | Tries backup key automatically |
| LLM timeout (all keys failed) | Reply "ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้" |
| DB connection failure | Log error, reply fallback |
| Unknown admin command | Reply "ไม่พบคำสั่ง..." |
| Malformed command | Reply "รูปแบบคำสั่งไม่ถูกต้อง..." |
| Expired invite code | Reply "ลิงก์เชิญหมดอายุแล้ว กรุณาติดต่อแอดมินหลัก" |
| Used invite code | Reply "ลิงก์เชิญนี้ถูกใช้ไปแล้ว กรุณาติดต่อแอดมินหลัก" |

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
- `LLMServiceTest` — prompt building with FAQ + settings + conversation context
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
4. **Conversation context (C2) instead of LLM-generated summary** — Keep last 2-3 raw message rounds. LLM decides if context is relevant. No summary table, no extra LLM call. Saves tokens over summary approach.
5. **LLMProvider interface** — Even though only Zen is planned, the interface ensures loose coupling and testability.
6. **Separate `api_keys` table** — System manages primary/backup order automatically (latest added = primary). No admin overhead.
7. **API key failover** — Try keys in reverse chronological order. All fail → fallback message.
8. **First-user-as-admin** — First person to message bot after deploy = แอดมินหลัก. No setup code needed. Simple and secure (bot has no public visibility before deployment).
9. **Invite via LINE URL** — `/เชิญ` generates LINE URL with pre-filled `INVITE_<code>`. Recipient taps link, sends first message → auto-activated. No "accept" command needed.
10. **`/รีวิว` 1 topic at a time** — LLM evaluates with reasoning. Good → auto-pass. Not good → admin decides. Fail → admin teaches criteria → improves FAQ. Loop.
11. **Default eval criteria** — 6 built-in criteria so `/รีวิว` works immediately. Admin can add/edit via `/เกณฑ์ประเมิน`.
12. **Repository pattern** — Data access isolated in repositories. Services call repositories, never write SQL directly.
13. **Readonly models** — Models are immutable value objects with `readonly` properties. Repositories are responsible for creation.
14. **Database migration runner** — `migrate.php` reads raw `.sql` files, tracks execution in `migrations` table. Prevents double execution.
15. **AI clarification** — LLM prompted to ask when ambiguous, answer directly when confident. No state machine.
16. **All commands in Thai** — No English aliases. `/help`, `/เพิ่มfaq`, `/ลบfaq`, `/เชิญ`, etc.
17. **AI-guided interactive flow** — Type `/คำสั่ง` only, bot asks for missing info step by step in multiple bubbles (max 5). Delete requires confirmation.
18. **Text messages + Quick Reply** — No Flex Message for MVP. All responses are text bubbles. Quick Reply used for pagination and choices.
19. **No `training_logs` table** — Not needed. All admin actions logged via `chat_logs`.
