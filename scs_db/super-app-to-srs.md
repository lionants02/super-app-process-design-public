# การนำข้อมูล super app ขึ้นไปถัง Oracle SCS

```mermaid
gantt
    title ตารางงานฝั่ง ETL
    dateFormat  YYYY-MM-DD
    axisFormat %Y-%m-%d

    section SCS ถังแดง
    1. พัฒนาส่งข้อมูลไปประมวลผลจาก Super App ไป SCS   :flow, 2024-03-18, 13d
    2. พัฒนาส่งข้อมูลผลลัพธ์จากผ SCS ไป Super App    :flow2, 2024-03-31, 5d
    3. พัฒนาส่วนอัพเดทสถานะการจ่ายจาก Super App ไป SCS :flow3, 2024-04-5, 5d
```

- [การนำข้อมูล super app ขึ้นไปถัง Oracle SCS](#การนำข้อมูล-super-app-ขึ้นไปถัง-oracle-scs)
  - [โฟลภาพรวม](#โฟลภาพรวม)
    - [ภาพการไหลของข้อมูลเข้าแต่ละ Database](#ภาพการไหลของข้อมูลเข้าแต่ละ-database)
    - [โฟลการไหลข้อมูลเข้า SCS DB ถังแดง](#โฟลการไหลข้อมูลเข้า-scs-db-ถังแดง)
      - [Ref.scs\_process](#refscs_process)
      - [Ref.process\_by\_request\_id](#refprocess_by_request_id)
    - [ภาพความสัมพันธ์การเกิดข้อมูล NHSO\_PAY\_RESULT กับ Super App](#ภาพความสัมพันธ์การเกิดข้อมูล-nhso_pay_result-กับ-super-app)
    - [โฟลตาราง NHSO\_PAY\_RESULT ขานำข้อมูลจาก Super App มาใส่](#โฟลตาราง-nhso_pay_result-ขานำข้อมูลจาก-super-app-มาใส่)
  - [ภาคผนวก](#ภาคผนวก)
    - [โฟลเงื่อนไขการการเริ่มการทำงาน ว่าจะประมวลผลข้อมูลจากข้อมูลชุดใด](#โฟลเงื่อนไขการการเริ่มการทำงาน-ว่าจะประมวลผลข้อมูลจากข้อมูลชุดใด)


## โฟลภาพรวม

### ภาพการไหลของข้อมูลเข้าแต่ละ Database

```mermaid
sequenceDiagram
    participant sup as Super App Database
    participant proc as Process Database
    participant temp as Temp SCS(ถังแดง) Database
    participant scs as SCS(ถังแดง) Database

    sup->>proc: แปลงเป็นโครงสร้าง SCS(ถังแดง) ใน scs_db.nhso_***
    proc->>temp: ใส่ข้อมูลลงในถังทดสอบ ใน SUPERAPP_APPL.SUP_APPL_***
    temp->>scs: ใส่ข้อมูลเข้าถัง SCS ใน SCS_OWNER.NHSO_***
```

### โฟลการไหลข้อมูลเข้า SCS DB ถังแดง
```mermaid
flowchart TD
    version["Version\n1.2-31032024"]
    pref[("scs_db.scs_audit_request\nrequest_id\n PostgreSQL Database")]
    pref_proc[("Process data\n scs_db.nhso_* \n PostgreSQL Database")]
    pref_temp_scs[("ORA Temp SCS(ถัง สปสช.)\nECLAIM_API.SUP_APPL_*\n Oracle Database")]
    pref_scs[("ORA SCS(ถังแดง สปสช.)\n SCS_OWNER.NHSO_* \n Oracle Database")]

    start("การส่งข้อมูลเข้า\nSCS DB สปสช.(ถังแดง)\nทำหลัง 00.00 น. ต้องเสร็จก่อน 05.30 น.")

    start --> process_air[["ประมวลผลข้อมูลต่อจาก super app\n ใช้ข้อมูลจาก big_table.audit_request \n ด้วยเงื่อนไข where state=0"]]
    subgraph "&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; call 10_pre_job_process"
        process_air --> check1[["สร้างรายการ request_id \n ที่ยังไม่เคยส่งเข้า SCS DB"]]
    end

    check1 -->|"รายการ request_id"| process_a[["Link Ref.scs_process\nสร้างชุดข้อมูล SCS DB ด้วย ETL\nจาก audit_request\nโดยที่ใส่ข้อมูลไว้ที่ schema scs_db\n call 50_run_job"]]
    process_a -.->|"ข้อมูล"| pref_proc

    process_a-->process_a_[["คัดลอกข้อมูล  schema scs_db \nไปยังฐานพักบน สปสช. \n call 120_to_database_oracle('ECLAIM_API','SUP_APPL_')"]]

    process_a_ -.->|"ข้อมูล"| pref_temp_scs
    pref_proc -.->|"ข้อมูล"| process_a_
    
    process_a_-->process_red[["นำข้อมูลใส่ลงlบนโครงสร้าง SCS DB สปสช. \n call 120_to_database_oracle('SCS_OWNER','NHSO_')"]]
    pref_temp_scs -.->|"ข้อมูล"| process_red
    process_red -.->|"ข้อมูล"| pref_scs
    

    process_red-->rec_request_id["บันทึกรายการ \naudit_reques id \nที่เคยประมวลผลแล้ว\n call 99_stamp_job_request_id_done"]
    
    rec_request_id---->END("end")
    rec_request_id -..-> |"ข้อมูล\n INSERT INTO \n request_pref (request_id,status)) \n VALUES('request_id,1')"| pref
    pref -.->|"ข้อมูล\n SELECT status \n from request_pref \n where request_id"| check1
```

#### Ref.scs_process
    อ้างอิงเป็น subfunction มาจากโฟลหลัก
```mermaid
flowchart TD

  s("Link Ref.scs_process\nสร้างชุดข้อมูล SCS DB ด้วย ETL\nจาก audit_request\nโดยที่ใส่ข้อมูลไว้ที่ schema scs_db")
  s--> tuncate["TRUNCATE \n ลบข้อมูลในตาราง process"]
  tuncate-->|"รายการ request_id\n 0cedfab6d64d48b9bb0aea2912660430 \n 61e756358ef149a2b5d65c2c5183d878 \n 8e66bd4167684ac8adbee09a26afa583"| loop(("ทำทีละ\nรายการ"))

  loop --> call_l8[["เรียกใช้ Ref.process_by_request_id('request_id')"]]
  call_l8 --> check{"ครบ\nทุกรายการ"}
  check-->|"ยังไม่ครบ"| loop
  check-->|"ครบแล้ว"| return("Return")
```

#### Ref.process_by_request_id  
    อ้างอิง subfunction จากอันบน
```mermaid
flowchart TD
    s("Ref.process_by_request_id\nประมวลผลข้อมูลครั้งละ request_id")
    s-->collect["call collect \nรวบรวมข้อมูลที่เกี่ยวข้องกับ request_id"]
    collect-->process["call 2_process_by_request_id \nประมวลผลโครงสร้าง SCS DB \nใส่ไว้ในโครงสร้างไฟล์"]
    process-->send_process_db["call 3_send_to_process_db \nนำข้อมูลโครงสร้างใส่ลงในฐานข้อมูลประมวลผล"]
    send_process_db-->R("Return")
```

### ภาพความสัมพันธ์การเกิดข้อมูล NHSO_PAY_RESULT กับ Super App

```mermaid
sequenceDiagram
    activate m_register_std
        NHSO_PAY_RESULT->m_register_std:HCODE ---- hcode
        NHSO_PAY_RESULT->m_register_std:HCODE_PAID ---- hcode
        NHSO_PAY_RESULT->m_register_std:PID ---- cid
        NHSO_PAY_RESULT->m_register_std:REFERENCE_ID ---- trans_no
        NHSO_PAY_RESULT->m_register_std:DATE_ADMIT ---- dateopd
    deactivate m_register_std
    activate m_process_result
        NHSO_PAY_RESULT->m_process_result:SUBFUND ---- m_serv_result.subFund
        NHSO_PAY_RESULT->m_process_result:ITEM_CODE ---- m_serv_result.localcode
        NHSO_PAY_RESULT->m_process_result:AMOUNT_PAID ---- m_serv_result.totpayProcessed
    deactivate m_process_result
    activate m_serv_std
        NHSO_PAY_RESULT->m_serv_std:AMOUNT_CHARGE ---- chargeamt
        NHSO_PAY_RESULT->m_serv_std:REIMB_UNIT_PRICE ---- reimbprice
    deactivate m_serv_std

```

### โฟลตาราง NHSO_PAY_RESULT ขานำข้อมูลจาก Super App มาใส่


```mermaid
flowchart TD
  s("การสร้างชุดข้อมูล\nNHSO_PAY_RESULT\nหลังเวลาเที่ยงคืน จาก super app\nเป็นโฟลที่ข้อมูลไหลต่อมาจากส่วนคัดเลือกข้อมูลด้วย audit_request")
  
  s -->m_serv_group[["จัดกลุ่มข้อมมูลในตาราง m_serv_std\nด้วยฟิวด์  stdcode,trans_no\nเพื่อให้รู้จำนวนแถว NHSO_PAY_RESULT"]]
  subgraph "กลุ่มตาราง m_serv"
      ref["select trans_no, \nstdcode as &quot;ITEM_CODE&quot; ,count(*) as &quot;count row&quot; \nfrom eclaimowner.m_serv_std \ngroup by trans_no,&quot;ITEM_CODE&quot; \norder by &quot;count row&quot; desc ;"]

      m_serv_group-->|"trans_no|ITEM_CODE|count row\n14818-6712523-123|55020|1\n11482-2919089-23|H9339|1\n11482-2919089-68|781380|1\n11482-2919089-68|813745|1\n11482-2919089-68|D3MON|1"|m_serv_subfun[["หา SUBFUND จากตาราง m_serv_std.subfund\nโดยที่ trans_no==trans_no และ ITEM_CODE==stdcode"]]
  end

  m_serv_subfun -->|"trans_no|ITEM_CODE|SUBFUND\n14818-6712523-123|55020|OPBKK_FS\n11482-2919089-23|H9339|OPBKK_REHAB\n11482-2919089-68|781380|OPBKK_DRUG\n11482-2919089-68|813745|OPBKK_DRUG\n11482-2919089-68|D3MON|OPBKK_OTHER"| AMOUNT_CHARGE[["หา AMOUNT_CHARGE\nจาก m_process_result.m_serv_result.totpayProcessed\nโดยที่ trans_no,m_serv_result.stdCode เหมือนกัน\nและ processFlag เป็น Y หรือ P"]]

  AMOUNT_CHARGE -->|"trans_no|ITEM_CODE|SUBFUND|AMOUNT_CHARGE\n14818-6712523-123|55020|OPBKK_FS|100\n11482-2919089-23|H9339|OPBKK_REHAB|150\n11482-2919089-68|781380|OPBKK_DRUG|null\n11482-2919089-68|813745|OPBKK_DRUG|null\n11482-2919089-68|D3MON|OPBKK_OTHER|1"| create_pay[["จะได้มูลตั้งต้น NHSO_PAY_RESULT\nหา SEQ ของชุดข้อมูลเพื่อนำมาใส่ค่าใน REFERENCE_ID ในรูปแบบ\nREFERENCE_ID=SEQ"]]

  create_pay-->create_pay2[["ดึงข้อมูลจากตาราง m_register_std ด้วย trans_no เดียวกัน\nใส่ในตาราง NHSO_PAY_RESULT ตามนี้\nhcode --> HCODE\nhcode_send --> HCODE_PAID\ncid --> PID\ndateopd --> DATE_ADMIT\ntotal_claim --> AMOUNT_PAID"]]

  create_pay2-->PARENT_REFERENCE_ID[["หา PARENT_REFERENCE_ID\nกรณีมีการแก้ไข/อุทธรณ์ต้องอ้างอิง seq เดิมด้วย"]]

  PARENT_REFERENCE_ID-->create_date_send["คำนวนหา DATE_SEND จาก\nวันที่ปัจุบันที่เป็นเวลาหลังเที่ยงคืน\nลบด้วย 1 วัน เช่น ตอนนี้เวลา\n2024-02-25 01:39:55.000\nถอยหลังไป 1 วัน\nDATE_SEND=2024-02-24 01:39:55.000"]
  create_date_send-->contant["กำหนดฟิวด์ค่าคงที่\nSOURCE_ID=SPA\nRUN_NO=รูปแบบ yyyymmdd [เวลาปัจจุบัน]\nPROCESS_STATUS=1\nAPPEAL_STATUS=N"]
  contant-->E("end")

  stamp["flow\nversion\n0.1"]
```

## ภาคผนวก

### โฟลเงื่อนไขการการเริ่มการทำงาน ว่าจะประมวลผลข้อมูลจากข้อมูลชุดใด
    คุยกับพี่โต้ง พี่แอร์ เกี่ยวกับเงื่อนไขการประมวลผล ไม่ได้ใช้แล้ว เป็นบันทึกเฉยๆ
```mermaid
flowchart TD
    s("โฟลขาลงถังแดง ตอน xx:xx น.")
    s-->tong1[["พี่โต้งประมวลผลเสร็จ จะเพิ่มข้อมูลใน PostgreSQL\nตาราง audit_request\nโดยใช้เงื่อนไข audit_request.state = 0"]]

    tong1-->tong2[["เลือกแถวข้อมูลใน audit_request\nที่ยังไม่เคยส่งไปถังแดง"]]

    tong1-->tong3[["หา trans_no ที่เกี่ยวข้องจากตาราง m_process_result\nโดย audit_request.audit_request_id เท่ากับ m_process_result.request_id"]]
    tong2-->joe1[["หา trans_no สำหรับรวบรวมข้อมูลจาก m_*\nโดย join audit_request.audit_request_id = m_register_std.request_id\n และ ยังไม่เคยส่งไปถังแดง\nจะได้ trans_no จากตาราง m_register_std"]]
    joe1-->collect1[["นำ trans_no ที่ได้ไปรวบรวมข้อมูลจากตารางชุด m_*\nโดยจะใช้เฉพาะข้อมูลวันก่อนหน้า"]]
    collect1-->collect2[["นำข้อมูลมาจัดเป็นโครงสร้างถังแดง\n"]]
    collect2-->send[["เข้าสู่กระบวนการส่งข้อมูลเข้าถึงแดง\n***บันทึกสถานะ audit_request_id ว่าเคยส่งไปถังแดงแล้ว***"]]
    send-->E("end")
```