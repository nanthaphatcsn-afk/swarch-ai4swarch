**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** นันทภัทร์ จั่นสนิท ID: 6970148821
**agent ที่ใช้:** Claude Code
**โมเดล:** Opus 5.5
**แพ็กเกจที่ใช้:** Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-recover-architecture.md` run A (evidence: `system/online-boutique/protos/demo.proto`) | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | อยู่ในขอบเขต | 5 | |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-recover-architecture.md` run B (evidence: `system/online-boutique/protos/demo.proto`, `system/online-boutique/kubernetes-manifests/`) + ขอให้ตอบเป็นภาษาไทย | `prompts/1-recover-architecture.md`, `AGENTS.md` และ `CLAUDE.md` (โหลดอัตโนมัติ), `system/online-boutique/protos/demo.proto`, `system/online-boutique/kubernetes-manifests/`: `README.md`, `kustomization.yaml`, `adservice.yaml`, `cartservice.yaml`, `checkoutservice.yaml`, `currencyservice.yaml`, `emailservice.yaml`, `frontend.yaml`, `loadgenerator.yaml`, `paymentservice.yaml`, `productcatalogservice.yaml`, `recommendationservice.yaml`, `shippingservice.yaml` | อยู่ในขอบเขต (ไฟล์ที่เปิดอ่านอยู่ใน evidence ทั้งหมด) | 4 | agent รัน `ls` บน terminal 1 ครั้ง เพื่อดูรายชื่อไฟล์ใน `kubernetes-manifests/` ซึ่งขัดกับกฎ "Don't run terminal commands" ใน `AGENTS.md` agent รายงานเรื่องนี้เอง คำสั่งนี้แสดงแค่รายชื่อไฟล์ในโฟลเดอร์ที่อนุญาต ไม่ได้เปิดไฟล์นอกขอบเขต และไม่ได้แก้ไขไฟล์ใด |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-write-adr.md` run a, context: none, requirement 3 ข้อ (ส่งอีเมลยืนยันผ่าน MailFast แทน logic จำลองใน emailservice, ให้แต้มสะสมผ่าน loyaltyservice ใหม่, บันทึก analytics "มีการสั่งซื้อ" ให้ทีมการตลาด), language: Thai, output: `outputs/2a-adr.md` | `AGENTS.md` และ `CLAUDE.md` (โหลดอัตโนมัติ), `prompts/2-write-adr.md`, `docs/adr/template.md` | | | ได้ ADR-0002 สถานะ Proposed หัวข้อตรงกับ template agent เลือก event-driven (event `OrderPlaced` ผ่าน message broker) จาก 3 ตัวเลือก ระบุความเสี่ยง (แก้ด้วย idempotent consumer และ outbox pattern) และ open questions ที่ requirement ไม่ได้ให้ไว้โดยไม่กำหนดตัวเลขเอง ข้อมูลระบบปัจจุบันติดป้าย [Inferred] ทั้งหมดเพราะ context: none agent บันทึกไฟล์ที่ `exercise/outputs/2a-adr.md` ตามที่ไฟล์ task กำหนด แทน `outputs/2a-adr.md` ที่ระบุใน prompt |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-write-adr.md` run b, context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`, requirement 3 ข้อเดียวกับ 2-a, language: Thai, output: `outputs/2b-adr.md` | `AGENTS.md` และ `CLAUDE.md` (โหลดอัตโนมัติ), `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | | | ได้ ADR-0002 สถานะ Proposed หัวข้อตรงกับ template บันทึกที่ `exercise/outputs/2b-adr.md` ตามที่ไฟล์ task กำหนด agent พิจารณา 4 ตัวเลือก (sync gRPC, goroutine fire-and-forget, RabbitMQ, Kafka) และเลือก RabbitMQ เพราะมี context จึงอ้างตัวเลขจริงจาก scenario ได้ (MailFast p95 1.5 s เกิน budget 500 ms, rate limit 50/s ต่ำกว่า peak 200 orders/s) และอ้างโค้ด (`order_id` สร้างที่ `main.go` บรรทัด 233) เพื่อออกแบบ loyaltyservice ให้ idempotent ความเสี่ยงที่ event หายเมื่อ RabbitMQ ล่มถูกใส่ไว้ใน Open questions เพราะ checkoutservice ยังไม่มี DB สำหรับ outbox ต่างจาก 2-a ที่ข้อมูลระบบเป็น [Inferred] ทั้งหมด |
| 5 | 3 | | `prompts/3-…` | | | | |
| 6 | | | | | | | |
| 7 | | | | | | | |