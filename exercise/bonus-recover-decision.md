**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | checkout เรียก 6 service แบบ sync | Evidenced | Evidenced | `src/checkoutservice/main.go` มี `mustConnGRPC` 6 ครั้งสำหรับ service ของร้าน อีก 1 ครั้งเป็น collector ของ tracing (`initTracing`) และอีก 1 ครั้งเป็นตัวนิยามฟังก์ชัน |
| 2 | ไม่มี timeout ที่ใช้งานจริง | Evidenced | Evidenced | `src/checkoutservice/main.go` ใน `mustConnGRPC` รับ context ที่มี timeout ไว้ด้วย `_` (ทิ้ง) และไม่ได้ส่งให้ `grpc.NewClient` และใน `convertCurrency` เรียก `Convert(context.TODO(), ...)` แทน `ctx` ของ request |
| 3 | ไม่มี message broker | Evidenced (โดยการไม่พบ) | Evidenced เฉพาะ 2 ไฟล์ที่อ่าน | ส่วน `import` ของ `src/checkoutservice/main.go` และ `src/frontend/main.go` ไม่มี library ของ broker (amqp, kafka, pubsub) แต่ frontend มี handler อยู่ในไฟล์อื่นที่ไม่ได้อ่าน (เช่น `placeOrderHandler`, `chatBotHandler`) จึงสรุปได้แค่ว่า "ไม่พบในไฟล์ที่อ่าน" ไม่ใช่ "ทั้งระบบไม่มี broker" |
| 4 | เหตุผลหลักคือความเรียบง่ายสำหรับ demo ขนาดเล็ก | Inferred | Inferred | ส่วนที่ว่า "เป็น demo" มีหลักฐาน คือ import path `github.com/GoogleCloudPlatform/microservices-demo/...` ใน `main.go` แต่ไม่มี comment หรือเอกสารใดบอกว่าเลือก gRPC + proto ไฟล์เดียว **เพราะ** ความเรียบง่าย ส่วนที่เป็นเหตุผลจึงยังเป็นการอนุมาน |
| 5 | เลือก sync เพราะ checkout ต้องได้ผลทันทีใน request เดียว | Inferred | Inferred (สมเหตุสมผล) | ใน `PlaceOrder` ต้องได้ `txID` จาก `chargeCard` ก่อน แล้วจึงได้ `shippingTrackingID` จาก `shipOrder` เพื่อใส่ใน `OrderResult` ที่ตอบกลับ โค้ดจึงสอดคล้องกับเหตุผลนี้ แต่ไม่ได้เขียนไว้ว่านี่คือเหตุผลของการเลือก |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ **17** ตามที่นิสิตตรวจ **ตรวจ 3 ข้อ ยืนยันได้ทั้ง 3 ข้อ แต่ข้อ "ไม่มี message broker" ต้องจำกัดขอบเขตเป็น "เฉพาะไฟล์ที่อ่าน"**

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี

   ในตาราง Claim check ไม่พบข้อที่เป็นการเดาล้วน ๆ แต่ติดป้าย Evidenced แต่มี 2 จุดที่เลยหลักฐานไป จุดแรกคือข้อ "ไม่มี message broker" ซึ่งเป็นจริงเฉพาะในไฟล์ที่อ่าน แต่ frontend มี handler อยู่ในไฟล์อื่นที่ไม่ได้อ่าน จุดที่สองคือในหัวข้อ Decision มีประโยค "การใช้ gRPC ทำให้ใส่ `otelgrpc` และ health service ได้สม่ำเสมอ" ที่เขียนเป็นเหตุผลของการเลือกโดยไม่มีป้าย [Inferred] ทั้งที่โค้ดพิสูจน์ได้แค่ว่าระบบ**ใช้** otelgrpc ไม่ได้พิสูจน์ว่าเป็น**เหตุผล**ที่เลือก gRPC

2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด

   เขาอาจเชื่อว่าทีมเดิมตั้งใจเลือก gRPC และ proto ไฟล์เดียวเพราะความเรียบง่ายและ observability ทั้งที่เป็นเพียงการอนุมานจากโค้ด และอาจเชื่อว่าทั้งระบบไม่มี message broker เลย ทั้งที่ AI อ่านมาแค่ 3 ไฟล์ ถ้าเอาไปใช้ตัดสินใจเปลี่ยนสถาปัตยกรรม อาจตั้งอยู่บนเหตุผลที่ทีมเดิมไม่ได้คิดจริง

3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต

   AI กู้คืน**สิ่งที่ระบบทำ**ได้ดี เช่น ใช้ gRPC แบบ unary, checkout เรียก 6 service ตามลำดับ, ไม่มี TLS และ timeout ไม่ถูกใช้งานจริง แต่**เหตุผล**ที่เลือกแบบนั้นมีเฉพาะสถาปนิกเดิมที่รู้ ตัวอย่างจาก ADR-0002 ของผม คือการตัดสินใจแบ่งเป็น 2 เฟส (เรียก loyalty แบบ sync ก่อน 11.11 แล้วค่อยขึ้น RabbitMQ หลังงาน) เหตุผลคือทีมไม่เคยรัน broker บน production, on-call มีแค่ 2 คน และมีเวลาเพียง 3 สัปดาห์ ถ้าคนในอนาคตอ่านแค่โค้ด จะเห็นแค่ว่า loyalty ถูกเรียกแบบ sync และอาจคิดว่าเป็นการออกแบบที่ไม่ดี โดยไม่รู้ว่าเป็นการเลือกเพื่อลดความเสี่ยงชั่วคราว