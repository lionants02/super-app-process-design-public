# สารบัญ
- [สารบัญ](#สารบัญ)
- [SCS DB(ถังแดง) คือ](#scs-dbถังแดง-คือ)
- [การคำนวณหาเงินที่จ่ายตอนส่งเบิกระหว่าง SCS DB(ถังแดง) กับ Super APP](#การคำนวณหาเงินที่จ่ายตอนส่งเบิกระหว่าง-scs-dbถังแดง-กับ-super-app)
  - [ความหมายความเป็นมา](#ความหมายความเป็นมา)
  - [สาเหตุที่ต้องทำการคำนวณค่าใหม่](#สาเหตุที่ต้องทำการคำนวณค่าใหม่)

# SCS DB(ถังแดง) คือ
เป็นฐานข้อมูลกำเบิกจ่ายกลางของ สปสช. เพื่อเช็คความซ้ำซ้อนของการส่งเบิก และ ติดตามข้อมูลการส่งเบิกแต่ละระบบ
```mermaid
flowchart TD

mop[("Moxx App")]
kxx[("KXX App")]
sup[("Super App")]
scs[("SCS DB\nถังแดง")]

mop-->scs
kxx-->scs
sup-->scs
```
# การคำนวณหาเงินที่จ่ายตอนส่งเบิกระหว่าง SCS DB(ถังแดง) กับ Super APP

```mermaid
sequenceDiagram
    autonumber
    Actor sup as Super App
    participant etl as การคำนวณ
    Actor scs as ถังแดง

    sup->>etl:ข้อมูลการเบิกจ่าย
    rect yellow
        etl-->>etl:คำนวณค่า AMOUNT_PAID
    end
    etl-->>scs:ข้อมูลการเบิกจ่าย
```

## ความหมายความเป็นมา  
เนื่องจากโครงสร้างฐานข้อมูล Super App ถูกพัฒนามาก่อนที่โครงสร้าง SCS DB(ถังแดง) จะประกาศใช้ จึงจำเป็นต้องทำการแปลงโครงสร้างให้อยู่ในรูปแบบของ SCS DB(ถังแดง) ก่อน   
- Super App ฐานข้อมูล Super App  
    ตารางฐานข้อมูลที่ใช้จะมี `eclaimowner.m_process_result` ตารางเดียว
  - ฟิวด์ที่เกี่ยวข้อง
    - `eclaimowner.m_process_result.m_serv_result` รายการผลของการประมวลผลเบิกจ่ายจาก Super App แยกตาม `localcode` และ `subFund`
      - `totpayProcessed/AMOUNT_PAID` ที่จ่าย
      - `subFund`
      - `localcode` แยกตาม local code
    - `eclaimowner.m_process_result.deduct` ผลการประมวลผลยอดที่มีการหักออกสำหรับการเบิกจ่าย Super App
      - `amount` จำนวนที่ต้องหักออกจากรายการ `m_serv_result.totpayProcessed/AMOUNT_PAID` เป็นตามสมการ $` จำนวนเงินที่ส่งเบิก\ สปสช. = totpayProcessed - amount`$ โดยนำแต่ละรายการใน `m_serv_result` มาหักออกด้วย `amount` จนกว่าค่า `amount` จะเป็น 0
      - `subfund`
      - `is_paid_by_nhso` จ่ายโดย สปสช. ต้องเป็น `true`

- SCS DB(ถังแดง)  
    ในที่นี้จะเจาะจงเฉพาะฟิวด์ `SCS_OWNER.NHSO_PAY_RESULT.AMOUNT_PAID` ที่เป็นการกำหนดเงินที่จ่าย โดยถ้าเกิดเป็นการเหมาะจ่ายจะเป็น 0  
  - ฟิวด์ที่เกี่ยวข้อง
    - `SCS_OWNER.NHSO_PAY_RESULT.AMOUNT_PAID` ที่เป็นการกำหนดเงินที่จ่าย โดยถ้าเกิดเป็นการเหมาะจ่ายจะเป็น 0

## สาเหตุที่ต้องทำการคำนวณค่าใหม่
เนื่องจากบางครั้งมีเงื่อนไขการเบิกจ่ายจากกองทุน เช่น ถ้าตรงเงื่อนไข__<u>aaa</u>__ให้เหมาจ่ายส่วนนี้ได้ __<u>xxx</u>__บาท

    เช่น  สมมุติผู้ป่วยมารับบริการ มีรายการดังนี้  
  1. ค่าทำแผล 200 บาท
  2. ค่ายาชา 200 บาท
  3. พาราเซตามอล 500 mg จำนวน 1 กระปุ๋ก 40 บาท
  4. ยาฆ่าเชื้อ amoxicillin 500 mg 1 แผง 60 บาท

  รวม 500 บาท  
  แล้วตรงกับเงื่อนไขพิเศษบางอย่างที่บอกว่า ในรายการนี้ให้เบิกได้ **\_\_<u>200</u>\_\_** บาท  
   ฟิวด์ `eclaimowner.m_process_result.deduct` จะได้ค่าออกมาเป็น **$-300$**

```mermaid
sequenceDiagram
    autonumber
    Actor patient as ผู้ป่วย
    participant hos as สถานพยาบาล
    Actor nhso as สปสช.

    rect yellow
        nhso ->>hos:ประกาศ สมมุติ ตอนนี้มีนโยบาย .... ก่อนส่งเบิก
    end
    patient ->> hos:เกิดอุบัติเหตุ มารับการรักษา

    rect rgb(167, 199, 250)
        hos -->> hos:ค่ารักษารวม 500 บาท
            hos -->> hos:ประมวลผลเบิกจ่าย
            note over hos:**deduct.amount** = -300
            note over hos:ทำแผล totpayProcessed/AMOUNT_PAID = 200
            note over hos:ยาชา totpayProcessed/AMOUNT_PAID = 200
            note over hos:พาราเซตามอล... totpayProcessed/AMOUNT_PAID = 40
            note over hos:amoxicillin... totpayProcessed/AMOUNT_PAID = 60
    end

    rect yellow
        hos-->>hos:นำยอด deduct.amount มาหักออกทีละรายการ totpayProcessed/AMOUNT_PAID จนเหลือ 0
    end
    rect rgb(255, 163, 255)
        hos ->> nhso:ทำเรื่องเบิกไป สปสช. 200 บาท เพราะว่า นำ -300 มาหักออกจากยอดที่มี
            note over hos,nhso:ทำแผล totpayProcessed/AMOUNT_PAID = 0 เพราะว่า หักออก 200
            note over hos,nhso:ยาชา totpayProcessed/AMOUNT_PAID = 100 เพราะว่า หักออก 100
            note over hos,nhso:พาราเซตามอล... totpayProcessed/AMOUNT_PAID = 40 ไม่ได้หัก
            note over hos,nhso:amoxicillin... totpayProcessed/AMOUNT_PAID = 60 ไม่ได้หัก
    end

```
