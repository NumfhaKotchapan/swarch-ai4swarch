**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

> **ชื่อ / รหัสนิสิต:** กชพรรณ เลิศปัญญาโรจน์ / 6630001321
> **ขอบเขตการตรวจ:** เปรียบเทียบผล Run B กับ config, proto และจุดเรียกใน source โดยไม่ได้รันระบบ รายละเอียดการใช้เครื่องมือบันทึกใน `00-prompt-log.md`

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

การตรวจรอบนี้เทียบทั้ง Dependencies table และลูกศร `Rel(...)` ใน Run B กับไฟล์หลักฐาน โดยเลือก dependency ของ checkout 5 รายการ และตรวจเพิ่มเติมอีก 2 รายการ ได้แก่ checkout → shipping และ cart → Redis การตรวจจากไฟล์ช่วยยืนยันการตั้งค่าและจุดเรียกในโค้ด แต่ไม่ได้ยืนยันผลการรันระบบ

ตำแหน่งอ้างอิง: ผล AI คือ [`outputs/1B-architecture.md`](outputs/1B-architecture.md); YAML อยู่ใน `system/online-boutique/kubernetes-manifests/`; proto คือ [`demo.proto`](../system/online-boutique/protos/demo.proto); โค้ดที่ตรวจเพิ่มคือ [`src/checkoutservice/main.go`](../system/online-boutique/src/checkoutservice/main.go) เลขหลัง `:` คือบรรทัดในไฟล์

ผู้เรียกของ 6 แถวแรกยืนยันจาก `checkoutservice.yaml:16–18` (`kind: Deployment`, `metadata.name: checkoutservice`) โค้ด `main.go:111–123,202–218` อ่านตัวแปร address เหล่านี้และสร้าง connection ผ่าน `grpc.NewClient` ชื่อ Service และ selector ใน YAML ปลายทางตรงกับ Deployment; `name: grpc`, proto contract และจุดเรียก client ในโค้ดรองรับ protocol gRPC

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| checkoutservice → emailservice | Run B ตาราง:60 / แผนภาพ:100; `checkoutservice.yaml:63–64` (`EMAIL_SERVICE_ADDR`); `emailservice.yaml:73–83`; `demo.proto:199–201` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `emailservice:5000`; Service ส่งต่อ `5000 → 8080` ตรงกับ `containerPort` และ `PORT` ที่ `emailservice.yaml:48–51` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `SendOrderConfirmation`; `main.go:273,379–384` พบ `PlaceOrder` เรียก `sendOrderConfirmation` ซึ่งใช้ `EmailServiceClient.SendOrderConfirmation` |
| checkoutservice → paymentservice | Run B ตาราง:59 / แผนภาพ:99; `checkoutservice.yaml:61–62` (`PAYMENT_SERVICE_ADDR`); `paymentservice.yaml:71–81`; `demo.proto:177–179` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `paymentservice:50051`; Service ส่งต่อ `50051 → 50051` ตรงกับ `containerPort:50051` ที่ `paymentservice.yaml:48` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `Charge`; `main.go:252,369–376` พบ `PlaceOrder` เรียก `chargeCard` ซึ่งใช้ `PaymentServiceClient.Charge` |
| checkoutservice → cartservice | Run B ตาราง:62 / แผนภาพ:102; `checkoutservice.yaml:67–68` (`CART_SERVICE_ADDR`); `cartservice.yaml:72–82`; `demo.proto:23–27` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `cartservice:7070`; Service ส่งต่อ `7070 → 7070` ตรงกับ `containerPort:7070` ที่ `cartservice.yaml:48` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `GetCart` และ `EmptyCart`; `main.go:324–336` พบ `CartServiceClient` เรียกทั้งสอง RPC โดยมีจุดใช้ helper ที่ `main.go:290,263` |
| checkoutservice → productcatalogservice | Run B ตาราง:57 / แผนภาพ:97; `checkoutservice.yaml:57–58` (`PRODUCT_CATALOG_SERVICE_ADDR`); `productcatalogservice.yaml:71–81`; `demo.proto:71–75` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `productcatalogservice:3550`; Service ส่งต่อ `3550 → 3550` ตรงกับ `containerPort:3550` ที่ `productcatalogservice.yaml:48` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `GetProduct`; `main.go:294,339–356` พบ `prepOrderItems` สร้าง `ProductCatalogServiceClient` แล้วเรียก `GetProduct` |
| checkoutservice → currencyservice | Run B ตาราง:61 / แผนภาพ:101; `checkoutservice.yaml:65–66` (`CURRENCY_SERVICE_ADDR`); `currencyservice.yaml:72–82`; `demo.proto:140–143` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `currencyservice:7000`; Service ส่งต่อ `7000 → 7000` ตรงกับ `containerPort:7000` ที่ `currencyservice.yaml:49` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `Convert`; `main.go:302,348,359–366` พบ `convertCurrency` ใช้ `CurrencyServiceClient.Convert` |
| checkoutservice → shippingservice | Run B ตาราง:58 / แผนภาพ:98; `checkoutservice.yaml:59–60` (`SHIPPING_SERVICE_ADDR`); `shippingservice.yaml:71–81`; `demo.proto:107–110` | ✅ หลักฐาน config, proto และจุดเรียกใน source สอดคล้องกัน | ตั้งค่า `shippingservice:50051`; Service ส่งต่อ `50051 → 50051` ตรงกับ `containerPort:50051` ที่ `shippingservice.yaml:47` ตารางและลูกศรของ AI ระบุ gRPC และทิศทางตรงกัน; proto มี `GetQuote` และ `ShipOrder`; `main.go:313–321,386–395` พบ client เรียกทั้งสอง RPC โดยมีจุดใช้ helper ที่ `main.go:298,258` |
| cartservice → redis-cart | Run B ตาราง:64 / แผนภาพ:104; `cartservice.yaml:18,50–51` (`REDIS_ADDR`), `118–120` (`redis:alpine`), `146–156` (Service `redis-cart`, `tcp-redis`) | ✅ ตรงกับ configuration; ❓ ยังไม่ได้ตรวจ source ของ cartservice หรือ runtime | ตั้งค่า `redis-cart:6379`; Service ส่งต่อ `6379 → 6379` ตรงกับ container port ตารางและลูกศรของ AI ระบุ `cartservice → redis-cart`, TCP และพอร์ตตรงกัน; ยังยืนยัน wire format จาก YAML อย่างเดียวไม่ได้ ตามข้อจำกัดที่ AI ระบุไว้ ไม่ใช้ proto เป็นหลักฐานของเส้นนี้ |

- **สิ่งที่ผิดหรือขาดหายไป:** จาก dependency 5 รายการที่เลือกตรวจ และอีก 2 แถวเดิมที่ตรวจซ้ำ ยังไม่พบชื่อผู้เรียก ชื่อปลายทาง พอร์ต protocol หรือทิศทางลูกศรที่ขัดกับหลักฐาน ตารางและแผนภาพ Run B สอดคล้องกันในรายการเหล่านี้
- พอร์ต emailservice `5000` กับ container `8080` สอดคล้องกัน: Service ส่งต่อ `port:5000` ไป `targetPort:8080` จึงไม่ใช่ข้อผิดพลาด
- YAML ยืนยันการตั้งค่า; proto ยืนยัน RPC ที่มีให้เรียก; source ที่อ่านเพิ่มยืนยันจุดเรียกในโค้ดของ checkout แต่ไม่ได้ยืนยันว่าการเรียกสำเร็จจริงในสภาพแวดล้อมที่ deploy และยังไม่ได้ทดสอบพฤติกรรม retry หรือความล้มเหลวของระบบ
- Run B ล่าสุดแยกประเด็น shopping assistant ไว้ใน Gaps (`outputs/1B-architecture.md:117`) และระบุว่า loadgenerator ไม่อยู่ใน base Kustomization (`:118`) แล้ว จึงไม่บันทึกสองประเด็นนี้ว่า AI พลาดโดยอัตโนมัติ รอบนี้ไม่ได้เปิดไฟล์ของสองประเด็นนี้เพื่อตรวจซ้ำ

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. `.proto` บอกชื่อ service และ RPC contracts แต่ไม่บอกว่าใครเป็น caller, deploy แยกกันจริงหรือไม่, ใช้ภาษาอะไร หรือเก็บข้อมูลที่ไหน Run B เพิ่ม manifest จึงระบุ frontend, Redis และ configured dependency ได้ แต่ยังต้องอ่าน source เพื่อยืนยัน call sequence และ failure handling
2. ความเข้าใจผิดที่อาจส่งผลต่อสมาชิกใหม่มากที่สุดคือการเห็น address ของบริการใน config แล้วสรุปว่าบริการนั้นเปิดใช้งานจริง เช่น `SHOPPING_ASSISTANT_SERVICE_ADDR` เพราะอาจทำให้เข้าใจ topology ผิดและพยายามเรียกบริการที่ยังไม่เปิดใช้ อย่างไรก็ตาม Run B รอบนี้แยก shopping assistant ไว้ใน Gaps แล้ว จึงยกประเด็นนี้เป็นตัวอย่างความเสี่ยงหากไม่ตรวจหลักฐาน ไม่ใช่ข้อผิดพลาดที่พบในผลรอบนี้
