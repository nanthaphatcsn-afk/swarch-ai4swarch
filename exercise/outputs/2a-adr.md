> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, `context: none`)

# ADR-0002: Publish an "order placed" event to a message broker for post-order side effects

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

## Context and problem

ADR นี้เขียนจาก requirement เพียงอย่างเดียว (`context: none`) จึงไม่ได้อ่านโค้ดหรือ manifest ของ Online Boutique ข้อเท็จจริงเกี่ยวกับระบบปัจจุบันทั้งหมดจึงเป็น **[Inferred]** หรือ "Not evidenced in the provided files"

Requirement ใหม่: เมื่อมีการสั่งซื้อสำเร็จ ระบบต้องทำงานเพิ่ม 3 อย่าง

| # | งาน | ผู้รับผิดชอบ | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อผ่าน MailFast (ผู้ให้บริการอีเมลภายนอกรายใหม่) | `emailservice` | แทนที่ logic จำลองเดิมใน `emailservice` |
| 2 | มอบแต้มสะสม (loyalty points) | `loyaltyservice` (ใหม่ ยังต้องสร้าง) | |
| 3 | บันทึกเหตุการณ์ analytics "มีการสั่งซื้อ" | ทีมการตลาด | ยังไม่ระบุว่าส่งไปที่ระบบใด |

ปัญหาที่ต้องตัดสินใจ: **service ที่ทำการสั่งซื้อควรแจ้งงานทั้ง 3 นี้อย่างไร** ให้การสั่งซื้อไม่ช้าลงหรือล้มเหลวเพราะ dependency ใหม่ (โดยเฉพาะ MailFast ซึ่งเป็นระบบภายนอกที่เราควบคุมไม่ได้) และให้เพิ่มผู้รับเหตุการณ์รายใหม่ได้ง่ายในอนาคต

สมมติฐาน (ยังไม่ได้ยืนยันจากไฟล์):

- **[Inferred]** มี service หนึ่ง (น่าจะเป็น `checkoutservice`) ที่ประสานการสั่งซื้อและเรียก `emailservice` แบบ synchronous อยู่แล้ว
- **[Inferred]** service ภายในสื่อสารกันด้วย gRPC
- **[Inferred]** ปัจจุบันยังไม่มี message broker ในระบบ

## Decision drivers

เรียงจากสำคัญที่สุด (ลำดับนี้เป็นข้อเสนอ ยังไม่ได้รับการยืนยันจากผู้มีส่วนได้ส่วนเสีย):

- **Availability ของการสั่งซื้อ:** การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือระบบ analytics ล่ม
- **Performance (latency ของการสั่งซื้อ):** ไม่ควรเพิ่ม latency ที่ผู้ใช้รู้สึกได้จากการเรียก dependency เพิ่ม 2–3 ตัว
- **Reliability ของ side effect:** อีเมลและแต้มสะสมต้องไม่หาย และไม่ถูกทำซ้ำ (เช่น แต้มไม่ถูกให้สองครั้ง)
- **Modifiability:** เพิ่มผู้รับเหตุการณ์ใหม่ (เช่น analytics ของทีมอื่น) ได้โดยไม่ต้องแก้ service ที่ทำการสั่งซื้อ
- **Cost / ความซับซ้อนในการดูแลระบบ:** โครงสร้างพื้นฐานใหม่ต้องคุ้มค่ากับที่ทีมดูแลได้

## Considered options

1. **Synchronous fan-out:** service ที่ทำการสั่งซื้อเรียก `emailservice` (ซึ่งเรียก MailFast), `loyaltyservice` และระบบ analytics ตรง ๆ ทีละตัวด้วย gRPC/HTTP ภายใน request เดียวกัน
2. **Event-driven ผ่าน message broker:** service ที่ทำการสั่งซื้อ publish event `OrderPlaced` ครั้งเดียวไปที่ message broker (เช่น RabbitMQ, Kafka หรือ Pub/Sub) แล้ว `emailservice`, `loyaltyservice` และตัวรับ analytics ต่าง subscribe แยกกัน
3. **Hybrid:** เรียก `emailservice` และ `loyaltyservice` แบบ synchronous (ให้ได้ผลทันที) แต่ส่ง analytics แบบ asynchronous (fire-and-forget หรือผ่าน queue)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous fan-out | ง่ายที่สุด ไม่ต้องมีโครงสร้างพื้นฐานใหม่; ใช้รูปแบบเดียวกับที่ระบบน่าจะใช้อยู่ **[Inferred]**; debug และ trace ได้ตรงไปตรงมา; รู้ผลลัพธ์ทันที | latency ของการสั่งซื้อเพิ่มตามผลรวมของทุก call; ถ้า MailFast ช้าหรือล่ม การสั่งซื้อจะช้าหรือล้มด้วย (availability ลดลง); service ที่ทำการสั่งซื้อต้องรู้จักผู้รับทุกราย (coupling สูง, modifiability ต่ำ); ต้องเขียน retry/timeout เองทุก call |
| 2. Event-driven ผ่าน message broker | การสั่งซื้อไม่ขึ้นกับ availability ของผู้รับ; latency เพิ่มแค่การ publish ครั้งเดียว; เพิ่มผู้รับใหม่ได้โดยไม่แก้ผู้ส่ง (modifiability สูง); broker ช่วยเรื่อง retry และ buffer เมื่อ MailFast ล่มชั่วคราว | ต้องติดตั้งและดูแล message broker ใหม่ (cost, ความซับซ้อน); เป็น eventual consistency ผู้ใช้อาจได้อีเมล/แต้มช้าไปเล็กน้อย; ต้องจัดการ duplicate delivery (idempotency) และการ publish ให้สอดคล้องกับการบันทึกคำสั่งซื้อ (เช่น outbox pattern); trace ข้าม service ยากขึ้น |
| 3. Hybrid | analytics ไม่กระทบการสั่งซื้อ; อีเมล/แต้มได้ผลทันที; ไม่จำเป็นต้องมี broker ถ้า analytics ใช้ fire-and-forget | ยังมีปัญหาเรื่อง MailFast ช้าหรือล่มทำให้การสั่งซื้อช้าหรือล้มเหมือน option 1; มีสองรูปแบบการสื่อสารให้ดูแล; fire-and-forget อาจทำ event analytics หาย; เพิ่มผู้รับใหม่ยังต้องแก้ผู้ส่ง |

## Decision

เลือก **Option 2: Event-driven ผ่าน message broker**

- service ที่ทำการสั่งซื้อ publish event `OrderPlaced` (เช่น order id, user/email, รายการสินค้า, ยอดรวม) หลังบันทึกคำสั่งซื้อสำเร็จ
- `emailservice` subscribe แล้วเรียก MailFast; `loyaltyservice` subscribe แล้วคำนวณและบันทึกแต้ม; ตัวรับ analytics subscribe แล้วส่งต่อให้ระบบของทีมการตลาด
- ผู้รับแต่ละรายมี queue/subscription ของตัวเอง ทำให้ผู้รับที่ล่มไม่กระทบผู้รับรายอื่น

เหตุผล: เป็นตัวเลือกเดียวที่ตอบ driver อันดับ 1–2 (availability และ latency ของการสั่งซื้อ) ได้ครบ เพราะตัดการพึ่งพา MailFast ซึ่งเป็นระบบภายนอกออกจากเส้นทางการสั่งซื้อ และตอบ modifiability ได้ดีที่สุดเพราะงาน 3 ประเภทนี้เป็น "ผู้รับเหตุการณ์เดียวกัน" ตามธรรมชาติ ราคาที่ต้องจ่ายคือโครงสร้างพื้นฐานใหม่และ eventual consistency

การเลือกผลิตภัณฑ์ broker (RabbitMQ, Kafka, Pub/Sub ฯลฯ) ยังไม่ตัดสินใน ADR นี้ เพราะไม่มีข้อมูลเรื่อง platform, ปริมาณ event และทักษะของทีม (ดู Open questions)

## Consequences

- **Positive:** การสั่งซื้อยังสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม (quality attribute: availability)
- **Positive:** latency ของการสั่งซื้อเพิ่มเพียงการ publish หนึ่งครั้ง ไม่ใช่ผลรวมของ 3 call (quality attribute: performance)
- **Positive:** เพิ่มหรือถอดผู้รับเหตุการณ์ได้โดยไม่ต้องแก้ service ที่ทำการสั่งซื้อ; เปลี่ยนผู้ให้บริการอีเมลในอนาคตกระทบแค่ `emailservice` (quality attribute: modifiability)
- **Positive:** broker เก็บ event ไว้ระหว่างที่ MailFast ล่ม แล้วส่งต่อเมื่อกลับมา (quality attribute: reliability / fault tolerance)
- **Negative:** ต้อง deploy, monitor และดูแล message broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** อีเมลและแต้มสะสมเกิดแบบ eventual ผู้ใช้อาจเห็นแต้มช้ากว่าการสั่งซื้อเล็กน้อย (quality attribute: consistency / usability)
- **Negative:** การ trace คำสั่งซื้อหนึ่งรายการข้ามหลาย service ยากขึ้น (quality attribute: observability / testability)
- **Risks and mitigations:**
  - *Duplicate delivery ทำให้ส่งอีเมลซ้ำหรือให้แต้มซ้ำ* → ผู้รับทุกรายต้อง idempotent โดยใช้ order id เป็น key
  - *บันทึกคำสั่งซื้อสำเร็จแต่ publish event ไม่สำเร็จ (หรือกลับกัน)* → ใช้ transactional outbox หรือ retry ฝั่งผู้ส่ง **[Inferred]** ขึ้นกับว่าระบบเก็บคำสั่งซื้อไว้ที่ใด ซึ่ง Not evidenced in the provided files
  - *MailFast ล่มนานหรือปฏิเสธ request* → retry แบบ exponential backoff และ dead-letter queue พร้อม alert
  - *event มีข้อมูลส่วนบุคคล (อีเมลลูกค้า) ไปถึงทีมการตลาด* → กำหนด schema ของ event ให้ analytics ได้เฉพาะ field ที่จำเป็น
  - *Schema ของ event เปลี่ยนแล้วผู้รับพัง* → ทำ versioning ของ schema (เช่น protobuf พร้อมกฎ backward compatibility)

## Validation

- **Fault injection:** ปิด MailFast (หรือ mock ให้ timeout) และ `loyaltyservice` แล้วยืนยันว่าการสั่งซื้อยังสำเร็จ และอีเมล/แต้มถูกส่งครบหลังระบบกลับมา
- **Load test:** เปรียบเทียบ latency ของการสั่งซื้อ (เช่น p95/p99) ก่อนและหลังเพิ่ม event; เป้าหมายตัวเลข: Not evidenced in the provided files
- **Metrics:** queue depth/consumer lag, จำนวนข้อความใน dead-letter queue, เวลาตั้งแต่สั่งซื้อถึงส่งอีเมลสำเร็จ, จำนวนแต้มที่ถูกให้ซ้ำ (ควรเป็น 0)
- **Review date:** ทบทวน ADR นี้หลังใช้งานจริงระยะหนึ่ง วันที่: Not evidenced in the provided files

## Open questions

- ใครคือ Deciders ของ ADR นี้?
- service ใดเป็นผู้ประสานการสั่งซื้อในปัจจุบัน และเรียก `emailservice` อย่างไร (sync/async, protocol)? ระบบมี message broker อยู่แล้วหรือไม่?
- เป้าหมาย latency และ availability ของการสั่งซื้อเป็นตัวเลขเท่าใด?
- ปริมาณคำสั่งซื้อ (ปกติและช่วง peak) เป็นเท่าใด? ใช้เลือกชนิดและขนาดของ broker
- อีเมลยืนยันต้องถึงผู้ใช้ภายในเวลาเท่าใด? แต้มสะสมต้องเห็นทันทีหลังสั่งซื้อหรือไม่? (ถ้าต้องทันที อาจต้องทบทวน Option 3)
- MailFast มี API แบบใด, rate limit, SLA, ค่าใช้จ่ายต่ออีเมล และรองรับ idempotency key หรือไม่?
- กฎการคำนวณแต้มสะสมคืออะไร และ `loyaltyservice` จะเก็บข้อมูลไว้ที่ data store ใด?
- ทีมการตลาดต้องการรับ event analytics ไปที่ระบบใด (เครื่องมือ analytics ภายนอก, data warehouse ฯลฯ) และต้องการ field ใดบ้าง? มีข้อกำหนดด้าน privacy/PDPA หรือไม่?
- platform ที่ deploy (เช่น Kubernetes บน cloud ใด) และทักษะของทีมกับ broker แต่ละชนิดเป็นอย่างไร?
- มีงบประมาณหรือข้อจำกัดด้านโครงสร้างพื้นฐานใหม่หรือไม่?
