- [ความสัมพันธ์](#ความสัมพันธ์)
  - [ความสัมพันธ์ของตาตรางฝั่ง NHSO\_](#ความสัมพันธ์ของตาตรางฝั่ง-nhso_)
  - [diagram ความสัมพันธ์คร้าว ๆ](#diagram-ความสัมพันธ์คร้าว-ๆ)

# ความสัมพันธ์
## ความสัมพันธ์ของตาตรางฝั่ง NHSO_

```mermaid
sequenceDiagram
    CHARD->NHSO_PAY_RESULT:LOCALCODE เท่ากับ ITEM_CODE
```

## diagram ความสัมพันธ์คร้าว ๆ

```mermaid
flowchart LR
    subgraph NHSO_*
        NHSO_AER["NHSO_AER\nแฟ้มข้อมูลอุบัติเหตุ"]
        NHSO_CHA["NHSO_CHA\nข้อมูลรายละเอียดทางการเงินของผู้เข้ารับบริการ"]
        NHSO_CHAD["NHSO_CHAD\nแฟ้มข้อมูลรายละเอียด \nค่าใช้จ่ายรายรายการ ของผู้เข้ารับบริการ"]
        NHSO_CMHS["NHSO_CMHS\nแฟ้มข้อมูลการให้บริการ \nผู้ป่วยจิตเวชเรื้อรัง \nในชุมชนข้อมูลการให้บริการผู้ป่วยจิตเวชเรื้อรัง ในชุมชน"]
        NHSO_DIAGNOSIS["NHSO_DIAGNOSIS\nแฟ้มข้อมูลวินิจฉัยโรค"]
        NHSO_DISABILITY["NHSO_DISABILITY\nแฟ้มข้อมูลการให้บริการ ผู้พิการ"]
        NHSO_INSCL["NHSO_INSCL\nแฟ้มเพิ่มเข้ามาในถังแดง สิทธิ์"]
        NHSO_NEWBORN["NHSO_NEWBORN\nแฟ้มข้อมูลประวัติ การคลอดของทารก"]
        NHSO_OPD["NHSO_OPD\nแฟ้มข้อมูลการรับบริการ ผู้ป่วยนอก"]
        NHSO_PATIENT["NHSO_PATIENT\nแฟ้มข้อมูลทั่วไป ข้อมูลทั่วไปของผู้เข้ารับบริการ"]
        NHSO_PAY_RESULT["NHSO_PAY_RESULT\nแฟ้มที่เพิ่มเข้ามาในถังแดง ข้อมูลการจ่าย"]
        NHSO_PRACTITIONER["NHSO_PRACTITIONER\nแฟ้มข้อมูลผู้ให้บริการ \nข้อมูลผู้ให้บริการหน่วยบริการ"]
        NHSO_PRENATAL["NHSO_PRENATAL\nแฟ้มข้อมูลประวัติ การตั้งครรภ์"]
        NHSO_PROCEDURE["NHSO_PROCEDURE\nแฟ้มข้อมูลการทำหัตถการ"]
        NHSO_PROVIDER["NHSO_PROVIDER\nแฟ้มข้อมูลหน่วยบริการ"]
    end

    subgraph Super app
        m_mental_std
        m_process_result
        m_register_std
        m_serv_std
        m_serviceitem_std
        m_diagnosis_std
        m_impairment_std
        m_birth_std
        m_pregnancy_std
        m_procedure_std
    end

    NHSO_AER---m_mental_std
    NHSO_AER---|cid| m_register_std

    NHSO_CHA---m_serviceitem_std

    NHSO_CHAD---m_serv_std

    NHSO_CMHS---m_mental_std
    NHSO_CMHS---|cid| m_register_std

    NHSO_DIAGNOSIS---m_diagnosis_std

    NHSO_DISABILITY---m_impairment_std

    NHSO_INSCL---|cid| m_register_std
    NHSO_INSCL---|inscl| m_process_result

    NHSO_NEWBORN---m_birth_std

    NHSO_OPD---m_register_std

    NHSO_PATIENT---m_register_std

    NHSO_PAY_RESULT---m_register_std
    NHSO_PAY_RESULT---|budget| m_process_result
    NHSO_PAY_RESULT-----|localcode,chargeamt|m_serv_std

    NHSO_PRACTITIONER---|"เอาเฉพาะข้อมูลแพทย์"|m_diagnosis_std

    NHSO_PRENATAL---m_pregnancy_std

    NHSO_PROCEDURE---m_procedure_std

    NHSO_PROVIDER---m_register_std
```
