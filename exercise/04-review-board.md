**ภาษา:** ภาษาไทย | [English](04-review-board.en.md)

# ส่วน B: บันทึกการอภิปรายกลุ่ม — Architecture Review Board

**สถานะ:** Approved with changes — อนุมัติโดยมีเงื่อนไข
**ผู้บันทึก / AI Analyst:** กชพรรณ เลิศปัญญาโรจน์ / 6630001321
**วันที่จัดทำบันทึก:** 27 ตุลาคม 2026
**ที่มาของบันทึก:** ข้อมูลการอภิปรายและมติกลุ่มที่ผู้จัดทำให้ไว้ใน Activity 4

## 1. ADR ที่รีวิวและสมาชิก

**ADR:** ADR-0002 — Publish an "OrderPlaced" event to RabbitMQ for post-order actions
**ไฟล์ ADR ที่กลุ่มเลือก:** [ADR ของ Pitchanan]( https://github.com/Preem-pcn/swarch-ai4swarch/blob/11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e/docs/adr/0002-publish-order-events-via-rabbitmq.md)
**เจ้าของ ADR:** Pitchanan Thaiphayak
**การตัดสินใจที่รีวิว:** ใช้ RabbitMQ ที่ดูแลเอง ร่วมกับ outbox และ consumers แยกกันสำหรับงานหลังสั่งซื้อ
**ผลรีวิว AI ที่ใช้ในส่วน B:** [รีวิว ADR ของ Pitchanan](https://github.com/Preem-pcn/swarch-ai4swarch/blob/11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e/exercise/outputs/3-review.md)
**Agent ของผลรีวิวกลุ่ม:** Claude Code / Claude Opus 5.5 ตามหัวเอกสารต้นทาง
**เวอร์ชันอ้างอิง:** commit 11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e

| บทบาท | สมาชิก |
|---|---|
| Architect (เจ้าของ ADR) | Pitchanan Thaiphayak |
| AI Analyst | Kotchapan (กชพรรณ เลิศปัญญาโรจน์) |
| Product Owner (PO) | Pon |
| Site Reliability Engineer (SRE) | Piyapanchanok |
| Security Officer | Chayutpong |

## 2. สรุปการอภิปรายกลุ่ม

### Architect — นำเสนอ ADR

Pitchanan อธิบายว่า RabbitMQ ช่วยแยกงาน email, loyalty และ analytics ออกจาก checkout ให้ประมวลผลได้อิสระและเพิ่ม consumers ในอนาคตได้โดยไม่ผูกกับ order service มากเกินไป พร้อมยอมรับว่าต้องอธิบายกรณี payment สำเร็จแต่ยังไม่มี event ที่บันทึกอย่าง durable ให้ชัดเจน และขอให้กลุ่มพิจารณาเป้าหมายเวลา loyalty ระหว่าง broker outage

### AI Analyst — นำเสนอผลรีวิว AI

Kotchapan นำเสนอข้อค้นพบของ AI โดยแยกจากความเห็นส่วนตัว ได้แก่ ช่องว่างระหว่าง payment กับ outbox รายละเอียด relay และ recovery ที่ยังต้องกำหนด ความล่าช้าของ loyalty และ backlog ของ email เมื่อความต้องการเกินกำลังส่งของ MailFast ตามบันทึก Activity 4 ยังมีการนำเสนอข้อทักท้วงเรื่องหลักฐานเปรียบเทียบ RabbitMQ กับ Kafka และการใช้ตัวอย่าง peak 30 นาทีซึ่งเป็นสมมติฐาน ไม่ใช่ข้อกำหนดของ scenario

**หมายเหตุแยกเอกสาร:** ส่วน B ใช้ ADR และผลรีวิวของ Pitchanan ตามลิงก์ข้างต้น ซึ่งมีทั้งประเด็น Kafka และตัวอย่าง peak 30 นาที โดยผลรีวิวระบุระยะ 30 นาทีเป็นตัวอย่าง [Inferred] ไม่ใช่ข้อกำหนดของ scenario ส่วน `exercise/outputs/3-review.md` ใน repository นี้เป็นผลรีวิว ADR รายบุคคลของกชพรรณสำหรับงานที่ 3 จึงเป็นคนละเอกสารกับผลรีวิวที่กลุ่มใช้

### Product Owner — ท้าทายสมมติฐานทางธุรกิจ

Pon ถามว่าลูกค้าและธุรกิจยอมรับการรออีเมลยืนยันได้เพียงใด หากมี 200 orders/s และส่งได้สูงสุด 50 emails/s โดยสมมติให้มีอีเมลหนึ่งฉบับต่อ order และอัตรานี้คงอยู่ backlog จะเพิ่มประมาณ 9,000 ฉบับต่อนาที นอกจากนี้ยังถามว่าจะจัดการข้อกำหนด loyalty ภายในหนึ่งนาทีอย่างไรเมื่อ broker ล่ม กลุ่มจึงต้องตกลงเวลาล่าช้าและความคาดหวังการกู้คืนอย่างชัดเจน

### SRE — ท้าทายความน่าเชื่อถือและการปฏิบัติการ

Piyapanchanok ถามถึงวิธีรัน outbox relay, retry และการจัดการข้อความซ้ำ พร้อมเสนอให้กำหนด monitoring, alerting, เกณฑ์ queue depth, dead-letter handling และขั้นตอน recovery รวมทั้งทดสอบ throughput และ availability ในสภาพใช้งานจริง ก่อนอ้างว่าระบบรองรับ peak และเป้าหมายเวลา loyalty ได้

### Security Officer — ตรวจความปลอดภัยและความเป็นส่วนตัว

Chayutpong ชี้ว่า queued events อาจมี email address และข้อมูลคำสั่งซื้อ โดยตามข้อมูลที่กลุ่มอภิปราย ADR ยังเปิดประเด็นข้อมูลส่วนบุคคลไว้แต่ไม่ได้กำหนดมาตรการครบถ้วน จึงต้องระบุ fields ที่จำเป็น สิทธิ์ publish/consume การยืนยันตัวตนและการปกป้อง connection ตลอดจน retention ของข้อความและ dead-letter records พร้อมจำกัดการเข้าถึงข้อความและ operational logs

### Architect — ตอบข้อกังวลและสรุป

Pitchanan เห็นด้วยว่าต้องกำหนด failure semantics เกณฑ์ยอมรับทางธุรกิจ และข้อกำหนดความปลอดภัยให้ชัดขึ้น โดยเสนอให้ปรับ ADR ก่อน implementation และยังไม่ถือว่าการออกแบบผ่านการตรวจสอบครบแล้ว

## 3. มติและสิ่งที่กลุ่มตกลงให้แก้ไข

**มติ:** Accept with changes — อนุมัติโดยมีเงื่อนไข
**ผลลงคะแนน:** 5–0 เห็นชอบให้รับ ADR ภายใต้เงื่อนไขต่อไปนี้

| # | สิ่งที่ต้องแก้ไขตามมติกลุ่ม |
|---|---|
| 1 | ทำขอบเขต transaction ระหว่าง payment, order persistence และ event publication ให้ชัด พร้อมกำหนด recovery และ reconciliation เมื่อเกิดความล้มเหลวระหว่างขั้นตอน |
| 2 | ระบุ delivery และ recovery guarantees: relay, retry, duplicate handling, dead-letter processing, monitoring และ recovery พร้อมตรวจว่ารองรับ workload และ latency targets ได้ |
| 3 | ตกลง service expectations ทางธุรกิจเรื่องเวลารอ email การระบาย backlog และ loyalty ภายในหนึ่งนาทีระหว่าง broker outage; ใช้หลักฐานรองรับการเปรียบเทียบ broker |
| 4 | เพิ่มข้อกำหนด security/privacy: ลดข้อมูลส่วนบุคคลใน event และกำหนด authentication, authorization, การปกป้อง connection, retention และสิทธิ์เข้าถึง dead-letter queue |
| 5 | ตรวจสอบการออกแบบก่อน production โดยทดสอบ peak throughput, broker outage, consumer failure, duplicate delivery และ recovery |

มตินี้กำหนดงานที่ต้องทำต่อ ยังไม่ใช่หลักฐานว่าปิดเงื่อนไขหรือผ่านการทดสอบแล้ว

## 4. ตารางข้อกังวล

| ข้อกังวล | ผู้เสนอ | อยู่ในผลรีวิว AI หรือไม่ | AI มีข้อมูลพอจะรู้หรือไม่ |
|---|---|---|---|
| Payment สำเร็จแต่ไม่มี durable event | Architect และ SRE | มี — Risk ในผลรีวิวล่าสุด | อนุมานความเสี่ยงจาก transaction boundary ที่ระบุได้ แต่นโยบาย recovery ที่ถูกต้องต้องตกลงโดยมนุษย์ |
| Loyalty อาจเกินหนึ่งนาทีเมื่อเกิด outage | PO และ SRE | มี — สถานการณ์ Availability และ Verdict ข้อ 2 | ระบุข้อขัดแย้งได้ แต่ลำดับความสำคัญทางธุรกิจและการยอมรับ degradation ต้องให้ผู้เกี่ยวข้องตัดสินใจ |
| Email backlog โตเมื่อโหลดพีก | PO | มี — Sensitivity point เรื่อง MailFast quota | คำนวณอัตรา backlog ได้ แต่ต้องยืนยันระยะ peak และเวลารอที่ยอมรับได้ |
| ข้ออ้างเปรียบเทียบ broker ยังขาดหลักฐาน | Architect | มี — ผลรีวิวของ Pitchanan ทักว่าข้ออ้างเรื่อง rate limiting ของ Kafka ไม่ได้แยกจาก RabbitMQ และ Kafka อาจตอบ drivers 1–5 ได้เช่นกัน | มีข้อมูลจากตัวเลือกและเหตุผลใน ADR ของ Pitchanan จึงตรวจความสอดคล้องของข้ออ้างได้ แต่ยังต้องมีหลักฐานเรื่องภาระ operation และความเหมาะสมกับทีม |
| ข้อมูลส่วนบุคคลอาจถูกเปิดเผยหรือเก็บค้างใน queues และ dead-letter records | Security Officer | ไม่เป็น finding หลักในผลรีวิวล่าสุด; ตามบันทึกกลุ่ม ADR มี privacy เป็นคำถามเปิด | ชี้คำถามเปิดและเสนอ controls ได้ แต่องค์กรต้องกำหนดข้อกำหนด security/privacy ที่ใช้จริง |

## 5. ประเมินผลรีวิว AI ตามการอภิปราย

**ข้อค้นพบที่มีประโยชน์ที่สุด:** ช่องว่างระหว่าง payment กับ outbox ทำให้กลุ่มเห็น failure scenario ที่ต้องออกแบบรองรับอย่างชัดเจน และแยกได้ว่าการมี outbox ไม่ได้แปลว่า payment กับ event workflow สอดคล้องกันทุกกรณี

**ข้อค้นพบที่อ่อนที่สุดตามบันทึกกลุ่ม:** ตัวอย่าง peak 30 นาทีต้องใช้อย่างระมัดระวัง เพราะ scenario ไม่กำหนดระยะ peak แม้ผลรีวิวของ Pitchanan จะระบุ [Inferred] ไว้แล้วก็ตาม กลุ่มจึงมองว่าเป็นตัวอย่างที่ช่วยอธิบาย backlog แต่ยังใช้แทน capacity plan จากระยะ peak จริงไม่ได้ ไม่ถือว่าระยะ 30 นาทีเป็นข้อกำหนดหรือเป็นตัวเลขที่ยืนยันแล้ว

## 6. การใช้บันทึกร่วมกัน

สมาชิกใช้บันทึกการอภิปรายฉบับเดียวกันใน repository ของแต่ละคน ส่วน Reflection เขียนแยกจากมุมมองและบทบาทของแต่ละคน บันทึกนี้แทนร่างแบบจำลองเดิม และนำบทสนทนาตัวละครสมมติออกแล้ว
