**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt และหลักฐานทางเทคนิค

**ชื่อ / รหัสนิสิต:** กชพรรณ เลิศปัญญาโรจน์ / 6630001321
**Agent:** Codex desktop
**โมเดล:** รอบเดิมบันทึกเป็น GPT-6 family; งานที่ 3 รอบล่าสุดระบุ Codex / GPT-6 ในหัวไฟล์ผลรีวิว
**แพ็กเกจ:** ไม่ระบุ
**วันที่:** 2026-10-27 (Asia/Bangkok)

บันทึกนี้สรุปการตั้งค่างาน ไฟล์หลักฐาน และผลลัพธ์ทางเทคนิค ช่องข้อความเป็นสรุปงานที่ดำเนินการ ไม่ใช่ transcript ของบทสนทนาส่วนตัว

| # | งาน / run | Agent + โมเดล | ข้อความ/การตั้งค่างานโดยย่อ | ไฟล์หลักฐานที่ใช้ | ขอบเขตและวิธีดำเนินการ | มีประโยชน์ (1–5) | ผลลัพธ์ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Codex / GPT-6 family | prompts/1; run A; evidence demo.proto | demo.proto; AGENTS.md; prompts/1 | อ้างระบบจาก proto; มีบริบทใบงานจากการวางแผน | 4 | outputs/1A-architecture.md |
| 2 | 1-B | Codex / GPT-6 family | prompts/1; run B; evidence proto + manifests | demo.proto; manifests12ไฟล์ตามรายการด้านล่าง | แยก configured dependency จากพฤติกรรม runtime | 5 | outputs/1B-architecture.md |
| 3 | 2-a (รอบแรก) | Codex / GPT-6 family | prompts/2; run a; requirement R-12; language Thai | template; R-12 | ร่างเน้น requirement แต่ session เห็น scenario.th.md มาก่อน จึงไม่ใช่ zero-shot ที่ควบคุมบริบทได้ | 3 | outputs/2a-adr.md |
| 4 | 2-b (รอบแรก) | Codex / GPT-6 family | prompts/2; run b; context scenario + example + checkout source | scenario.md; ADR0001; checkoutservice/main.go; template | ใช้ตัวเลขและข้อจำกัดจาก scenario; component ใหม่ระบุเป็นข้อเสนอ | 4 | outputs/2b-adr.md |
| 5 | 3 (รอบแรก) | Codex / GPT-6 family | prompts/3; adr ADR-0002; language Thai | ADR-0002; scenario.md; prompts/3 | reviewer อยู่ใน session เดียวกับผู้ร่าง จึงไม่ใช่ independent review | 4 | outputs/3-review.md |
| 6 | ตรวจ dependency และจัดเอกสาร | Codex / GPT-6 family | ตรวจ config เทียบ source และสรุปทางเลือก | manifests; checkoutservice/main.go; ใบงาน01–03 | ตารางตรวจและร่างคำอธิบายจัดทำโดย AI พร้อม path/symbol | 5 | ใบงาน01–03 และ ADR0002สำหรับตรวจรับ |
| 7 | ส่วน B แบบจำลองรายบุคคล | Codex / GPT-6 family | วิเคราะห์ ADR ผ่านมุมมอง architect, analyst, PO และ SRE | ADR0002; review3; scenario.md | แบบจำลองจากข้อมูลร่วม ไม่อ่านการ์ดบทบาท และไม่อ้างว่ามีการประชุมจริง | 3 | ใบงาน04 และร่างreflection05 |
| 8 | บทสนทนาซ้อม | Codex / GPT-6 family | จัดบทสนทนาจำลอง 4 บทบาท | ใบงาน04; outputs/3-review.md; ADR0002และscenarioที่อ่านแล้ว | ตัวละครและบทพูดเป็นเรื่องสมมติ; ไม่เพิ่มผู้ร่วมประชุมจริง | 3 | ใบงาน04ส่วน7 |
| 9 | 3 (รอบล่าสุด) | Codex / GPT-6 | ใช้ `prompts/3-review-adr.md` รีวิว ADR-0002; `language: Thai` | `AGENTS.md`; `prompts/3-review-adr.md`; `docs/adr/0002-use-rabbitmq-for-post-order-processing.md`; `docs/scenario.md` | เปิดแชตใหม่; ยืนยันรายการอ่านจากประวัติการเรียกเครื่องมือแล้ว ไม่พบการเปิดหลักฐานนอก ADR และ scenario นอกเหนือจากไฟล์กติกาและโจทย์ | 5 | outputs/3-review.md |

## วิธีดำเนินการและข้อจำกัด
- รอบแรกดำเนินทุก run ใน session เดียว เปรียบเทียบเนื้อหาได้ แต่ไม่ใช้สรุปผลเชิงเหตุของการเพิ่ม context แบบ controlled experiment; Run a/Run b รอบล่าสุดทำแยกแชตตามบันทึกท้ายไฟล์
- ใช้ terminal สำหรับเตรียม checkout และจัดเก็บ Git; ใช้ file API อ่านหลักฐาน/เขียนเอกสาร ไม่มีการรันระบบ microservices หรือ load test
- ADR0002 และคำตอบใบงานมี AI ช่วยร่าง; ล่าสุดเจ้าของงานตรวจรับข้อเสนอ ADR0002 และระบุชื่อใน Deciders โดยคง Proposed ส่วนคำตอบใบงานอื่นยังต้องตรวจรับ; Validation เป็นแผนทดสอบ ไม่ใช่ผลที่วัดแล้ว
- ส่วน B ร่างแรกเป็นแบบจำลอง ต่อมาได้อภิปราย ADR กับกลุ่มแล้วและลงมติอนุมัติโดยมีเงื่อนไข 5–0; ใบงาน 04 เป็นบันทึกการอภิปรายจริงตามข้อมูลที่ผู้จัดทำให้มา ส่วนแถว 7–8 เก็บเป็นประวัติขั้นเตรียมงาน และนำบทสนทนาสมมติออกจากบันทึกฉบับปัจจุบันแล้ว
- ไม่แก้ source code, prompts, AGENTS.md หรือตัวอย่าง ADR0001 และไม่อ่าน docs/review-board/ หรือ repo ต้นฉบับ Online Boutique

## ไฟล์หลักฐานและเอกสารที่เปิดอ่าน

รายการรวมจากหลายงานและหลายรอบ; รายการที่ยืนยันเฉพาะงานที่ 3 รอบล่าสุดอยู่ในแถว 9 และบันทึกท้ายไฟล์
- README.md
- AGENTS.md
- prompts/1-recover-architecture.md
- prompts/2-write-adr.md
- prompts/3-review-adr.md
- exercise/00-prompt-log.md
- exercise/01-recover-architecture.md
- exercise/02-write-adr.md
- exercise/03-ai-review.md
- exercise/04-review-board.md
- exercise/05-reflection.md
- docs/adr/template.md
- docs/scenario.th.md
- docs/scenario.md
- docs/adr/0001-use-redis-for-cart-storage.md
- system/online-boutique/protos/demo.proto
- system/online-boutique/kubernetes-manifests/adservice.yaml
- system/online-boutique/kubernetes-manifests/cartservice.yaml
- system/online-boutique/kubernetes-manifests/checkoutservice.yaml
- system/online-boutique/kubernetes-manifests/currencyservice.yaml
- system/online-boutique/kubernetes-manifests/emailservice.yaml
- system/online-boutique/kubernetes-manifests/frontend.yaml
- system/online-boutique/kubernetes-manifests/kustomization.yaml
- system/online-boutique/kubernetes-manifests/loadgenerator.yaml
- system/online-boutique/kubernetes-manifests/paymentservice.yaml
- system/online-boutique/kubernetes-manifests/productcatalogservice.yaml
- system/online-boutique/kubernetes-manifests/recommendationservice.yaml
- system/online-boutique/kubernetes-manifests/shippingservice.yaml
- system/online-boutique/src/checkoutservice/main.go
- docs/adr/0002-use-rabbitmq-for-post-order-processing.md
- exercise/outputs/3-review.md

ไฟล์ผลลัพธ์ที่สร้างถูกอ่านซ้ำเพื่อตรวจความครบของเอกสาร; คำแนะนำเครื่องมือที่อ่านนอก repo คือ educators/index/SKILL.md และ write-like-me/SKILL.md

## การตรวจเอกสาร
- มีไฟล์ที่เตรียม12ไฟล์: ใบงาน00–05, outputs1A/1B/2a/2b/3-review และ ADR0002
- ตรวจ dependency7รายการเทียบ config/source; ตรวจ symbol17เส้นในแผนภาพ B และ alias ที่ใช้ใน Mermaid
- ตรวจหัวข้อ ADR, code fences และ whitespace; ยังไม่ได้ render Mermaid หรือรัน benchmark

## การทำรอบล่าสุด

- ทำ Run a ใหม่ในแชตใหม่ โดยให้ requirement R-12 และกำหนด context: none
- ทำ Run b ใหม่ในอีกแชต โดยให้ scenario, ตัวอย่าง ADR และ source ของ checkoutservice
- ใช้ไฟล์ outputs/2a-adr.md และ outputs/2b-adr.md ล่าสุดในการเปรียบเทียบ
- หมายเหตุเดิมเรื่องการทำใน session เดียวเป็นข้อมูลของร่างรอบแรก

- งานที่ 3 รอบล่าสุดทำในแชตใหม่ชื่อ “Review RabbitMQ order ADR” แยกจากแชตผู้ร่าง ใช้ `prompts/3-review-adr.md`, ADR-0002 ที่เลือกแล้ว และ `language: Thai`; ผลอยู่ใน `outputs/3-review.md`
- ตรวจประวัติการเรียกเครื่องมือของรอบล่าสุดแล้ว: เปิดอ่าน `AGENTS.md`, `prompts/3-review-adr.md`, `docs/adr/0002-use-rabbitmq-for-post-order-processing.md` และ `docs/scenario.md` จริง ไม่ใช่เพียงรายชื่อที่อ้างในเนื้อหา; ไม่พบการเปิด source code, ADR-0001 หรือ `docs/scenario.th.md` ในรอบนี้
- รอบรีวิวล่าสุดมีการใช้ terminal ค้นชื่อไฟล์และอ่าน `AGENTS.md` ช่วงเริ่มต้น จากนั้นใช้ file API อ่าน prompt และหลักฐาน จึงมีข้อคลาดเคลื่อนด้านวิธีใช้เครื่องมือจากกติกา “Don't run terminal commands” แม้ไม่พบการอ่านหลักฐานเกินขอบเขต
- ผู้จัดทำประเมินประโยชน์ของรีวิวรอบล่าสุด 5/5; ได้อภิปรายกลุ่มแล้ว โดย Kotchapan รับบท AI Analyst และใช้บันทึกกับมติในใบงาน 04 ส่วน Reflection ในใบงาน 05 เป็นมุมมองรายบุคคล

## การตรวจเอกสารอ้างอิงของส่วน B

- ส่วน B ใช้ ADR ของ Pitchanan Thaiphayak ใน repository Preem-pcn/swarch-ai4swarch แยกจาก ADR รายบุคคลของกชพรรณใน repository นี้
- ยืนยันชื่อเจ้าของและตัวเลือกใน [ADR ต้นทาง](https://github.com/Preem-pcn/swarch-ai4swarch/blob/11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e/docs/adr/0002-publish-order-events-via-rabbitmq.md) และยืนยัน Kafka/ตัวอย่าง peak 30 นาทีใน [ผลรีวิวต้นทาง](https://github.com/Preem-pcn/swarch-ai4swarch/blob/11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e/exercise/outputs/3-review.md) ซึ่งระบุระยะ peak เป็น [Inferred]
- เปิดอ่านผ่าน GitHub API: รายการ docs/adr/, docs/adr/0002-publish-order-events-via-rabbitmq.md, exercise/outputs/3-review.md และ exercise/04-review-board.md ของ repository ต้นทาง ไม่อ่าน docs/review-board/
- ใช้ commit 11a30aaf29a9aea5a17abf0a49c0e41bc2619c0e เป็นเวอร์ชันอ้างอิงคงที่ ปรับเฉพาะลิงก์และข้อความที่สับสนระหว่างผลรีวิวรายบุคคลกับผลรีวิวของกลุ่มในใบงาน04–05
- ข้อมูลการอภิปราย สมาชิก และมติมาจากบันทึกกลุ่มที่ผู้จัดทำให้มา การอ่าน repository ใช้ตรวจเนื้อหา ADR และผลรีวิว ไม่ใช่หลักฐานยืนยันว่าเกิดการประชุม
