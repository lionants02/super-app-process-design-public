# การดึงข้อมูลกลับจากถังแดง

- [การดึงข้อมูลกลับจากถังแดง](#การดึงข้อมูลกลับจากถังแดง)
  - [โฟลการทำงาน](#โฟลการทำงาน)
    - [โฟลตาราง NHSO\_PAY\_RESULT ขากลับเข้า Super App](#โฟลตาราง-nhso_pay_result-ขากลับเข้า-super-app)
      - [โฟลการหา request\_id จาก trans\_no](#โฟลการหา-request_id-จาก-trans_no)
  - [ภาคผนวก](#ภาคผนวก)
    - [โฟล์กลับจากถังแดงที่เคยคุยกับพี่โต้งพี่แอร์เก่า](#โฟล์กลับจากถังแดงที่เคยคุยกับพี่โต้งพี่แอร์เก่า)


## โฟลการทำงาน

### โฟลตาราง NHSO_PAY_RESULT ขากลับเข้า Super App


```mermaid
flowchart TD
    version["Version\n1.3-04042024"]

    pref_super_app[("Super App Database")]
    pref_db_scs[("SCS DB(ถังแดง) \n SCS Oracle Database")]

    s("รับข้อมูลผลจาก SCS DB(ถังแดง)\n เข้าสู่ระบบ Super App\nหลังเวลา 15.05 น.");
    s--->nhso_pay["อ่าข้อมูลผลจากตาราง NHSO_PAY_RESULT\nโดยใช้เงื่อนไข RESULT IS NOT NULL \n และ trunc(updated_date)>=trunc(sysdate-3)"];
    pref_db_scs -.->|"ข้อมูล"| nhso_pay
    
    nhso_pay-->result0[["อัพเดทข้อมูลที่มีการเปลี่ยนแปลง \n หรือ \n เพิ่มข้อมูลที่ใหม่ \n ลงตาราง big_table.audit_result"]]
    result0-.->|"ข้อมูล"| pref_super_app

    result0-->split_taska((p))

    split_taska-->|"ข้อมูลชุดที่ลง audit_result"| result[["ปรับ ลงตาราง \n big_table.audit_result_latest --> เก็บค่าสถานะล่าสุดแต่ละฟิวด์ตาม trans_no \nbig_table.audit_result_archive --> เพิ่มข้อมูลแบบ history ข้อมูลมีอายุ xxx เดือน"]];

    result-.->|"ข้อมูล"| pref_super_app

    split_taska-->|"รายการ request_id ที่ลง audit_result"| record_super_app[["บันทึกว่ารับข้อมูลประมวลผลมาแล้ว\n กำหนด big_table.audit_request.state=2 \n เงื่อนไข audit_request_id = request_id"]]
    record_super_app-.->|"ข้อมูล"| pref_super_app
    
    record_super_app-->E("end");
```

#### โฟลการหา request_id จาก trans_no
    ด้านบนต้องใช้  
```mermaid
flowchart TD
    s("Find request_id (input trans_no)")
    s-->lookup["หา request_id จาก \n eclaimowner.m_register_std.request_id \n โดยเงื่อนไข trans_no=trans_no"]
    lookup-->return("Return request_id")
```

## ภาคผนวก

### โฟล์กลับจากถังแดงที่เคยคุยกับพี่โต้งพี่แอร์เก่า
เป็นโฟลตัวเก่าที่คุยในครั้งแรก
```mermaid
flowchart TD
    s("โฟลการดึงข้อมูลจากถังแดง  ตอน xx:xx น.\nเข้าสูงถัง super app")
    s-->condition{"สถานะ"}
    
    condition-->|"3, 4"| to_audit_result[["นำข้อมูลใส่ลงใน audit_result\nโดย ......"]]
    to_audit_result-->tong1[["พี่โต้งอ่านข้อมูล\nอ่าน audit_result วันละ 1 รอบ\nโดยข้อมูลที่อ่านไปแล้วให้ audit_request=2"]]
    tong1-->tong2[["นำข้อมูลที่อ่านมาใส่ใน........\nโดยคืนมาเป็น audit_result.result= 3,4 or 7\nและทำการลบทิ้ง (หาวิธีเก็บ log........ ต่างหาก)"]]
    tong2-->E("end")


    condition-->|"7"| seven_process[["พี่โจ้หาวิธี กรณี 7\n7>3 ,7>4\nใส่ในตาราง..........\nพร้อมใส่วันเวลาของ........"]]
    seven_process-->seven_process_tong[["พี่โต้งดึงข้อมูล\nจาก........ ใส่ใน........."]]
    seven_process_tong-->E
```