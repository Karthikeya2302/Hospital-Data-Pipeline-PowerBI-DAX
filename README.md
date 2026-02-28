# 🏥 End-to-End Healthcare Analytics Pipeline

A complete data analytics project built on a hospital management dataset covering **data cleaning, normalization, data modeling, DAX measure development, and interactive dashboard creation in Power BI.**

---

## 📌 Project Overview

| Detail | Description |
|--------|-------------|
| **Domain** | Healthcare / Hospital Management |
| **Tools Used** | Microsoft Excel, Power Query, Power BI Desktop, DAX |
| **Dataset** | Single flat table with 33+ columns |
| **Records** | ~10,000 patient records |
| **Dashboards** | 6 interactive pages |

---

## 📂 Project Structure

```
Healthcare-Analytics-Pipeline/
│
├── Data/
│   └── hospital_data.xlsx          # Cleaned and normalized Excel file (4 tables)
│
├── PowerBI/
│   └── Hospital_Data_Analysis.pbix # Power BI dashboard file
│
├── Screenshots/
│   ├── Overview.png
│   ├── Patient_Analysis.png
│   ├── Doctor_Department.png
│   ├── Billing_Revenue.png
│   ├── Admission_Surgery.png
│   └── Feedback.png
│
└── README.md
```

---

## 🗂️ Dataset Description

The raw dataset was a single denormalized flat table containing the following categories of information:

- **Patient Info** — PatientID, PatientName, Age, Gender, City, State, RegistrationDate
- **Doctor Info** — DoctorID, DoctorName, Specialization, Department, ConsultationFee
- **Admission Info** — AdmissionDate, DischargeDate, RoomType, AdmissionReason, Diagnosis, Disease, SurgeryRequired, SurgeryDate, FollowUpDate
- **Billing Info** — TotalBillAmount, AmountPaid, PendingAmount, PaymentMode, BillingDate, InsuranceProvider
- **Feedback** — FeedbackRating

---

## 🧹 Phase 1 — Data Cleaning & Transformation (Power Query)

### 1. Data Type Assignment
Assigned correct data types to all columns:
- **Text** — categorical columns (PatientName, Gender, City, State, Disease, Diagnosis, PaymentMode etc.)
- **GUID/UUID** format IDs (PatientID, DoctorID) set as **Text** since they are non-numeric identifiers
- **Currency** — monetary columns (TotalBillAmount, AmountPaid, PendingAmount, ConsultationFee)
- **Whole Number** — integer columns (Age, FeedbackRating)
- **Boolean** — binary columns (SurgeryRequired)
- **Date** — all temporal columns (AdmissionDate, DischargeDate, RegistrationDate, SurgeryDate, FollowUpDate, BillingDate)

### 2. Composite Key Deduplication
Applied deduplication using a **composite key** instead of a single column:
- Unique key: **PatientID + AdmissionDate**
- Rationale: Same patient can have multiple valid admissions. Deduplicating on PatientID alone would delete valid revisit records.

### 3. Null Value Treatment
- Used **Column Quality profiling** (View tab) to identify null percentages across all columns
- Conditionally populated columns retained as null: SurgeryDate, FollowUpDate, InsuranceProvider, DischargeDate, Email
- Replaced nulls with `0` for AmountPaid and PendingAmount to enable accurate aggregations
- Validated critical columns (PatientName, DoctorName, Diagnosis, TotalBillAmount) for unexpected nulls
- Note: Empty cells in Date columns automatically convert to null upon data type assignment — expected behavior

### 4. Text Standardization
- Applied **Trim** and **Clean** transformations to all text columns to remove whitespace and non-printable characters
- Standardized text casing using **Capitalize Each Word** for name and location columns
- Enforced consistent categorical values in Gender and SurgeryRequired columns

### 5. Data Validation
- **Age** — filtered and removed out-of-range values (below 0 or above 120)
- **FeedbackRating** — validated values are within 1–5 range only
- **Gender** — standardized to consistent Male/Female values
- **SurgeryRequired** — converted to Boolean data type (True/False)

### 6. Data Integrity Check — PendingAmount Verification
Verified accuracy of PendingAmount by creating a calculated validation column:

```
PendingCheck = [TotalBillAmount] - [AmountPaid]
```

Created a Match/Mismatch flag to identify inconsistencies:

```
= if Number.Round([PendingAmount],2) = Number.Round([PendingCheck],2) 
  then "Match" else "Mismatch"
```

- Identified rows where original PendingAmount had data entry errors
- **Resolution**: Dropped original PendingAmount column and replaced with calculated PendingCheck column

### 7. Feature Engineering (Calculated Columns)
Created new derived columns to enrich the dataset:

```
LengthOfStay  = Duration.Days([DischargeDate] - [AdmissionDate])

AgeGroup      = if [Age] < 18 then "Minor"
                else if [Age] < 40 then "Young Adult"
                else if [Age] < 60 then "Middle Aged"
                else "Senior"

PaymentStatus = if [PendingAmount] > 0 then "Pending" else "Cleared"
```

### 8. Column Pruning
Removed low-value, PII, and redundant columns:

| Column Removed | Reason |
|----------------|--------|
| Address | Too granular, location data unreliable |
| PhoneNumber | PII, not needed for analysis |
| Email | PII, not needed for analysis |
| DoctorPhone | PII, not needed for analysis |
| RoomNumber | Too granular |
| BedNumber | Too granular |
| TreatmentGiven | Unstructured free text, not usable in visuals |
| PendingCheck | Validation column only, not needed after verification |

> **Data Quality Note**: Location data (Address, City, State) was found to be inconsistent — address fields contained different cities/states than their respective columns. This is a known data quality issue. Map visuals were not used. State column retained for filtering purposes only.

---

## 🗃️ Phase 2 — Data Normalization (Star Schema Design)

The single denormalized flat table was decomposed into **4 normalized tables** using Reference queries in Power Query to maintain dependency on the cleaned base query.

### Table Structure

```
                    ┌─────────────────┐
                    │  Patient Table  │
                    │  (Dimension)    │
                    │  PK: PatientID  │
                    └────────┬────────┘
                             │ 1:Many
              ┌──────────────┴──────────────┐
              │                             │
   ┌──────────▼──────────┐      ┌──────────▼──────────┐
   │   Admission Table   │      │    Billing Table     │
   │     (Fact)          │      │      (Fact)          │
   │  FK: PatientID      │      │  FK: PatientID       │
   │  FK: DoctorID       │      └─────────────────────┘
   └──────────▲──────────┘
              │ Many:1
   ┌──────────┴──────────┐
   │   Doctor Table      │
   │   (Dimension)       │
   │   PK: DoctorID      │
   └─────────────────────┘
```

### Table Details

| Table | Type | Primary Key | Columns |
|-------|------|-------------|---------|
| Patient Table | Dimension | PatientID | PatientID, PatientName, Age, AgeGroup, Gender, City, State, RegistrationDate |
| Doctor Table | Dimension | DoctorID | DoctorID, DoctorName, Specialization, Department, ConsultationFee |
| Admission Table | Fact | PatientID + AdmissionDate | PatientID, DoctorID, AdmissionDate, DischargeDate, RoomType, AdmissionReason, Diagnosis, Disease, SurgeryRequired, SurgeryDate, FollowUpDate, LengthOfStay, FeedbackRating |
| Billing Table | Fact | PatientID + BillingDate | PatientID, TotalBillAmount, AmountPaid, PendingAmount, PaymentMode, BillingDate, InsuranceProvider, PaymentStatus |

> FeedbackRating was moved into Admission Table since feedback is per admission, not per patient.

---

## 🔗 Phase 3 — Data Modeling (Power BI)

### Relationships

| From | To | Cardinality | Filter Direction |
|------|----|-------------|-----------------|
| Patient Table [PatientID] | Admission Table [PatientID] | One-to-Many | Single |
| Patient Table [PatientID] | Billing Table [PatientID] | One-to-Many | Single |
| Doctor Table [DoctorID] | Admission Table [DoctorID] | One-to-Many | Single |

### DAX Measures
All measures stored in a dedicated `_Measures` table — corporate best practice.

```dax
-- Patient Measures
Total Patients       = DISTINCTCOUNT(Patient_Table[PatientID])
Average Age          = AVERAGE(Patient_Table[Age])

-- Billing Measures
Total Revenue        = SUM(Billing_Table[TotalBillAmount])
Total Amount Paid    = SUM(Billing_Table[AmountPaid])
Total Pending Amount = SUM(Billing_Table[PendingAmount])
Collection Rate %    = DIVIDE([Total Amount Paid], [Total Revenue], 0) * 100

-- Admission Measures
Total Admissions     = COUNT(Admission_Table[AdmissionDate])
Avg Length of Stay   = AVERAGE(Admission_Table[LengthOfStay])
Surgery Rate %       = DIVIDE(
                         COUNTROWS(FILTER(Admission_Table, 
                         Admission_Table[SurgeryRequired] = TRUE())),
                         [Total Admissions], 0) * 100

-- Doctor Measures
Avg Consultation Fee = AVERAGE(Doctor_Table[ConsultationFee])

-- Feedback Measures
Avg Feedback Rating  = AVERAGE(Admission_Table[FeedbackRating])
```

---

## 📊 Phase 4 — Dashboard (Power BI)

6 interactive dashboard pages with synced slicers for cross-page filtering.

### Slicers (Synced across all pages)
- 📅 Admission Date Range
- 👤 Gender
- 🏥 Department

---

### Page 1 — Hospital Overview
> High level KPIs and summary of the entire hospital dataset

| Visual | Type | Insight |
|--------|------|---------|
| Total Patients, Revenue, Pending, Avg Rating, Admissions, Avg LOS | Cards | Key KPIs at a glance |
| Gender Distribution | Pie Chart | Male vs Female patient split |
| Patients by AgeGroup | Bar Chart | Which age group visits most |
| Patients by Disease | Bar Chart | Most common diseases |
| Admission Trend | Line Chart | Monthly admission patterns |

---

### Page 2 — Patient Analysis
> Deep dive into patient demographics and admission patterns

| Visual | Type | Insight |
|--------|------|---------|
| Total Patients, Avg Age, Total Admissions | Cards | Patient summary |
| Disease vs Gender | Stacked Bar | Which diseases affect which gender more |
| Disease vs AgeGroup | Stacked Bar | Which age groups are prone to which diseases |
| Gender Distribution | Donut Chart | Overall gender split |
| Monthly Admission Trend | Line Chart | Seasonal admission patterns |

---

### Page 3 — Doctor & Department Analysis
> Performance analysis of doctors and departments

| Visual | Type | Insight |
|--------|------|---------|
| Total Admissions, Avg Fee, Avg Rating | Cards | Doctor performance summary |
| Top 10 Doctors by Patient Count | Bar Chart | Most in-demand doctors |
| Patients by Department | Treemap | Department wise patient load |
| Revenue by Department | Donut Chart | Revenue contribution per department |
| Avg Feedback Rating by Doctor | Column Chart | Doctor performance ranking |

---

### Page 4 — Billing & Revenue Analysis
> Financial performance and payment analysis

| Visual | Type | Insight |
|--------|------|---------|
| Total Revenue, Amount Paid, Pending, Collection Rate % | Cards | Financial KPIs |
| Revenue vs Pending by Department | Clustered Bar | Department wise financial health |
| Payment Mode Distribution | Donut Chart | How patients prefer to pay |
| Monthly Revenue Trend | Line Chart | Revenue growth over time |

---

### Page 5 — Admission & Surgery Analysis
> Operational insights on admissions and surgeries

| Visual | Type | Insight |
|--------|------|---------|
| Total Admissions, Avg LOS, Surgery Rate % | Cards | Operational KPIs |
| Surgery Required Distribution | Donut Chart | Surgery vs non surgery cases |
| Avg Length of Stay by RoomType | Bar Chart | Room type efficiency |
| Admission Trend over Time | Line Chart | Admission volume patterns |
| AdmissionReason Breakdown | Treemap | Most common reasons for admission |

---

### Page 6 — Feedback Analysis
> Patient satisfaction and doctor performance ratings

| Visual | Type | Insight |
|--------|------|---------|
| Avg Feedback Rating, Total Patients | Cards | Overall satisfaction |
| Avg Rating by Doctor | Bar Chart | Individual doctor performance |
| Rating Distribution (1–5) | Column Chart | Overall satisfaction spread |

---

## 💡 Key Business Insights

### Patient Insights
- Identifies the most common diseases by age group and gender
- Tracks seasonal admission trends — helps with resource planning
- Shows which age groups require the most hospital care

### Financial Insights
- Tracks revenue vs pending amount per department — identifies cash flow issues
- Shows preferred payment modes — helps streamline billing processes
- Monthly revenue trend helps with financial forecasting

### Operational Insights
- Average length of stay by room type — identifies efficiency gaps
- Surgery rate % helps with surgical team resource planning
- Peak admission months help with staff scheduling

### Doctor & Department Insights
- Identifies top performing and underperforming doctors by rating
- Shows which departments generate the most revenue
- Highlights departments with highest patient load

---

## 🛠️ Technical Skills Demonstrated

- **ETL Pipeline** — Extract, Transform, Load using Power Query
- **Data Cleaning** — Null handling, deduplication, type casting, text standardization
- **Feature Engineering** — LengthOfStay, AgeGroup, PaymentStatus derived columns
- **Data Normalization** — Star Schema with Fact and Dimension tables
- **Data Modeling** — Relationships, cardinality, filter direction in Power BI
- **DAX** — Measures using SUM, DISTINCTCOUNT, AVERAGE, DIVIDE, FILTER, COUNTROWS
- **Dashboard Design** — Multi-page interactive dashboard with synced slicers
- **Data Quality Assessment** — Column profiling, integrity checks, validation

---

## 📸 Dashboard Screenshots

> *(Add screenshots of each dashboard page here)*

---

## 🚀 How to Use

1. Download the repository
2. Open `Data/hospital_data.xlsx` to view the cleaned and normalized tables
3. Open `PowerBI/Hospital_Data_Analysis.pbix` in Power BI Desktop
4. Refresh data if prompted
5. Use slicers on Page 1 to filter across all pages

---

## 👤 Author

**Your Name**  
MS Computer Science — University of Alabama at Birmingham (UAB)  
[LinkedIn](#) | [GitHub](#) | [Portfolio](#)
