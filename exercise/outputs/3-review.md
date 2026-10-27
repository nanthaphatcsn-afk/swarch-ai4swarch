> Reviewer: Claude Code (model: Claude Opus 5.5, `claude-opus-5-5`)

# Review: ADR-0002 Publish an "OrderPlaced" event to RabbitMQ

แหล่งที่อ่าน: `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md`, `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 (Performance) | ลูกค้า | กด place order | peak 11.11, 200 orders/s, MailFast p99 8 s | `PlaceOrder` ตอบกลับหลัง publish `OrderPlaced` สำเร็จหรือหมด timeout | p95 ≤ 500 ms (เหลือ budget ประมาณ 200 ms จาก 300 ms วันนี้) |
| QA2 (Availability + correctness) | RabbitMQ node / ทั้ง cluster | ล่มหรือ publish confirm ไม่กลับมาระหว่าง checkout | peak 200 orders/s | order ยังสำเร็จ; event ไม่หาย | checkout success rate ไม่ลดลง; order ที่ไม่ได้แต้มภายใน 60 s = 0; แต้มซ้ำ = 0 |
| QA3 (External constraint) | MailFast | ช้า (p99 8 s) และ fail 0.5 % | peak 200 orders/s นาน *T* วินาที | email consumer คุม rate และ retry | ส่ง ≤ 50 emails/s; ไม่มีอีเมลซ้ำ; backlog ระบายหมดภายในประมาณ 3 × *T* วินาทีหลัง peak |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | เมื่อ publish ล้มเหลว ADR ให้ order สำเร็จและพึ่ง "retry buffer" ใน `checkoutservice` ซึ่งเป็น buffer ในหน่วยความจำ — คือจุดอ่อนเดียวกับที่ใช้ตัด Option 2 ทิ้ง ("งานสูญหายเมื่อ pod restart") | event หาย = ลูกค้าไม่ได้แต้ม ขัด R-12 ("within 1 minute, exactly once") โดยตรง; เหตุผลการเลือกจึงไม่สอดคล้องกันเอง |
| **Risk** | retry ไปยัง MailFast (fail 0.5 %, p99 8 s) ไม่มีกลไกกันอีเมลซ้ำ — consumer timeout ก่อน MailFast ตอบ แล้ว redeliver จะส่งซ้ำ | ADR ทำ idempotency ให้ loyalty แต่ไม่ทำให้ email; อีเมลยืนยันซ้ำกระทบลูกค้าโดยตรง (ADR ยกเป็น Open question แต่ไม่มี mitigation) |
| **Risk** | ถ้า `loyaltyservice` ล่มเกิน 1 นาที event จะรอใน queue ได้ แต่ SLO 60 s พังแน่นอน — failure-injection test ในส่วน Validation ตรวจแค่ "ประมวลผลครบเมื่อกลับมา" | durable queue แก้ availability ของ checkout แต่ไม่แก้ timeliness ของ loyalty; ต้องกำหนด availability target ของ `loyaltyservice` เอง |
| **Sensitivity point** | publish timeout + publisher confirms บน quorum queue (ต้อง replicate ให้ majority ก่อน confirm) | ค่านี้กิน budget 200 ms โดยตรง; ADR ระบุแค่ "timeout สั้น" และ "ไม่กี่ ms [Inferred]" โดยไม่มีตัวเลข |
| **Sensitivity point** | ความยาว peak *T* และ prefetch/จำนวน consumer ของ email | backlog = 150 × *T*, drain ประมาณ 3 × *T* (คำนวณถูก); กำหนดขนาด disk ของ RabbitMQ และความล่าช้าของอีเมล |
| **Trade-off point** | "publish fail → order ยังสำเร็จ" | ได้ availability ของ checkout แต่เสีย correctness ของ loyalty; ถ้าเลือก outbox จะได้ correctness แต่เพิ่ม DB + latency + operability cost |
| **Trade-off point** | เพิ่ม RabbitMQ cluster (stateful) | ได้ modifiability (action ใหม่ไม่แตะ checkout) และ buffer ช่วง peak แต่เสีย operability/cost สำหรับทีม 4 คน และ observability ข้าม async |
| **Non-risk** | queue แยกต่อ action | backlog ของ email (150/s ช่วง peak) ไม่หน่วง loyalty — สำคัญต่อ SLO 60 s |
| **Non-risk** | idempotency ของ loyalty: `order_id` + unique constraint ใน transaction เดียวกับการเพิ่มแต้ม | เปลี่ยน at-least-once เป็น effectively exactly once ได้จริง และมี redelivery test ยืนยัน |
| **Non-risk** | ตัด Kafka ด้วยเหตุผล 200 events/s + ทีมเคยใช้ RabbitMQ 2 คน | ตรงกับ constraint ด้าน team/ops ใน scenario |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:**
  - **Hybrid:** เรียก `loyaltyservice` แบบ sync gRPC (บริการภายใน คุม latency ได้, idempotent ด้วย `order_id`) และใช้ queue เฉพาะ email/analytics — แก้ปัญหา event หายสำหรับ requirement ที่เข้มที่สุด
  - **Outbox ที่ไม่ต้องให้ checkout มี DB ใหม่เต็มรูป** หรือ reconciliation job ที่เทียบ order กับแต้มย้อนหลัง — ADR เลื่อนไปเป็น "ระยะยาว" ทั้งที่เป็นแกนของ requirement
  - broker self-hosted ตัวอื่นที่เบากว่า (เช่น NATS JetStream) ไม่ได้ถูกเทียบ
- **Consequence ที่ไม่ได้ระบุ:** event มี email ลูกค้า แต่ fanout ไปทุก consumer รวม analytics (การเปิดเผยข้อมูลส่วนบุคคลเกินจำเป็น); retry ที่ fail 0.5 % กิน rate limit 50/s ช่วง peak
- **Claim ที่อ่อน/ไม่มีหลักฐาน:**
  - "เป็นตัวเลือกเดียวที่ตอบ driver 1–5 พร้อมกัน" — ไม่จริงจนกว่าจะแก้กรณี publish fail (driver 3)
  - "publish ใช้ไม่กี่ ms" — [Inferred] โดยไม่มีตัวเลขสำหรับ quorum queue + confirms
  - Validation "p100 ≤ 60 s" ทดสอบเฉพาะ happy path; ไม่มีการทดสอบ RabbitMQ ล่มระหว่าง load test (QA2)

## 4. Verdict: **Accept with changes**

1. **แก้เส้นทาง publish failure** ให้ไม่ทำ event หาย: เลือก outbox, hybrid (sync loyalty) หรือ reconciliation job อย่างใดอย่างหนึ่ง และเอา "retry buffer ในหน่วยความจำ" ออก เพื่อให้เหตุผลที่ตัด Option 2 ยังใช้ได้
2. **ใส่ตัวเลข**: publish timeout และ latency ของ confirm บน quorum queue ต้องอยู่ใน budget 200 ms และเพิ่ม test "ปิด RabbitMQ ระหว่าง load test 200 orders/s"
3. **ทำ email ให้ idempotent** (dedupe ด้วย `order_id` ฝั่ง `emailservice` หรือ idempotency key ของ MailFast) และกำหนด availability target ของ `loyaltyservice` ให้สอดคล้องกับ SLO 60 s
