# PRODUCT REQUIREMENTS DOCUMENT (PRD)

## Project Name

LINE OA AI Assistant with In-Chat Training

---

# 1. Overview

ระบบ Chatbot สำหรับ LINE Official Account (LINE OA)

ผู้ใช้งานสามารถสนทนากับ AI ผ่าน LINE OA ได้ตามปกติ

ผู้ดูแลระบบ (Admin) สามารถสอน ปรับปรุง และแก้ไขพฤติกรรมของ AI ได้ผ่าน LINE Chat โดยตรง โดยไม่ต้องแก้ไขโค้ด ไม่ต้อง Deploy ใหม่ และไม่ต้อง Fine-tune โมเดล

ระบบใช้:

* PHP 8.2+
* MySQL 8+
* LINE Messaging API
* OpenAI API

---

# 2. Business Goals

1. ลดภาระการตอบคำถามซ้ำๆ
2. ให้ Admin สามารถสอนบอทได้เอง
3. ลดการพึ่งพานักพัฒนาในการแก้ FAQ
4. รองรับ FAQ ไม่เกิน 100 รายการ
5. ใช้งานง่าย ดูแลรักษาง่าย
6. ไม่ใช้ Vector Database
7. ไม่ใช้ Embedding
8. ไม่ใช้ Fine-Tuning

---

# 3. System Scope

## Included

* LINE OA Chatbot
* FAQ Management ผ่าน Chat
* Rule Management ผ่าน Chat
* AI Response Generation
* Chat History Storage
* Admin Command System

## Excluded

* Dashboard
* Vector Search
* Embedding
* Fine-Tuned Model
* Multi-Tenant
* CRM Integration
* Payment Integration

---

# 4. User Roles

## Customer

สามารถ

* ส่งข้อความ
* รับคำตอบจาก AI

ไม่สามารถ

* สอนระบบ
* แก้ไขข้อมูล

---

## Admin

สามารถ

* เพิ่ม FAQ
* แก้ FAQ
* ลบ FAQ
* เพิ่ม Rule
* ลบ Rule
* ดูสถิติพื้นฐาน

ทั้งหมดผ่าน LINE Chat

---

# 5. Functional Requirements

---

## FR-01 User Registration

เมื่อมีผู้ใช้ส่งข้อความเข้ามาครั้งแรก

ระบบต้อง

1. ตรวจสอบ line_user_id
2. หากไม่พบในฐานข้อมูล
3. สร้างผู้ใช้ใหม่
4. role = user

---

## FR-02 Chat Logging

ทุกข้อความต้องถูกบันทึก

เก็บ

* user_id
* role
* message
* timestamp

ทั้งข้อความจาก User และ AI

---

## FR-03 FAQ Knowledge Base

ระบบต้องสามารถเก็บ FAQ ได้

ข้อมูล

* Question
* Answer
* Active Status

จำนวน FAQ สูงสุดที่ออกแบบไว้

100 รายการ

---

## FR-04 Rule Engine

ระบบต้องรองรับ Rule แบบ Keyword Match

ตัวอย่าง

Keyword:

คืนสินค้า

Response:

กรุณากรอกแบบฟอร์มคืนสินค้า

หากพบ Keyword

ระบบต้องตอบทันที

โดยไม่เรียก OpenAI

---

## FR-05 AI Response Generation

หากไม่พบ Rule

ระบบต้อง

1. โหลด FAQ
2. โหลด Settings
3. สร้าง Prompt
4. ส่งไปยัง OpenAI
5. รับผลลัพธ์
6. ตอบกลับผู้ใช้

---

## FR-06 Admin Command: Add FAQ

Command

```text
/faq

Q: ร้านเปิดกี่โมง
A: เปิดทุกวัน 09:00-18:00
```

Expected Result

Insert FAQ

---

## FR-07 Admin Command: Edit FAQ

Command

```text
/editfaq

id: 3

answer: เปิดทุกวัน 08:00-20:00
```

Expected Result

Update FAQ

---

## FR-08 Admin Command: Delete FAQ

Command

```text
/deletefaq

id: 3
```

Expected Result

Set active = 0

---

## FR-09 Admin Command: Add Rule

Command

```text
/rule

keyword: คืนสินค้า

response: กรุณากรอกแบบฟอร์มคืนสินค้า
```

Expected Result

Insert Rule

---

## FR-10 Admin Command: Statistics

Command

```text
/stats
```

Expected Result

Return

* FAQ Count
* Rule Count
* Chat Count

---

# 6. Non Functional Requirements

---

## Performance

* Response Time < 5 seconds
* Support 100 FAQ records
* Support 10,000 chat logs

---

## Security

* Admin Command ต้องตรวจ role ก่อนเสมอ
* SQL Injection Protection
* Prepared Statement Only
* API Keys ต้องอยู่ใน .env

---

## Reliability

* หาก OpenAI ล่ม
* ต้องตอบ fallback message

ตัวอย่าง

```text
ขออภัย ระบบไม่สามารถให้บริการได้ในขณะนี้
```

---

# 7. Database Design

## users

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100) UNIQUE,
    display_name VARCHAR(255),
    role ENUM('admin','user') DEFAULT 'user',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## bot_settings

```sql
CREATE TABLE bot_settings (
    setting_key VARCHAR(100) PRIMARY KEY,
    setting_value TEXT
);
```

---

## faq

```sql
CREATE TABLE faq (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question TEXT,
    answer TEXT,
    active TINYINT DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NULL
);
```

---

## rules

```sql
CREATE TABLE rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    trigger_text VARCHAR(255),
    response_text TEXT,
    active TINYINT DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## chat_logs

```sql
CREATE TABLE chat_logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100),
    role ENUM('user','assistant'),
    message TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## training_logs

```sql
CREATE TABLE training_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    admin_id INT,
    command_type VARCHAR(50),
    content TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

# 8. AI Prompt Specification

System Prompt Template

```text
You are a customer support assistant.

Company Information

{SETTINGS}

Available FAQ

{FAQ_DATA}

Rules

1. Use FAQ information first.
2. Never invent information.
3. Never guess prices.
4. Never create promotions.
5. If information is unavailable, return fallback message.
6. Respond in Thai language.
7. Keep answers concise.

Customer Message

{USER_MESSAGE}
```

---

# 9. Processing Flow

```text
LINE Message
        │
        ▼
Webhook
        │
        ▼
Load User
        │
        ▼
Save Chat Log
        │
        ▼
Admin ?
        │
 ┌──────┴──────┐
 │             │
YES            NO
 │             │
 ▼             ▼
Process      Check Rules
Command         │
 │              ▼
 ▼         Rule Found?
Reply           │
                │
         ┌──────┴──────┐
         │             │
         YES           NO
         │             │
         ▼             ▼
      Reply       Build Prompt
                        │
                        ▼
                   OpenAI API
                        │
                        ▼
                   Save Log
                        │
                        ▼
                    Reply
```

---

# 10. Project Structure

```text
project/

config/
    database.php
    app.php

app/

    Services/
        UserService.php
        FAQService.php
        RuleService.php
        ChatService.php
        OpenAIService.php
        TrainingService.php

    Repositories/
        UserRepository.php
        FAQRepository.php
        RuleRepository.php
        ChatRepository.php

    Controllers/
        WebhookController.php

webhook/
    line.php

storage/
    logs/

vendor/

.env
```

---

# 11. Acceptance Criteria

System is accepted when:

1. Customer can chat with AI via LINE OA
2. Admin can add FAQ via LINE
3. Admin can edit FAQ via LINE
4. Admin can add Rules via LINE
5. Rules respond without calling AI
6. FAQ is used in AI responses
7. All chats are logged
8. Unauthorized users cannot execute admin commands
9. OpenAI failure is handled gracefully
10. No code changes are required when updating FAQ or Rules
