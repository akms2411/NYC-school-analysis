# NYC Schools SAT Analysis

ETL pipeline for cleaning and analyzing NYC high school SAT results.

## Overview

This project cleans NYC SAT data and loads it into PostgreSQL.

**Key feature**: Replaces invalid values with NaN instead of deleting entire rows.

## The Data

- **Original records**: 493 schools
- **After cleaning**: 479 schools
- **Records removed**: 14 duplicates
- **Invalid values replaced with NaN**: 5 math scores

## Results

Average SAT scores:
- Math: 414
- Reading: 401
- Writing: 394

## Files

- `sat_modeling.ipynb` - Main notebook
- `sat_modeling_documentation.md` - Detailed documentation

## How to Run

```bash
pip install pandas psycopg2-binary jupyter numpy
jupyter notebook sat_modeling.ipynb
```

## Author

Alexander Kuhn - [GitHub](https://github.com/akms2411)

---

**Last Updated**: January 21, 2026
