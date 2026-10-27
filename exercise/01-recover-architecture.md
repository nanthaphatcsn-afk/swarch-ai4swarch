**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR="emailservice:5000"`) | ✅ | `src/checkoutservice/main.go` อ่าน `EMAIL_SERVICE_ADDR` แล้วเปิด connection ด้วย `mustConnGRPC` และ `sendOrderConfirmation` เรียก `SendOrderConfirmation` จริงใน `PlaceOrder` ถ้าล้มเหลวจะแค่ `log.Warnf` order ไม่ล้ม |
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` (`CART_SERVICE_ADDR="cartservice:7070"`) | ✅ | `main.go` เรียก `GetCart` (ใน `getUserCart`) และ `EmptyCart` (ใน `emptyUserCart`) ตรงกับ label "Gets and empties cart" แต่ผลของ `EmptyCart` ถูกทิ้งด้วย `_ =` |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR="paymentservice:50051"`) | ✅ | `main.go` ใน `chargeCard` เรียก `Charge` และนำ `transaction_id` มาใช้ต่อใน `PlaceOrder` |
| `checkoutservice` → `shippingservice` | `checkoutservice.yaml` (`SHIPPING_SERVICE_ADDR="shippingservice:50051"`) | ✅ | `main.go` เรียกทั้ง `GetQuote` (ใน `quoteShipping`) และ `ShipOrder` (ใน `shipOrder`) ตรงกับ label "Quotes and ships order" |
| `frontend` → `shippingservice` | `frontend.yaml` (`SHIPPING_SERVICE_ADDR="shippingservice:50051"`) | ❓ | `src/frontend/main.go` อ่าน env var และเปิด connection ไป shipping จริง แต่การเรียก RPC อยู่ในไฟล์ handler อื่นที่ไม่ได้ตรวจ จึงยืนยันได้แค่ว่ามี connection ไม่ได้ยืนยันว่าเรียก `GetQuote` ตาม label ใน diagram |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **dependency ที่ขาดไป (tracing):** ทั้ง `checkoutservice` และ `frontend` เปิด gRPC connection ไปที่ `COLLECTOR_SERVICE_ADDR` เมื่อ `ENABLE_TRACING=1` (ใน `initTracing()`) แต่ diagram ไม่มี collector เลย เพราะ AI ดูแค่ manifest ไม่ได้ดูโค้ด
  - **dependency ที่ขาดไป:** `frontend.yaml` มี `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ AI ไม่วาดเส้น `frontend` → `shoppingassistantservice` เพราะไม่มี manifest ของ service นี้ในโฟลเดอร์
  - **container ที่น่าสงสัย:** `loadgenerator` ถูกวาดไว้ใน `System_Boundary` เหมือนเป็นส่วนหนึ่งของระบบ ทั้งที่ถูก comment ออกจาก `kustomization.yaml` และเป็นเพียงเครื่องมือสร้าง traffic ทดสอบ
  - **label ที่เป็นการอนุมาน:** ชื่อ operation บนทุกเส้น `Rel` เช่น "Gets shipping quote" อนุมานจากชื่อ RPC ใน proto ไม่ได้มาจาก env var แต่ใน diagram ไม่ได้ติดป้าย [Inferred]

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   จาก `.proto` อย่างเดียว agent ไม่รู้ว่า **ใครเรียกใคร** Run A จึงมีเส้น dependency 0 เส้น ส่วน Run B มี 17 เส้น เพราะ proto ประกาศแค่ RPC ที่ฝั่ง server เปิดให้เรียก ไม่มีข้อมูลฝั่ง client ซึ่งอยู่ใน env var `*_ADDR` ของ manifest นอกจากนี้ agent ยังมองไม่เห็น component ที่ไม่ได้เป็น gRPC server เช่น `frontend` (จุดเข้าระบบผ่าน HTTP), `redis-cart` และ `loadgenerator` เพราะ component เหล่านี้ไม่ได้อยู่ในสัญญาของ interface

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   แผนภาพ B แสดงเส้น `Rel` ทั้ง 17 เส้นพร้อมชื่อ operation ราวกับเป็นข้อเท็จจริง ทั้งที่หลักฐานของแต่ละเส้นมีแค่ env var ที่ตั้งค่าไว้ และชื่อ operation ก็อนุมานจากชื่อ RPC โดยไม่มีป้าย [Inferred] สมาชิกใหม่จึงอาจเชื่อว่าทุกเส้นเป็นการเรียกที่เกิดขึ้นจริงและรู้แน่ว่าเรียก RPC ไหน ถ้าเอาแผนภาพนี้ไปวิเคราะห์ผลกระทบเวลาแก้ service ใด service หนึ่ง ก็จะประเมินผิดได้