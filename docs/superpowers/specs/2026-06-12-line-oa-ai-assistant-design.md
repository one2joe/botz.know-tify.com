# Design Document: LINE OA AI Assistant with In-Chat Training

Date: 2026-06-12
Based on: PRD.md (same directory root)

---

## 1. Architecture Overview

| Component | Decision | Rationale |
|-----------|----------|-----------|
| Language | PHP 8.4 | Per PRD, stable for production |
| Framework | Vanilla PHP (no framework) | Single webhook endpoint, 30 files total, no framework overhead |
| HTTP Client | Raw cURL (via PHP `ext-curl`) | Only 3 HTTP endpoints (LINE Reply, LINE Profile, Zen Chat). Avoids Guzzle/LINE SDK vendor bloat (~2 MB) |
| LLM Provider | Zen API (OpenAI-compatible) | Unified access to multiple models via single API key. Model: `deepseek-v4-flash-free` |
| LLM Abstraction | `LLMProvider` interface | Strategy pattern for future provider swaps. Zen implements `LLMProvider` |
| Database | MySQL 8 via PDO | Per PRD, prepared statements only |
| Dependencies | `vlucas/phpdotenv` only | Vendor size ~200 KB |

---

## 2. Project Structure

```
line-oa-bot/
├── public/
│   └── webhook.php              ← Entry point (LINE Webhook)
├── src/
│   ├── Contracts/
│   │   └── LLMProvider.php       ← Interface for LLM providers
│   ├── Controllers/
│   │   └── WebhookController.php ← Route webhook events
│   ├── Services/
│   │   ├── LineService.php       ← LINE Messaging API (cURL)
│   │   ├── LLMService.php        ← Prompt builder + LLM call
│   │   ├── UserService.php       ← User findOrCreate logic
│   │   ├── FaqService.php        ← FAQ CRUD
│   │   ├── RuleService.php       ← Rule matching
│   │   ├── ChatService.php       ← Chat logging
│   │   └── CommandService.php    ← Admin command parser
│   ├── Repositories/
│   │   ├── UserRepository.php
│   │   ├── FaqRepository.php
│   │   ├── RuleRepository.php
│   │   └── ChatRepository.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── Faq.php
│   │   ├── Rule.php
│   │   └── ChatLog.php
│   └── Providers/
│       └── ZenProvider.php       ← Zen API implementation
├── config/
│   └── Database.php
├── storage/
│   └── logs/
├── composer.json
├── .env
└── .env.example
```

---

## 3. Processing Flow

```
LINE POST → public/webhook.php
    │
    ▼
Verify signature (HMAC-SHA256, no LINE SDK)
    │ (fail → 401)
    ▼
Parse JSON body → LineEvent DTO
    │
    ▼
UserService::findOrCreate($lineUserId)
    │
    ▼
ChatService::log('user', $message)
    │
    ▼
User IS Admin? ──YES──→ CommandService::parse($message)
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
  → build prompt with FAQ + Settings
  → ZenProvider::ask() via cURL
    │
    ▼
ChatService::log('assistant', $response)
    │
    ▼
LineService::reply($replyToken, $response)
```

---

## 4. Database (MySQL 8)

Based on PRD section 7, with additions:
- Indexes on frequently queried columns
- `ON UPDATE CURRENT_TIMESTAMP` on `faq.updated_at`
- Default `bot_settings` rows

```sql
-- Same as PRD with added indexes. See PRD.md section 7 for full DDL.
```

---

## 5. LLM Provider (Zen API)

### Interface

```php
interface LLMProvider
{
    public function ask(string $systemPrompt, string $userMessage): string;
}
```

### Zen Implementation

```php
class ZenProvider implements LLMProvider
{
    public function __construct(
        private readonly string $apiKey,
        private readonly string $model,  // 'deepseek-v4-flash-free'
        private readonly string $baseUrl // 'https://api.zen.run/v1'
    ) {}

    public function ask(string $systemPrompt, string $userMessage): string
    {
        $response = $this->curlPost('/chat/completions', [
            'model' => $this->model,
            'messages' => [
                ['role' => 'system', 'content' => $systemPrompt],
                ['role' => 'user', 'content' => $userMessage],
            ],
            'max_tokens' => 500,
            'temperature' => 0.3,
        ]);

        return $response['choices'][0]['message']['content']
            ?? Fallback message;
    }
}
```

### Model selection via .env

```
LLM_PROVIDER=zen
ZEN_API_KEY=sk-...
ZEN_MODEL=deepseek-v4-flash-free
```

Future providers can be added by implementing `LLMProvider` and adding a new class in `src/Providers/`.

---

## 6. LINE Integration (No SDK)

All LINE API calls are raw cURL:

| Operation | Endpoint | Method |
|-----------|----------|--------|
| Verify signature | Local HMAC-SHA256 | - |
| Reply message | `POST /v2/bot/message/reply` | Bearer token |
| Get profile | `GET /v2/bot/profile/{userId}` | Bearer token |

Signature verification is a single function:

```php
function verifyLineSignature(string $body, string $signature, string $channelSecret): bool
{
    $hash = base64_encode(hash_hmac('sha256', $body, $channelSecret, true));
    return hash_equals($hash, $signature);
}
```

---

## 7. Error Handling

| Scenario | Response |
|----------|----------|
| Invalid LINE signature | HTTP 401, no processing |
| OpenAI/Zen API timeout | Log error, reply fallback message |
| OpenAI/Zen API 4xx/5xx | Log error, reply fallback message |
| DB connection failure | Log error, reply fallback message |
| Unknown admin command | Reply "ไม่พบคำสั่ง..." |
| Malformed command format | Reply "รูปแบบคำสั่งไม่ถูกต้อง..." |
| OpenAI unavailable | Reply "ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้" |

---

## 8. Admin Commands

| Command | Handler | Format |
|---------|---------|--------|
| `/faq` | Add FAQ | Q: ... A: ... |
| `/editfaq` | Edit FAQ | id: N | answer: ... |
| `/deletefaq` | Delete FAQ | id: N |
| `/rule` | Add Rule | keyword: ... | response: ... |
| `/deleterule` | Delete Rule | id: N |
| `/stats` | Statistics | (no args) |

Admin identification: First admin must be inserted into DB manually (INSERT INTO users). After that, LINE Chat commands work normally.

---

## 9. PHP Extensions Required

| Extension | Reason |
|-----------|--------|
| curl | LINE API + Zen API calls |
| pdo_mysql | Database connection |
| mbstring | Thai string handling, Rule matching |
| json | Webhook/API serialization |
| openssl | HTTPS (required by cURL) |
| opcache | Performance (recommended) |

---

## 10. composer.json

```json
{
  "name": "botz/line-oa-assistant",
  "require": {
    "php": ">=8.2",
    "vlucas/phpdotenv": "^5.6"
  },
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```

---

## 11. Key Design Decisions

1. **No LINE SDK** — LINE Bot SDK v8 was ~6-8 MB. Even v9 pulls Guzzle (~2 MB). For 3 API calls, raw cURL is sufficient.
2. **No Guzzle** — PHP's built-in ext-curl handles everything. No extra dependencies.
3. **LLMProvider interface** — Even though only Zen is planned, the interface ensures loose coupling and testability.
4. **Single entry point** — `public/webhook.php` handles all LINE webhook events. No routing needed.
5. **Repository pattern** — Data access is isolated in repositories, keeping services focused on business logic.
6. **Readonly models** — Models are immutable value objects. Repositories are responsible for creation.
