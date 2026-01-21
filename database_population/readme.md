# 📈 SAT Analysis

## Overview

ETL pipeline for cleaning NYC high school SAT results and loading into PostgreSQL.

## Data

- **Original records**: 493 schools
- **After cleaning**: 479 schools
- **Records removed**: 14 duplicates
- **Invalid values replaced with NaN**: 5 math scores

## Key Features

- **Smart Data Handling**: Replaces invalid values with NaN instead of deleting rows
- **Data Validation**: SAT score range validation (200-800)
- **Duplicate Detection**: Identifies and removes exact duplicates
- **PostgreSQL Integration**: Secure database connection with UPSERT

## Results

Average SAT scores:
- **Math**: 414
- **Reading**: 401
- **Writing**: 394

## Files

- `sat_modeling.ipynb` - Main analysis notebook
- `sat_modeling_documentation.md` - Detailed documentation

## Technologies

- Python (pandas, numpy, psycopg2)
- PostgreSQL (Neon)
- Jupyter Notebook

---

**Part of**: [NYC Schools Analysis](https://github.com/akms2411/NYC-school-analysis)
