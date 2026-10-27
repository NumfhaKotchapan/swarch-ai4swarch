เอเจนต์: Codex | โมเดล: GPT-6

## Quality-attribute scenarios

อ้างอิงย่อ: **S** = `docs/scenario.md`; **A** = `docs/adr/0002-use-rabbitmq-for-post-order-processing.md`; ตัวเลขหลัง : คือบรรทัด

- **Performance:** source: ลูกค้า · stimulus: สั่งซื้อ 200 orders/s · environment: พีก 11.11, MailFast p95 1.5 s/p99 8 s · response: commit การยอมรับคำสั่งซื้อพร้อม outbox แล้วตอบโดยไม่รอ consumers · response measure: PlaceOrder p95 ≤ 500 ms รวม durable commit; ค่าเดิมประมาณ 300 ms ไม่ใช่งบเพิ่มที่รับประกันได้ (S:24–26; A:46, 60, 73)
- **Correctness/timeliness:** source: relay/consumer · stimulus: ส่งซ้ำหรือล่มหลัง commit ก่อน ack · environment: โหลดพีก · response: loyaltyservice บันทึกแต้มกับ unique order_id ใน transaction เดียว · response measure: แต้มหนึ่งครั้งต่อคำสั่งซื้อภายใน 60 s โดยต้องตกลงจุดเริ่มเวลา; DLQ ไม่ถือว่าสำเร็จ (S:25, 27; A:49, 63, 86)
- **Availability:** source: MailFast, loyaltyservice หรือ analytics · stimulus: บริการใช้ไม่ได้ · environment: ระบบรับคำสั่งซื้อและ outbox ยังพร้อม · response: รับคำสั่งซื้อ เก็บงานค้าง แล้วกู้คืน · response measure: ไม่มีคำสั่งซื้อล้มเหลวเพราะบริการทั้งสาม; loyalty outage เกิน 60 s ขัดเป้าหมายเวลาและต้องตัดสินใจเรื่องข้อกำหนด (S:27–28; A:74, 87)

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | แยกคิวตามงาน และบันทึก ledger/แต้มกับ unique order_id ก่อน ack (A:48–49) | **[Inferred]** ลดการขวางกันของคิวและป้องกันแต้มซ้ำ สอดคล้องกับ availability/correctness (S:27–28); ผลแต้มต้องอยู่ใน transaction จริง |
| Risk | payment/shipping มาก่อน durable commit; reconciliation และ request idempotency ยังไม่ปิด (A:46, 53, 76) | **[Inferred]** อาจมีการชำระเงินแล้วแต่ไม่มีงานหลังสั่งซื้อ หรือเกิดผลซ้ำเมื่อ retry; outbox ไม่ปิดช่องว่างนี้ กระทบ correctness/recoverability |
| Sensitivity point | quota MailFast 50/s เทียบกับพีก 200 orders/s และระยะพีก (S:25–26; A:50, 88) | **[Inferred]** ที่ 1 email/order คิวโตอย่างน้อย 150/s ก่อน retry; จะระบายได้เมื่อโหลดใหม่ต่ำกว่ากำลังส่งที่เหลือ กระทบ timeliness และพื้นที่เก็บ |
| Sensitivity point | เวลา relay, อายุคิวแต้ม, retry และกำลังกู้คืนยังไม่มีงบเวลาที่เลือก (A:47–49, 63, 98) | **[Inferred]** เวลารวมต้องไม่เกิน 60 s; คิวแยกอย่างเดียวไม่พิสูจน์ว่าให้แต้มทัน (S:27) |
| Trade-off point | store ใน request path เพิ่ม durability แต่เพิ่ม dependency/latency และภาระ broker/relay (A:46, 60–64) | **[Inferred]** durability แลกกับ performance, availability ของ checkout และ cost สำหรับทีม 4 คน; ยังไม่มีผลทดสอบยืนยัน (S:24, 29; A:69) |

## Missing or weak

- **การเทียบตัวเลือก:** มีสามทางเลือกแล้ว แต่ยังไม่เทียบ RabbitMQ กับคิวในฐานข้อมูลด้วย capacity, recovery และภาระดูแลที่วัดได้ ทั้งที่ตัวเลือก 3 ก็เพิ่ม store; ควรพิจารณาตัวเลือก 2 แบบแยก workers ตามงานด้วย **[Inferred]** (A:30–40, 46, 61, 85)
- **สัญญา delivery:** “ตรวจสอบ routing ครบทุกคิว” ยังไม่ระบุวิธีตรวจและกู้คืนเมื่อถึงเพียงบางคิว ก่อน relay ทำเครื่องหมายว่าส่งแล้ว; ควรทดสอบคิว/binding ขาดระหว่าง publish **[Inferred]** (A:47, 75)
- **ขอบเขตหลักฐาน:** ADR ยอมรับว่าแผนทดสอบยังไม่มีผล และระบุช่องว่างหลักไว้แล้ว จึงเป็นเงื่อนไขค้าง ไม่ใช่ประเด็นที่ผู้เขียนละเลย (A:69, 92–98) รายละเอียดโค้ด, ADR-0001 และ `docs/scenario.th.md` ที่ ADR อ้าง ตรวจสอบโดยตรงไม่ได้ภายใต้ขอบเขตงานนี้; ไม่ถือว่าเป็นหลักฐานที่ผู้รีวิวตรวจแล้ว (A:13, 17–18, 53, 96–98)

## Verdict

**accept with changes** — รับทิศทางแบบมีเงื่อนไข คง **Proposed** จนปิดสามประเด็นต่อไปนี้ ยังไม่พร้อมนำไปใช้จริง (A:7, 69, 92–98)

1. เลือก durable store และปิด request idempotency/reconciliation พร้อมพิสูจน์ crash หลัง payment/shipping ก่อน commit ว่าไม่สูญงานหรือคิดเงิน/จัดส่งซ้ำ (A:46, 53, 76, 97)
2. ตกลงระยะพีก โหลดหลังพีก เวลารออีเมล จุดเริ่ม 60 s และข้อขัดแย้งเมื่อ loyalty outage กับผู้รับผิดชอบข้อกำหนด ห้ามลด SLA เอง; ทดสอบ 200 orders/s ให้ PlaceOrder p95 ≤ 500 ms พร้อมตรวจแต้มครบ/ไม่ซ้ำและเวลาระบาย backlog (S:24–28; A:73–77, 86–88)
3. กำหนด routing/replay กรณีสำเร็จบางคิว ผู้ดูแล ขนาด deployment และเป้าหมาย recovery พร้อมเปรียบเทียบกับคิวในฐานข้อมูลให้เห็นว่าประโยชน์คุ้มภาระทีม 4 คน (S:29–30; A:39–40, 47, 61, 75, 84–85)
