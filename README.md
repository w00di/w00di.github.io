# Notes
## ClickHouse
### S3 Table Function
```sql
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi_0_0_0.snappy.parquet')
LIMIT 100;
```
```sql
DESCRIBE s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi_0_0_0.snappy.parquet');
```
```sql
SELECT
  PROJECT,
  count() AS c
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi_0_0_0.snappy.parquet')
GROUP BY PROJECT
ORDER BY c DESC;
```
```sql
SELECT
  toStartOfMonth(TIMESTAMP),
  PROJECT,
  count()
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi_0_0_0.snappy.parquet')
GROUP BY month, PROJECT;
```
### Merge Tree
```sql
CREATE TABLE uk_price_paid
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);
```
```sql
INSERT INTO uk_price_paid
WITH
   splitByChar(' ', postcode) AS p
SELECT
    toUInt32(price_string) AS price,
    parseDateTimeBestEffortUS(time) AS date,
    p[1] AS postcode1,
    p[2] AS postcode2,
    transform(a, ['T', 'S', 'D', 'F', 'O'], ['terraced', 'semi-detached', 'detached', 'flat', 'other']) AS type,
    b = 'Y' AS is_new,
    transform(c, ['F', 'L', 'U'], ['freehold', 'leasehold', 'unknown']) AS duration,
    addr1,
    addr2,
    street,
    locality,
    town,
    district,
    county
FROM url(
    'http://prod.publicdata.landregistry.gov.uk.s3-website-eu-west-1.amazonaws.com/pp-complete.csv',
    'CSV',
    'uuid_string String,
    price_string String,
    time String,
    postcode String,
    a String,
    b String,
    c String,
    addr1 String,
    addr2 String,
    street String,
    locality String,
    town String,
    district String,
    county String,
    d String,
    e String'
) SETTINGS max_http_get_redirects=10;
```
```sql
select formatReadableQuantity(count()) from uk_price_paid;
```
```sql
select * from uk_price_paid
limit 100;
```
```sql
select
  avg(price),
  town
from uk_price_paid
group by town
order by 1 desc;
```
### avg() & count()
```sql
SELECT *
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet')
LIMIT 100;
```
```sql
SELECT count()
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');
```
```sql
SELECT formatReadableQuantity(count())
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');
```
```sql
SELECT
    avg(volume)
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet')
WHERE
    crypto_name = 'Bitcoin';
```
```sql
SELECT
    crypto_name,
    count() AS count
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet')
GROUP BY crypto_name
ORDER BY crypto_name;
```
```sql
SELECT
    trim(crypto_name) as name,
    count() AS count
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet')
GROUP BY name
ORDER BY name;
```
### Order By
* In a Merge Tree table the ```Order By``` is the Primary Key
* The Primary Key is how the data gets sorted on disk
* Every time an insert takes place the rows from that insert get put into their own folder (part)
* Inserts should be LARGE
* If doing smaller inserts turn on ASYNC INSERT
* ClickHouse MergeTree merges parts into larger parts
* Each 8192 rows or 10 MB of table are separated into ```granules```
* Each ```granule``` is given a primary key based on the the first row of that ```granule```
* The primary.idx consits of all the ```granules``` for a table. This index lives in memory
* ```granule``` is the smallest indivisible data set that ClickHouse reads when searching rows
* Each ```stripe``` of granules that need to be searhed are sent to a threat for processing

```sql
--Step 1:
DESCRIBE s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi/2023/pypi_0_7_34.snappy.parquet');

--Step 2:
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi/2023/pypi_0_7_34.snappy.parquet')
LIMIT 10;

--Step 3:
SELECT count()
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi/2023/pypi_0_7_34.snappy.parquet');

--Step 4:
CREATE TABLE pypi (
    TIMESTAMP DateTime,
    COUNTRY_CODE String,
    URL String,
    PROJECT String
)
ENGINE = MergeTree
PRIMARY KEY TIMESTAMP;

--Step 5:
INSERT INTO pypi
    SELECT TIMESTAMP, COUNTRY_CODE, URL, PROJECT
    FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/pypi/2023/pypi_0_7_34.snappy.parquet');

--Step 6:
SELECT
    PROJECT,
    count() AS c
FROM pypi
GROUP BY PROJECT
ORDER BY c DESC
LIMIT 100;

--Step 7:
/*
 * All of the rows were read, because the query had no WHERE clause - so
 * ClickHouse needed to process every granule.
 */

--Step 8:
SELECT
    PROJECT,
    count() AS c
FROM pypi
WHERE toStartOfMonth(TIMESTAMP) = '2023-04-01'
GROUP BY PROJECT
ORDER BY c DESC
LIMIT 100;

SELECT
    PROJECT,
    count() AS c
FROM pypi
WHERE TIMESTAMP >= toDate('2023-04-01') AND TIMESTAMP < toDate('2023-05-01')
GROUP BY PROJECT
ORDER BY c DESC
LIMIT 100;

--Step 9:
/*
 * Your answer may vary by a granule or two, but the query only has to process
 * 565,248 rows, which is exactly 8,192 x 69. So the query processed 69
 * granules instead of performing a scan of the entire table. Why? Because the
 * primary key is the TIMESTAMP column, which allows ClickHouse to skip about
 * 1/3 of the data.
*/

--Step 10:
SELECT
    PROJECT,
    count() AS c
FROM pypi
WHERE PROJECT LIKE 'boto%'
GROUP BY PROJECT
ORDER BY c DESC;

--Step 11:
/*
 * The PROJECT column is not in the primary key, so the primary index is no
 * help in skipping granules.
 */

--Step 12:
CREATE TABLE pypi2 (
    TIMESTAMP DateTime,
    COUNTRY_CODE String,
    URL String,
    PROJECT String
)
ENGINE = MergeTree
PRIMARY KEY (TIMESTAMP, PROJECT);

INSERT INTO pypi2
    SELECT *
    FROM pypi;

SELECT
    PROJECT,
    count() AS c
FROM pypi2
WHERE PROJECT LIKE 'boto%'
GROUP BY PROJECT
ORDER BY c DESC;

--Step 13:
/*
 * None. Even though PROJECT was added to the primary key, it did not allow
 * ClickHouse to skip any granules. Why? Because the TIMESTAMP has a high
 * cardinality that is making any subsequent columns in the primary key
 * difficult to be useful.
 */


--Step 14:
CREATE OR REPLACE TABLE pypi2 (
    TIMESTAMP DateTime,
    COUNTRY_CODE String,
    URL String,
    PROJECT String
)
ENGINE = MergeTree
PRIMARY KEY (PROJECT, TIMESTAMP);

INSERT INTO pypi2
    SELECT *
    FROM pypi;

SELECT
    PROJECT,
    count() AS c
FROM pypi2
WHERE PROJECT LIKE 'boto%'
GROUP BY PROJECT
ORDER BY c DESC;

--Step 15:
/*
 * The first column of the primary key is an important and powerful design
 * decision. By putting PROJECT first, we are assuring that our queries that
 * filter by PROJECT will process a minimum amount of rows.
 */
```
```sql
--Step 1:
SELECT
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    count() AS num_of_active_parts
FROM system.parts
WHERE (active = 1) AND (table = 'pypi');

--Step 3:
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    count() AS num_of_active_parts
FROM system.parts
WHERE (active = 1) AND (table LIKE '%pypi%')
GROUP BY table;

--Step 4:
CREATE TABLE test_pypi (
    TIMESTAMP DateTime,
    COUNTRY_CODE String,
    URL String,
    PROJECT String
)
ENGINE = MergeTree
PRIMARY KEY (PROJECT, COUNTRY_CODE, TIMESTAMP);

INSERT INTO test_pypi
    SELECT * FROM pypi2;

--Step 5:
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    count() AS num_of_active_parts
FROM system.parts
WHERE (active = 1) AND (table LIKE '%pypi%')
GROUP BY table;
```
### Modeling Data
#### Special Table Engines
* Dictionary: For representing dictionary as a table
* View: For implementing views - it only stores a SELECT query, no data
* MaterializedView: stores the actual data from a corresponding SELECT query
* File: Useful for exporting table data to a file or converting data from one format to another (CSV, TSV, JSON XML, etc.)
* URL: Similar to File, but queries data from a remote HTTP/HTTPS server
* Memory: Stores data only in memory (data is lost on restart), useful for testing
* More: [Table Engines](https://clickhouse.com/docs/en/engines/table-engines)

#### Data Types
* Store floating point numbers as decimals instead
* To define an array: use square brackets [] (or the array() function)
* Nullable allows null to be used for missing values
  * ```metric Nullable(UInt64)```
  * Nullable types can not be a part of the primary key
  * Values that are nullable will be skipped when running mathematical functions
  * Only use it when its business logic and important to be ```null```
* Enums: define enumerations
  * ```device_type Enum('server' = 1, 'container` = 2, 'router' = 3)```
  * Can only contain values in the ```Enum``` defintition
* LowCardinality: Useful when you have a column with a relatively small number of unique values (10,000 or less)
  *  Stores values as integers (uses a dictionary encoding)
  *  You can dynamically add new values

### Primary Keys & Primary Indexes
* Primary Key can be defined inside or outside column list
* ```Order By``` can be used as well
* If both are defined ```PRIMARY KEY``` must be a prefix of the ```ORDER BY```
* Query execution is significantly more effective and faster on a table where the primary key columns are order by cardinality in ascending order
* Only add a column to a primary key if:
  * You have lots of queries on the added column
  * Adding another column allows you to skip quite long data ranges
* Primary Keys have to fit in memory, if not, Clickhouse will not start

```sql
create database mike;

create table mike.friends (
    name String,
    birthday Date,
    age UInt8
)
Engine = MergeTree
PRIMARY KEY name;

ALTER TABLE mike.friends
    ADD COLUMN meetings Array(DateTime);

show create table mike.friends

insert into mike.friends values
    ('Michael','1970-12-02',54,['2024-11-05','1718632394']),
    ('Thomas','1970-12-02',35,[now(),now() - interval 1 week]);

select * from mike.friends;

select meetings[1] from mike.friends;

alter table mike.friends
    add column size UInt64,
    add column metric Nullable(UInt64);

insert into mike.friends (size,metric) values
    (100, 234234),
    (NULL, 3245234),
    (200, NULL);

select * from mike.friends
```
```sql
select uniqExact(COUNTRY)
from pypi;

select uniqExact(PROJECT)
from pypi

CREATE TABLE pypi3 (
    TIMESTAMP DateTime,
    COUNTRY_CODE LowCardinality(String),
    URL String,
    PROJECT LowCardinality(String)
)
ENGINE = MergeTree
PRIMARY KEY (PROJECT, TIMESTAMP);

show create table pypi2;

INSERT INTO pypi3
    SELECT * FROM pypi2;


SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    count() AS num_of_active_parts
FROM system.parts
WHERE (active = 1) AND (table LIKE 'pypi%')
GROUP BY table;

SELECT
    toStartOfMonth(TIMESTAMP) AS month,
    count() AS count
FROM pypi2
WHERE COUNTRY_CODE = 'US'
GROUP BY
    month
ORDER BY
    month ASC,
    count DESC;
```
```sql
DESCRIBE s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');

CREATE TABLE crypto_prices (
    trade_date Date,
    crypto_name LowCardinality(String),
    volume Float32,
    price Float32,
    market_cap Float32,
    change_1_day Float32
) Engine =  MergeTree
PRIMARY KEY (crypto_name, trade_date)

INSERT INTO crypto_prices
SELECT *
FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');

SELECT count()
FROM crypto_prices

SELECT *
FROM crypto_prices
WHERE volume >= 1000_000

SELECT avg(price), crypto_name
FROM crypto_prices
WHERE crypto_name LIKE 'B%'
GROUP BY crypto_name
```




