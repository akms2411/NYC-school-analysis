# SAT Modeling - Documentation

## Project Overview

This project implements a comprehensive ETL (Extract, Transform, Load) pipeline for cleaning, validating, and integrating NYC high school SAT results data into a PostgreSQL database. The workflow encompasses data quality assessment, duplicate detection, validation checks, and structured database integration.

**Author:** Alexander Kuhn  
**Date:** January 19, 2026  
**Platform:** Jupyter Notebook with Python 3.x

---

## Table of Contents

1.  [Project Objectives](#project-objectives)
2.  [Data Source](#data-source)
3.  [Technology Stack](#technology-stack)
4.  [Workflow Overview](#workflow-overview)
5.  [Detailed Steps](#detailed-steps)
6.  [Data Quality Issues](#data-quality-issues)
7.  [Database Schema](#database-schema)
8.  [Results](#results)
9.  [Usage](#usage)
10. [Key Insights](#key-insights)

---

## Project Objectives

The main objectives of this project are:

1. **Data Cleaning**: Replace invalid values with NaN, remove duplicates and irrelevant records
2. **Standardization**: Normalize column names and data types
3. **Validation**: Ensure SAT scores fall within valid range (200-800), setting invalid values to NaN
4. **Integration**: Securely store cleaned data in PostgreSQL
5. **Reproducibility**: Create a documented, reusable ETL process

---

## Data Source

**File:** `sat-results.csv`  
**Origin:** NYC High School SAT Results  
**Initial Records:** 493  
**Final Records:** 479  

### Original Columns

| Column Name | Type | Description |
|------------|------|-------------|
| `DBN` | Text | District Borough Number (unique school identifier) |
| `SCHOOL NAME` | Text | High school name |
| `Num of SAT Test Takers` | Text | Number of students who took the SAT |
| `SAT Critical Reading Avg. Score` | Text | Average reading score |
| `SAT Math Avg. Score` | Text | Average math score |
| `SAT Writing Avg. Score` | Text | Average writing score |
| `SAT Critical Readng Avg. Score` | Text | Duplicate column with typo |
| `internal_school_id` | Integer | Internal ID (not relevant for analysis) |
| `contact_extension` | Text | Contact extension (not relevant for analysis) |
| `pct_students_tested` | Text | Percentage of students tested |
| `academic_tier_rating` | Float | Academic tier rating (1-4) |

---

## Technology Stack

- **Python 3.x**
- **pandas** - Data manipulation and analysis
- **psycopg2** - PostgreSQL database connectivity
- **PostgreSQL (Neon)** - Cloud-based database platform
- **Jupyter Notebook** - Interactive development environment

---

## Workflow Overview

```
Load Raw Data
    ↓
Remove Redundant Columns
    ↓
Normalize Column Names
    ↓
Convert Data Types
    ↓
Validation Checks
    ↓
Replace Invalid Values with NaN
    ↓
Detect and Remove Duplicates
    ↓
Remove Non-Analytical Columns
    ↓
Convert pd.NA to Python None
    ↓
Insert into PostgreSQL
    ↓
Verify Data Integrity
```

---

## Detailed Steps

### 1. Import Required Libraries

```python
import pandas as pd
import psycopg2
from psycopg2.extras import execute_batch
```

**Purpose**: Load essential libraries for data manipulation and database connectivity.

---

### 2. Load Dataset

```python
df = pd.read_csv('sat-results.csv')
display(df.head())
print(df.info())
display(df.describe(include='all'))
```

**Result**: 493 rows, 11 columns loaded

**Initial Inspection Findings**:
- All SAT score columns stored as text (need conversion)
- Presence of duplicate column with typo
- Mixed data types requiring standardization

---

### 3. Cleaning Raw Dataset

#### 3.1 Remove Redundant/Invalid Columns

**Problem**: Column `SAT Critical Readng Avg. Score` is a duplicate with typo.

```python
df = df.drop(columns=['SAT Critical Readng Avg. Score'])
```

**Result**: Reduced to 10 columns

---

#### 3.2 Normalize Column Names

**Applied Transformations**:
- Strip leading/trailing whitespace
- Convert to lowercase
- Replace spaces with underscores
- Remove special characters

```python
df.columns = (
    df.columns
    .str.strip()
    .str.lower()
    .str.replace(' ', '_', regex=True)
    .str.replace(r'[^\w]', '', regex=True)
)
```

**Before**:
```
['DBN', 'SCHOOL NAME', 'Num of SAT Test Takers', ...]
```

**After**:
```
['dbn', 'school_name', 'num_of_sat_test_takers', ...]
```

**Result**: Consistent snake_case naming convention

---

#### 3.3 Type Conversion and Parsing

##### Numeric Columns

```python
numeric_cols = [
    'num_of_sat_test_takers',
    'sat_critical_reading_avg_score',
    'sat_math_avg_score',
    'sat_writing_avg_score'
]
df[numeric_cols] = df[numeric_cols].apply(pd.to_numeric, errors='coerce')
```

##### Percentage Field

```python
df['pct_students_tested'] = (
    df['pct_students_tested']
    .str.rstrip('%')
    .pipe(pd.to_numeric, errors='coerce')
)
```

**Example Transformation**:
- `"78%"` → `78.0`
- `"85%"` → `85.0`

##### Academic Tier Rating

```python
df['academic_tier_rating'] = (
    pd.to_numeric(df['academic_tier_rating'], errors='coerce')
    .astype('Int64')
)
```

**Distribution**:
- Tier 4: 112 schools
- Tier 2: 101 schools
- Tier 3: 96 schools
- Tier 1: 93 schools

**Result**: All fields have correct data types for analysis

---

### 4. Validation Checks

#### 4.1 SAT Score Validation

**Valid Range**: 200 - 800 per section

```python
# Check for invalid math scores
invalid_math = df[(df["sat_math_avg_score"] < 200) | (df["sat_math_avg_score"] > 800)]
```

**Identified Invalid Math Scores**:
- `999.0` (2 cases)
- `850.0` (1 case)
- `-10.0` (1 case)
- `1100.0` (1 case)

**Identified Invalid Reading/Writing Scores**: None

**Analysis**: 5 schools have invalid SAT math scores that will be replaced with NaN.

---

#### 4.2 Replace Invalid SAT Scores with NaN

```python
import numpy as np

df_filtered = df.copy()

for col in ['sat_critical_reading_avg_score', 'sat_math_avg_score', 'sat_writing_avg_score']:
    # Set values outside valid range (200-800) to NaN
    df_filtered.loc[(df_filtered[col] < 200) | (df_filtered[col] > 800), col] = np.nan

# No need to reset index as we're keeping all rows
```

**Result**: All 493 records are retained

**Approach**:
- Invalid SAT scores are set to NaN instead of removing entire rows
- This preserves schools with partial data (e.g., valid reading/writing but invalid math)
- Allows for more comprehensive analysis and data visibility

**Invalid Values Replaced with NaN**:
- Invalid SAT scores: 5 values replaced
- Missing SAT scores: Already NaN (53 schools with missing data)


---

#### 4.3 Duplicate Detection and Removal

```python
# Identify fully duplicated rows
df_full_duplicates = df_filtered[df_filtered.duplicated(keep=False)]

# Remove exact duplicates
df_filtered_unique = df_filtered.drop_duplicates()
```

**Identified Schools with Duplicates**:
- Landmark High School
- Murry Bergtraum High School for Business Careers
- Mott Hall High School
- South Bronx Preparatory
- Bronx Leadership Academy High School

**Result**: 479 unique records (14 duplicates removed)

**Duplicate Rate**: 2.8% of total records

---

#### 4.4 Remove Non-Analytical/Non-Relational Columns

```python
df_filtered_unique.drop(
    columns=['internal_school_id', 'contact_extension'],
    inplace=True
)
```

**Rationale**: These columns are not relevant for SAT performance analysis.

**Result**: 8 columns remain

**Final Column Schema**:
1. `dbn`
2. `school_name`
3. `num_of_sat_test_takers`
4. `sat_critical_reading_avg_score`
5. `sat_math_avg_score`
6. `sat_writing_avg_score`
7. `pct_students_tested`
8. `academic_tier_rating`

---

### 5. Converting Missing Values: pandas format → PostgreSQL format

**Problem**: psycopg2 cannot directly handle pandas' `pd.NA`.

**Solution**:

```python
# Step 1: Convert pd.NA to Python None
df_cleaned = df_filtered_unique.copy()
df_cleaned = df_cleaned.replace({pd.NA: None})

# Step 2: Ensure Python-native types for database compatibility
for col in df_cleaned.columns:
    if pd.api.types.is_integer_dtype(df_cleaned[col]):
        df_cleaned[col] = df_cleaned[col].astype(object)  # Int64 -> Python int/None
    elif pd.api.types.is_float_dtype(df_cleaned[col]):
        df_cleaned[col] = df_cleaned[col].astype(object)  # float64 -> Python float/None

# Convert to list of tuples for batch insertion
records = list(df_cleaned.itertuples(index=False, name=None))
```

**Technical Details**:
- pandas uses `pd.NA` for nullable types
- PostgreSQL expects `NULL` (represented as `None` in Python)
- psycopg2 is a C-based library that only understands Python native types

**Result**: All NULL values are PostgreSQL-compatible

---

### 6. PostgreSQL Database Integration

#### 6.1 Connection Parameters

```python
conn = psycopg2.connect(
    dbname="neondb",
    user="neondb_owner",
    password="<password>",
    host="ep-falling-glitter-a5m0j5gk-pooler.us-east-2.aws.neon.tech",
    port="5432",
    sslmode="require"
)
```

**Security Features**:
- SSL/TLS encryption enabled
- Parameterized queries prevent SQL injection
- Connection pooling for efficiency

---

#### 6.2 Table Creation

```sql
DROP TABLE IF EXISTS nyc_schools.alex_sat_scores CASCADE;

CREATE TABLE IF NOT EXISTS nyc_schools.alex_sat_scores (
    dbn TEXT PRIMARY KEY,
    school_name TEXT,
    num_of_sat_test_takers INTEGER,
    sat_critical_reading_avg_score FLOAT,
    sat_math_avg_score FLOAT,
    sat_writing_avg_score FLOAT,
    pct_students_tested FLOAT,
    academic_tier_rating INTEGER
);
```

**Design Decisions**:
- `dbn` as PRIMARY KEY ensures uniqueness
- FLOAT data type for SAT scores allows flexibility
- No NOT NULL constraints to preserve data integrity
- CASCADE on DROP prevents orphaned dependencies

---

#### 6.3 Data Insertion

```python
insert_query = """
    INSERT INTO nyc_schools.alex_sat_scores (
        dbn, school_name, num_of_sat_test_takers,
        sat_critical_reading_avg_score, sat_math_avg_score,
        sat_writing_avg_score, pct_students_tested,
        academic_tier_rating
    )
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
    ON CONFLICT (dbn) DO UPDATE SET
        school_name = EXCLUDED.school_name,
        num_of_sat_test_takers = EXCLUDED.num_of_sat_test_takers,
        sat_critical_reading_avg_score = EXCLUDED.sat_critical_reading_avg_score,
        sat_math_avg_score = EXCLUDED.sat_math_avg_score,
        sat_writing_avg_score = EXCLUDED.sat_writing_avg_score,
        pct_students_tested = EXCLUDED.pct_students_tested,
        academic_tier_rating = EXCLUDED.academic_tier_rating;
"""

execute_batch(cur, insert_query, records, page_size=100)
```

**Features**:
- `ON CONFLICT` handles potential duplicates gracefully
- `execute_batch` with `page_size=100` optimizes performance
- Parameterized queries (`%s`) prevent SQL injection
- UPSERT pattern allows re-running script safely

**Result**: 479 records successfully inserted

---

#### 6.4 Verification Query

```python
verification_query = """
    SELECT COUNT(*) as total_records,
           ROUND(AVG(sat_math_avg_score), 3) as avg_math_score,
           ROUND(AVG(sat_critical_reading_avg_score), 3) as avg_reading_score,
           ROUND(AVG(sat_writing_avg_score), 3) as avg_writing_score
    FROM nyc_schools.alex_sat_scores;
"""

df_verification = pd.read_sql_query(verification_query, conn)
df_verification = df_verification.round(3)
display(df_verification)
```

**Purpose**: Verify data integrity and calculate aggregate statistics.

---

## Data Quality Issues

### Issues Identified

| Issue | Count | Percentage | Impact |
|-------|-------|------------|--------|
| Duplicate column with typo | 1 | - | Column removed |
| Invalid SAT scores (outside 200-800) | 5 | 1.0% | Values replaced with NaN |
| Missing SAT scores | 53 | 10.8% | Already NaN |
| Exact row duplicates | 14 | 2.8% | Records removed |
| Missing `contact_extension` | 87 | 17.6% | Retained as NULL |
| Missing `pct_students_tested` | 104 | 21.1% | Retained as NULL |
| Missing `academic_tier_rating` | 91 | 18.5% | Retained as NULL |
| Incorrect data types | All | 100% | Converted |

### Solutions Applied

| Problem | Solution | Impact |
|---------|----------|--------|
| Duplicate column | Drop column | Reduced to 10 columns |
| Invalid scores | Replace with NaN | 5 values replaced, all rows retained |
| Row duplicates | `drop_duplicates()` | 14 rows removed |
| Missing values | Retain as NULL | Data integrity preserved |
| Type errors | `pd.to_numeric()` with `errors='coerce'` | Consistent types |
| Non-analytical columns | Drop columns | 8 columns remain |

---

## Database Schema

### Table: `nyc_schools.alex_sat_scores`

| Column | Data Type | Constraints | Description |
|--------|-----------|-------------|-------------|
| `dbn` | TEXT | PRIMARY KEY | Unique school identifier |
| `school_name` | TEXT | | School name |
| `num_of_sat_test_takers` | INTEGER | | Number of SAT test takers |
| `sat_critical_reading_avg_score` | FLOAT | | Average reading score (200-800) |
| `sat_math_avg_score` | FLOAT | | Average math score (200-800) |
| `sat_writing_avg_score` | FLOAT | | Average writing score (200-800) |
| `pct_students_tested` | FLOAT | | Percentage of students tested (0-100) |
| `academic_tier_rating` | INTEGER | | Academic tier rating (1-4) |

### Indexes

- **Primary Key**: `dbn` (automatic index)

### Constraints

- `ON CONFLICT (dbn) DO UPDATE`: Prevents duplicates, updates existing records

---

## Results

### Data Reduction Pipeline

| Phase | Records | Removed | Reason |
|-------|---------|---------|--------|
| Initial Load | 493 | - | Raw data |
| After Column Removal | 493 | 0 | Removed duplicate column |
| After Type Conversion | 493 | 0 | Converted data types |
| After Invalid Value Replacement | 493 | 0 | Invalid values replaced with NaN |
| After Duplicate Removal | 479 | 14 | Exact duplicates |
| **Final Database** | **479** | **14 total** | **Clean dataset** |

### Completeness Rate

| Field | Complete Records | Completeness Rate |
|-------|------------------|-------------------|
| `sat_critical_reading_avg_score` | 435 | 90.8% |
| `sat_math_avg_score` | 430 | 89.7% |
| `sat_writing_avg_score` | 435 | 90.8% |
| `pct_students_tested` | 375 | 78.3% |
| `academic_tier_rating` | 410 | 85.6% |

### Expected Average Scores

The average SAT scores after cleaning should fall in the realistic range of 400-500 per section, reflecting actual NYC high school performance.

---

## Usage

### Prerequisites

```bash
pip install pandas psycopg2-binary jupyter
```

### Step-by-Step Instructions

1. **Prepare CSV file**:
   - Ensure `sat-results.csv` is in your working directory
   - Verify file has correct column headers

2. **Configure database credentials**:
   - Update connection parameters in the notebook
   - Replace `<password>` with your actual password
   - Replace host with your database endpoint

3. **Run notebook**:
   ```bash
   jupyter notebook day4_sat_modeling.ipynb
   ```

4. **Execute cells sequentially**:
   - Run each cell in order
   - Review intermediate results after each step
   - Check for errors or warnings

5. **Verify results**:
   - Run the verification query
   - Check record count and average scores
   - Confirm data integrity in database

---

## Key Insights

### Technical Learnings

1. **Data Validation is Critical**:
   - 5 invalid SAT scores (1.0%) were identified and replaced with NaN
   - 53 schools (10.8%) had missing SAT scores
   - Early validation prevents errors in downstream analysis
   - Range validation (200-800) caught impossible values
   - Retaining rows with partial data provides more comprehensive visibility

2. **Duplicate Detection Requires Care**:
   - Full row duplicates were present (2.9% of total data)
   - `drop_duplicates()` is effective but manual inspection recommended
   - DBN should be unique but wasn't in raw data

3. **Type Conversion is Non-Trivial**:
   - pandas nullable integers (Int64) vs. Python int
   - `pd.NA` must be converted to `None` for psycopg2
   - String percentage values need parsing and conversion

4. **Batch Insertion is Efficient**:
   - `execute_batch()` with `page_size=100` optimizes performance
   - `ON CONFLICT` avoids errors on duplicate keys
   - Parameterized queries prevent SQL injection


**End of Documentation**
