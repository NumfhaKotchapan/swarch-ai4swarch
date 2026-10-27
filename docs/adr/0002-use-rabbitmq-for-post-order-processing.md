# ADR-0002: ใช้ RabbitMQ แยกงานหลังสั่งซื้อเป็น asynchronous consumers

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** กชพรรณ เลิศปัญญาโรจน์

> **สถานะ:** ผู้จัดทำเลือกแนวทาง RabbitMQ พร้อม outbox และ consumers แยก โดยมี AI ช่วยร่างและปรับเอกสาร คงสถานะ Proposed และต้องปิด Conditions before implementation ก่อนนำไปใช้จริง

## Context and problem

R-12 กำหนดให้ส่งอีเมลยืนยันผ่าน MailFast แทน mock emailservice เพิ่มแต้มผ่าน loyaltyservice ใหม่ และบันทึกเหตุการณ์สำหรับฝ่ายการตลาด โดยปีหน้าจะเพิ่ม SMS, fraud scoring และการปรับ recommendations (docs/scenario.md:7–14; ข้อกำหนดผู้ใช้)

- ปัจจุบัน PlaceOrder เรียก chargeCard แล้ว shipOrder ก่อน emptyUserCart และ sendOrderConfirmation; อีเมลล้มเหลวจะบันทึกคำเตือนแต่ยังคืนผลสำเร็จ การเรียก SendOrderConfirmation เป็น synchronous gRPC (system/online-boutique/src/checkoutservice/main.go:230–280, 369–393)
- เป้าหมาย p95 ของ PlaceOrder ≤ 500 ms จากปัจจุบันประมาณ 300 ms; โหลดพีก 200 orders/s; MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และจำกัด 50 emails/s (docs/scenario.md:24–26)
- ต้องให้แต้มภายใน 1 นาทีและให้ครั้งเดียวต่อคำสั่งซื้อ; คำสั่งซื้อต้องไม่ล้มเหลวเพราะอีเมล แต้ม หรือ analytics ใช้งานไม่ได้ (docs/scenario.md:27–28)
- ทีมมี 4 คน ถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ระบบทำงานบน Kubernetes และองค์กรต้องการ open-source แบบ self-hosted (docs/scenario.md:29–30)
- ADR-0001 เลือก Redis สำหรับตะกร้าและยอมรับความทนทานที่จำกัด เพราะยังไม่มีการจ่ายเงิน (docs/adr/0001-use-redis-for-cart-storage.md:11, 20, 38–45) จึงไม่ควรนำเหตุผลเดียวกันมาใช้ยอมรับการสูญเสียงานหลังชำระเงิน **[Inferred]**
- ใน main.go ที่ให้มา ไม่พบการบันทึกคำสั่งซื้ออย่างถาวร, outbox, broker หรือ loyaltyservice; การมีอยู่ที่อื่นคือ **Not evidenced in the provided files** (system/online-boutique/src/checkoutservice/main.go:66–86, 230–280)

## Decision drivers

1. **Availability:** แยกความล้มเหลวของงานหลังสั่งซื้อจากผลสำเร็จของคำสั่งซื้อ (docs/scenario.md:28)
2. **Performance:** รักษา p95 ≤ 500 ms โดยไม่รอ MailFast ซึ่งมี latency สูงกว่าเป้าหมาย (docs/scenario.md:24, 26)
3. **Correctness และ timeliness:** ให้แต้มครั้งเดียวต่อคำสั่งซื้อและภายใน 1 นาที (docs/scenario.md:27)
4. **Modifiability:** เพิ่มงานใหม่โดยไม่เพิ่มการเรียกใน checkout ทุกครั้ง (docs/scenario.md:14)
5. **Operability และ cost:** จำกัดภาระดูแลให้เหมาะกับทีม 4 คน ใช้ประสบการณ์ RabbitMQ และแนวทาง self-hosted บน Kubernetes (docs/scenario.md:29–30)

## Considered options

1. เพิ่ม synchronous gRPC orchestration ใน checkoutservice โดยเปลี่ยน emailservice ให้เรียก MailFast และเรียก loyaltyservice กับ analytics โดยตรง
2. ใช้ตัวจัดการงานเบื้องหลังส่วนกลางกับคิวงานถาวรในฐานข้อมูล ให้ workers เรียกงานหลังสั่งซื้อแต่ละประเภท
3. เผยแพร่ OrderPlaced ผ่าน RabbitMQ แบบ publish/subscribe โดยมี durable outbox และ consumer แยกสำหรับอีเมล แต้ม และ analytics

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. synchronous gRPC orchestration | สอดคล้องกับรูปแบบเดิมและทักษะ Go/gRPC ของทีม; ลำดับการเรียกมองเห็นง่าย (main.go:252–279; docs/scenario.md:29) | **[Inferred]** การรอ MailFast เสี่ยงเกิน 500 ms แม้เรียกขนาน; การละเลย error อย่างปัจจุบันไม่ทำให้งานสำเร็จภายหลัง; เพิ่มงานใหม่ต้องแก้ checkout (main.go:273–279; docs/scenario.md:14, 24–28) |
| 2. คิวงานในฐานข้อมูลและตัวจัดการส่วนกลาง | **[Inferred]** แยก latency ออกจาก checkout และบันทึกงานกับการยอมรับคำสั่งซื้อใน transaction เดียวได้; ไม่เพิ่ม broker หากเลือกใช้ฐานข้อมูลสำหรับงานถาวรอยู่แล้ว | **[Inferred]** ต้องออกแบบ polling, locking, retry และการแบ่งทรัพยากรเอง; การเพิ่มประเภทงานผูกกับตัวจัดการส่วนกลาง และต้องดูแลฐานข้อมูลใหม่หากยังไม่มี |
| 3. RabbitMQ พร้อม outbox และ consumers แยก | **[Inferred]** แยก failure, backlog และการขยาย workers ตามประเภทงาน; เพิ่ม subscriber ใหม่ได้โดยไม่เพิ่มการเรียกใน checkout; มีประสบการณ์ในทีมช่วยลดความเสี่ยงการเรียนรู้ (docs/scenario.md:14, 29) | **[Inferred]** เพิ่ม broker, durable store และ relay; ต้องจัดการ duplicate, schema และการติดตามข้ามบริการ; การส่งข้อความเพียงอย่างเดียวไม่รับประกันผลให้แต้มครั้งเดียวหรือภายใน 1 นาที |

## Decision

เสนอเลือก **ตัวเลือก 3 — RabbitMQ พร้อม outbox และ consumers แยก** โดยให้ความสำคัญกับ availability เพื่อแยกความล้มเหลวของงานหลังสั่งซื้อ และ modifiability เพื่อเพิ่มงานปีหน้าโดยไม่เพิ่ม RPC ใน checkout พร้อมรักษาเป้าหมาย p95 ≤ 500 ms และใช้ประสบการณ์ RabbitMQ ในทีม (docs/scenario.md:14, 24–30) การเลือกนี้มีต้นทุนในการดูแล broker, durable store และ relay รวมถึงผลลัพธ์แบบ eventual consistency ตาม Consequences; ต้องยอมรับภาระดังกล่าวและปิด Conditions before implementation ก่อนนำไปใช้จริง รายละเอียดต่อไปนี้เป็น **ข้อเสนอใหม่** ยังไม่ได้มีหลักฐานว่าใช้งานอยู่ในระบบปัจจุบัน

1. **จุดยอมรับคำสั่งซื้อและ durable outbox:** หลัง payment และ shipping สำเร็จ ให้ checkoutservice บันทึกสถานะยอมรับคำสั่งซื้อพร้อม OrderPlaced ใน transaction เดียวของ durable transactional store ใหม่ ก่อนตอบสำเร็จ ไม่เรียก post-order consumers หรือรอ broker ใน request path และไม่ใช้เพียง goroutine ในหน่วยความจำแทนงานถาวร การเลือกฐานข้อมูลยังต้องตกลงใน Open questions; ห้ามถือว่า Redis สำหรับตะกร้ามีคุณสมบัติที่ต้องการโดยอัตโนมัติ (จุดแทรกอ้างอิง main.go:252–279; ขอบเขต Redis ตาม ADR-0001:38–45)
2. **เผยแพร่และกระจายงาน:** relay อ่าน outbox แล้วส่งไป RabbitMQ exchange พร้อม publisher confirms; ใช้ persistent messages, durable replicated queues และตรวจสอบ routing ครบทุกคิวก่อนทำเครื่องหมายว่าส่งแล้ว เมื่อ broker ใช้งานไม่ได้ ให้เก็บ pending events และลองใหม่ ไม่ทำให้คำสั่งซื้อที่บันทึกสำเร็จล้มเหลว ออกแบบการส่งเป็น at-least-once และรองรับการส่งซ้ำทุก consumer
3. **แยกคิวตามงาน:** ให้ emailservice ที่ปรับใหม่เป็น consumer เรียก MailFast แทน mock logic; ให้ loyaltyservice ใหม่และ analytics consumer มีคิวและ workers ของตนเอง ใช้ retry พร้อม backoff/jitter และ dead-letter queue สำหรับงานที่ต้องตรวจสอบ คิวอีเมลที่ค้างต้องไม่ขวางแต้ม เพิ่ม SMS หรือ consumer อื่นด้วยการเพิ่ม queue/binding โดยคงสัญญา OrderPlaced เดิมเมื่อข้อมูลเพียงพอ (ความต้องการจาก docs/scenario.md:9–14)
4. **ผลแต้มครั้งเดียว:** loyaltyservice ต้องบันทึก order_id ที่ประมวลผลแล้วพร้อม ledger/การเพิ่มแต้มใน local transaction เดียว โดยบังคับ unique key ของการให้แต้มต่อ order_id; ack หลัง commit เท่านั้น เมื่อข้อความมาซ้ำให้ยืนยันผลเดิมโดยไม่เพิ่มแต้มอีก นี่คือผลทางธุรกิจแบบ exactly-once บนการส่งแบบ at-least-once ไม่ใช่คำรับประกัน exactly-once จาก broker หากแต้มอยู่ในอีกระบบหนึ่ง ต้องมี idempotency ที่ระบบนั้นด้วยก่อนยอมรับแบบนี้
5. **รองรับ MailFast:** จำกัดการส่งรวมทุก replica ไม่เกิน 50 calls/s โดยนับ retries ในงบเดียวกัน และเก็บอีเมลรอส่งอย่างถาวร ถ้ามี 1 อีเมลต่อคำสั่งซื้อและโหลดคงที่ 200 orders/s คิวจะเพิ่มอย่างน้อย 150 รายการ/s ก่อนคิด retry **[Inferred: คำนวณจาก docs/scenario.md:25–26]** การเพิ่ม workers อย่างเดียวแก้ข้อจำกัดนี้ไม่ได้ ต้องตกลงเวลารออีเมลและเจรจา quota หากจำเป็น การส่งสำเร็จแต่ ack สูญหายอาจทำให้อีเมลซ้ำ; ใช้ idempotency key หาก MailFast รองรับ และอย่าอ้างว่ารองรับโดยไม่มีข้อมูล
6. **สัญญาและข้อมูล:** เสนอ event ที่มี event_id คงที่เมื่อ retry, order_id, user_id, เวลาเกิดเหตุการณ์, schema_version และข้อมูลคำสั่งซื้อที่จำเป็นต่อการคำนวณแต้ม/analytics; จำกัดข้อมูลติดต่อไว้เฉพาะเส้นทางอีเมลผ่านข้อมูลแยกหรือ reference ที่ควบคุมสิทธิ์ ห้ามส่งข้อมูลบัตรเครดิตซึ่ง request ปัจจุบันใช้ใน chargeCard (main.go:252, 369–376) กำหนด schema แบบเพิ่มข้อมูลโดยยังรองรับ consumer เดิม และให้ analytics deduplicate ตาม order_id กับประเภทเหตุการณ์

การ commit ฐานข้อมูลใหม่นี้ไม่ทำให้ payment/shipping ซึ่งเป็น remote calls อยู่ใน transaction เดียวกันได้ ปัจจุบัน payment สำเร็จก่อน shipping และ error ของ shipping ทำให้ PlaceOrder คืน error (main.go:252–260) จึงต้องออกแบบ reconciliation สำหรับการล่มหลังผลข้างเคียงเหล่านี้แต่ก่อน durable commit และพิจารณา request idempotency ก่อนใช้งานจริง ความสามารถ idempotency ของ payment/shipping คือ **Not evidenced in the provided files**; ADR นี้ยังไม่ถือว่าปิดช่องว่างดังกล่าวแล้ว

## Consequences

- **Positive:** ไม่รอ MailFast, loyalty หรือ analytics ก่อนตอบ; เมื่อ consumer ใดล่ม งานนั้นยังรอประมวลผลแยกจากงานอื่นได้ (quality attribute: performance, availability)
- **Positive:** เพิ่ม post-order action ด้วย subscriber ใหม่ และขยาย loyalty workers แยกจากอีเมลได้ (quality attribute: modifiability, scalability)
- **Positive:** outbox และการบันทึกแต้มแบบ atomic ช่วยให้กู้คืนงานและป้องกัน double points แม้ส่งข้อความซ้ำ (quality attribute: reliability, correctness)
- **Negative:** ลูกค้าอาจได้รับอีเมลหรือเห็นแต้มภายหลัง; คิวอีเมลเพิ่มระหว่างพีก และ durable commit เพิ่ม latency ใน checkout ซึ่งต้องวัดจริง (quality attribute: timeliness, performance)
- **Negative:** ทีม 4 คนต้องดูแล broker, store, relay, consumers และการกู้คืนข้อมูล; ยังไม่มีงบประมาณหรือขนาด deployment ที่ยืนยัน (quality attribute: operability, cost)
- **Risks and mitigations:** event สูญหายในช่องว่างระหว่างยอมรับคำสั่งซื้อกับ publish → บันทึกสถานะและ outbox ใน transaction เดียว ใช้ confirms/routing checks และ replay; relay อาจส่งซ้ำ → ใช้ idempotent consumers (quality attribute: durability, correctness)
- **Risks and mitigations:** consumer ล่มเกิน 1 นาทีทำให้เป้าหมายแต้มเป็นไปไม่ได้ในช่วงนั้น → แยกคิว จัด capacity/ความพร้อมใช้งานให้ loyalty ติดตามอายุ event ตั้ง alert ก่อนถึงเส้นตาย และกำหนดวิธีจัดการ SLA breach; DLQ ไม่ถือว่าให้แต้มสำเร็จ (quality attribute: availability, timeliness)
- **Risks and mitigations:** outbox/store เต็มหรือใช้ไม่ได้ทำให้รับงานถาวรเพิ่มไม่ได้ → วางแผน capacity, replication, backup และการกู้คืน; การแยก consumers ไม่ได้ทำให้ checkout ไร้ dependency ใหม่ และไม่ควรตอบสำเร็จแล้วทิ้ง event (quality attribute: availability, durability)
- **Risks and mitigations:** งานใหม่อย่าง fraud scoring อาจต้องใช้ผลก่อนยอมรับคำสั่งซื้อ → ยืนยันความหมายทางธุรกิจก่อนจัดเป็น post-order subscriber เพราะการประมวลผลภายหลังย้อนกลับการชำระเงินไม่ได้เอง (quality attribute: correctness, modifiability; ความคาดหวังจาก docs/scenario.md:14)

## Validation

รายการต่อไปนี้เป็นแผนทดสอบก่อนนำไปใช้จริง ยังไม่ได้ดำเนินการและยังไม่มีผล benchmark หรือหลักฐานยืนยันว่าผ่านเกณฑ์

| การตรวจสอบที่เสนอ | เกณฑ์หรือหลักฐานที่ต้องได้ |
|---|---|
| ทดสอบโหลด 200 orders/s รวม durable commit และ post-order pipeline โดยจำลอง latency/error ของ MailFast ตาม scenario | p95 ของ PlaceOrder ≤ 500 ms; แต้มของคำสั่งซื้อที่ยอมรับแล้วครบและไม่ซ้ำภายใน 60 s ในภาวะที่ตกลงไว้ (docs/scenario.md:24–28) |
| ปิด emailservice/MailFast, loyaltyservice และ analytics ทีละส่วน | คำสั่งซื้อยังสำเร็จเมื่อระบบรับคำสั่งซื้อกับ outbox พร้อมใช้งาน; งานอื่นเดินต่อ งานค้างไม่สูญหายและกู้คืนได้หลังเปิดบริการ; ระบุแต้มที่เกิน 60 s เป็น SLA breach |
| หยุด relay/broker แล้วเปิดใหม่ รวมทั้งหยุด process หลัง publish ก่อน mark และหลัง loyalty commit ก่อน ack | outbox ส่งต่อครบทั้งสามคิว; จำนวนแต้มเท่ากับการให้หนึ่งครั้งต่อ order_id; analytics ไม่บันทึกเหตุการณ์ซ้ำในการ replay |
| จำลอง crash หลัง payment/shipping แต่ก่อน commit รวมถึง client retry | ต้องมีหลักฐานว่าระบุและกู้คืนคำสั่งซื้อที่มีผลข้างเคียงแล้วได้ และไม่เรียกเก็บเงิน/จัดส่งซ้ำ ก่อนเปลี่ยนสถานะ ADR เป็น Accepted |
| ทดสอบ MailFast limiter ข้ามหลาย replicas และ retry; จำลองโหลดพีกต่อเนื่อง | รวมไม่เกิน 50 calls/s, คิวแต้มไม่ถูกคิวอีเมลขวาง; วัดอายุอีเมลเก่าสุด ขนาด backlog และเวลาระบายหลังพีก โดยระยะเวลาพีกและเกณฑ์รออีเมลต้องตกลงก่อน (docs/scenario.md:25–27) |
| เพิ่ม consumer จำลองสำหรับ SMS และทดสอบ event schema รุ่นถัดไป | ไม่ต้องเพิ่ม RPC ใน checkout; consumers เดิมยังประมวลผลได้ |

เก็บ metrics ของ PlaceOrder p95, outbox age, queue age แยกประเภท, retry/DLQ, event-to-loyalty-commit latency และการตัด duplicate; เชื่อม trace/log ด้วย order_id และ event_id ผลทดสอบทั้งหมดข้างต้นยังไม่ได้ดำเนินการ เพราะงานนี้จัดทำเฉพาะเอกสารข้อเสนอ

## Open questions

- ใครเป็นเจ้าของการดูแล broker, outbox, loyaltyservice และ analytics?
- จะใช้ durable transactional store ใดสำหรับสถานะยอมรับคำสั่งซื้อกับ outbox และ ledger ของ loyalty; ขนาด cluster, งบประมาณ, retention, RPO/RTO และแนวทาง backup/recovery เป็นเท่าใด? **Not evidenced in the provided files**
- นิยาม “order placed” และเวลาเริ่มนับ 1 นาทีคือจุดใด? จะกู้คืนหรือชดเชยอย่างไรเมื่อล่มหลัง chargeCard/shipOrder แต่ก่อน commit; ต้องเพิ่ม request idempotency ในส่วนใด และบริการปลายทางรองรับหรือไม่?
- เป้าหมายแต้มภายใน 1 นาทีครอบคลุม outage แบบใดและนานเท่าใด? หาก loyaltyservice ล่มเกิน 1 นาที จะรักษาการยอมรับคำสั่งซื้อตามข้อกำหนดและจัดการ SLA breach อย่างไร? ไม่สามารถรับประกันทั้งสองอย่างโดยไม่มีเงื่อนไข
- ช่วงพีก 200 orders/s นานเท่าใด โหลดหลังพีกเท่าใด และยอมรับอีเมลล่าช้าได้สูงสุดเท่าใด? MailFast เพิ่ม quota ได้หรือไม่ มี protocol/API, timeout, retry policy และ idempotency semantics อย่างไร? **Not evidenced in the provided files**
- กติกาคำนวณแต้ม การปัดเศษ/สกุลเงิน การคืนสินค้า และการยกเลิกคำสั่งซื้อเป็นอย่างไร? ต้องรักษาลำดับเหตุการณ์ต่อ user/order หรือไม่?
- Analytics ใช้ปลายทางและ schema ใด ต้องเก็บข้อมูลส่วนบุคคลใด ใครเข้าถึงได้ และเก็บนานเท่าใด? ข้อกำหนด SMS, fraud scoring และ recommendations ใหม่เป็นงานหลังสั่งซื้อจริงทั้งหมดหรือไม่?

## Conditions before implementation

เงื่อนไขเหล่านี้รวบรวมจาก Decision, Consequences, Validation และ Open questions ข้างต้น ยังไม่ถือว่าปิดประเด็นแล้ว

1. **Capacity และเวลารออีเมล:** ตกลงระยะ peak 200 orders/s, โหลดหลังพีก, เวลารออีเมลที่ยอมรับได้ และ quota plan ของ MailFast แล้วใช้ผลทดสอบยืนยันขนาด backlog และเวลาระบาย ภายใต้สมมติฐาน 1 email/order และ quota เดิม 50/s คิวจะเพิ่มอย่างน้อย 150 รายการ/s ก่อนคิด retry; queue อย่างเดียวไม่แก้ข้อจำกัดอัตราส่ง (docs/scenario.th.md: ตารางเป้าหมายด้านคุณภาพและข้อจำกัด)
2. **Durable acceptance และ recovery:** เลือก durable transactional store และพิสูจน์การกู้คืนเมื่อ payment/shipping สำเร็จแต่ยังไม่ commit รวมถึง client retry โดยมีหลักฐานเรื่อง idempotency/reconciliation ว่าป้องกันการคิดเงินหรือจัดส่งซ้ำได้ การมี outbox อย่างเดียวไม่ปิดช่องว่างของ remote side effects (อ้างอิงประเด็นใน Decision และแถว crash test ของ Validation ที่สืบทอดจาก Run b)
3. **ขอบเขต SLA แต้ม:** ตกลงจุดเริ่มนับ 60 s, ขอบเขต outage ของ loyaltyservice และวิธีจัดการ SLA breach พร้อมพิสูจน์ capacity และ recovery throughput ตามแผนทดสอบ การเข้า DLQ ไม่ถือว่าให้แต้มสำเร็จ และยังรับประกันแต้มภายใน 60 s ไม่ได้เมื่อบริการล่มนานกว่าเส้นตาย (docs/scenario.th.md: ข้อกำหนดแต้มสะสม; Consequences และ Open questions)
