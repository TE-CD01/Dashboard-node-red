# TE-CDBU1 Factory System — Block Diagram

ระบบทั้งหมดของ TE-CDBU1: จากเครื่องเทส Chroma → SQL Server → Dashboard

---

## 1. ภาพรวมทั้งระบบ (Full System Overview)

```mermaid
flowchart TB
    subgraph FLOOR["🏭 FACTORY FLOOR — Station ATS2_CDBU"]
        CHROMA["⚡ Chroma ATS Tester<br/>(Test Equipment)"]
        TXTFILE["📄 Log File .txt<br/>D:\Chroma Temp Data\<br/>1085033-04 261607XXXXXX-HH-MM-DD-Mon-YYYY.txt"]
        CHROMA -->|"เทสเสร็จ → สร้างไฟล์"| TXTFILE
    end

    subgraph WATCHER["🐍 PYTHON WATCHER — chroma_watcher.py"]
        SCAN["🔍 Scan Folder ทุก 5 วินาที"]
        READY["✅ ตรวจไฟล์เขียนเสร็จ<br/>(File Size นิ่ง 1 วิ)"]
        PARSEFN["📛 Parse ชื่อไฟล์<br/>→ PartNumber, SerialNumber"]
        PARSEC["📋 Parse เนื้อหาไฟล์<br/>→ Header + Step Blocks"]
        DUPCHECK{"🔁 เช็คซ้ำ<br/>PartNumber+SerialNumber<br/>+StationName"}
        INSERTS["💾 INSERT TestSummary<br/>(1 แถว/ไฟล์)"]
        INSERTD["💾 INSERT TestDetail<br/>(1 แถว/Step)"]
        MOVEFILE["📦 Move File → Done Folder"]
        SKIP["⏭️ ข้าม (ซ้ำแล้ว)"]

        SCAN --> READY --> PARSEFN --> PARSEC --> DUPCHECK
        DUPCHECK -->|ไม่ซ้ำ| INSERTS --> INSERTD --> MOVEFILE
        DUPCHECK -->|ซ้ำ| SKIP --> MOVEFILE
    end

    subgraph SQLSERVER["🗄️ SQL SERVER — TE-CDBU1-Testdata<br/>THBPOSFPPDB\\THBPOSFPPDB"]
        TBL_SUM[("📊 TestSummary<br/>━━━━━━━━━━━━<br/>id, FileName, SerialNo<br/>PartNumber, SerialNumber<br/>CaseNumber, ComponentSN<br/>StationName, Testresult<br/>TotalSteps/Pass/Fail<br/>StartTime, ElapsedTime")]
        TBL_DET[("📋 TestDetail<br/>━━━━━━━━━━━━<br/>id, SummaryID(FK)<br/>StepNo, StepName<br/>StepResult, MeasuredValue<br/>ValueUnit, MinLimit, MaxLimit<br/>RawBlock")]
        TBL_SET[("⚙️ DashboardSettings<br/>━━━━━━━━━━━━<br/>SettingKey, SettingValue<br/>yield_target_*, alert_emails<br/>smtp_user/password")]

        TBL_SUM -.->|"1 : N"| TBL_DET
    end

    subgraph BACKEND["🚀 FASTAPI BACKEND — main.py (port 8000)"]
        DBPY["db.py<br/>Connection Pool<br/>+ Query Functions"]
        CPKPY["cpk.py<br/>คำนวณ CpK<br/>+ สร้าง Excel"]
        ALERTPY["alert.py<br/>ตรวจ Yield/CpK<br/>ทุก 5 นาที<br/>ส่ง Email SMTP"]

        subgraph APIS["API Endpoints"]
            A1["/api/summary"]
            A2["/api/stations"]
            A3["/api/station/{name}"]
            A4["/api/station/{name}/rows"]
            A5["/api/serial/{sn}"]
            A6["/api/cpk/{name}"]
            A7["/api/cpk/{name}/export"]
            A8["/api/alerts"]
            A9["/api/settings"]
        end

        APIS --> DBPY
        A6 & A7 --> CPKPY
        ALERTPY --> DBPY
        ALERTPY --> CPKPY
    end

    subgraph DASHBOARD["💻 DASHBOARD — db.html (Browser)"]
        UI_OVER["📊 Overview<br/>KPI + Station Cards<br/>+ Alert Banner"]
        UI_STA["◫ Station Detail<br/>Trend / Pareto / Records / CpK"]
        UI_SER["≡ Serial Detail<br/>Steps + Raw Log"]
        UI_CPK["∿ CpK Analysis<br/>Preview + Export Excel"]
        UI_ALERT["⚡ Alerts"]
        UI_SET["⚙️ Settings<br/>Yield Target + Email"]
    end

    subgraph EMAIL["📧 EMAIL ALERT"]
        OUTLOOK["Outlook SMTP<br/>smtp.office365.com:587"]
        RECIPIENT["👤 Engineer / Manager<br/>(Inbox)"]
        OUTLOOK --> RECIPIENT
    end

    %% Connections between groups
    TXTFILE -->|"อ่านไฟล์"| SCAN
    INSERTS -->|"pyodbc INSERT"| TBL_SUM
    INSERTD -->|"pyodbc INSERT"| TBL_DET

    DBPY <-->|"pyodbc SELECT/INSERT"| TBL_SUM
    DBPY <-->|"pyodbc SELECT/INSERT"| TBL_DET
    DBPY <-->|"GET/POST Settings"| TBL_SET

    A1 -->|"JSON"| UI_OVER
    A2 -->|"JSON"| UI_OVER
    A3 -->|"JSON"| UI_STA
    A4 -->|"JSON Pagination"| UI_STA
    A5 -->|"JSON"| UI_SER
    A6 -->|"JSON Preview"| UI_CPK
    A7 -->|"📥 .xlsx Download"| UI_CPK
    A8 -->|"JSON"| UI_ALERT
    A9 <-->|"GET/POST"| UI_SET

    ALERTPY -->|"smtplib"| OUTLOOK

    style FLOOR fill:#1a2235,stroke:#00d4aa,color:#fff
    style WATCHER fill:#0d1323,stroke:#3b82f6,color:#fff
    style SQLSERVER fill:#1a2235,stroke:#f59e0b,color:#fff
    style BACKEND fill:#0d1323,stroke:#22c55e,color:#fff
    style DASHBOARD fill:#1a2235,stroke:#00d4aa,color:#fff
    style EMAIL fill:#0d1323,stroke:#ef4444,color:#fff
```

---

## 2. Pipeline การ Import ข้อมูล (Data Ingestion Flow)

```mermaid
sequenceDiagram
    autonumber
    participant C as Chroma Tester
    participant F as D:\Chroma Temp Data\
    participant W as chroma_watcher.py
    participant DB as SQL Server

    C->>F: เทสเสร็จ → เขียนไฟล์ .txt
    loop ทุก 5 วินาที
        W->>F: สแกนหาไฟล์ .txt
    end
    F-->>W: เจอไฟล์ใหม่
    W->>W: รอ 1 วิ เช็ค File Size นิ่ง (เขียนเสร็จ)
    W->>W: Parse ชื่อไฟล์ → PartNumber, SerialNumber
    W->>W: อ่านเนื้อหา → Header + แยก Step Blocks (===)
    W->>DB: เช็คซ้ำ (PartNumber+SerialNumber+Station)
    alt ไม่ซ้ำ
        W->>DB: INSERT TestSummary → ได้ SummaryID
        W->>DB: INSERT TestDetail (ทุก Step, executemany)
        DB-->>W: Commit สำเร็จ
        W->>F: ย้ายไฟล์ → Done Folder
    else ซ้ำแล้ว
        W->>F: ย้ายไฟล์ → Done Folder (ไม่ INSERT)
    end
```

---

## 3. Lazy Load Flow บน Dashboard

```mermaid
flowchart LR
    START["👤 เปิด Dashboard"] --> KPI["GET /api/summary<br/>GET /api/stations<br/>━━━━━━━━━<br/>โหลด KPI รวม<br/>+ Station Cards<br/>~0.2 วิ"]

    KPI --> CLICK1{"คลิก<br/>Station Card?"}
    CLICK1 -->|Yes| STADETAIL["GET /api/station/{name}<br/>GET /api/station/{name}/rows<br/>GET /api/cpk/{name}<br/>━━━━━━━━━<br/>Trend + Pareto<br/>+ Records + CpK<br/>~0.5 วิ"]

    STADETAIL --> CLICK2{"คลิก<br/>SerialNo?"}
    CLICK2 -->|Yes| SERDETAIL["GET /api/serial/{sn}<br/>━━━━━━━━━<br/>ทุก Step<br/>+ Raw Log<br/>~0.3 วิ"]

    STADETAIL --> CLICK3{"กด<br/>Export Excel?"}
    CLICK3 -->|Yes| EXPORT["GET /api/cpk/{name}/export<br/>━━━━━━━━━<br/>คำนวณ CpK<br/>+ สร้าง .xlsx<br/>~1-2 วิ"]

    KPI -.->|"Auto ทุก 30 วิ"| KPI
    KPI -.->|"Auto ทุก 5 นาที"| ALERTCHK["GET /api/alerts<br/>━━━━━━━━━<br/>ตรวจ Yield Target"]

    style START fill:#00d4aa,color:#000
    style KPI fill:#1a2235,stroke:#3b82f6,color:#fff
    style STADETAIL fill:#1a2235,stroke:#22c55e,color:#fff
    style SERDETAIL fill:#1a2235,stroke:#f59e0b,color:#fff
    style EXPORT fill:#1a2235,stroke:#ef4444,color:#fff
    style ALERTCHK fill:#1a2235,stroke:#ef4444,color:#fff
```

---

## 4. Database Schema (ER Diagram)

```mermaid
erDiagram
    TestSummary ||--o{ TestDetail : "1 : N (SummaryID)"

    TestSummary {
        int id PK
        nvarchar FileName
        nvarchar Testprogram
        nvarchar SerialNo
        nvarchar PartNumber "1085033-04"
        nvarchar SerialNumber "261607000487"
        nvarchar CaseNumber "ECD16010089"
        nvarchar ComponentSN "BUNC1B598949#"
        nvarchar ModelName
        nvarchar TesterID
        nvarchar StationName "INDEX"
        nvarchar WorkOrder
        nvarchar LineName
        nvarchar GroupName
        nvarchar FixtureId
        nvarchar CarrierId
        datetime2 StartTime "INDEX DESC"
        nvarchar ElapsedTime
        nvarchar Testresult "PASS/FAIL"
        int TotalSteps
        int PassSteps
        int FailSteps
        datetime2 ImportedAt
    }

    TestDetail {
        int id PK
        int SummaryID FK
        nvarchar SerialNo
        int StepNo
        nvarchar StepName
        nvarchar StepResult "PASS/FAIL"
        float MeasuredValue "สำหรับ CpK"
        nvarchar ValueUnit
        nvarchar ValueLabel
        float MinLimit "LSL"
        float MaxLimit "USL"
        nvarchar RawBlock "Log ดิบ Step นี้"
    }

    DashboardSettings {
        nvarchar SettingKey PK
        nvarchar SettingValue
        nvarchar Description
        datetime2 UpdatedAt
    }
```

---

## 5. CpK & Alert Flow

```mermaid
flowchart TB
    subgraph TRIGGER["⏰ Trigger ทุก 5 นาที (Background Thread)"]
        T1["alert.check_and_alert()"]
    end

    T1 --> Q1["db.get_alerts(targets)<br/>━━━━━━━━━<br/>Query Yield ย้อนหลัง 24 ชม.<br/>GROUP BY StationName"]
    T1 --> Q2["db.get_cpk_data(station)<br/>━━━━━━━━━<br/>Query TestDetail<br/>WHERE MinLimit/MaxLimit NOT NULL"]

    Q1 --> CHECK1{"Yield < Target?"}
    Q2 --> CALC["cpk.calculate_cpk()<br/>━━━━━━━━━<br/>numpy: Mean, Std<br/>Cpu = (USL-Mean)/3σ<br/>Cpl = (Mean-LSL)/3σ<br/>CpK = min(Cpu,Cpl)"]
    CALC --> CHECK2{"CpK < 1.00?"}

    CHECK1 -->|Yes| COOL1{"Cooldown<br/>ผ่านแล้ว?"}
    CHECK2 -->|Yes| COOL2{"Cooldown<br/>ผ่านแล้ว?"}

    COOL1 -->|Yes| MAIL1["📧 ส่ง Email<br/>Yield Alert"]
    COOL2 -->|Yes| MAIL2["📧 ส่ง Email<br/>CpK Alert"]

    COOL1 -->|No, รออยู่| SKIP1["⏭️ ข้าม"]
    COOL2 -->|No, รออยู่| SKIP2["⏭️ ข้าม"]

    MAIL1 --> SMTP["smtplib → smtp.office365.com:587"]
    MAIL2 --> SMTP
    SMTP --> INBOX["📬 Inbox ผู้รับ<br/>(alert_emails)"]

    style TRIGGER fill:#0d1323,stroke:#f59e0b,color:#fff
    style MAIL1 fill:#ef4444,color:#fff
    style MAIL2 fill:#ef4444,color:#fff
    style CALC fill:#1a2235,stroke:#00d4aa,color:#fff
```

---

## 6. Step Parsing Logic (chroma_watcher.py)

```mermaid
flowchart TB
    FILE["📄 ไฟล์ .txt<br/>1085033-04 261607000487-08-33-22-April-2026.txt"]

    FILE --> SPLIT1["แยกชื่อไฟล์ด้วย Space<br/>━━━━━━━━━<br/>PartNumber = 1085033-04<br/>ส่วนที่เหลือ = 261607000487-08-33-22-April-2026"]

    SPLIT1 --> SPLIT2["Regex หา DateTime<br/>HH-MM-SS-Month-YYYY<br/>━━━━━━━━━<br/>SerialNumber = 261607000487<br/>FileDateTime = 08-33-22-April-2026"]

    FILE --> READ["อ่านเนื้อหาไฟล์ทั้งหมด<br/>(ลอง encoding: utf-8/latin-1/cp1252/cp874)"]

    READ --> HEADER["extract_header()<br/>━━━━━━━━━<br/>Testprogram, ComponentSN(Serial No)<br/>CaseNumber(Model Name)<br/>StartTime, ElapsedTime, Operator<br/>Testresult, StationName, LineName<br/>FixtureId, CarrierId, GroupName, WorkOrder"]

    READ --> STEPS["parse_step_blocks()<br/>━━━━━━━━━<br/>แยกด้วย ===== "]

    STEPS --> REGEX["Regex จับหัว Step<br/>STEP.N(UUT Test seq.N) : ชื่อ ... PASS/FAIL"]

    REGEX --> MEASURE["หาค่าวัดในแต่ละบรรทัด<br/>━━━━━━━━━<br/>Pattern: Label : LSL : Value : USL : Unit<br/>→ MeasuredValue, MinLimit, MaxLimit, Unit"]

    MEASURE --> COUNT["count_steps()<br/>━━━━━━━━━<br/>TotalSteps, PassSteps, FailSteps"]

    SPLIT2 --> COMBINE["รวมข้อมูลทั้งหมด"]
    HEADER --> COMBINE
    COUNT --> COMBINE

    COMBINE --> OUT1["→ INSERT TestSummary<br/>(1 record)"]
    COMBINE --> OUT2["→ INSERT TestDetail<br/>(N records, executemany)"]

    style FILE fill:#00d4aa,color:#000
    style OUT1 fill:#1a2235,stroke:#f59e0b,color:#fff
    style OUT2 fill:#1a2235,stroke:#f59e0b,color:#fff
```

---

## สรุปไฟล์ทั้งหมดในระบบ

| ไฟล์ | ภาษา | หน้าที่ |
|------|------|---------|
| `chroma_watcher.py` | Python | จับตามองโฟลเดอร์ → Parse .txt → INSERT SQL (Real-time) |
| `ChromaDataParser.cs` | C# | ทางเลือก: Chroma เรียกตรงเมื่อเทสเสร็จ |
| `db.py` | Python | Connection Pool + Query Functions |
| `cpk.py` | Python | คำนวณ CpK + สร้าง Excel |
| `alert.py` | Python | ตรวจ Yield/CpK ทุก 5 นาที + ส่ง Email |
| `main.py` | Python | FastAPI App รวมทุก Endpoint |
| `db.html` | HTML/JS | Dashboard หน้าเว็บ (Lazy Load) |
