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
```sql
--Step 1:
DESCRIBE pypi;

--Step 2:
SELECT uniqExact(COUNTRY_CODE)
FROM pypi;

/*
 * You will notice there are only 186 unique values of the country code, which
 * makes it a great candidate for LowCardinality.
 */

--Step 3:
SELECT
    uniqExact(PROJECT),
    uniqExact(URL)
FROM pypi;

/*
 * There are over 24,000 unique values of PROJECT, which is large - but not too
 * large. We will try LowCardinality on this column as well and see if it
 * improves storage and query performance. The URL has over 79,000 unique
 * values, and we can assume that a URL could have a lot of different values,
 * so it is probably a bad choice for LowCardinality.
 */

--Step 4:
CREATE TABLE pypi3 (
    TIMESTAMP DateTime,
    COUNTRY_CODE LowCardinality(String),
    URL String,
    PROJECT LowCardinality(String)
)
ENGINE = MergeTree
PRIMARY KEY (PROJECT, TIMESTAMP);

INSERT INTO pypi3
    SELECT * FROM pypi2;

--Step 5:
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    count() AS num_of_active_parts
FROM system.parts
WHERE (active = 1) AND (table LIKE 'pypi%')
GROUP BY table;

--Step 6:
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
--Step 1:
DESCRIBE s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');

--Step 2:
CREATE TABLE crypto_prices (
   trade_date Date,
   crypto_name LowCardinality(String),
   volume Float32,
   price Float32,
   market_cap Float32,
   change_1_day Float32
)
ENGINE = MergeTree
PRIMARY KEY (crypto_name, trade_date);

--Step 3:
INSERT INTO crypto_prices
   SELECT *
   FROM s3('https://learnclickhouse.s3.us-east-2.amazonaws.com/datasets/crypto_prices.parquet');

--Step 4:
SELECT count()
FROM crypto_prices;

--Step 5:
SELECT count()
FROM crypto_prices
WHERE volume >= 1_000_000;

/*
 * It read all of the rows because volume is not part of the primary key.
 */

--Step 6:
SELECT
   avg(price)
FROM crypto_prices
WHERE crypto_name = 'Bitcoin';

/*
 * Only a single granule was processed. As crypto_name is a primary key,
 * ClickHouse use it to optmize the query.
 */

--Step 7:
SELECT
   avg(price)
FROM crypto_prices
WHERE crypto_name LIKE 'B%';
```
### Insert Data
```sql
DESCRIBE s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz','TabSeparatedWithNames')
SETTINGS schema_inference_make_columns_nullable=false;


SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz','TabSeparatedWithNames')
LIMIT 100;

CREATE TABLE taxi (
    trip_id	Int64,
    vendor_id	Int64,
    pickup_date	Date,
    pickup_datetime	DateTime64(9),
    dropoff_date	Date,
    dropoff_datetime	DateTime64(9),
    store_and_fwd_flag	Int64,
    rate_code_id	Int64,
    pickup_longitude	Float64,
    pickup_latitude	Float64,
    dropoff_longitude	Float64,
    dropoff_latitude	Float64,
    passenger_count	Int64,
    trip_distance	String,
    fare_amount	String,
    extra	String,
    mta_tax	String,
    tip_amount	String,
    tolls_amount	Float64,
    ehail_fee	Int64,
    improvement_surcharge	String,
    total_amount	String,
    payment_type	String,
    trip_type	Int64,
    pickup	String,
    dropoff	String,
    cab_type	String,
    pickup_nyct2010_gid	Int64,
    pickup_ctlabel	Float64,
    pickup_borocode	Int64,
    pickup_ct2010	String,
    pickup_boroct2010	String,
    pickup_cdeligibil	String,
    pickup_ntacode	String,
    pickup_ntaname	String,
    pickup_puma	Int64,
    dropoff_nyct2010_gid	Int64,
    dropoff_ctlabel	Float64,
    dropoff_borocode	Int64,
    dropoff_ct2010	String,
    dropoff_boroct2010	String,
    dropoff_cdeligibil	String,
    dropoff_ntacode	String,
    dropoff_ntaname	String,
    dropoff_puma	Int64
) ENGINE = MergeTree
PRIMARY KEY (vendor_id, pickup_date);

INSERT INTO taxi
    SELECT *
    FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/nyc-taxi/trips_{0..2}.gz','TabSeparatedWithNames');

select formatReadableQuantity(count())
from taxi;

select *
from taxi
limit 100

select any(pickup_ntaname), count()
from taxi
group by pickup_ctlabel
order by 2 desc
limit 20

select
    min(pickup_date),
    max(pickup_date),
from taxi
```

```sql
--Step 1:
DESCRIBE s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/uk_property_prices.snappy.parquet')
SETTINGS
   schema_inference_make_columns_nullable=false;

--Step 2:
CREATE TABLE uk_price_paid
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, date);

--Step 3:
INSERT INTO uk_price_paid
    SELECT *
    FROM url('https://learn-clickhouse.s3.us-east-2.amazonaws.com/uk_property_prices.snappy.parquet');

--Step 4:
SELECT count()
FROM uk_price_paid;

--Step 5:
SELECT avg(price)
FROM uk_price_paid
WHERE postcode1 = 'LU1' AND postcode2 = '5FT';

/*
 * The primary key contains postcode1 and postcode2 as the first two columns,
 * so filtering by both allows ClickHouse to skip the most granules.
 */

--Step 6:
SELECT avg(price)
FROM uk_price_paid
WHERE postcode2 = '5FT';

/*
 * The postcode2 column is the second column in the primary key, which allows
 * ClickHouse to avoid about 1/3 of the table. Not bad, but note that the
 * second value of a primary key is not as helpful in our dataset as the first
 * column of the primary key. This all depends on your dataset, but this query
 * gives you an idea of how you should think through and test if a column will
 * be useful before adding it to the primary key. In this example, postcode2
 * seems beneficial (assuming we need to filter by postcode2 regularly.)
 */

--Step 7:
SELECT avg(price)
FROM uk_price_paid
WHERE town = 'YORK';

/*
 * The town column is not a part of the primary key, so the primary index does
 * not provide any skipping of granules.
 */
```
#### Modify Data During Insert
```sql
--Step 1:
SELECT count()
FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS format_csv_delimiter = '~';

--Step 2:
SELECT formatReadableQuantity(sum(actual_amount))
FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS format_csv_delimiter = '~';

--Step 3:
SELECT formatReadableQuantity(sum(approved_amount))
FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS format_csv_delimiter = '~';

/*
 * You get an exception telling you that trying to sum a String column is not
 * allowed. Apparently, the approved_amount column is not entirely numeric
 * data, and ClickHouse inferred that column as a String.
 */

--Step 4:
DESCRIBE s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS format_csv_delimiter = '~';

--Step 5:
SELECT
    formatReadableQuantity(sum(toUInt32OrZero(approved_amount))),
    formatReadableQuantity(sum(toUInt32OrZero(recommended_amount)))
FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS format_csv_delimiter = '~';

--Step 6:
SELECT
    formatReadableQuantity(sum(approved_amount)),
    formatReadableQuantity(sum(recommended_amount))
FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv')
SETTINGS
format_csv_delimiter='~',
schema_inference_hints='approved_amount UInt32, recommended_amount UInt32';

--Step 7:
CREATE TABLE operating_budget (
    fiscal_year LowCardinality(String),
    service LowCardinality(String),
    department LowCardinality(String),
    program LowCardinality(String),
    program_code LowCardinality(String),
    description String,
    item_category LowCardinality(String),
    approved_amount UInt32,
    recommended_amount UInt32,
    actual_amount Decimal(12,2),
    fund LowCardinality(String),
    fund_type Enum8('GENERAL FUNDS' = 1, 'FEDERAL FUNDS' = 2, 'OTHER FUNDS' = 3)
)
ENGINE = MergeTree
PRIMARY KEY (fiscal_year, program);

--Step 8:
INSERT INTO operating_budget
    WITH
        splitByChar('(', c4) AS result
    SELECT
        c1 AS fiscal_year,
        c2 AS service,
        c3 AS department,
        result[1] AS program,
        splitByChar(')',result[2])[1] AS program_code,
        c5 AS description,
        c6 AS item_category,
        toUInt32OrZero(c7) AS approved_amount,
        toUInt32OrZero(c8) AS recommended_amount,
        toDecimal64(c9, 2) AS actual_amount,
        c10 AS fund,
        c11 AS fund_type
    FROM s3(
        'https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv',
        'CSV',
        'c1 String,
        c2 String,
        c3 String,
        c4 String,
        c5 String,
        c6 String,
        c7 String,
        c8 String,
        c9 String,
        c10 String,
        c11 String'
        )
    SETTINGS
        format_csv_delimiter = '~',
        input_format_csv_skip_first_lines=1;

--Step 9:
SELECT * FROM operating_budget;

--Step 10:
SELECT formatReadableQuantity(sum(approved_amount))
FROM operating_budget
WHERE fiscal_year = '2022';

--Step 11:
SELECT sum(actual_amount)
FROM operating_budget
WHERE fiscal_year = '2022'
AND program_code = '031';
```
### Analyzing Data
```sql
SELECT
    town
    count() as c
FROM uk_price_paid
GROUP BY town
LIMIT 20
FORMAT Vertical

with
    'LONDON' as my_town
select
    avg(price)
from uk_price_paid
where town = my_town

with most_expensive AS (
        select * from uk_price_paid
        order by price desc
        limit 10
)
select
    avg(price)
from most_expensive

SELECT
    any(town),
    district,
    count() as c
FROM uk_price_paid
GROUP BY district
Order by c desc
LIMIT 20

SELECT
    avg(price) OVER (PARTITION BY postcode1),
    *
FROM uk_price_paid
WHERE type='terraced'
AND postcode1 != ''
LIMIT 100

SELECT DISTINCT lower(town)
FROM uk_price_paid
LIMIT 10

SELECT sum(price)
FROM uk_price_paid

select
    count()
from uk_price_paid
where position(street, 'KING') > 0;

select
    count()
from uk_price_paid
where multiFuzzyMatchAny(street, 1, ['KING']);

select distinct
    street,
    multiSearchAllPositionsCaseInsensitive(
        street,
        ['abbey','road']
    ) AS positions
FROM uk_price_paid
WHERE NOT has(positions, 0);

SELECT
    max(price),
    toStartOfDay(date) AS day
FROM uk_price_paid
GROUP BY day
ORDER BY day desc;

select now() as today;

With now() as today
select today - INTERVAL 1 HOUR

SELECT
    town,
    max(price),
    argMax(street,price)
FROM uk_price_paid
GROUP BY town

CREATE FUNCTION mergePostcode AS (p1,p2) -> concat(p1,p2)

select mergePostcode(postcode1, postcode2)
from uk_price_paid;

SELECT quantiles(0.90)(price) from uk_price_paid
WHERE toYear(date) >= '2020';

SELECT uniq(street) FROM uk_price_paid;

SELECT uniqExact(street) FROM uk_price_paid;

SELECT topK(10)(street)
FROM uk_price_paid;

SELECT topKIf(10)(street, street != '')
FROM uk_price_paid;

SELECT arrayJoin(splitByChar(' ', street)) FROM uk_price_paid LIMIT 1000
```
```sql
--Step 1:
SELECT *
FROM uk_price_paid
WHERE price >= 100_000_000
ORDER BY price desc;

--Step 2:
SELECT count()
FROM uk_price_paid
WHERE
    price > 1_000_000
    AND date >= toDate('2022-01-01') AND date <= toDate('2022-12-31');

--Step 3:
SELECT uniqExact(town)
FROM uk_price_paid;

--Step 4:
SELECT
    town,
    count() AS c
FROM uk_price_paid
GROUP BY town
ORDER BY c DESC
LIMIT 1;

--Step 5:
SELECT topKIf(10)(town, town != 'LONDON')
FROM uk_price_paid;

--Step 6:
SELECT
    town,
    avg(price) AS avg_price
FROM uk_price_paid
GROUP BY town
ORDER BY avg_price DESC
LIMIT 10;

--Step 7:
SELECT
    addr1,
    addr2,
    street,
    town
FROM uk_price_paid
ORDER BY price DESC
LIMIT 1;

--Step 8:
SELECT
    avgIf(price, type = 'detached'),
    avgIf(price, type = 'semi-detached'),
    avgIf(price, type = 'terraced'),
    avgIf(price, type = 'flat'),
    avgIf(price, type = 'other')
FROM uk_price_paid;

SELECT type, avg(price) as avg_price
FROM uk_price_paid
GROUP BY type;

--Step 9:
SELECT
    formatReadableQuantity(sum(price))
FROM uk_price_paid
WHERE
    county IN ['AVON','ESSEX','DEVON','KENT','CORNWALL']
    AND
    date >= toDate('2020-01-01') AND date <= toDate('2020-12-31');


--Step 10:
SELECT
    toStartOfMonth(date) AS month,
    avg(price) AS avg_price
FROM uk_price_paid
WHERE
    date >= toDate('2005-01-01') AND date <= toDate('2010-12-31')
GROUP BY month
ORDER BY month ASC;

--Step 11:
SELECT
    toStartOfDay(date) AS day,
    count()
FROM uk_price_paid
WHERE
    town = 'LIVERPOOL'
    AND date >= toDate('2020-01-01') AND date <= toDate('2020-12-31')
GROUP BY day
ORDER BY day ASC;

--Step 12:
WITH (
    SELECT max(price)
    FROM uk_price_paid
) AS overall_max
SELECT
    town,
    max(price) / overall_max
FROM uk_price_paid
GROUP BY town
ORDER BY 2 DESC;
```
### Materialized Views
#### Views
- Not efficient
- Works like a subquery
Example
```sql
SELECT count() FROM (
    SELECT * FROM uk_price_paid
    WHERE type = 'terraced'
)
```
### Materialized Views
- Insert Trigger
- Whatever the FROM clause is is the trigger
- Don't use populate if you're actively inserting into a table
- Only happens on insert (not delete or update)

1. Define the destination table
```sql
CREATE TABLE uk_prices_by_town_dest (
    price UInt32,
    date Date,
    street LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String)
)
ENGINE = MergeTree
ORDER BY town;
```
2. Define the materialized view
```sql
CREATE MATERIALIZED VIEW uk_prices_by_town_view
TO uk_prices_by_town_dest
AS
    SELECT
        price,
        date,
        street,
        town,
        district
    FROM uk_price_paid
    WHERE date >= toDate('2024-02-19 12:30:00');
```
3. Populate the destination table
```sql
INSERT INTO uk_prices_by_town_dest
    SELECT
        price,
        date,
        street,
        town,
        district,
    FROM uk_price_paid
    WHERE date < toDate('2024-02-19 12:30:00');
```
