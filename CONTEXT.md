# CONTEXT — LINE OA AI Assistant

## Glossary

### Repository
ตัวกลางระหว่าง Service (business logic) กับ Database — มีหน้าที่ query DB และแปลงผลลัพธ์เป็น Model object ไม่มี business logic

### Service
Business logic layer — เรียก Repository เพื่ออ่าน/เขียนข้อมูล, ตัดสินใจ, เรียก LLM, ฯลฯ ไม่รู้ว่าข้อมูลเก็บใน DB ยังไง

### Container (Pimple)
Service locator สำหรับ dependency injection — register factory functions, service เรียก `$container->get(...)` เพื่อขอ dependency

### LLMProvider
Interface สำหรับ AI provider — ปัจจุบันใช้ Zen API ผ่าน `ZenProvider` implements `LLMProvider`

### Migration
SQL script ที่ version controlled — รันผ่าน PHP migrate script, track สถานะใน `migrations` table

### ลูกค้า (Customer)
ผู้ใช้ LINE OA ทั่วไป — `role = 'user'` — ใช้ LINE Chat กับ AI ได้อย่างเดียว ไม่มีสิทธิ์สั่ง admin command
_ไม่ใช้_: User, ผู้ใช้

### แอดมินหลัก (Super Admin)
Admin ระดับสูงสุด — คนแรกที่ส่งข้อความถึงบอทหลัง Deploy จะกลายเป็นแอดมินหลักอัตโนมัติ — มีสิทธิ์ทำได้ทุกอย่าง รวมถึงเชิญ ถอดสิทธิ์ และจัดการ API Keys
_ไม่ใช้_: Super Admin, Root Admin

### ผู้ช่วยแอดมิน (Helper Admin)
Admin ที่ได้รับเชิญจากแอดมินหลัก — `role = 'helper_admin'` — ทำได้ทุกอย่างยกเว้น: เชิญ admin ใหม่, ถอดสิทธิ์ admin คนอื่น, จัดการ API Keys
_ไม่ใช้_: Sub Admin, Co-Admin

### Invite Code
รหัส `INVITE_xxx` ที่แอดมินหลักสร้างผ่าน `/เชิญ` — ฝังใน LINE URL (`line.me/R/oaMessage/@340bzlph/?text=INVITE_xxx`) — ผู้รับกดลิงก์ → LINE เปิดแชทบอท → ข้อความ pre-filled → ส่ง → auto-activate เป็นผู้ช่วยแอดมิน — อายุ 24 ชม., ใช้ครั้งเดียว
_ไม่ใช้_: Accept code, Invitation code

### API Keys
Key ของ LLM provider (Zen) เก็บในตาราง `api_keys` — ระบบจัดการ primary/backup อัตโนมัติ (key ที่เพิ่มล่าสุด = primary, ก่อนหน้า = backup)

### Rule
Keyword-based matching — ถ้าเจอ trigger_text ในข้อความ → ตอบ response_text ทันที โดยไม่เรียก LLM

### Command
Admin command ภาษาไทยล้วน เช่น `/เพิ่มfaq`, `/ลบfaq`, `/เชิญ`, `/รีวิว`, `/เพิ่มคีย์` — parser อยู่ใน `CommandService`
_ไม่ใช้_: English aliases

### Review (/รีวิว)
ระบบประเมินบทสนทนาที่ยังไม่ได้รีวิว (`is_reviewed = 0`) — LLM ประเมินทีละ 1 ประเด็น — ถ้าดี → auto-pass, ถ้าไม่ดี → ให้แอดมินตัดสิน → แอดมินสอนกฎ → ปรับปรุง FAQ

### Eval Criteria
เกณฑ์ที่ใช้ในการประเมินบทสนทนา — มี 6 ข้อเริ่มต้น (default) — แอดมินเพิ่ม/แก้/ลบได้ผ่าน `/เกณฑ์ประเมิน` — ใช้ใน `/รีวิว` เพื่อให้ LLM มีแนวทางประเมิน

### Conversation Context (C2)
วิธีการส่งประวัติการสนทนาให้ LLM — เก็บ 2 รอบล่าสุดจาก `chat_logs` — LLM ตัดสินว่าเกี่ยวข้องกันไหม ถ้าเกี่ยว → ขยายเป็น 3 รอบ — ไม่มีตาราง `conversation_summaries`, ไม่มี LLM call เพื่อสรุป

### AI Clarification
พฤติกรรมของ LLM เมื่อลูกค้าส่งข้อความคลุมเครือ — LLM จะถามกลับเพื่อขอรายละเอียดเพิ่ม ถ้าชัดเจนแล้วจึงตอบตรงๆ — ไม่มี state machine แยก

### Zen API
OpenAI-compatible API Gateway — ใช้ key เดียวเข้าถึงหลาย model ผ่าน `https://api.zen.run/v1`

### Timezone
ระบบใช้ Asia/Bangkok (UTC+7) — ทุกอย่าง: DB, log, LINE timestamp

### Entry Point
`webhook.php` ที่ project root — ไม่มี `public/` directory, ทั้ง dev (Herd) และ prod (Apache) ใช้ไฟล์เดียวกัน

### Non-text messages
LINE รองรับ message types หลายประเภท (image, sticker, location, ฯลฯ) — หาก user ส่งสิ่งเหล่านี้, bot จะ reply ว่า "ขออภัย ระบบรองรับเฉพาะข้อความเท่านั้นค่ะ" และไม่ log ลง chat_logs

### Interactive Command Flow
รูปแบบการทำงานของ Command — แอดมินพิมพ์แค่ `/คำสั่ง` โดยไม่ต้องมีข้อมูลต่อท้าย → บอทจะถามทีละขั้นตอน (แยกหลายบอลลูน, ไม่เกิน 5) — คำสั่งลบทุกประเภทต้องถามยืนยันก่อน
_ไม่ใช้_: AI-guided mode
