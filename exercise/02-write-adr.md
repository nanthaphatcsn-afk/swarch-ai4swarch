**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ ไม่ได้อ่าน scenario จึงย้ายตัวเลขทั้งหมดไปเป็น Open questions | ใช้ เช่น budget 500 ms, MailFast p95 1.5 s และ 50 emails/s, peak 200 orders/s แล้วใช้ตัวเลขเหล่านี้ตัดตัวเลือก |
| ทางเลือกที่พิจารณา (ระบุ) | sync fan-out, broker, hybrid เลือก broker แต่ไม่ได้ระบุว่าใช้ตัวไหน | sync gRPC, goroutine, RabbitMQ, Kafka เลือก RabbitMQ |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: idempotent ด้วย order id (ระดับหลักการ) / rate limit: ไม่ได้จัดการ | แต้ม: idempotent ด้วย `order_id` + unique constraint / rate limit: consumer จำกัด ≤ 50/s และใช้ queue พักงานช่วง peak |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี ข้อสมมติทุกข้อติดป้าย [Inferred] | มี: ระบุ frontend เป็น "Go", อ้างว่าใช้ Locust, เขียนว่าทีมไม่เคยใช้ Kafka ซึ่งไม่มีใน scenario |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | ลบคำว่า "(Locust)" ออกจากหัวข้อ Validation | แม้ Locust อาจถูกต้องในความเป็นจริง แต่ไม่มีหลักฐานในไฟล์ context (`docs/scenario.md`, ADR-0001, `main.go`) จึงลบออก เพื่อให้ ADR อ้างเฉพาะสิ่งที่ตรวจสอบได้ |
| 2 | แก้ข้อเสียของ Kafka จาก "ไม่มีใครในทีมเคยใช้" เป็น "ไม่ทราบว่าทีมมีประสบการณ์กับ Kafka หรือไม่" และเพิ่มเรื่องนี้ใน Open questions | scenario ระบุแค่ว่ามี 2 คนเคยใช้ RabbitMQ ไม่ได้บอกเรื่อง Kafka AI จึงเติมข้อมูลเอง ต้องถามทีมก่อน แต่ยังคงเลือก RabbitMQ เพราะ 200 events/s ไม่จำเป็นต้องใช้ Kafka |
| 3 | ลบ "Go" ออกจาก container `frontend` ใน diagram | ไฟล์ context มีแค่โค้ดของ `checkoutservice` ไม่มีไฟล์ของ frontend ภาษาของ frontend จึงไม่มีหลักฐาน แม้ในระบบจริงอาจเป็น Go ก็ตาม |