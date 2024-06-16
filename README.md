# Notes
## ClickHouse
### S3 Table Function
```sql
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi_0_0_0.snappy.parquet')
LIMIT 100;
```
