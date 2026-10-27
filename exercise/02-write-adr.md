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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่กำหนดตัวเลข latency, throughput หรือ rate limit; ถาม SLO ปริมาณงาน และข้อจำกัด MailFast ไว้ใน Open questions | ใช้ checkout p95 ≤ 500 ms (ปัจจุบันประมาณ 300 ms), peak 200 orders/s, MailFast p95 1.5 s/p99 8 s, failure 0.5%, limit 50 emails/s และแต้มภายใน 60 s; ตัวเลขตรงกับ docs/scenario.th.md |
| ทางเลือกที่พิจารณา (ระบุ) | เรียกงานโดยตรงแบบ synchronous; ตัวประสานงาน asynchronous ควบคุม workflow; durable event ผ่าน message broker ที่ยังไม่เลือกผลิตภัณฑ์ | synchronous gRPC จาก checkoutservice; ตัวจัดการงานส่วนกลางกับคิวถาวรในฐานข้อมูล; RabbitMQ พร้อม outbox และ consumer แยก |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | ใช้ orderId ร่วมกับชนิดรายการคะแนนเป็นกุญแจกันซ้ำ และบันทึกพร้อมผลคะแนนแบบ atomic; ยังไม่ทราบ rate limit ของ MailFast จึงถามไว้ | ใช้ unique key ของการให้แต้มต่อ order_id บันทึกพร้อม ledger ใน transaction เดียว และ ack หลัง commit; จำกัด MailFast รวมทุก replica และ retry ≤ 50 calls/s; หาก 1 email/order ที่ 200 orders/s จะมี backlog เพิ่มอย่างน้อย 150 รายการ/s |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แยก outbox, event fields และ consumer เป็นข้อเสนอใหม่ ไม่อ้างว่ามีอยู่แล้ว; ไม่ตั้งตัวเลข performance หรืออ้างว่ารันทดสอบแล้ว | แยก RabbitMQ, outbox, durable store และ queue topology เป็นข้อเสนอใหม่ ไม่อ้างว่ามีอยู่แล้วหรือทดสอบแล้ว; ตัวเลขข้อกำหนดตรงกับ scenario และระบุ backlog 150/s เป็นผลคำนวณภายใต้สมมติฐาน 1 email/order |

> **วิธีเปรียบเทียบ:** รอบล่าสุดทำ Run a และ Run b ในแชตใหม่แยกกัน ตารางนี้เปรียบเทียบไฟล์ outputs/2a-adr.md และ outputs/2b-adr.md ล่าสุด ส่วนข้อมูลร่างเดิมและการทำรอบใหม่บันทึกแยกไว้ใน exercise/00-prompt-log.md

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | AI ปรับเหตุผลใน Decision ให้ชัดว่าเสนอ RabbitMQ โดยให้ความสำคัญกับ availability และ modifiability พร้อมคงเป้าหมาย latency และระบุภาระ broker, durable store และ eventual consistency | ทำให้เหตุผลและต้นทุนของทางเลือกที่ผู้จัดทำตรวจรับชัดเจน |
| 2 | AI เพิ่ม Conditions before implementation สำหรับระยะ peak/เวลารออีเมล/quota, recovery หลัง payment/shipping และขอบเขต SLA แต้ม 60 s | รวบรวมประเด็นที่มีอยู่ใน Run b ให้เป็นเงื่อนไขที่ต้องปิดก่อนใช้จริง โดยคง Open questions ไว้ |
| 3 | AI เพิ่มข้อความเปิด Validation ว่าเป็นแผนทดสอบที่ยังไม่ได้ดำเนินการ และเพิ่มหมายเหตุสถานะฉบับเตรียมส่ง | แยกแผนทดสอบออกจากผลทดสอบจริง; การระบุผู้ตัดสินใจหลังตรวจรับบันทึกในแถว 4 |
| 4 | ระบุชื่อใน Deciders และปรับหมายเหตุหลังตรวจรับข้อเสนอ | ให้สถานะผู้ตัดสินใจตรงกับการตรวจรับจริง |



## สถานะข้อเสนอหลังตรวจรับ

ADR ฉบับนี้ใช้ Run b ล่าสุดเป็นฐาน โดยมี AI ช่วยปรับเอกสาร ผู้จัดทำตรวจและเลือกแนวทาง RabbitMQ พร้อม outbox และ consumers แยก พร้อมยอมรับภาระ operation และ eventual consistency จึงระบุชื่อใน Deciders คงสถานะ Proposed และเก็บเงื่อนไขที่ต้องพิสูจน์ก่อนนำไปใช้จริงไว้ใน Conditions before implementation
