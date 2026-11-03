# UMKM Investment Decision Support System

Aplikasi desktop berbasis **Java Swing** yang membantu pemilik Usaha Mikro, Kecil, dan Menengah (UMKM) menentukan **satu prioritas investasi terbaik** dari beberapa alternatif, dengan mempertimbangkan kondisi keuangan, keterbatasan modal, dan potensi keuntungan. Proyek mata kuliah Pemrograman Berorientasi Objek (PBO).

## Daftar Isi

1. [Deskripsi](#1-deskripsi)
2. [Alur dan Fungsionalitas](#2-alur-dan-fungsionalitas)
3. [Rancangan Database](#3-rancangan-database)
4. [Rancangan Class](#4-rancangan-class)
5. [Cara Menjalankan](#5-cara-menjalankan)

---

## 1. Deskripsi

| Komponen | Pilihan |
|---|---|
| Bahasa dan GUI | Java (JDK 17+), Swing |
| Database | MySQL 8, nama database `umkm_investment_dss` |
| Akses data | JDBC (MySQL Connector/J) |
| Metode keputusan | SAW (Simple Additive Weighting) |
| Pengguna | Pemilik UMKM (semua kategori usaha), single user |

**Cakupan investasi:** peralatan, stok, ekspansi, dan lainnya. Semuanya diperlakukan sama karena sistem hanya membutuhkan biaya investasi, tambahan pendapatan, tambahan biaya operasional, dan periode.

**Metode SAW:** tiap alternatif dinilai dengan 4 indikator, dinormalisasi, dikalikan bobot, lalu dijumlahkan menjadi skor. Skor tertinggi menjadi rekomendasi.

| Kode | Indikator | Tipe | Bobot awal |
|---|---|---|---|
| C1 | ROI | Benefit | 0,30 |
| C2 | Net Profit | Benefit | 0,25 |
| C3 | Payback Period | Cost | 0,25 |
| C4 | Investment Cost | Cost | 0,20 |

---

## 2. Alur dan Fungsionalitas

### 2.1 Input, Proses, dan Output

| Tahap | Isi |
|---|---|
| **Input** | Profil usaha (nama, kategori, skala); kondisi keuangan (modal tersedia, pendapatan bulanan, pengeluaran operasional bulanan, dana operasional, dana cadangan); alternatif investasi (nama, jenis, biaya, tambahan pendapatan per bulan, tambahan biaya operasional per bulan, periode dalam bulan) |
| **Proses** | **A.** Financial Assessment: hitung modal teralokasi (modal tersedia − dana operasional − dana cadangan). **B.** Feasibility Check: biaya investasi harus ≤ modal teralokasi. **C.** Financial Analysis: hitung Net Profit, ROI, dan Payback Period. **D.** Decision Making: ranking dengan SAW |
| **Output** | Financial Overview, Feasibility Report, Investment Comparison (ROI, Net Profit, Payback, skor), dan satu Investment Recommendation beserta alasannya |

Status setiap alternatif setelah dianalisis:

| Status | Arti | Ikut ranking? |
|---|---|---|
| `ELIGIBLE` | Terjangkau dan menguntungkan | Ya |
| `NOT_AFFORDABLE` | Biaya melebihi modal teralokasi | Tidak |
| `NOT_VIABLE` | Net Profit ≤ 0 | Tidak |

### 2.2 Alur Proses

```mermaid
flowchart TD
    A([Mulai]) --> B["Isi Business Profile"]
    B --> C["Isi Financial Condition"]
    C --> D["Input Investment Alternatives"]
    D --> E["Jalankan Analisis"]
    E --> F["A. Financial Assessment: hitung modal teralokasi"]
    F --> G{"B. Biaya investasi ≤ modal teralokasi?"}
    G -->|Tidak| H["Status: NOT_AFFORDABLE"]
    G -->|Ya| I["C. Financial Analysis: Net Profit, ROI, Payback"]
    I --> J{"Net Profit positif?"}
    J -->|Tidak| K["Status: NOT_VIABLE"]
    J -->|Ya| L["Status: ELIGIBLE"]
    L --> M{"Ada alternatif ELIGIBLE?"}
    M -->|Ya| N["D. SAW: normalisasi, bobot, skor, ranking"]
    N --> O["Rekomendasi: peringkat 1"]
    M -->|Tidak| P["Tidak ada rekomendasi"]
    H --> Q["Simpan hasil ke database"]
    K --> Q
    O --> Q
    P --> Q
    Q --> R["Tampilkan hasil di Dashboard, Analysis, Reports"]
    R --> S([Selesai])
```

### 2.3 Menu Aplikasi

Aplikasi memakai satu `MainFrame` dengan menu samping (sidebar) dan area konten berbasis `CardLayout`.

| Menu | Fungsi |
|---|---|
| Dashboard | Ringkasan keuangan, jumlah alternatif, dan rekomendasi terakhir |
| Business Profile | Mengelola profil UMKM |
| Financial Condition | Memasukkan data keuangan; modal teralokasi tampil otomatis |
| Investment Alternatives | Tambah, ubah, hapus, dan lihat alternatif investasi |
| Financial Analysis | Menampilkan Net Profit, ROI, Payback Period, dan status kelayakan |
| Decision Analysis | Menjalankan analisis; menampilkan skor, ranking, dan rekomendasi |
| Reports | Hasil analisis terakhir dan riwayat analisis |

---

## 3. Rancangan Database

**Database:** `umkm_investment_dss` (MySQL). Skrip pembuatan tabel berada di `src/main/resources/script.sql`.

```mermaid
erDiagram
    BUSINESS_PROFILE ||--|| FINANCIAL_CONDITION : has
    BUSINESS_PROFILE ||--o{ INVESTMENT_ALTERNATIVE : considers
    BUSINESS_PROFILE ||--o{ ANALYSIS_SESSION : runs
    ANALYSIS_SESSION ||--o{ ANALYSIS_RESULT : contains
    INVESTMENT_ALTERNATIVE ||--o{ ANALYSIS_RESULT : evaluated_in

    BUSINESS_PROFILE {
        int id PK
        varchar name
        varchar category
        enum business_scale
    }
    FINANCIAL_CONDITION {
        int id PK
        int business_id FK
        decimal available_capital
        decimal avg_monthly_revenue
        decimal avg_monthly_expense
        decimal operational_fund
        decimal reserve_fund
    }
    INVESTMENT_ALTERNATIVE {
        int id PK
        int business_id FK
        varchar name
        enum investment_type
        decimal investment_cost
        decimal additional_revenue
        decimal additional_opex
        int period_months
    }
    CRITERIA {
        int id PK
        varchar code
        varchar name
        enum criteria_type
        decimal weight
    }
    ANALYSIS_SESSION {
        int id PK
        int business_id FK
        timestamp analyzed_at
        decimal allocable_capital
    }
    ANALYSIS_RESULT {
        int id PK
        int session_id FK
        int alternative_id FK
        varchar alternative_name
        decimal investment_cost
        decimal monthly_net_profit
        decimal net_profit
        decimal roi
        decimal payback_period
        enum status
        decimal saw_score
        int rank_no
    }
```

| Tabel | Fungsi |
|---|---|
| `business_profile` | Profil UMKM |
| `financial_condition` | Kondisi keuangan UMKM (relasi 1:1 dengan profil) |
| `investment_alternative` | Alternatif investasi yang dipertimbangkan |
| `criteria` | Master 4 indikator SAW: tipe (benefit/cost) dan bobot |
| `analysis_session` | Satu kali eksekusi analisis (riwayat) |
| `analysis_result` | Hasil tiap alternatif pada satu sesi analisis; menyimpan salinan nilai agar riwayat tetap utuh walau alternatif diubah atau dihapus |

---

## 4. Rancangan Class

### 4.1 Struktur Proyek

```
src/main/
├── java/com/github/umkminvestmentdss/
│   ├── Main.java
│   ├── entity/        BusinessProfile, FinancialCondition, InvestmentAlternative,
│   │                  Criterion, AnalysisSession, AnalysisResult
│   │                  + enum: BusinessScale, InvestmentType, CriterionType, FeasibilityStatus
│   ├── model/         InvestmentAlternativeTableModel, AnalysisResultTableModel,
│   │                  AnalysisSessionTableModel
│   ├── repository/    DatabaseUtil, CrudRepository, BusinessProfileRepository,
│   │                  FinancialConditionRepository, InvestmentAlternativeRepository,
│   │                  CriterionRepository, AnalysisRepository
│   ├── service/       AnalysisService, FinancialAnalysisService, DecisionMethod, SawMethod
│   └── view/          MainFrame, BasePanel, DashboardPanel, BusinessProfilePanel,
│                      FinancialConditionPanel, InvestmentAlternativesPanel,
│                      FinancialAnalysisPanel, DecisionAnalysisPanel, ReportPanel
└── resources/
    └── script.sql
```

| Paket | Peran |
|---|---|
| `entity` | Data yang dipetakan ke tabel, beserta enum |
| `model` | `TableModel` yang menghubungkan data dengan `JTable` |
| `repository` | Akses database dengan JDBC |
| `service` | Logika analisis finansial dan SAW |
| `view` | Antarmuka Swing, tanpa rumus |

```mermaid
flowchart LR
    V["view"] --> S["service"]
    V --> R["repository"]
    V --> M["model"]
    S --> R
    S --> E["entity"]
    R --> E
    M --> E
    R --> DB[("MySQL")]
```

CRUD sederhana dilakukan `view` → `repository`. Proses analisis dilakukan `view` → `AnalysisService` → `repository`.

### 4.2 Diagram Class: Entity

```mermaid
classDiagram
    direction LR

    class BusinessScale {
        <<enumeration>>
        MIKRO
        KECIL
        MENENGAH
    }
    class InvestmentType {
        <<enumeration>>
        PERALATAN
        STOK
        EKSPANSI
        LAINNYA
    }
    class CriterionType {
        <<enumeration>>
        BENEFIT
        COST
    }
    class FeasibilityStatus {
        <<enumeration>>
        ELIGIBLE
        NOT_AFFORDABLE
        NOT_VIABLE
    }

    class BusinessProfile {
        -int id
        -String name
        -String category
        -BusinessScale businessScale
    }
    class FinancialCondition {
        -int id
        -int businessId
        -double availableCapital
        -double avgMonthlyRevenue
        -double avgMonthlyExpense
        -double operationalFund
        -double reserveFund
        +getAllocableCapital() double
        +getMonthlyCashFlow() double
    }
    class InvestmentAlternative {
        -int id
        -int businessId
        -String name
        -InvestmentType investmentType
        -double investmentCost
        -double additionalRevenue
        -double additionalOpex
        -int periodMonths
    }
    class Criterion {
        -int id
        -String code
        -String name
        -CriterionType type
        -double weight
        +isBenefit() boolean
    }
    class AnalysisResult {
        -String alternativeName
        -double investmentCost
        -double monthlyNetProfit
        -double netProfit
        -double roi
        -Double paybackPeriod
        -FeasibilityStatus status
        -double sawScore
        -int rank
        +getValue(Criterion c) double
    }
    class AnalysisSession {
        -int id
        -int businessId
        -LocalDateTime analyzedAt
        -double allocableCapital
        -List~AnalysisResult~ results
        -String recommendationReason
        +getRecommended() AnalysisResult
    }

    BusinessProfile --> BusinessScale
    InvestmentAlternative --> InvestmentType
    Criterion --> CriterionType
    AnalysisResult --> FeasibilityStatus
    AnalysisSession "1" *-- "many" AnalysisResult
```

### 4.3 Diagram Class: Repository dan Service

```mermaid
classDiagram
    direction TB

    class DatabaseUtil {
        +getConnection() Connection
    }
    class CrudRepository~T~ {
        <<interface>>
        +insert(T entity) int
        +update(T entity) boolean
        +delete(int id) boolean
        +findById(int id) T
        +findAll() List~T~
    }
    class BusinessProfileRepository
    class FinancialConditionRepository
    class InvestmentAlternativeRepository
    class CriterionRepository
    class AnalysisRepository {
        +save(AnalysisSession session) int
        +findSessionsByBusinessId(int id) List~AnalysisSession~
    }

    class DecisionMethod {
        <<interface>>
        +rank(List~AnalysisResult~ candidates, List~Criterion~ criteria) List~AnalysisResult~
    }
    class SawMethod
    class FinancialAnalysisService {
        +evaluate(InvestmentAlternative alt, double allocableCapital) AnalysisResult
    }
    class AnalysisService {
        -DecisionMethod decisionMethod
        +runAnalysis(int businessId) AnalysisSession
    }

    CrudRepository <|.. BusinessProfileRepository
    CrudRepository <|.. FinancialConditionRepository
    CrudRepository <|.. InvestmentAlternativeRepository
    DecisionMethod <|.. SawMethod

    BusinessProfileRepository ..> DatabaseUtil
    FinancialConditionRepository ..> DatabaseUtil
    InvestmentAlternativeRepository ..> DatabaseUtil
    CriterionRepository ..> DatabaseUtil
    AnalysisRepository ..> DatabaseUtil

    AnalysisService --> DecisionMethod
    AnalysisService --> FinancialAnalysisService
    AnalysisService --> AnalysisRepository
```

| Class | Tanggung jawab |
|---|---|
| `FinancialAnalysisService` | Tahap B dan C untuk satu alternatif: status kelayakan, Net Profit, ROI, Payback Period |
| `SawMethod` | Tahap D: normalisasi, skor, dan ranking |
| `AnalysisService` | Mengorkestrasi tahap A sampai D, menyusun alasan rekomendasi, dan menyimpan hasil |

### 4.4 Diagram Class: View dan Model

```mermaid
classDiagram
    direction TB

    class JFrame
    class JPanel
    class AbstractTableModel

    class MainFrame {
        +showPanel(String name) void
    }
    class BasePanel {
        <<abstract>>
        +refreshData() void
    }
    class DashboardPanel
    class BusinessProfilePanel
    class FinancialConditionPanel
    class InvestmentAlternativesPanel
    class FinancialAnalysisPanel
    class DecisionAnalysisPanel
    class ReportPanel

    JFrame <|-- MainFrame
    JPanel <|-- BasePanel
    BasePanel <|-- DashboardPanel
    BasePanel <|-- BusinessProfilePanel
    BasePanel <|-- FinancialConditionPanel
    BasePanel <|-- InvestmentAlternativesPanel
    BasePanel <|-- FinancialAnalysisPanel
    BasePanel <|-- DecisionAnalysisPanel
    BasePanel <|-- ReportPanel
    MainFrame o-- BasePanel

    AbstractTableModel <|-- InvestmentAlternativeTableModel
    AbstractTableModel <|-- AnalysisResultTableModel
    AbstractTableModel <|-- AnalysisSessionTableModel
```

### 4.5 Penerapan OOP

| Konsep | Penerapan |
|---|---|
| **Encapsulation** | Atribut `entity` bersifat `private` dengan getter dan setter; validasi dasar ada di setter; perhitungan turunan seperti modal teralokasi dibungkus di `FinancialCondition` |
| **Inheritance** | `MainFrame` mewarisi `JFrame`; tujuh panel mewarisi `BasePanel` (turunan `JPanel`); tiga TableModel mewarisi `AbstractTableModel` |
| **Abstraction** | `BasePanel` (abstract class), interface `CrudRepository<T>`, dan interface `DecisionMethod` |
| **Polymorphism** | `MainFrame` memanggil `refreshData()` pada tipe `BasePanel` dan tiap panel menjalankan versinya sendiri; `AnalysisService` menerima `DecisionMethod` (`new AnalysisService(new SawMethod())`); TableModel meng-*override* `getValueAt()` dan `getColumnCount()` |

---

## 5. Cara Menjalankan

1. Pastikan JDK 17+ dan MySQL 8 sudah terpasang.
2. Jalankan `src/main/resources/script.sql` pada MySQL untuk membuat database, tabel, dan data indikator awal.
3. Sesuaikan `URL`, `USERNAME`, dan `PASSWORD` di `repository/DatabaseUtil.java`.
4. Tambahkan MySQL Connector/J ke classpath proyek.
5. Jalankan `Main.java`.