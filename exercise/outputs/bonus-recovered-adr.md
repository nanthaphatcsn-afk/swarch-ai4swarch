Agent: Claude Code — model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R001: Use synchronous gRPC with one shared `demo.proto` for all inter-service communication

- **Status:** Reconstructed
- **Date:** 2026-10-27 (วันที่ reconstruct; วันที่ตัดสินใจจริง Not evidenced in the provided files — มีเพียง copyright 2018 ใน `main.go` และ 2020 ใน `demo.proto`)
- **Deciders:** Not evidenced in the provided files (copyright `Google LLC` ในทุกไฟล์ → น่าจะเป็นทีม Google **[Inferred]**)

## Context and problem

ข้อเท็จจริงจากไฟล์ evidence:

- ระบบแบ่งเป็นหลาย service ที่นิยามไว้ใน **ไฟล์เดียว** `system/online-boutique/protos/demo.proto` ภายใต้ `package hipstershop` รวม 9 service: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`
- ทุก `rpc` เป็นแบบ unary (request → response) ไม่มี `stream` เลยใน `demo.proto`
- message กลางถูกใช้ข้าม service หลายตัว: `Money` (Currency, Payment, Shipping, Checkout, Email), `Address` (Shipping, Checkout), `CartItem` (Cart, Shipping, Email), `Empty`
- `checkoutservice` เป็น orchestrator: `PlaceOrder` เรียก 6 service ตามลำดับแบบ blocking — `GetCart` → `GetProduct` + `Convert` (ต่อ item) → `GetQuote` → `Convert` → `Charge` → `ShipOrder` → `EmptyCart` → `SendOrderConfirmation` (`src/checkoutservice/main.go`, `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`)
- `frontend` รับ HTTP จากผู้ใช้ (`gorilla/mux`, `http.ListenAndServe`) แล้วเปิด gRPC connection ไป 7 service (`src/frontend/main.go`, `mustConnGRPC` บรรทัด 141–147)
- ยกเว้น 1 จุด: `SHOPPING_ASSISTANT_SERVICE_ADDR` ถูกอ่านใน `frontend` แต่ **ไม่มี** gRPC connection และไม่มีใน `demo.proto` → คำกล่าวว่า "ทุก service ใช้ gRPC" ไม่จริงทั้งหมด; protocol ของ shopping assistant Not evidenced in the provided files
- ไม่มี client ของ message broker ใดๆ ในไฟล์ evidence

ปัญหาที่ต้องตัดสินใจ: service จำนวนมากต้องคุยกันอย่างไร (sync/async, protocol, วิธีนิยามและแชร์ contract)

## Decision drivers

เรียงตามความสำคัญที่ **[Inferred]** จาก code:

1. **Simplicity / understandability** — ลำดับการสั่งซื้ออ่านได้ตรงๆ เป็นโค้ดเชิงลำดับใน `PlaceOrder`
2. **Interoperability / modifiability ของ contract** — นิยาม API แบบ typed ที่เดียว generate code ได้ (`go_package` option, import `.../genproto`)
3. **Observability** — trace ข้าม service ได้ด้วย `otelgrpc` stats handler บนทั้ง client และ server
4. **Performance (latency)** — binary protocol บน HTTP/2 **[Assumption]**
5. **Operability บน Kubernetes** — gRPC health check มาตรฐาน (`grpc_health_v1`)

## Considered options

1. **A — Synchronous gRPC + `demo.proto` ไฟล์เดียวที่ทุก service แชร์** (สิ่งที่ระบบทำอยู่)
2. **B — Synchronous gRPC แต่แยก `.proto` ต่อ service** (แต่ละทีมเป็นเจ้าของ contract ของตัวเอง)
3. **C — Synchronous REST/JSON over HTTP** (OpenAPI หรือไม่มี schema)
4. **D — Hybrid: gRPC สำหรับ query + asynchronous messaging (message broker) สำหรับขั้นที่ไม่ต้องรอผล** เช่น `SendOrderConfirmation`, `EmptyCart`

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | contract ที่เดียว เห็นภาพรวมทั้งระบบ; reuse `Money`/`Address`/`CartItem` ได้ไม่ต้อง map; typed client/server ที่ generate ได้; รวม OTel + health check ได้ง่าย; flow อ่านง่าย | ทุก service coupled กับไฟล์เดียว — แก้ message กลางกระทบทุกคน; ต้อง regenerate และ copy `genproto` ทุก service; temporal coupling: service ปลายทางล่ม = checkout ล้ม; latency สะสมตามจำนวน call |
| B | ownership ชัด, version contract แยกได้, เปลี่ยน service หนึ่งไม่กระทบ build ของคนอื่น | type กลาง (`Money`) ต้อง duplicate หรือแยกเป็น common proto; ต้องมีการจัดการ dependency ระหว่าง proto **[Assumption]** |
| C | เครื่องมือทั่วไป debug ด้วย curl ได้, เบราว์เซอร์เรียกตรงได้ **[Assumption]** | ไม่มี schema บังคับ (ถ้าไม่ใช้ OpenAPI); JSON ใหญ่/ช้ากว่า binary; ต้องเขียน client เอง **[Assumption]** |
| D | ลด temporal coupling — email ล่มไม่บล็อกคำสั่งซื้อ; รองรับ retry/buffer ได้ดี | ต้องเพิ่มและดูแล broker (cost, ops); eventual consistency; flow ยากต่อการ trace/อ่าน **[Assumption]** |

## Decision

เลือก **Option A** — ทุก service ภายใน (ที่อยู่ใน evidence) คุยกันแบบ synchronous unary gRPC โดยใช้ contract จาก `demo.proto` ไฟล์เดียว

เหตุผลที่ reconstruct ได้ (ส่วนใหญ่เป็น **[Inferred]**):

- ระบบนี้เป็น demo ขนาดเล็ก การเห็น API ทั้งระบบในไฟล์เดียวช่วยให้เข้าใจง่าย และ reuse message กลางได้โดยไม่ต้อง map **[Inferred]**
- flow `PlaceOrder` ต้องใช้ผลลัพธ์ทันที (ราคา, ค่าส่ง, `transaction_id`, `tracking_id`) เพื่อตอบผู้ใช้ใน request เดียว — เหมาะกับ request/response **[Inferred]** จาก `PlaceOrderResponse{OrderResult}`
- การใช้ gRPC ทำให้ใส่ `otelgrpc` และ health service มาตรฐานได้สม่ำเสมอทุก service (ดู `main.go` ทั้งสองไฟล์)
- เหตุผลเรื่อง polyglot (ภาษาหลายภาษา generate จาก proto เดียวกัน) Not evidenced in the provided files — ไฟล์ evidence เป็น Go ทั้งหมด

## Consequences

- **Positive:** contract typed และรวมศูนย์ ลด mismatch ของ field ระหว่าง service (quality attribute: **modifiability / interoperability**)
- **Positive:** distributed tracing ข้าม service ผ่าน `otelgrpc.NewClientHandler/NewServerHandler` และ `propagation.TraceContext` (quality attribute: **observability**)
- **Positive:** flow สั่งซื้ออ่านและ debug ได้ตรงไปตรงมา (quality attribute: **understandability / testability**)
- **Negative:** temporal coupling — `PlaceOrder` คืน error ทันทีถ้า Cart/ProductCatalog/Currency/Shipping/Payment ตัวใดล้ม (quality attribute: **availability**)
- **Negative:** latency ของ checkout = ผลรวมของ call แบบลำดับ และ `GetProduct` + `Convert` ถูกเรียก **ต่อ item** ใน `prepOrderItems` (N+1) (quality attribute: **performance**)
- **Negative:** การแก้ `demo.proto` (เช่น `Money`) กระทบทุก service; code ถูก generate/copy ไว้ใน `genproto` ของแต่ละ service (quality attribute: **modifiability**, **deployability**)
- **Negative:** ใช้ `insecure.NewCredentials()` — ไม่มี TLS ระหว่าง service (quality attribute: **security**)
- **Risks and mitigations:**
  - ไม่มี timeout ต่อ call จริง: `mustConnGRPC` สร้าง `context.WithTimeout` แต่ทิ้งค่า (`_, cancel`) และ `convertCurrency` ใช้ `context.TODO()` (ทำให้ deadline/trace จาก request หลุดด้วย) → mitigation: ส่ง `ctx` ต่อ และตั้ง deadline ต่อ RPC
  - ไม่พบ retry / circuit breaker ใน evidence → mitigation **[Inferred]**: ใช้ gRPC retry policy หรือ service mesh
  - payment สำเร็จแล้วแต่ `ShipOrder` ล้ม → คืน `codes.Unavailable` โดยไม่มี compensation (เงินถูกตัดแต่ไม่ส่งของ) → mitigation: saga/compensating action หรือย้ายเป็น async (Option D)
  - ขั้นที่ไม่สำคัญถูกทำเป็น best-effort แล้ว: `EmptyCart` ทิ้ง error (`_ =`), `SendOrderConfirmation` แค่ `log.Warnf` → เป็นจุดที่ย้ายไป async ได้ง่าย

## Validation

- Load test `POST /cart/checkout` แล้ววัด p95 latency เทียบกับจำนวน item ในตะกร้า (ตรวจผลกระทบ N+1)
- Fault injection: ปิด `emailservice` / `shippingservice` ทีละตัว แล้ววัด success rate ของ checkout
- ใช้ trace (OTel) ดูสัดส่วน latency ต่อ downstream call
- Review ทุกครั้งที่จะแก้ message กลางใน `demo.proto` (ตรวจ backward compatibility ของ field number)

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | contract ทุก service อยู่ในไฟล์เดียว `package hipstershop` (9 service) | Evidenced | `protos/demo.proto` (`package hipstershop`, `service ...` ทั้ง 9) |
| 2 | ทุก RPC เป็น unary ไม่มี streaming | Evidenced | `protos/demo.proto` (ไม่มีคำว่า `stream`) |
| 3 | message กลาง (`Money`, `Address`, `CartItem`, `Empty`) ถูก reuse ข้าม service | Evidenced | `protos/demo.proto` (`ChargeRequest.amount`, `GetQuoteRequest.address`, `OrderItem.item`, …) |
| 4 | `checkoutservice` เรียก 6 service แบบ synchronous ตามลำดับ | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `mustConnGRPC` ×6) |
| 5 | `frontend` รับ HTTP และเรียก 7 service ผ่าน gRPC | Evidenced | `src/frontend/main.go` (`mux.NewRouter`, `http.ListenAndServe`, `mustConnGRPC` บรรทัด 141–147) |
| 6 | shopping assistant ไม่ได้ใช้ gRPC / ไม่อยู่ใน `demo.proto` | Evidenced (ส่วนที่ว่าไม่มี gRPC conn) ; protocol จริง Not evidenced | `src/frontend/main.go` (`shoppingAssistantSvcAddr` ไม่มี `mustConnGRPC`); `protos/demo.proto` |
| 7 | ไม่มี message broker ในไฟล์ evidence | Evidenced (โดยการไม่พบ) | imports ใน `src/checkoutservice/main.go`, `src/frontend/main.go` |
| 8 | ใช้ code ที่ generate จาก proto ต่อ service | Evidenced | `src/checkoutservice/main.go` (import `.../src/checkoutservice/genproto`); `demo.proto` (`option go_package`) |
| 9 | service ไม่แชร์ package กัน (code ซ้ำกัน) | Evidenced | `src/checkoutservice/main.go`, `src/frontend/main.go` (comment `TODO(ahmetb) ... not sharing packages`) |
| 10 | OTel tracing รวมกับ gRPC ทั้ง client และ server | Evidenced | `otelgrpc.NewServerHandler` (checkout), `otelgrpc.NewClientHandler` (`mustConnGRPC` ทั้งสองไฟล์) |
| 11 | มี gRPC health service มาตรฐาน | Evidenced | `src/checkoutservice/main.go` (`healthpb.RegisterHealthServer`) |
| 12 | ไม่มี TLS ระหว่าง service | Evidenced | `insecure.NewCredentials()` ใน `mustConnGRPC` |
| 13 | ไม่มี timeout ต่อ call ที่ใช้งานจริง | Evidenced | `mustConnGRPC` (`_, cancel := context.WithTimeout`); `convertCurrency` (`context.TODO()`) |
| 14 | `GetProduct` + `Convert` ถูกเรียกต่อ item (N+1) | Evidenced | `src/checkoutservice/main.go` (`prepOrderItems` loop) |
| 15 | ShipOrder ล้มหลัง Charge สำเร็จ → ไม่มี compensation | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`: `chargeCard` แล้ว `shipOrder` คืน `codes.Unavailable`) |
| 16 | `EmptyCart` และ email เป็น best-effort | Evidenced | `_ = cs.emptyUserCart(...)`; `log.Warnf("failed to send order confirmation ...")` |
| 17 | ไม่มี retry / circuit breaker | Evidenced (ในไฟล์ที่อ่าน) ; อาจมีใน infra อื่น Not evidenced | `mustConnGRPC` ไม่มี retry option |
| 18 | ผู้ตัดสินใจคือทีม Google | Inferred | มีเพียง copyright `Google LLC` |
| 19 | เหตุผลหลักคือความเรียบง่ายสำหรับ demo ขนาดเล็ก | Inferred | ไม่มีเอกสารเหตุผลใน evidence |
| 20 | เลือก sync เพราะ checkout ต้องได้ผลลัพธ์ทันทีใน request เดียว | Inferred | สอดคล้องกับ `PlaceOrderResponse{OrderResult}` แต่ไม่มีการระบุเหตุผล |
| 21 | ไฟล์ proto เดียวเพื่อให้เห็น API ทั้งระบบและ reuse type | Inferred | โครงสร้าง `demo.proto` สอดคล้อง แต่ไม่มีคำอธิบายเหตุผล |
| 22 | gRPC เหมาะกับ polyglot service | Assumption | ไฟล์ evidence เป็น Go ทั้งหมด |
| 23 | gRPC (binary/HTTP/2) เร็วกว่า REST/JSON | Assumption | ความรู้ทั่วไป |
| 24 | Option B แยก ownership/versioning ได้ดีกว่า | Assumption | ความรู้ทั่วไป |
| 25 | Option C debug ง่ายกว่าแต่ไม่มี schema บังคับ | Assumption | ความรู้ทั่วไป |
| 26 | Option D ลด temporal coupling แต่เพิ่ม ops cost และ eventual consistency | Assumption | ความรู้ทั่วไป |
| 27 | Mitigation: deadline ต่อ RPC, retry policy/service mesh, saga | Assumption | ความรู้ทั่วไป |
