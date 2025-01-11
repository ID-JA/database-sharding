# Distributed Database Implementation with Oracle and Docker

> **_NOTE:_** This project is an assignment for my Master's degree program.

you can download the full documentation here [![Google Drive](https://img.shields.io/badge/Google%20Drive-4285F4?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1l22aSvolZ7MQjur8ZywhvzftMnT4FayJ/view?usp=sharing)

## Introduction
This project implements a distributed database system using Oracle Database and Docker. The objective is to leverage sharding techniques for improved scalability and performance.

---

## Environment Setup

### Downloading Oracle Database Image
Retrieve the Oracle Database image from Docker Hub:
```bash
docker pull gvenzl/oracle-free
```

### Creating Docker Network
Set up a shared Docker network for container communication:
```bash
docker network create oracle_network
```

### Creating Containers
Run the following commands to create and start the containers:
```bash
docker run -d --name inventory_eu -e ORACLE_PASSWORD=password -p 1531:1521 -v inventory_eu_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name inventory_na -e ORACLE_PASSWORD=password -p 1532:1521 -v inventory_na_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name product_core -e ORACLE_PASSWORD=password -p 1533:1521 -v product_core_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name product_other -e ORACLE_PASSWORD=password -p 1534:1521 -v product_other_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name central -e ORACLE_PASSWORD=password -p 1535:1521 -v central_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name customer_eu_core -e ORACLE_PASSWORD=password -p 1538:1521 -v customer_eu_core_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe

docker run -d --name customer_eu_other -e ORACLE_PASSWORD=password -p 1539:1521 -v customer_eu_other_data:/opt/oracle/oradata --network app_net gvenzl/oracle-xe
```

### Verifying Containers
Check the running containers:
```bash
docker ps
```

---

## Table Creation and Configuration

### Horizontal Fragmentation
#### Inventory and Orders Tables
Connect to `inventory_eu` container:
```bash
docker exec -it inventory_eu bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE TABLE Inventory (
    inventory_id NUMBER(10) PRIMARY KEY,
    product_id NUMBER(10),
    region VARCHAR(100),
    quantity_on_hand NUMBER(10)
);

CREATE TABLE Orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10, 2),
    status VARCHAR(50)
);

CREATE TABLE OrderItems (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10, 2),
    CONSTRAINT fk_order_item_order_id FOREIGN KEY (order_id) REFERENCES Orders(order_id)
);


CREATE TABLE Shipments (
    shipment_id INT PRIMARY KEY,
    order_id INT,
    from_location INT,
    to_location INT,
    shipment_date DATE,
    region VARCHAR(100),
    status VARCHAR(50),
    CONSTRAINT fk_shipment_order_id FOREIGN KEY (order_id) REFERENCES Orders(order_id)
);

CREATE TABLE Suppliers (
    supplier_id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone_number VARCHAR(255) NOT NULL,
    region VARCHAR2(200)
);
```
Repeat for `inventory_na`:
```bash
docker exec -it inventory_na bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Use the same SQL script as above.

### Vertical Fragmentation
#### Products Table
Connect to `product_core` container:
```bash
docker exec -it product_core bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE TABLE Products_Core_Details (
    product_id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description VARCHAR(4000),
    price DECIMAL(10,2),
    category VARCHAR(100)
);
```
Connect to `product_other` container:
```bash
docker exec -it product_other bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE TABLE Product_Other_Details (
    product_id INT PRIMARY KEY,
    weight DECIMAL(10,2),
    color VARCHAR(100),
    "size" VARCHAR(100),
    warranty VARCHAR(255),
    manufacturer VARCHAR(255),
    supplier_id INT
);
```

### Mixed Fragmentation
#### Customers Table
Connect to `customers_eu_core` container:
```bash
docker exec -it customers_eu_core bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE TABLE Customer_Core_Details (
    customer_id INT PRIMARY KEY,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    phone_number VARCHAR(50),
);
```
Connect to `customers_eu_other` container:
```bash
docker exec -it customers_eu_other bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE TABLE Customer_Other_Details (
    customer_id INT PRIMARY KEY,
    billing_address_id INT,
    shipping_address_id INT,
    registration_date DATE,
    region VARCHAR(100)
);
```
---

## Database Connectivity

### Creating DBLinks
Connect to the central container and create DBLinks:
```bash
docker exec -it central bash
sqlplus SYSTEM/password@//localhost:1521/XE
```
Execute the SQL script:
```sql
CREATE DATABASE LINK inventory_eu_link CONNECT TO SYSTEM IDENTIFIED BY password USING 'inventory_eu:1521/XE';

CREATE DATABASE LINK inventory_na_link CONNECT TO system IDENTIFIED BY password USING 'inventory_na:1521/XE';

CREATE DATABASE LINK product_core_link CONNECT TO system IDENTIFIED BY password USING 'product_core:1521/XE';

CREATE DATABASE LINK product_other_link CONNECT TO system IDENTIFIED BY password USING 'product_other:1521/XE';

CREATE DATABASE LINK customer_eu_core_link CONNECT TO system IDENTIFIED BY password USING 'customer_eu_core:1521/XE';

CREATE DATABASE LINK customer_eu_other_link CONNECT TO system IDENTIFIED BY password USING 'customer_eu_other:1521/XE';
```

---

## Materialized Views and Jobs

### Creating Materialized Views
```sql
CREATE MATERIALIZED VIEW inventory_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT * FROM INVENTORY@inventory_na_link WHERE 1 = 0;

CREATE MATERIALIZED VIEW order_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT * FROM ORDERS@inventory_na_link WHERE 1 = 0;

CREATE MATERIALIZED VIEW order_items_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT * FROM OrderItems@inventory_na_link;

CREATE MATERIALIZED VIEW shipment_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT * FROM SHIPMENTS@inventory_eu_link;

CREATE MATERIALIZED VIEW supplier_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT * FROM SUPPLIER@inventory_eu_link;


CREATE MATERIALIZED VIEW customer_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
SELECT 
  cp_eu.CUSTOMER_ID,
  cp_eu.first_name,
  cp_eu.last_name,
  cp_eu.email,
  cp_eu.phone_number,
  ca_eu.billing_address,
  ca_eu.shipping_address,
  cr_eu.registration_date,
  cr_eu.region
 FROM CUSTOMER_PERSONAL@customer_eu_core_link cp_eu JOIN CUSTOMER_ADDRESS@customer_eu_other_link ca_eu
  ON cp_eu.CUSTOMER_ID = ca_eu.CUSTOMER_ID 
  JOIN CUSTOMER_REGISTRATION@customer_eu_other_link cr_eu
  ON cp_eu.CUSTOMER_ID = cr_eu.CUSTOMER_ID;


CREATE MATERIALIZED VIEW product_mv
BUILD IMMEDIATE 
REFRESH FORCE
ON DEMAND
AS
  SELECT
      pc.product_id,
      pc.name,
      pc."DESCRIPTION",
      pc.price,
      pc.category,
      po.weight,
      po.color,
      po."size",
      po.warranty,
      po.manufacturer,
      po.supplier_id
  FROM Products_Core_Detail@product_core_link pc
  JOIN product_other_details@product_other_link po 
    ON pc.product_id = po.product_id
```

### Refresh Procedure
```sql
CREATE OR REPLACE PROCEDURE import_data AS
BEGIN
    DBMS_MVIEW.REFRESH('customer_mv');
    DBMS_MVIEW.REFRESH('inventory_mv');
    DBMS_MVIEW.REFRESH('order_items_mv');
    DBMS_MVIEW.REFRESH('order_mv');
    DBMS_MVIEW.REFRESH('shipment_mv');
    DBMS_MVIEW.REFRESH('supplier_mv');
    DBMS_MVIEW.REFRESH('product_mv');
END;
/
```

### Scheduling Jobs
```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'IMPORT_DATA_JOB',
    job_type        => 'STORED_PROCEDURE',
    job_action      => 'IMPORT_DATA',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY;BYHOUR=23;BYMINUTE=59;BYSECOND=59',
    enabled         => TRUE,
    comments        => 'import data from shards at the end of the day'
  );
END;
/

```

---

## Integrity Checks

### Verification Procedure
```sql
CREATE OR REPLACE PROCEDURE VERIFY_MATERIALIZED_VIEW_CUSTOMER_SYN IS
  v_mv_count INTEGER := 0;
  v_source_count INTEGER := 0;
  v_mv_last_refresh TIMESTAMP;
  v_error_message VARCHAR2(1000);
BEGIN
  SELECT last_refresh_date INTO v_mv_last_refresh
  FROM user_mviews
  WHERE mview_name = 'CUSTOMER_MV';

  BEGIN
    SELECT COUNT(*) INTO v_mv_count
    FROM customer_mv;

    SELECT COUNT(*) INTO v_source_count
    FROM customer_personal@customer_eu_core_link cp_eu
    JOIN customer_address@customer_eu_other_link ca_eu
      ON cp_eu.customer_id = ca_eu.customer_id
    JOIN customer_registration@customer_eu_other_link cr_eu
      ON cp_eu.customer_id = cr_eu.customer_id;

    IF v_mv_count != v_source_count THEN
      v_error_message := 'Mismatch in row count: Materialized view has ' || v_mv_count || ' rows, but source has ' || v_source_count || ' rows.';
      DBMS_OUTPUT.PUT_LINE(v_error_message);
    ELSE
      DBMS_OUTPUT.PUT_LINE('Row count is consistent between materialized view and source.');
    END IF;
  EXCEPTION
    WHEN OTHERS THEN
      DBMS_OUTPUT.PUT_LINE('Error comparing row counts: ' || SQLERRM);
  END;

  IF v_mv_last_refresh < SYSDATE - INTERVAL '1' DAY THEN
    v_error_message := 'Materialized view is not up-to-date. Last refresh was at ' || TO_CHAR(v_mv_last_refresh, 'YYYY-MM-DD HH24:MI:SS');
    DBMS_OUTPUT.PUT_LINE(v_error_message);
  ELSE
    DBMS_OUTPUT.PUT_LINE('Materialized view is up-to-date.');
  END IF;

  IF v_mv_count = v_source_count AND v_mv_last_refresh >= SYSDATE - INTERVAL '1' DAY THEN
    DBMS_OUTPUT.PUT_LINE('Materialized view is in sync with the source data.');
  ELSE
    DBMS_OUTPUT.PUT_LINE('Data sync issues found.');
  END IF;
  
END VERIFY_MATERIALIZED_VIEW_CUSTOMER_SYN;
/

```

### Scheduling Integrity Checks
```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'VERIFY_MV_SYNC_JOB',
    job_type        => 'STORED_PROCEDURE',
    job_action      => 'VERIFY_MATERIALIZED_VIEW_CUSTOMER_SYN', 
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY;BYHOUR=0;BYMINUTE=0;BYSECOND=0',
    enabled         => TRUE,
    comments        => 'Verifies if the materialized view is in sync with the source data'
  );
END;
/
```

---

## Contributors ✨

<table>
  <tbody>
    <tr>
      <td align="center"><a href="https://jamalidaissa.vercel.app"><img src="https://avatars.githubusercontent.com/u/69154853?v=4" width="100px;" alt="Jamal Id Aissa"/><br /><sub><b>Jamal Id Aissa</b></sub></a></td>
    </tr>
  </tbody>
</table>


