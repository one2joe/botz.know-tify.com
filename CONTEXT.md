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

### Admin
ผู้ใช้ LINE OA ที่มี `role = 'admin'` — คนแรกต้องส่ง `init <ADMIN_SETUP_CODE>` ผ่าน LINE Chat เพื่อยืนยันตัวตน หลังจากนั้น Admin คนแรกสามารถสั่ง `/addadmin` และ `/removeadmin` เพื่อจัดการ Admin คนอื่นได้

### ADMIN_SETUP_CODE
รหัสลับใน `.env` ใช้สำหรับ init admin คนแรกผ่าน LINE Chat — ถ้าลบออกจาก .env แล้ว จะไม่สามารถตั้ง admin คนแรกผ่าน chat ได้อีก

### Customer / User
ผู้ใช้ LINE OA ทั่วไป — `role = 'user'` — ใช้ LINE Chat กับ AI ได้อย่างเดียว ไม่มีสิทธิ์สั่ง admin command

### Command
Admin command เช่น `/faq`, `/editfaq`, `/rule`, `/stats` — parser อยู่ใน `CommandService`

### Rule
Keyword-based matching — ถ้าเจอ trigger_text ในข้อความ → ตอบ response_text ทันที โดยไม่เรียก LLM

### Zen API
OpenAI-compatible API Gateway — ใช้ key เดียวเข้าถึงหลาย model ผ่าน `https://api.zen.run/v1`

### Timezone
ระบบใช้ Asia/Bangkok (UTC+7) — ทุกอย่าง: DB, log, LINE timestamp ภาษาไทยล้วน ตลอดทั้งระบบ ผู้ใช้ ลูกค้า แอดมิน คนไทยทั้งหมด

### Entry Point
`webhook.php` ที่ project root — ไม่มี `public/` directory, ทั้ง dev (Herd) และ prod (Apache) ใช้ไฟล์เดียวกัน

### API Keys
เก็บ key ของ LLM provider (Zen) ในตาราง `api_keys` — มี primary ตัวหลัก + backups สำหรับ auto failover ถ้าตัวหลักเรียกไม่ได้

### Non-text messages
LINE รองรับ message types หลายประเภท (image, sticker, location, ฯลฯ) — หาก user ส่งสิ่งเหล่านี้, bot จะ reply ว่า "ขออภัย ระบบรองรับเฉพาะข้อความเท่านั้นค่ะ" และไม่ log ลง chat_logs
