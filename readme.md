# TRAINING TIDB SQL TUNING - LABS HANDS ON
================================================================================================

# TiDB SQL Tuning Lab 1: Clustered and Non-Clustered Indexes
## Module 1: Best Practices For Clustered Indexes
In this module, you will create clustered indexes for primary key tables and configure related settings to prevent data insertion hotspots.

## Tasks
- Created a table named orders1 by a given DDL. The table will have a high concurrency of insertions by the application design.
- Identify any potential performance issues that may arise with table orders1.
- Recreate the orders1 table to ensure it includes a clustered index for the primary key.
- Perform multiple data insertions into the orders1 table and analyze the resulting data distribution.
## Step 1. Log in to the remote Linux
## Step 2. Connect to database test
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
> Database changed
> ```

### Step 3. Observe the table definition of orders1, point out the potential performance problem
- The table definition is shown below, click the Explain with AI button to get more information.
```
CREATE TABLE orders1 (
 id INT UNSIGNED AUTO_INCREMENT NOT NULL,
 code VARCHAR(30) NOT NULL,
 order_no VARCHAR(200) NOT NULL DEFAULT '', 
 status INT NOT NULL,
 cancel_flag INT DEFAULT NULL,
 create_user VARCHAR(50) DEFAULT NULL,
 update_user VARCHAR(50) DEFAULT NULL,
 create_time DATETIME DEFAULT NULL, 
 update_time DATETIME DEFAULT NULL,
 PRIMARY KEY (id) 
);
```
- The table orders1 utilizes AUTO_INCREMENT for its primary key. It is advisable to replace AUTO_INCREMENT with AUTO_RANDOM to prevent hotspot issues that may arise from sequential data insertion.

- Considering the expected large volume of data insertion, it is recommended to preemptively partition the table by splitting the TiKV regions during the initial table creation.

### Step 4. Recreate table orders1 as per the best practice
- Create the orders1 table, since we need to use AUTO_RANDOM instead of AUTO_INCREMENT, the id column needs to be changed from INT type to BIGINT type.

```
DROP TABLE orders1;
CREATE TABLE orders1 
( id BIGINT unsigned AUTO_RANDOM NOT NULL,
  code VARCHAR(30) NOT NULL,
  order_no VARCHAR(200) NOT NULL DEFAULT '', 
  status INT NOT NULL,
  cancel_flag INT DEFAULT NULL,
  create_user VARCHAR(50) DEFAULT NULL,
  update_user VARCHAR(50) DEFAULT NULL,
  create_time DATETIME DEFAULT NULL, 
  update_time DATETIME DEFAULT NULL,
  PRIMARY KEY (id) CLUSTERED
);
SHOW CREATE TABLE orders1\G
```

> **sample output**
> ```
> *************************** 1. row ***************************
>Table: orders1
>Create Table: CREATE TABLE `orders1` (
>  `id` BIGINT unsigned NOT NULL /*T![auto_rand] AUTO_RANDOM(5) */,
>  `code` VARCHAR(30) NOT NULL,
>  `order_no` VARCHAR(200) NOT NULL DEFAULT '',
>  `status` INT NOT NULL,
>  `cancel_flag` INT(4) DEFAULT NULL,
>  `create_user` VARCHAR(50) DEFAULT NULL,
>  `update_user` VARCHAR(50) DEFAULT NULL,
>  `create_time` DATETIME DEFAULT NULL,
>  `update_time` DATETIME DEFAULT NULL,
>  PRIMARY KEY (`id`) /*T![clustered_index] CLUSTERED */
>) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
>1 row in set (0.01 sec)
> ```

- Check the Region distribution of the orders1 table.

```
SHOW TABLE orders1 REGIONS\G
```
> **sample output**
> ```
> *************************** 1. row ***************************
>              REGION_ID: 34
>              START_KEY: t_94_
>                END_KEY: t_281474976710649_
>              LEADER_ID: 139
>        LEADER_STORE_ID: 2
>                  PEERS: 35, 139, 186
>             SCATTERING: 0
>          WRITTEN_BYTES: 273
>             READ_BYTES: 0
>   APPROXIMATE_SIZE(MB): 1
>       APPROXIMATE_KEYS: 0
> SCHEDULING_CONSTRAINTS: 
>       SCHEDULING_STATE: 
> 1 row in set (0.01 sec)
> ```

- Split orders1 table into multiple TiKV regions.

```
SPLIT TABLE orders1 BETWEEN(0) AND (9223372036854775807) REGIONS 16;
SHOW TABLE orders1 REGIONS\G
```

> **sample output**
> ```
> +--------------------+----------------------+
>| TOTAL_SPLIT_REGION | SCATTER_FINISH_RATIO |
>+--------------------+----------------------+
>|                 15 |                    1 |
>+--------------------+----------------------+
>1 row in set (0.48 sec)
>*************************** 1. row ***************************
>             REGION_ID: 244
>             START_KEY: t_94_
>               END_KEY: t_94_r_576460752303423487
>             LEADER_ID: 246
>       LEADER_STORE_ID: 2
>                 PEERS: 245, 246, 247
>            SCATTERING: 0
>         WRITTEN_BYTES: 39
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>*************************** 2. row ***************************
>             REGION_ID: 248
>             START_KEY: t_94_r_576460752303423487
>               END_KEY: t_94_r_1152921504606846974
>             LEADER_ID: 251
>       LEADER_STORE_ID: 5
>                 PEERS: 249, 250, 251
>            SCATTERING: 0
>         WRITTEN_BYTES: 39
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>... ...
>*************************** 16. row ***************************
>             REGION_ID: 34
>             START_KEY: t_94_r_8646911284551352305
>               END_KEY: t_281474976710649_
>             LEADER_ID: 139
>       LEADER_STORE_ID: 2
>                 PEERS: 35, 139, 186
>            SCATTERING: 0
>         WRITTEN_BYTES: 0
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>16 rows in set (0.08 sec)
> ```

### Step 5. Insert data into the orders1 table five times

```
BEGIN;
INSERT INTO orders1 (code, status) VALUES (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1 (code, status) VALUES (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1 (code, status) VALUES (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1 (code, status) VALUES (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1 (code, status) VALUES (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
COMMIT;
```

> **sample output**
> ```
> Query OK, 0 rows affected (0.00 sec)
> ```

```
SELECT id, code, status FROM orders1 ORDER BY id;
```
> **sample output**
> ```
> +---------------------+----------------------+--------+
>| id                  | code                 | status |
>+---------------------+----------------------+--------+
>| 1152921504606846977 | b1f55f9268fb31cec7f5 |      0 |
>| 1152921504606846978 | c3fd54abaab48504539a |      3 |
>| 1152921504606846979 | 92c1b99e02e4e646b0b5 |      0 |
>| 1152921504606846980 | 67af6b9ceb8e758c5794 |      9 |
>| 1152921504606846981 | ed834b9ef841a67d8edb |      8 |
>+---------------------+----------------------+--------+
>5 rows in set (0.01 sec)
> ```

- Although the AUTO_RANDOM attribute has been specified for the primary key in the orders1 table, the generated primary key values are still continuous.

### Step 6. TRUNCATE the data in the orders1 table and insert new data
- Note that, after TRUNCATE, there will only be 1 region left in the orders1 table, so it needs to be split manually.
```
TRUNCATE TABLE orders1;
SPLIT TABLE orders1 BETWEEN(0) AND (9223372036854775807) REGIONS 16;
```
> **sample output**
> ```
> +--------------------+----------------------+
>| TOTAL_SPLIT_REGION | SCATTER_FINISH_RATIO |
>+--------------------+----------------------+
>|                 15 |                    1 |
>+--------------------+----------------------+
>1 row in set (0.06 sec)
> ```
```
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), \
(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), \
(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), (SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
SELECT id, code, status FROM orders1 ORDER BY id;
```

> **sample output**
> ```
> +---------------------+----------------------+--------+
>| id                  | code                 | status |
>+---------------------+----------------------+--------+
>| 5188146770730811393 | 4341550b1717c9e42c33 |      6 |
>| 5188146770730811394 | 8d5687b2a3fa0cbcc039 |      3 |
>| 5188146770730811395 | 87424ab86c4189f14488 |      3 |
>| 5188146770730811396 | 3952300c307266d9da4e |      2 |
>| 5188146770730811397 | 901e4f0fb4e0bcfd98bc |      9 |
>+---------------------+----------------------+--------+
>5 rows in set (0.01 sec)
> ```
- The conclusion is that as long as the insertion statements are within the same transaction, regardless of whether AUTO_RANDOM is specified or not, the primary key values will be continuous and there is still a possibility of hotspots.
### Step 7. Insert the data in different transactions
```
TRUNCATE TABLE orders1;
SPLIT TABLE orders1 BETWEEN(0) AND (9223372036854775807) REGIONS 16;
```
> **sample output**
> ```
> +--------------------+----------------------+
>| TOTAL_SPLIT_REGION | SCATTER_FINISH_RATIO |
>+--------------------+----------------------+
>|                 15 |                    1 |
>+--------------------+----------------------+
>1 row in set (0.06 sec)
> ```
```
SET autocommit=ON;
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders1(code, status) VALUES(SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
SELECT id, code, status FROM orders1 ORDER BY id;
```
> **sample output**
> ```
>+----------------------+----------------------+--------+
>| id                   | code                 | status |
>+----------------------+----------------------+--------+
>|  6341068275337658369 | a96328e33dd6b7ad2f15 |      8 |
>| 13835058055282163715 | 586ab8cf9fb194b5e4ca |      2 |
>| 14411518807585587205 | 21105132b2384ccab63b |      7 |
>|  8646911284551352324 | a80b9c0196c4e9256fd3 |      6 |
>|  7493989779944505346 | 3f43c82a9308a1467b85 |      2 |
>+----------------------+----------------------+--------+
>5 rows in set (0.02 sec)
> ```
- When different transactions perform insertion, we found that the values of the primary key column are random.

### Step 8. Clean up table orders1
```
DROP TABLE orders1;
```
> **sample output**
> ```
> Query OK, 0 rows affected (0.52 sec)
> ```
## Module 2: Best Practices For Non-Clustered Indexes
- In this module, you will create non-clustered indexes for primary key tables and perform related settings to avoid hotspots.
### Step 1. Observe the table definition of orders2, point out the potential performance problem
```
CREATE TABLE orders2 
( id INT(20) NOT NULL ,
  code VARCHAR(30) NOT NULL,
  order_no VARCHAR(200) NOT NULL DEFAULT '', 
  status INT(4) NOT NULL,
  cancel_flag INT(4) DEFAULT NULL,
  create_user VARCHAR(50) DEFAULT NULL,
  update_user VARCHAR(50) DEFAULT NULL,
  create_time DATETIME DEFAULT NULL, 
  update_time DATETIME DEFAULT NULL
);
```

- This table has a non-clustered index for primary key, it will cause a write hotspot problem. To avoid this, you can use SHARD_ROW_ID_BITS and PRE_SPLIT_REGIONS.

### Step 2. Rebuild the orders2 table
```
DROP TABLE orders2;
CREATE TABLE orders2
( id INT(20) NOT NULL ,
  code VARCHAR(30) NOT NULL,
  order_no VARCHAR(200) NOT NULL DEFAULT '', 
  status INT(4) NOT NULL,
  cancel_flag INT(4) DEFAULT NULL,
  create_user VARCHAR(50) DEFAULT NULL,
  update_user VARCHAR(50) DEFAULT NULL,
  create_time DATETIME DEFAULT NULL, 
  update_time DATETIME DEFAULT NULL
) SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=3;
SHOW CREATE TABLE orders2\G
```
> **sample output**
> ```
> *************************** 1. row ***************************
>      Table: orders2
>Create Table: CREATE TABLE `orders2` (
>   `id` INT(20) NOT NULL,
>   `code` VARCHAR(30) NOT NULL,
>   `order_no` VARCHAR(200) NOT NULL DEFAULT '',
>   `status` INT(4) NOT NULL,
>   `cancel_flag` INT(4) DEFAULT NULL,
>   `create_user` VARCHAR(50) DEFAULT NULL,
>   `update_user` VARCHAR(50) DEFAULT NULL,
>   `create_time` DATETIME DEFAULT NULL,
>   `update_time` DATETIME DEFAULT NULL
>) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin /*T! SHARD_ROW_ID_BITS=4 PRE_SPLIT_REGIONS=3 */
>1 row in set (0.00 sec)
> ```

### Step 3. Confirm orders2 table has a non-clustered index
```
SELECT TIDB_PK_TYPE FROM information_schema.tables WHERE table_schema = 'test' AND table_name = 'orders2';
```
> **sample output**
> ```
> +--------------+
>| TIDB_PK_TYPE |
>+--------------+
>| NONCLUSTERED |
>+--------------+
>1 row in set (0.02 sec)
> ```

### Step 4. Check the distribution of regions of the orders2 table
- Since PRE_SPLIT_REGIONS is 3, the orders2 table has 8 empty regions available for insertion at the time of creation.
```
SHOW TABLE orders2 REGIONS\G
```

> **sample output**
> ```
> *************************** 1. row ***************************
>             REGION_ID: 748
>             START_KEY: t_106_
>               END_KEY: t_106_r_1152921504606846976
>             LEADER_ID: 750
>       LEADER_STORE_ID: 2
>                 PEERS: 749, 750, 751
>            SCATTERING: 0
>         WRITTEN_BYTES: 39
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>*************************** 2. row ***************************
>             REGION_ID: 752
>             START_KEY: t_106_r_1152921504606846976
>               END_KEY: t_106_r_2305843009213693952
>             LEADER_ID: 754
>       LEADER_STORE_ID: 2
>                 PEERS: 753, 754, 755
>            SCATTERING: 0
>         WRITTEN_BYTES: 0
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE:
>         
>... ...
>
>*************************** 8. row ***************************
>             REGION_ID: 34
>             START_KEY: t_106_r_8070450532247928832
>               END_KEY: t_281474976710649_
>             LEADER_ID: 139
>       LEADER_STORE_ID: 2
>                 PEERS: 35, 139, 186
>            SCATTERING: 0
>         WRITTEN_BYTES: 0
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>8 rows in set (0.05 sec)
> ```

### Step 5. Insert data into the orders2 table
```
BEGIN;
INSERT INTO orders2(id, code, status) VALUES(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(9, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
COMMIT;
SELECT id, code, status FROM orders2;
SELECT _tidb_rowid, id, code, status FROM orders2 ORDER BY _tidb_rowid;
```
> **sample output**
> ```
> +----+----------------------+--------+
>| id | code                 | status |
>+----+----------------------+--------+
>|  1 | 9cb0ad99d83a62915d6b |      4 |
>|  1 | 329dadd330d53bbe8aeb |      3 |
>|  2 | 999ca3ad7c2c61f61777 |      5 |
>|  2 | 74760a7e6f1f3a16ef64 |      7 |
>|  9 | 1e264cb3f696c5d9b94c |      2 |
>+----+----------------------+--------+
>5 rows in set (0.01 sec)
>
>+---------------------+----+----------------------+--------+
>| _tidb_rowid         | id | code                 | status |
>+---------------------+----+----------------------+--------+
>| 5764607523034234881 |  1 | 9cb0ad99d83a62915d6b |      4 |
>| 5764607523034234882 |  1 | 329dadd330d53bbe8aeb |      3 |
>| 5764607523034234883 |  2 | 999ca3ad7c2c61f61777 |      5 |
>| 5764607523034234884 |  2 | 74760a7e6f1f3a16ef64 |      7 |
>| 5764607523034234885 |  9 | 1e264cb3f696c5d9b94c |      2 |
>+---------------------+----+----------------------+--------+
>5 rows in set (0.00 sec)
> ```

- You can find that, although the SHARD_ROW_ID_BITS is used, the generated hidden column _tidb_rowid is still continuous.

### Step 6. TRUNCATE the data in orders2
- Note that after TRUNCATE, orders2 stills has 8 default regions. This is because that orders2 specifies the PRE_SPLIT_REGIONS=3. There is no need to be manually split again.
```
TRUNCATE TABLE orders2;
INSERT INTO orders2(id, code, status) VALUES(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), \
(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), (2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), \
(2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10)), (9, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
SELECT id, code, status FROM orders2;
SELECT _tidb_rowid, id, code, status FROM orders2 ORDER BY _tidb_rowid;
```
> **sample output**
> ```
> +----+----------------------+--------+
>| id | code                 | status |
>+----+----------------------+--------+
>|  1 | 71f193f2ed99f16bed50 |      4 |
>|  1 | 83664e98ca7e5991ac06 |      0 |
>|  2 | 79639074670bc1b9dfaf |      0 |
>|  2 | 446775a4c9850a10ce95 |      8 |
>|  9 | 1a2e539cc7aac470f468 |      4 |
>+----+----------------------+--------+
>5 rows in set (0.00 sec)
>
>+---------------------+----+----------------------+--------+
>| _tidb_rowid         | id | code                 | status |
>+---------------------+----+----------------------+--------+
>| 1152921504606846977 |  1 | 71f193f2ed99f16bed50 |      4 |
>| 1152921504606846978 |  1 | 83664e98ca7e5991ac06 |      0 |
>| 1152921504606846979 |  2 | 79639074670bc1b9dfaf |      0 |
>| 1152921504606846980 |  2 | 446775a4c9850a10ce95 |      8 |
>| 1152921504606846981 |  9 | 1a2e539cc7aac470f468 |      4 |
>+---------------------+----+----------------------+--------+
>5 rows in set (0.01 sec)
> ```

- The conclusion is that as long as the insert statements are within the same transaction, regardless of whether the SHARD_ROW_ID_BITS property of the table is specified or not, the generated hidden column _tidb_rowid will still be continuous, and there may still be a possibility of hotspot. Below we will perform the insert operations in different transactions.

### Step 7. Insert data in different transactions
```
TRUNCATE TABLE orders2;
SET autocommit=ON;
INSERT INTO orders2(id, code, status) VALUES(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(1, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(2, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
INSERT INTO orders2(id, code, status) VALUES(9, SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10));
SELECT id, code, status FROM orders2;
SELECT _tidb_rowid, id, code, status FROM orders2 ORDER BY _tidb_rowid;
```

> **sample output**
> ```
> +----+----------------------+--------+
>| id | code                 | status |
>+----+----------------------+--------+
>|  9 | dbb9004292d00a8a0059 |      8 |
>|  1 | 219cf3b5989d4bf0973c |      7 |
>|  2 | e2d02bf318cca2040c71 |      4 |
>|  1 | 3c32ed407a200ed13f5c |      1 |
>|  2 | 74301f4a88de6434abd3 |      7 |
>+----+----------------------+--------+
>5 rows in set (0.00 sec)
>
>+---------------------+----+----------------------+--------+
>| _tidb_rowid         | id | code                 | status |
>+---------------------+----+----------------------+--------+
>| 8070450532247928837 |  9 | dbb9004292d00a8a0059 |      8 |
>| 2305843009213693953 |  1 | 219cf3b5989d4bf0973c |      7 |
>| 1729382256910270467 |  2 | 74301f4a88de6434abd3 |      7 |
>| 7493989779944505346 |  1 | 3c32ed407a200ed13f5c |      1 |
>| 4611686018427387908 |  2 | e2d02bf318cca2040c71 |      4 |
>+---------------------+----+----------------------+--------+
>5 rows in set (0.00 sec)
> ```

- When different transactions perform inserts, we found that the automatically generated hidden column _tidb_rowid is not continuous.

### Step 8. Clean up table orders2
```
DROP TABLE orders2;
```
> **sample output**
> ```
> Query OK, 0 rows affected (0.52 sec)
> ```

## Module 3: Splitting a Table

- In this exercise, we will create a clustered index for primary key table. After inserting data into the table, we will manually split the regions of the table to prevent hotspots in read and write operations.

### Step 1. Create a new clustered index for primary key table orders
```
USE test;
CREATE TABLE orders
( id bigINT(20) unsigned AUTO_RANDOM NOT NULL ,
  code VARCHAR(30) NOT NULL,
  order_no VARCHAR(200) NOT NULL DEFAULT '', 
  status INT(4) NOT NULL,
  cancel_flag INT(4) DEFAULT NULL,
  create_user VARCHAR(50) DEFAULT NULL,
  update_user VARCHAR(50) DEFAULT NULL,
  create_time DATETIME DEFAULT NULL, 
  update_time DATETIME DEFAULT NULL,
  PRIMARY KEY (id) CLUSTERED
);
SHOW CREATE TABLE orders\G
```

> **sample output**
> ```
> *************************** 1. row ***************************
>      Table: orders
>Create Table: CREATE TABLE `orders` (
>`id` bigINT(20) unsigned NOT NULL /*T![auto_rand] AUTO_RANDOM(5) */,
>`code` VARCHAR(30) NOT NULL,
>`order_no` VARCHAR(200) NOT NULL DEFAULT '',
>`status` INT(4) NOT NULL,
>`cancel_flag` INT(4) DEFAULT NULL,
>`create_user` VARCHAR(50) DEFAULT NULL,
>`update_user` VARCHAR(50) DEFAULT NULL,
>`create_time` DATETIME DEFAULT NULL,
>`update_time` DATETIME DEFAULT NULL,
>PRIMARY KEY (`id`) /*T![clustered_index] CLUSTERED */
>) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin
>1 row in set (0.00 sec)
> ```

### Step 2. Insert some data into the table orders
```
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Jones', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Garcia', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Davis', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Adams', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Edwards', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Anderson', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Morgan', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
 FLOOR(RAND()*10), FLOOR(RAND()*10), 'Clark', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Smith', now());
INSERT INTO orders(code, order_no, status, cancel_flag, create_user, create_time) \
VALUES(CONCAT('CODE_', LPAD(ROUND(RAND() * 1000), 3, 0)), SUBSTRING(MD5(RAND()),1,20), \
FLOOR(RAND()*10), FLOOR(RAND()*10), 'Tom', now());
SELECT COUNT(*) FROM orders;
SELECT code, create_user FROM orders;
```
> **sample output**
> ```
> +----------+
>| COUNT(*) |
>+----------+
>|       10 |
>+----------+
>1 row in set (0.01 sec)
>
>+----------+-------------+
>| code     | create_user |
>+----------+-------------+
>| CODE_525 | Edwards     |
>| CODE_502 | Davis       |
>| CODE_224 | Morgan      |
>| CODE_370 | Jones       |
>| CODE_525 | Smith       |
>| CODE_832 | Tom         |
>| CODE_189 | Anderson    |
>| CODE_483 | Clark       |
>| CODE_963 | Garcia      |
>| CODE_257 | Adams       |
>+----------+-------------+
>10 rows in set (0.00 sec)
> ```

### Step 3. Split the table orders to prevent from reading and writing hotspot
```
SHOW TABLE orders REGIONS\G
SPLIT TABLE orders BETWEEN (0) and (9223372036854775807) regions 4;
SHOW TABLE orders REGIONS\G
```

> **sample output**
> ```
> *************************** 1. row ***************************
>             REGION_ID: 28
>             START_KEY: t_110_
>               END_KEY: t_281474976710649_
>             LEADER_ID: 187
>       LEADER_STORE_ID: 2
>                 PEERS: 29, 187, 261
>            SCATTERING: 0
>         WRITTEN_BYTES: 0
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>1 row in set (0.01 sec)
>
>+--------------------+----------------------+
>| TOTAL_SPLIT_REGION | SCATTER_FINISH_RATIO |
>+--------------------+----------------------+
>|                  3 |                    1 |
>+--------------------+----------------------+
>1 row in set (0.28 sec)
>
>*************************** 1. row ***************************
>             REGION_ID: 426
>             START_KEY: t_110_
>               END_KEY: t_110_r_2305843009213693951
>             LEADER_ID: 427
>       LEADER_STORE_ID: 1
>                 PEERS: 427, 428, 429
>            SCATTERING: 0
>         WRITTEN_BYTES: 39
>            READ_BYTES: 0
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>
>... ...
>
>*************************** 4. row ***************************
>             REGION_ID: 28
>             START_KEY: t_110_r_6917529027641081853
>               END_KEY: t_281474976710649_
>             LEADER_ID: 187
>       LEADER_STORE_ID: 2
>                 PEERS: 29, 187, 261
>            SCATTERING: 0
>         WRITTEN_BYTES: 1624
>            READ_BYTES: 2290
>  APPROXIMATE_SIZE(MB): 1
>      APPROXIMATE_KEYS: 0
>SCHEDULING_CONSTRAINTS: 
>      SCHEDULING_STATE: 
>4 rows in set (0.01 sec)
> ```

### Step 4. Clean up the orders table
```
DROP TABLE orders;
```

> **sample output**
> ```
> Bye
> ```

## Module 4: Using Key Visualizer
- In this exercise, you will use the sysbench tool to simulate concurrent data insertion and adopt the AUTO_RANDOM option to attempt to split hotspots.

### Step 1. Config the key visualizer of TiDB PD Dashboard
- In the key visualizer of TiDB PD Dashboard, set the refresh interval to 30s. The default user name is root with no password.

### Step 2. Read the script oltp_common_1.lua
- Pay attention to the CREATE TABLE statement. Consider what problems may arise from the method of table creation used.
```
cat oltp_common_1.lua 
```
```
-- Copyright (C) 2006-2018 Alexey Kopytov <akopytov@gmail.com>

-- This program is free software; you can redistribute it and/or modify
-- it under the terms of the GNU General Public License as published by
-- the Free Software Foundation; either version 2 of the License, or
-- (at your option) any later version.

-- This program is distributed in the hope that it will be useful,
-- but WITHOUT ANY WARRANTY; without even the implied warranty of
-- MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
-- GNU General Public License for more details.

-- You should have received a copy of the GNU General Public License
-- along with this program; if not, write to the Free Software
-- Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA

-- -----------------------------------------------------------------------------
-- Common code for OLTP benchmarks.
-- -----------------------------------------------------------------------------

function init()
   assert(event ~= nil,
          "this script is meant to be included by other OLTP scripts and " ..
             "should not be called directly.")
end

if sysbench.cmdline.command == nil then
   error("Command is required. Supported commands: prepare, prewarm, run, " ..
            "cleanup, help")
end

-- Command line options
sysbench.cmdline.options = {
   table_size =
      {"Number of rows per table", 10000},
   range_size =
      {"Range size for range SELECT queries", 100},
   tables =
      {"Number of tables", 1},
   point_selects =
      {"Number of point SELECT queries per transaction", 10},
   simple_ranges =
      {"Number of simple range SELECT queries per transaction", 1},
   sum_ranges =
      {"Number of SELECT SUM() queries per transaction", 1},
   order_ranges =
      {"Number of SELECT ORDER BY queries per transaction", 1},
   distinct_ranges =
      {"Number of SELECT DISTINCT queries per transaction", 1},
   index_updates =
      {"Number of UPDATE index queries per transaction", 1},
   non_index_updates =
      {"Number of UPDATE non-index queries per transaction", 1},
   delete_inserts =
      {"Number of DELETE/INSERT combinations per transaction", 1},
   range_selects =
      {"Enable/disable all range SELECT queries", true},
   auto_inc =
   {"Use AUTO_INCREMENT column as Primary Key (for MySQL), " ..
       "or its alternatives in other DBMS. When disabled, use " ..
       "client-generated IDs", true},
   skip_trx =
      {"Don't start explicit transactions and execute all queries " ..
          "in the AUTOCOMMIT mode", false},
   secondary =
      {"Use a secondary index in place of the PRIMARY KEY", false},
   create_secondary =
      {"Create a secondary index in addition to the PRIMARY KEY", false},
   mysql_storage_engine =
      {"Storage engine, if MySQL is used", "innodb"},
   pgsql_variant =
      {"Use this PostgreSQL variant when running with the " ..
          "PostgreSQL driver. The only currently supported " ..
          "variant is 'redshift'. When enabled, " ..
          "create_secondary is automatically disabled, and " ..
          "delete_inserts is set to 0"}
}

-- Prepare the dataset. This command supports parallel execution, i.e. will
-- benefit from executing with --threads > 1 as long as --tables > 1
function cmd_prepare()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   for i = sysbench.tid % sysbench.opt.threads + 1, sysbench.opt.tables,
   sysbench.opt.threads do
     create_table(drv, con, i)
   end
end

-- Preload the dataset into the server cache. This command supports parallel
-- execution, i.e. will benefit from executing with --threads > 1 as long as
-- --tables > 1
--
-- PS. Currently, this command is only meaningful for MySQL/InnoDB benchmarks
function cmd_prewarm()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   assert(drv:name() == "mysql", "prewarm is currently MySQL only")

   -- Do not create on disk tables for subsequent queries
   con:query("SET tmp_table_size=2*1024*1024*1024")
   con:query("SET max_heap_table_size=2*1024*1024*1024")

   for i = sysbench.tid % sysbench.opt.threads + 1, sysbench.opt.tables,
   sysbench.opt.threads do
      local t = "sbtest" .. i
      print("Prewarming table " .. t)
      con:query("ANALYZE TABLE sbtest" .. i)
      con:query(string.format(
                   "SELECT AVG(id) FROM " ..
                      "(SELECT * FROM %s FORCE KEY (PRIMARY) " ..
                      "LIMIT %u) t",
                   t, sysbench.opt.table_size))
      con:query(string.format(
                   "SELECT COUNT(*) FROM " ..
                      "(SELECT * FROM %s WHERE k LIKE '%%0%%' LIMIT %u) t",
                   t, sysbench.opt.table_size))
   end
end

-- Implement parallel prepare and prewarm commands
sysbench.cmdline.commands = {
   prepare = {cmd_prepare, sysbench.cmdline.PARALLEL_COMMAND},
   prewarm = {cmd_prewarm, sysbench.cmdline.PARALLEL_COMMAND}
}


-- Template strings of random digits with 11-digit groups separated by dashes

-- 10 groups, 119 characters
local c_value_template = "###########-###########-###########-" ..
   "###########-###########-###########-" ..
   "###########-###########-###########-" ..
   "###########"

-- 5 groups, 59 characters
local pad_value_template = "###########-###########-###########-" ..
   "###########-###########"

function get_c_value()
   return sysbench.rand.string(c_value_template)
end

function get_pad_value()
   return sysbench.rand.string(pad_value_template)
end

function create_table(drv, con, table_num)
   local id_index_def, id_def
   local engine_def = ""
   local extra_table_options = ""
   local query

   if sysbench.opt.secondary then
     id_index_def = "KEY xid"
   else
     id_index_def = "PRIMARY KEY"
   end

   if drv:name() == "mysql" or drv:name() == "attachsql" or
      drv:name() == "drizzle"
   then
      if sysbench.opt.auto_inc then
         id_def = "INTEGER NOT NULL AUTO_INCREMENT"
      else
         id_def = "INTEGER NOT NULL"
      end
      engine_def = "/*! ENGINE = " .. sysbench.opt.mysql_storage_engine .. " */"
      extra_table_options = mysql_table_options or ""
   elseif drv:name() == "pgsql"
   then
      if not sysbench.opt.auto_inc then
         id_def = "INTEGER NOT NULL"
      elseif pgsql_variant == 'redshift' then
        id_def = "INTEGER IDENTITY(1,1)"
      else
        id_def = "SERIAL"
      end
   else
      error("Unsupported database driver:" .. drv:name())
   end

   print(string.format("Creating table 'sbtest%d'...", table_num))

   query = string.format([[
CREATE TABLE sbtest%d(
  id int(11) NOT NULL AUTO_INCREMENT,
  k int(11) NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad CHAR(60) DEFAULT '' NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin]],
      table_num)

   con:query(query)

   if sysbench.opt.create_secondary then
      print(string.format("Creating a secondary index on 'sbtest%d'...",
                          table_num))
      con:query(string.format("CREATE INDEX k_%d ON sbtest%d(k)",
                              table_num, table_num))
   end

   if (sysbench.opt.table_size > 0) then
      print(string.format("Inserting %d records into 'sbtest%d'",
                          sysbench.opt.table_size, table_num))
   end

   if sysbench.opt.auto_inc then
      query = "INSERT INTO sbtest" .. table_num .. "(k, c, pad) VALUES"
   else
      query = "INSERT INTO sbtest" .. table_num .. "(id, k, c, pad) VALUES"
   end

   con:bulk_insert_init(query)

   local c_val
   local pad_val

   for i = 1, sysbench.opt.table_size do

      c_val = get_c_value()
      pad_val = get_pad_value()

      if (sysbench.opt.auto_inc) then
         query = string.format("(%d, '%s', '%s')",
                               sb_rand(1, sysbench.opt.table_size), c_val,
                               pad_val)
      else
         query = string.format("(%d, %d, '%s', '%s')",
                               i, sb_rand(1, sysbench.opt.table_size), c_val,
                               pad_val)
      end

      con:bulk_insert_next(query)
   end

   con:bulk_insert_done()
end

local t = sysbench.sql.type
local stmt_defs = {
   point_selects = {
      "SELECT c FROM sbtest%u WHERE id=?",
      t.INT},
   simple_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ?",
      t.INT, t.INT},
   sum_ranges = {
      "SELECT SUM(k) FROM sbtest%u WHERE id BETWEEN ? AND ?",
       t.INT, t.INT},
   order_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
       t.INT, t.INT},
   distinct_ranges = {
      "SELECT DISTINCT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
      t.INT, t.INT},
   index_updates = {
      "UPDATE sbtest%u SET k=k+1 WHERE id=?",
      t.INT},
   non_index_updates = {
      "UPDATE sbtest%u SET c=? WHERE id=?",
      {t.CHAR, 120}, t.INT},
   deletes = {
      "DELETE FROM sbtest%u WHERE id=?",
      t.INT},
   inserts = {
      "INSERT INTO sbtest%u (id, k, c, pad) VALUES (?, ?, ?, ?)",
      t.INT, t.INT, {t.CHAR, 120}, {t.CHAR, 60}},
}

function prepare_begin()
   stmt.begin = con:prepare("BEGIN")
end

function prepare_commit()
   stmt.commit = con:prepare("COMMIT")
end

function prepare_for_each_table(key)
   for t = 1, sysbench.opt.tables do
      stmt[t][key] = con:prepare(string.format(stmt_defs[key][1], t))

      local nparam = #stmt_defs[key] - 1

      if nparam > 0 then
         param[t][key] = {}
      end

      for p = 1, nparam do
         local btype = stmt_defs[key][p+1]
         local len

         if type(btype) == "table" then
            len = btype[2]
            btype = btype[1]
         end
         if btype == sysbench.sql.type.VARCHAR or
            btype == sysbench.sql.type.CHAR then
               param[t][key][p] = stmt[t][key]:bind_create(btype, len)
         else
            param[t][key][p] = stmt[t][key]:bind_create(btype)
         end
      end

      if nparam > 0 then
         stmt[t][key]:bind_param(unpack(param[t][key]))
      end
   end
end

function prepare_point_selects()
   prepare_for_each_table("point_selects")
end

function prepare_simple_ranges()
   prepare_for_each_table("simple_ranges")
end

function prepare_sum_ranges()
   prepare_for_each_table("sum_ranges")
end

function prepare_order_ranges()
   prepare_for_each_table("order_ranges")
end

function prepare_distinct_ranges()
   prepare_for_each_table("distinct_ranges")
end

function prepare_index_updates()
   prepare_for_each_table("index_updates")
end

function prepare_non_index_updates()
   prepare_for_each_table("non_index_updates")
end

function prepare_delete_inserts()
   prepare_for_each_table("deletes")
   prepare_for_each_table("inserts")
end

function thread_init()
   drv = sysbench.sql.driver()
   con = drv:connect()

   -- Create global nested tables for prepared statements and their
   -- parameters. We need a statement and a parameter set for each combination
   -- of connection/table/query
   stmt = {}
   param = {}

   for t = 1, sysbench.opt.tables do
      stmt[t] = {}
      param[t] = {}
   end

   -- This function is a 'callback' defined by individual benchmark scripts
   prepare_statements()
end

-- Close prepared statements
function close_statements()
   for t = 1, sysbench.opt.tables do
      for k, s in pairs(stmt[t]) do
         stmt[t][k]:close()
      end
   end
   if (stmt.begin ~= nil) then
      stmt.begin:close()
   end
   if (stmt.commit ~= nil) then
      stmt.commit:close()
   end
end

function thread_done()
   close_statements()
   con:disconnect()
end

function cleanup()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   for i = 1, sysbench.opt.tables do
      print(string.format("Dropping table 'sbtest%d'...", i))
      con:query("DROP TABLE IF EXISTS sbtest" .. i )
   end
end

local function get_table_num()
   return sysbench.rand.uniform(1, sysbench.opt.tables)
end

local function get_id()
   return sysbench.rand.default(1, sysbench.opt.table_size)
end

function begin()
   stmt.begin:execute()
end

function commit()
   stmt.commit:execute()
end

function execute_point_selects()
   local tnum = get_table_num()
   local i

   for i = 1, sysbench.opt.point_selects do
      param[tnum].point_selects[1]:set(get_id())

      stmt[tnum].point_selects:execute()
   end
end

local function execute_range(key)
   local tnum = get_table_num()

   for i = 1, sysbench.opt[key] do
      local id = get_id()

      param[tnum][key][1]:set(id)
      param[tnum][key][2]:set(id + sysbench.opt.range_size - 1)

      stmt[tnum][key]:execute()
   end
end

function execute_simple_ranges()
   execute_range("simple_ranges")
end

function execute_sum_ranges()
   execute_range("sum_ranges")
end

function execute_order_ranges()
   execute_range("order_ranges")
end

function execute_distinct_ranges()
   execute_range("distinct_ranges")
end

function execute_index_updates()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.index_updates do
      param[tnum].index_updates[1]:set(get_id())

      stmt[tnum].index_updates:execute()
   end
end

function execute_non_index_updates()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.non_index_updates do
      param[tnum].non_index_updates[1]:set_rand_str(c_value_template)
      param[tnum].non_index_updates[2]:set(get_id())

      stmt[tnum].non_index_updates:execute()
   end
end

function execute_delete_inserts()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.delete_inserts do
      local id = get_id()
      local k = get_id()

      param[tnum].deletes[1]:set(id)

      param[tnum].inserts[1]:set(id)
      param[tnum].inserts[2]:set(k)
      param[tnum].inserts[3]:set_rand_str(c_value_template)
      param[tnum].inserts[4]:set_rand_str(pad_value_template)

      stmt[tnum].deletes:execute()
      stmt[tnum].inserts:execute()
   end
end

-- Re-prepare statements if we have reconnected, which is possible when some of
-- the listed error codes are in the --mysql-ignore-errors list
function sysbench.hooks.before_restart_event(errdesc)
   if errdesc.sql_errno == 2013 or -- CR_SERVER_LOST
      errdesc.sql_errno == 2055 or -- CR_SERVER_LOST_EXTENDED
      errdesc.sql_errno == 2006 or -- CR_SERVER_GONE_ERROR
      errdesc.sql_errno == 2011    -- CR_TCP_CONNECTION
   then
      close_statements()
      prepare_statements()
   end
end
```

- The table created has a clustered index for primary key. It uses AUTO_INCREMENT on the primary key column id. This may result in hotspot problems during concurrent insertion.

### Step 3. Check the configuration file of sysbench tool
```
cat config_lab01
```
```
mysql-host=10.90.1.249
mysql-port=4000 
mysql-user=root 
mysql-db=sbtest 
mysql-password=
time=600 
threads=32 
report-interval=10
```
- Here are more detailed explanation:
    - time: The running time of the test.
    - thread: The number of concurrent sessions.
    - report-interval: Printing report interval (unit: seconds).
    - db-driver: MySQL compatible syntax.

### Step 4. Create a new database for sysbench testing
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

```
CREATE DATABASE sbtest;
EXIT;
```

### Step 5. Run sysbench to simulate the scenario of concurrent data insertion

- This step may take up to 5 minutes, and during the process, we can monitor whether there are hotspots generated through the traffic visualization chart in TiDB PD Dashboard.
```
sysbench './oltp_common_1.lua' --config-file=config_lab01 --tables=1 --table-size=5000000 prepare
```
> **sample output**
> ```
> sysbench './oltp_common_1.lua' --config-file=config_lab01 --tables=1 --table-size=5000000 prepare
>sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)
>
>Initializing worker threads...
>
>Creating table 'sbtest1'...
>Inserting 20000000 records into 'sbtest1'
> ```

- table-size : The number of rows will be inserted to the table.

- After the completion of the sysbench test, observe the write traffic within 1 hour in the traffic visualization chart in TiDB PD Dashboard. The following figure is generated with 20,000,000 insertions. Your figure might be different. Consider why this occurs and whether it can be optimized.
![alt text](image.png)

- Because the table is a clustered index for primary key and uses AUTO_INCREMENT to define the primary key, when continuously inserting data, we found that at the same time, the write pressure is concentrated on a certain region, which means there is a hotspot problem.

### Step 6. Clean up the data of oltp_common_1.lua
```
sysbench './oltp_common_1.lua' --config-file=config_lab01 cleanup
```

> **sample output**
> ```
> sysbench './oltp_common_1.lua' --config-file=config_lab01 cleanup
>sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)
>
>Dropping table 'sbtest1'...
> ```

### Step 7. Read the script oltp_common_2.lua
```
cat oltp_common_2.lua 
```
```
-- Copyright (C) 2006-2018 Alexey Kopytov <akopytov@gmail.com>

-- This program is free software; you can redistribute it and/or modify
-- it under the terms of the GNU General Public License as published by
-- the Free Software Foundation; either version 2 of the License, or
-- (at your option) any later version.

-- This program is distributed in the hope that it will be useful,
-- but WITHOUT ANY WARRANTY; without even the implied warranty of
-- MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
-- GNU General Public License for more details.

-- You should have received a copy of the GNU General Public License
-- along with this program; if not, write to the Free Software
-- Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA

-- -----------------------------------------------------------------------------
-- Common code for OLTP benchmarks.
-- -----------------------------------------------------------------------------

function init()
   assert(event ~= nil,
          "this script is meant to be included by other OLTP scripts and " ..
             "should not be called directly.")
end

if sysbench.cmdline.command == nil then
   error("Command is required. Supported commands: prepare, prewarm, run, " ..
            "cleanup, help")
end

-- Command line options
sysbench.cmdline.options = {
   table_size =
      {"Number of rows per table", 10000},
   range_size =
      {"Range size for range SELECT queries", 100},
   tables =
      {"Number of tables", 1},
   point_selects =
      {"Number of point SELECT queries per transaction", 10},
   simple_ranges =
      {"Number of simple range SELECT queries per transaction", 1},
   sum_ranges =
      {"Number of SELECT SUM() queries per transaction", 1},
   order_ranges =
      {"Number of SELECT ORDER BY queries per transaction", 1},
   distinct_ranges =
      {"Number of SELECT DISTINCT queries per transaction", 1},
   index_updates =
      {"Number of UPDATE index queries per transaction", 1},
   non_index_updates =
      {"Number of UPDATE non-index queries per transaction", 1},
   delete_inserts =
      {"Number of DELETE/INSERT combinations per transaction", 1},
   range_selects =
      {"Enable/disable all range SELECT queries", true},
   auto_inc =
   {"Use AUTO_INCREMENT column as Primary Key (for MySQL), " ..
       "or its alternatives in other DBMS. When disabled, use " ..
       "client-generated IDs", true},
   skip_trx =
      {"Don't start explicit transactions and execute all queries " ..
          "in the AUTOCOMMIT mode", false},
   secondary =
      {"Use a secondary index in place of the PRIMARY KEY", false},
   create_secondary =
      {"Create a secondary index in addition to the PRIMARY KEY", false},
   mysql_storage_engine =
      {"Storage engine, if MySQL is used", "innodb"},
   pgsql_variant =
      {"Use this PostgreSQL variant when running with the " ..
          "PostgreSQL driver. The only currently supported " ..
          "variant is 'redshift'. When enabled, " ..
          "create_secondary is automatically disabled, and " ..
          "delete_inserts is set to 0"}
}

-- Prepare the dataset. This command supports parallel execution, i.e. will
-- benefit from executing with --threads > 1 as long as --tables > 1
function cmd_prepare()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   for i = sysbench.tid % sysbench.opt.threads + 1, sysbench.opt.tables,
   sysbench.opt.threads do
     create_table(drv, con, i)
   end
end

-- Preload the dataset into the server cache. This command supports parallel
-- execution, i.e. will benefit from executing with --threads > 1 as long as
-- --tables > 1
--
-- PS. Currently, this command is only meaningful for MySQL/InnoDB benchmarks
function cmd_prewarm()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   assert(drv:name() == "mysql", "prewarm is currently MySQL only")

   -- Do not create on disk tables for subsequent queries
   con:query("SET tmp_table_size=2*1024*1024*1024")
   con:query("SET max_heap_table_size=2*1024*1024*1024")

   for i = sysbench.tid % sysbench.opt.threads + 1, sysbench.opt.tables,
   sysbench.opt.threads do
      local t = "sbtest" .. i
      print("Prewarming table " .. t)
      con:query("ANALYZE TABLE sbtest" .. i)
      con:query(string.format(
                   "SELECT AVG(id) FROM " ..
                      "(SELECT * FROM %s FORCE KEY (PRIMARY) " ..
                      "LIMIT %u) t",
                   t, sysbench.opt.table_size))
      con:query(string.format(
                   "SELECT COUNT(*) FROM " ..
                      "(SELECT * FROM %s WHERE k LIKE '%%0%%' LIMIT %u) t",
                   t, sysbench.opt.table_size))
   end
end

-- Implement parallel prepare and prewarm commands
sysbench.cmdline.commands = {
   prepare = {cmd_prepare, sysbench.cmdline.PARALLEL_COMMAND},
   prewarm = {cmd_prewarm, sysbench.cmdline.PARALLEL_COMMAND}
}


-- Template strings of random digits with 11-digit groups separated by dashes

-- 10 groups, 119 characters
local c_value_template = "###########-###########-###########-" ..
   "###########-###########-###########-" ..
   "###########-###########-###########-" ..
   "###########"

-- 5 groups, 59 characters
local pad_value_template = "###########-###########-###########-" ..
   "###########-###########"

function get_c_value()
   return sysbench.rand.string(c_value_template)
end

function get_pad_value()
   return sysbench.rand.string(pad_value_template)
end

function create_table(drv, con, table_num)
   local id_index_def, id_def
   local engine_def = ""
   local extra_table_options = ""
   local query

   if sysbench.opt.secondary then
     id_index_def = "KEY xid"
   else
     id_index_def = "PRIMARY KEY"
   end

   if drv:name() == "mysql" or drv:name() == "attachsql" or
      drv:name() == "drizzle"
   then
      if sysbench.opt.auto_inc then
         id_def = "INTEGER NOT NULL AUTO_INCREMENT"
      else
         id_def = "INTEGER NOT NULL"
      end
      engine_def = "/*! ENGINE = " .. sysbench.opt.mysql_storage_engine .. " */"
      extra_table_options = mysql_table_options or ""
   elseif drv:name() == "pgsql"
   then
      if not sysbench.opt.auto_inc then
         id_def = "INTEGER NOT NULL"
      elseif pgsql_variant == 'redshift' then
        id_def = "INTEGER IDENTITY(1,1)"
      else
        id_def = "SERIAL"
      end
   else
      error("Unsupported database driver:" .. drv:name())
   end

   print(string.format("Creating table 'sbtest%d'...", table_num))

   query = string.format([[
CREATE TABLE sbtest%d(
  id bigint(11) NOT NULL AUTO_RANDOM(5),
  k int(11) NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad CHAR(60) DEFAULT '' NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin]],
      table_num)

   con:query(query)

   if sysbench.opt.create_secondary then
      print(string.format("Creating a secondary index on 'sbtest%d'...",
                          table_num))
      con:query(string.format("CREATE INDEX k_%d ON sbtest%d(k)",
                              table_num, table_num))
   end

   if (sysbench.opt.table_size > 0) then
      print(string.format("Inserting %d records into 'sbtest%d'",
                          sysbench.opt.table_size, table_num))
   end

   if sysbench.opt.auto_inc then
      query = "INSERT INTO sbtest" .. table_num .. "(k, c, pad) VALUES"
   else
      query = "INSERT INTO sbtest" .. table_num .. "(id, k, c, pad) VALUES"
   end

   con:bulk_insert_init(query)

   local c_val
   local pad_val

   for i = 1, sysbench.opt.table_size do

      c_val = get_c_value()
      pad_val = get_pad_value()

      if (sysbench.opt.auto_inc) then
         query = string.format("(%d, '%s', '%s')",
                               sb_rand(1, sysbench.opt.table_size), c_val,
                               pad_val)
      else
         query = string.format("(%d, %d, '%s', '%s')",
                               i, sb_rand(1, sysbench.opt.table_size), c_val,
                               pad_val)
      end

      con:bulk_insert_next(query)
   end

   con:bulk_insert_done()
end

local t = sysbench.sql.type
local stmt_defs = {
   point_selects = {
      "SELECT c FROM sbtest%u WHERE id=?",
      t.INT},
   simple_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ?",
      t.INT, t.INT},
   sum_ranges = {
      "SELECT SUM(k) FROM sbtest%u WHERE id BETWEEN ? AND ?",
       t.INT, t.INT},
   order_ranges = {
      "SELECT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
       t.INT, t.INT},
   distinct_ranges = {
      "SELECT DISTINCT c FROM sbtest%u WHERE id BETWEEN ? AND ? ORDER BY c",
      t.INT, t.INT},
   index_updates = {
      "UPDATE sbtest%u SET k=k+1 WHERE id=?",
      t.INT},
   non_index_updates = {
      "UPDATE sbtest%u SET c=? WHERE id=?",
      {t.CHAR, 120}, t.INT},
   deletes = {
      "DELETE FROM sbtest%u WHERE id=?",
      t.INT},
   inserts = {
      "INSERT INTO sbtest%u (id, k, c, pad) VALUES (?, ?, ?, ?)",
      t.INT, t.INT, {t.CHAR, 120}, {t.CHAR, 60}},
}

function prepare_begin()
   stmt.begin = con:prepare("BEGIN")
end

function prepare_commit()
   stmt.commit = con:prepare("COMMIT")
end

function prepare_for_each_table(key)
   for t = 1, sysbench.opt.tables do
      stmt[t][key] = con:prepare(string.format(stmt_defs[key][1], t))

      local nparam = #stmt_defs[key] - 1

      if nparam > 0 then
         param[t][key] = {}
      end

      for p = 1, nparam do
         local btype = stmt_defs[key][p+1]
         local len

         if type(btype) == "table" then
            len = btype[2]
            btype = btype[1]
         end
         if btype == sysbench.sql.type.VARCHAR or
            btype == sysbench.sql.type.CHAR then
               param[t][key][p] = stmt[t][key]:bind_create(btype, len)
         else
            param[t][key][p] = stmt[t][key]:bind_create(btype)
         end
      end

      if nparam > 0 then
         stmt[t][key]:bind_param(unpack(param[t][key]))
      end
   end
end

function prepare_point_selects()
   prepare_for_each_table("point_selects")
end

function prepare_simple_ranges()
   prepare_for_each_table("simple_ranges")
end

function prepare_sum_ranges()
   prepare_for_each_table("sum_ranges")
end

function prepare_order_ranges()
   prepare_for_each_table("order_ranges")
end

function prepare_distinct_ranges()
   prepare_for_each_table("distinct_ranges")
end

function prepare_index_updates()
   prepare_for_each_table("index_updates")
end

function prepare_non_index_updates()
   prepare_for_each_table("non_index_updates")
end

function prepare_delete_inserts()
   prepare_for_each_table("deletes")
   prepare_for_each_table("inserts")
end

function thread_init()
   drv = sysbench.sql.driver()
   con = drv:connect()

   -- Create global nested tables for prepared statements and their
   -- parameters. We need a statement and a parameter set for each combination
   -- of connection/table/query
   stmt = {}
   param = {}

   for t = 1, sysbench.opt.tables do
      stmt[t] = {}
      param[t] = {}
   end

   -- This function is a 'callback' defined by individual benchmark scripts
   prepare_statements()
end

-- Close prepared statements
function close_statements()
   for t = 1, sysbench.opt.tables do
      for k, s in pairs(stmt[t]) do
         stmt[t][k]:close()
      end
   end
   if (stmt.begin ~= nil) then
      stmt.begin:close()
   end
   if (stmt.commit ~= nil) then
      stmt.commit:close()
   end
end

function thread_done()
   close_statements()
   con:disconnect()
end

function cleanup()
   local drv = sysbench.sql.driver()
   local con = drv:connect()

   for i = 1, sysbench.opt.tables do
      print(string.format("Dropping table 'sbtest%d'...", i))
      con:query("DROP TABLE IF EXISTS sbtest" .. i )
   end
end

local function get_table_num()
   return sysbench.rand.uniform(1, sysbench.opt.tables)
end

local function get_id()
   return sysbench.rand.default(1, sysbench.opt.table_size)
end

function begin()
   stmt.begin:execute()
end

function commit()
   stmt.commit:execute()
end

function execute_point_selects()
   local tnum = get_table_num()
   local i

   for i = 1, sysbench.opt.point_selects do
      param[tnum].point_selects[1]:set(get_id())

      stmt[tnum].point_selects:execute()
   end
end

local function execute_range(key)
   local tnum = get_table_num()

   for i = 1, sysbench.opt[key] do
      local id = get_id()

      param[tnum][key][1]:set(id)
      param[tnum][key][2]:set(id + sysbench.opt.range_size - 1)

      stmt[tnum][key]:execute()
   end
end

function execute_simple_ranges()
   execute_range("simple_ranges")
end

function execute_sum_ranges()
   execute_range("sum_ranges")
end

function execute_order_ranges()
   execute_range("order_ranges")
end

function execute_distinct_ranges()
   execute_range("distinct_ranges")
end

function execute_index_updates()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.index_updates do
      param[tnum].index_updates[1]:set(get_id())

      stmt[tnum].index_updates:execute()
   end
end

function execute_non_index_updates()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.non_index_updates do
      param[tnum].non_index_updates[1]:set_rand_str(c_value_template)
      param[tnum].non_index_updates[2]:set(get_id())

      stmt[tnum].non_index_updates:execute()
   end
end

function execute_delete_inserts()
   local tnum = get_table_num()

   for i = 1, sysbench.opt.delete_inserts do
      local id = get_id()
      local k = get_id()

      param[tnum].deletes[1]:set(id)

      param[tnum].inserts[1]:set(id)
      param[tnum].inserts[2]:set(k)
      param[tnum].inserts[3]:set_rand_str(c_value_template)
      param[tnum].inserts[4]:set_rand_str(pad_value_template)

      stmt[tnum].deletes:execute()
      stmt[tnum].inserts:execute()
   end
end

-- Re-prepare statements if we have reconnected, which is possible when some of
-- the listed error codes are in the --mysql-ignore-errors list
function sysbench.hooks.before_restart_event(errdesc)
   if errdesc.sql_errno == 2013 or -- CR_SERVER_LOST
      errdesc.sql_errno == 2055 or -- CR_SERVER_LOST_EXTENDED
      errdesc.sql_errno == 2006 or -- CR_SERVER_GONE_ERROR
      errdesc.sql_errno == 2011    -- CR_TCP_CONNECTION
   then
      close_statements()
      prepare_statements()
   end
end
```
### Step 8. Run sysbench to simulate the scenario of concurrent data insertion
- Again, this step may take 5 minutes. During this process, we can monitor whether hotspots are generated by using the traffic visualization graph in the TiDB PD Dashboard.

```
sysbench './oltp_common_2.lua' --config-file=config_lab01 --tables=1 --table-size=5000000 prepare
```

> **sample output**
> ```
> sysbench './oltp_common_2.lua' --config-file=config_lab01 --tables=1 --table-size=5000000 prepare
>sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)
>
>Initializing worker threads...
>
>Creating table 'sbtest1'...
>Inserting 5000000 records into 'sbtest1'
> ```

- After the completion of the sysbench test, observe the incoming flow within 1 hour in the traffic visualization graph in TiDB PD Dashboard. The following figure is generated with 20,000,000 insertions. Your figure might be different. Consider the reasons and compare with the previous test to determine if the hotspot issue has been solved.
![alt text](image-1.png)

### Step 9. Clean up the data of oltp_common_2.lua

```
sysbench './oltp_common_2.lua' --config-file=config_lab01 cleanup
```
> **sample output**
> ```
>  sysbench './oltp_common_2.lua' --config-file=config_lab01 cleanup
>sysbench 1.0.17 (using bundled LuaJIT 2.1.0-beta2)
>
>Dropping table 'sbtest1'...
> ```

====================================================================================================================

# TiDB SQL Tuning Lab 2: Partitioned Tables

## Module 1: RANGE, RANGE COLUMN, and RANGE INTERVAL Partitioned Tables
- In this exercise, you will create and manage Range partitioning, Range COLUMNS partitioning and Range INTERVAL partitioned table.

### Step 1. Log in to the remote Linux
### Step 2. Connect to TiDB and create a non-partitioned table emp

```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

```
use test;
```

> **sample output**
> ```
>  Database changed
> ```

```
CREATE TABLE emp (
 id INT PRIMARY KEY,
 fname VARCHAR(30),
 lname VARCHAR(30),
 hired DATE NOT NULL DEFAULT '2018-01-01',
 dept_code INT NOT NULL,
 region_id INT NOT NULL
);
INSERT INTO emp VALUES(1, 'Chars', 'Smith', '2018-05-08', 1, 801);
INSERT INTO emp VALUES(2, 'Mott', 'Johnson', '2018-06-01', 2, 401);
INSERT INTO emp VALUES(3, 'Lundeen', 'Williams', '2018-06-22', 3, 201);
INSERT INTO emp VALUES(4, 'Broe', 'Brown', '2018-09-22', 4, 101);
INSERT INTO emp VALUES(5, 'Layla', 'Jones', '2018-10-12', 5, 301);
INSERT INTO emp VALUES(6, 'Tom', 'Garcia', '2019-01-30', 2, 501);
INSERT INTO emp VALUES(7, 'Donald', 'Miller', '2019-02-21', 2, 401);
INSERT INTO emp VALUES(8, 'Kevin', 'Davis', '2019-03-09', 3, 301);
INSERT INTO emp VALUES(9, 'Ken', 'Rodriguez', '2020-11-17', 1, 201);
INSERT INTO emp VALUES(10, 'Bill', 'Martinez', '2020-11-19', 2, 201);
INSERT INTO emp VALUES(11, 'Jack', 'Hernandez', '2020-12-01', 4, 201);
INSERT INTO emp VALUES(12, 'Fran', 'Jones', '2020-12-01', 4, 201);
INSERT INTO emp VALUES(13, 'Mark', 'Wilson', '2021-07-08', 4, 101);
INSERT INTO emp VALUES(14, 'Bob', 'Thomas', '2021-07-08', 4, 101);
INSERT INTO emp VALUES(15, 'Jeff', 'Taylor', '2022-04-03', 5, 101);
INSERT INTO emp VALUES(16, 'Jim', 'Lee', '2022-04-04', 2, 201);
INSERT INTO emp VALUES(17, 'Tim', 'Anderson', '2022-05-10', 2, 301);
INSERT INTO emp VALUES(18, 'Mike', 'Jackson', '2022-05-10', 3, 401);
INSERT INTO emp VALUES(19, 'Benjamin', 'Thompson', '2022-09-24', 1, 501);
INSERT INTO emp VALUES(20, 'Charlotte', 'Clark', '2022-09-24', 1, 801);
INSERT INTO emp VALUES(21, 'Mason', 'Jones', '2022-10-27', 1, 801);
INSERT INTO emp VALUES(22, 'Emma', 'Robinson', '2023-06-15', 2, 301);
INSERT INTO emp VALUES(23, 'Ava', 'Adams', '2023-07-03', 2, 301);
INSERT INTO emp VALUES(24, 'Olivia', 'Lewis', '2023-08-01', 5, 701);
INSERT INTO emp VALUES(25, 'Ava', 'Phillips', '2023-09-02', 3, 701);
INSERT INTO emp VALUES(26, 'Isabella', 'Morgan', '2023-09-03', 2, 101);
INSERT INTO emp VALUES(27, 'Donald', 'Edwards', '2023-09-03', 4, 101);
INSERT INTO emp VALUES(28, 'Somes', 'Alexander', '2023-12-10', 1, 101);
INSERT INTO emp VALUES(29, 'Crabb', 'Ford', '2024-06-30', 2, 401);
INSERT INTO emp VALUES(30, 'Sisika', 'Williams', '2024-06-30', 2, 401);
INSERT INTO emp VALUES(31, 'Noah', 'Anderson', '2024-07-02', 4, 101);
INSERT INTO emp VALUES(32, 'Jacob', 'Martinez', '2025-01-11', 4, 501);
INSERT INTO emp VALUES(33, 'Kim', 'Miller', '2025-01-30', 1, 401);
INSERT INTO emp VALUES(34, 'Jim', 'Smith', '2025-04-24', 2, 501);
INSERT INTO emp VALUES(35, 'Sam', 'Martinez', '2025-05-22', 3, 101);
INSERT INTO emp VALUES(36, 'Candy', 'Morgan', '2025-08-03', 1, 301);
SELECT COUNT(*) from emp;
```
> **sample output**
> ```
> +----------+
>| COUNT(*) |
>+----------+
>|       36 |
>+----------+
>1 row in set (0.00 sec)
> ```

### Step 3. Create a range partitioned table employees1 without PK
```
CREATE TABLE employees1 (
 id INT NOT NULL,
 fname VARCHAR(30),
 lname VARCHAR(30),
 hired DATE NOT NULL DEFAULT '2018-01-01',
 dept_code INT NOT NULL,
 region_id INT NOT NULL
)
PARTITION BY RANGE (hired) (
   PARTITION p0 VALUES LESS THAN ('2019-01-01'),
   PARTITION p1 VALUES LESS THAN ('2020-01-01'),
   PARTITION p2 VALUES LESS THAN ('2021-01-01'),
   PARTITION p3 VALUES LESS THAN ('2022-01-01'),
   PARTITION p4 VALUES LESS THAN ('2023-01-01'),
   PARTITION p5 VALUES LESS THAN ('2024-01-01'),
   PARTITION p6 VALUES LESS THAN ('2025-01-01'),
   PARTITION p7 VALUES LESS THAN ('2026-01-01'),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```

```
ERROR 1697 (HY000): VALUES value for partition 'p0' must have type INT
```
- The reason for the error ERROR 1697 (HY000): VALUES value for partition 'p0' must have type INT is that the KEY after PARTITION BY RANGE must be of INT type. We need to modify the SQL statement accordingly.

```
CREATE TABLE employees1 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE (YEAR(hired)) (
   PARTITION p0 VALUES LESS THAN (2019),
   PARTITION p1 VALUES LESS THAN (2020),
   PARTITION p2 VALUES LESS THAN (2021),
   PARTITION p3 VALUES LESS THAN (2022),
   PARTITION p4 VALUES LESS THAN (2023),
   PARTITION p5 VALUES LESS THAN (2024),
   PARTITION p6 VALUES LESS THAN (2025),
   PARTITION p7 VALUES LESS THAN (2026),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.13 sec)
> ```

### Step 4. Create a COLUMNS Range partitioned table employees2 without PK
```
CREATE TABLE employees2 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE COLUMNS(hired) (
   PARTITION p0 VALUES LESS THAN ('2019-01-01'),
   PARTITION p1 VALUES LESS THAN ('2020-01-01'),
   PARTITION p2 VALUES LESS THAN ('2021-01-01'),
   PARTITION p3 VALUES LESS THAN ('2022-01-01'),
   PARTITION p4 VALUES LESS THAN ('2023-01-01'),
   PARTITION p5 VALUES LESS THAN ('2024-01-01'),
   PARTITION p6 VALUES LESS THAN ('2025-01-01'),
   PARTITION p7 VALUES LESS THAN ('2026-01-01'),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.13 sec)
> ```

- The data types of partition columns can be integer, string (CHAR or VARCHAR), DATE, and DATETIME. Expressions, such as YEAR(), are not supported. You can use one or more columns as partitioning keys.

### Step 5. Create a Range partitioned table employees3 with PK
```
CREATE TABLE employees3 (
   id INT PRIMARY KEY,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE (YEAR(hired)) (
   PARTITION p0 VALUES LESS THAN (2019),
   PARTITION p1 VALUES LESS THAN (2020),
   PARTITION p2 VALUES LESS THAN (2021),
   PARTITION p3 VALUES LESS THAN (2022),
   PARTITION p4 VALUES LESS THAN (2023),
   PARTITION p5 VALUES LESS THAN (2024),
   PARTITION p6 VALUES LESS THAN (2025),
   PARTITION p7 VALUES LESS THAN (2026),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```

> **sample output**
> ```
>ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function
> ```
- The reason for the error ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function is that, for tables with unique keys, all columns in the partition expression must be included in the unique key. Primary key is also a unique key.
```
CREATE TABLE employees3 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL,
   PRIMARY KEY idx_id_hired(id, hired)
)
PARTITION BY RANGE (YEAR(hired)) (
   PARTITION p0 VALUES LESS THAN (2019),
   PARTITION p1 VALUES LESS THAN (2020),
   PARTITION p2 VALUES LESS THAN (2021),
   PARTITION p3 VALUES LESS THAN (2022),
   PARTITION p4 VALUES LESS THAN (2023),
   PARTITION p5 VALUES LESS THAN (2024),
   PARTITION p6 VALUES LESS THAN (2025),
   PARTITION p7 VALUES LESS THAN (2026),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.13 sec)
> ```

### Step 6. Create a COLUMNS Range partitioned table employees4 with PK
```
CREATE TABLE employees4 (
   id INT PRIMARY KEY,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE COLUMNS(hired) (
   PARTITION p0 VALUES LESS THAN ('2019-01-01'),
   PARTITION p1 VALUES LESS THAN ('2020-01-01'),
   PARTITION p2 VALUES LESS THAN ('2021-01-01'),
   PARTITION p3 VALUES LESS THAN ('2022-01-01'),
   PARTITION p4 VALUES LESS THAN ('2023-01-01'),
   PARTITION p5 VALUES LESS THAN ('2024-01-01'),
   PARTITION p6 VALUES LESS THAN ('2025-01-01'),
   PARTITION p7 VALUES LESS THAN ('2026-01-01'),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```
> **sample output**
> ```
>ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function
> ```

- The reason for the error ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function is that, for tables with unique keys, all columns in the partition expression must be included in the unique key. Primary key is also a unique key.

```
CREATE TABLE employees4 (
   id INT,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL,
   PRIMARY KEY idx_id_hired(id, hired)
)
PARTITION BY RANGE COLUMNS(hired) (
   PARTITION p0 VALUES LESS THAN ('2019-01-01'),
   PARTITION p1 VALUES LESS THAN ('2020-01-01'),
   PARTITION p2 VALUES LESS THAN ('2021-01-01'),
   PARTITION p3 VALUES LESS THAN ('2022-01-01'),
   PARTITION p4 VALUES LESS THAN ('2023-01-01'),
   PARTITION p5 VALUES LESS THAN ('2024-01-01'),
   PARTITION p6 VALUES LESS THAN ('2025-01-01'),
   PARTITION p7 VALUES LESS THAN ('2026-01-01'),
   PARTITION p8 VALUES LESS THAN MAXVALUE
);
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.14 sec)
> ```

### Step 7. Import data into partitioned tables

- Import the data from the non-partitioned table emp into the partitioned tables employees1, employees2, employees3, and employees4, and check the partition information. Then, Use the reorganize partition command to merge two partitions from employees1, employees2, employees3, and employees4 into one partition.

- Note that the ANALYZE TABLE command may need a few seconds to run.

```
INSERT INTO employees1 SELECT * FROM emp;
INSERT INTO employees2 SELECT * FROM emp;
INSERT INTO employees3 SELECT * FROM emp;
INSERT INTO employees4 SELECT * FROM emp;
ANALYZE TABLE employees1;
ANALYZE TABLE employees2;
ANALYZE TABLE employees3;
ANALYZE TABLE employees4;
SELECT table_name, partition_name, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name IN ('employees1', 'employees2', 'employees3', 'employees4') ORDER BY table_name, partition_name;
```

> **sample output**
> ```
>+------------+----------------+------------+
>| table_name | partition_name | table_rows |
>+------------+----------------+------------+
>| employees1 | p0             |          5 |
>| employees1 | p1             |          3 |
>| employees1 | p2             |          4 |
>| employees1 | p3             |          2 |
>| employees1 | p4             |          7 |
>| employees1 | p5             |          7 |
>| employees1 | p6             |          3 |
>| employees1 | p7             |          5 |
>| employees1 | p8             |          0 |
>| employees2 | p0             |          5 |
>| employees2 | p1             |          3 |
>| employees2 | p2             |          4 |
>| employees2 | p3             |          2 |
>| employees2 | p4             |          7 |
>| employees2 | p5             |          7 |
>| employees2 | p6             |          3 |
>| employees2 | p7             |          5 |
>| employees2 | p8             |          0 |
>| employees3 | p0             |          5 |
>| employees3 | p1             |          3 |
>| employees3 | p2             |          4 |
>| employees3 | p3             |          2 |
>| employees3 | p4             |          7 |
>| employees3 | p5             |          7 |
>| employees3 | p6             |          3 |
>| employees3 | p7             |          5 |
>| employees3 | p8             |          0 |
>| employees4 | p0             |          5 |
>| employees4 | p1             |          3 |
>| employees4 | p2             |          4 |
>| employees4 | p3             |          2 |
>| employees4 | p4             |          7 |
>| employees4 | p5             |          7 |
>| employees4 | p6             |          3 |
>| employees4 | p7             |          5 |
>| employees4 | p8             |          0 |
>+------------+----------------+------------+
>36 rows in set (0.01 sec)
> ```

- You might see warning message code 1105 after running the ANALYZE TABLE command. The warning message states that the dynamic pruning mode is disabled, since there is no global statistics available. You can check this documentation for details.

```
ALTER TABLE employees1 REORGANIZE PARTITION p6,p7 into (PARTITION pErr VALUES LESS THAN (2026));
ALTER TABLE employees2 REORGANIZE PARTITION p6,p7 into (PARTITION pErr VALUES LESS THAN ('2026-01-01'));
ALTER TABLE employees3 REORGANIZE PARTITION p6,p7 into (PARTITION pErr VALUES LESS THAN (2026));
ALTER TABLE employees4 REORGANIZE PARTITION p5,p7 into (PARTITION pErr VALUES LESS THAN ('2026-01-01'));
ALTER TABLE employees4 REORGANIZE PARTITION p6,p7 into (PARTITION pErr VALUES LESS THAN ('2026-01-01'));
ANALYZE TABLE employees1;
ANALYZE TABLE employees2;
ANALYZE TABLE employees3;
ANALYZE TABLE employees4;
SELECT table_name, partition_name, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name IN ('employees1', 'employees2', 'employees3', 'employees4') ORDER BY table_name, partition_name;
```

> **sample output**
> ```
>tidb>ALTER TABLE employees4 REORGANIZE PARTITION p5,p7 into (PARTITION pErr VALUES LESS THAN ('2026-01-01'));
>ERROR 8200 (HY000): Unsupported REORGANIZE PARTITION of RANGE; not adjacent partitions
>
>+------------+----------------+------------+
>| table_name | partition_name | table_rows |
>+------------+----------------+------------+
>| employees1 | p0             |          5 |
>| employees1 | p1             |          3 |
>| employees1 | p2             |          4 |
>| employees1 | p3             |          2 |
>| employees1 | p4             |          7 |
>| employees1 | p5             |          7 |
>| employees1 | p8             |          0 |
>| employees1 | pErr           |          8 |
>| employees2 | p0             |          5 |
>| employees2 | p1             |          3 |
>| employees2 | p2             |          4 |
>| employees2 | p3             |          2 |
>| employees2 | p4             |          7 |
>| employees2 | p5             |          7 |
>| employees2 | p8             |          0 |
>| employees2 | pErr           |          8 |
>| employees3 | p0             |          5 |
>| employees3 | p1             |          3 |
>| employees3 | p2             |          4 |
>| employees3 | p3             |          2 |
>| employees3 | p4             |          7 |
>| employees3 | p5             |          7 |
>| employees3 | p8             |          0 |
>| employees3 | pErr           |          8 |
>| employees4 | p0             |          5 |
>| employees4 | p1             |          3 |
>| employees4 | p2             |          4 |
>| employees4 | p3             |          2 |
>| employees4 | p4             |          7 |
>| employees4 | p5             |          7 |
>| employees4 | p8             |          0 |
>| employees4 | pErr           |          8 |
>+------------+----------------+------------+
>32 rows in set (0.01 sec)
> ```

- The reason for the error ERROR 8200 (HY000): Unsupported REORGANIZE PARTITION of RANGE; not adjacent partitions when creating the employees4 table is that for a Range partition table, you can only merge adjacent partitions.

### Step 8. Drop partitions
- Use the DROP PARTITION command to delete unnecessary partitions pErr from employees1, employees2, employees3, and employees4.
```
ALTER TABLE employees1 DROP PARTITION pErr;
ALTER TABLE employees2 DROP PARTITION pErr;
ALTER TABLE employees3 DROP PARTITION pErr;
ALTER TABLE employees4 DROP PARTITION pErr;
ANALYZE TABLE employees1;
ANALYZE TABLE employees2;
ANALYZE TABLE employees3;
ANALYZE TABLE employees4;
SELECT table_name, partition_name, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name IN ('employees1', 'employees2', 'employees3', 'employees4') ORDER BY table_name, partition_name;
```
> **sample output**
> ```
>+------------+----------------+------------+
>| table_name | partition_name | table_rows |
>+------------+----------------+------------+
>| employees1 | p0             |          5 |
>| employees1 | p1             |          3 |
>| employees1 | p2             |          4 |
>| employees1 | p3             |          2 |
>| employees1 | p4             |          7 |
>| employees1 | p5             |          7 |
>| employees1 | p8             |          0 |
>| employees2 | p0             |          5 |
>| employees2 | p1             |          3 |
>| employees2 | p2             |          4 |
>| employees2 | p3             |          2 |
>| employees2 | p4             |          7 |
>| employees2 | p5             |          7 |
>| employees2 | p8             |          0 |
>| employees3 | p0             |          5 |
>| employees3 | p1             |          3 |
>| employees3 | p2             |          4 |
>| employees3 | p3             |          2 |
>| employees3 | p4             |          7 |
>| employees3 | p5             |          7 |
>| employees3 | p8             |          0 |
>| employees4 | p0             |          5 |
>| employees4 | p1             |          3 |
>| employees4 | p2             |          4 |
>| employees4 | p3             |          2 |
>| employees4 | p4             |          7 |
>| employees4 | p5             |          7 |
>| employees4 | p8             |          0 |
>+------------+----------------+------------+
>28 rows in set (0.01 sec)
> ```

### Step 9. Create a Range INTERVAL and a Range COLUMNS partitioned table
```
CREATE TABLE employees5 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE (YEAR(hired))
INTERVAL (1) FIRST PARTITION LESS THAN (2019) LAST PARTITION LESS THAN (2026) MAXVALUE PARTITION;

CREATE TABLE employees6 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '2018-01-01',
   dept_code INT NOT NULL,
   region_id INT NOT NULL
)
PARTITION BY RANGE COLUMNS (hired)
INTERVAL (1 MONTH) FIRST PARTITION LESS THAN ('2019-01-01') LAST PARTITION LESS THAN ('2026-01-01') MAXVALUE PARTITION;

INSERT INTO employees5 SELECT * FROM emp;
INSERT INTO employees6 SELECT * FROM emp;

ANALYZE TABLE employees5;
ANALYZE TABLE employees6;

SELECT table_name, partition_name, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name IN ('employees5', 'employees6') ORDER BY table_name, partition_name;
```
> **sample output**
> ```
>+------------+-----------------+------------+
>| table_name | partition_name  | table_rows |
>+------------+-----------------+------------+
>| employees5 | P_LT_2019       |          5 |
>| employees5 | P_LT_2020       |          3 |
>| employees5 | P_LT_2021       |          4 |
>| employees5 | P_LT_2022       |          2 |
>| employees5 | P_LT_2023       |          7 |
>| employees5 | P_LT_2024       |          7 |
>| employees5 | P_LT_2025       |          3 |
>| employees5 | P_LT_2026       |          5 |
>| employees5 | P_MAXVALUE      |          0 |
>| employees6 | P_LT_2019-01-01 |          5 |
>| employees6 | P_LT_2019-02-01 |          1 |
>| employees6 | P_LT_2019-03-01 |          1 |
>| employees6 | P_LT_2019-04-01 |          1 |
>| employees6 | P_LT_2019-05-01 |          0 |
>| employees6 | P_LT_2019-06-01 |          0 |
>| employees6 | P_LT_2019-07-01 |          0 |
>| employees6 | P_LT_2019-08-01 |          0 |
>| employees6 | P_LT_2019-09-01 |          0 |
>| employees6 | P_LT_2019-10-01 |          0 |
>| employees6 | P_LT_2019-11-01 |          0 |
>| employees6 | P_LT_2019-12-01 |          0 |
>| employees6 | P_LT_2020-01-01 |          0 |
>| employees6 | P_LT_2020-02-01 |          0 |
>| employees6 | P_LT_2020-03-01 |          0 |
>| employees6 | P_LT_2020-04-01 |          0 |
>| employees6 | P_LT_2020-05-01 |          0 |
>| employees6 | P_LT_2020-06-01 |          0 |
>| employees6 | P_LT_2020-07-01 |          0 |
>| employees6 | P_LT_2020-08-01 |          0 |
>| employees6 | P_LT_2020-09-01 |          0 |
>| employees6 | P_LT_2020-10-01 |          0 |
>| employees6 | P_LT_2020-11-01 |          0 |
>| employees6 | P_LT_2020-12-01 |          2 |
>| employees6 | P_LT_2021-01-01 |          2 |
>| employees6 | P_LT_2021-02-01 |          0 |
>| employees6 | P_LT_2021-03-01 |          0 |
>| employees6 | P_LT_2021-04-01 |          0 |
>| employees6 | P_LT_2021-05-01 |          0 |
>| employees6 | P_LT_2021-06-01 |          0 |
>| employees6 | P_LT_2021-07-01 |          0 |
>| employees6 | P_LT_2021-08-01 |          2 |
>| employees6 | P_LT_2021-09-01 |          0 |
>| employees6 | P_LT_2021-10-01 |          0 |
>| employees6 | P_LT_2021-11-01 |          0 |
>| employees6 | P_LT_2021-12-01 |          0 |
>| employees6 | P_LT_2022-01-01 |          0 |
>| employees6 | P_LT_2022-02-01 |          0 |
>| employees6 | P_LT_2022-03-01 |          0 |
>| employees6 | P_LT_2022-04-01 |          0 |
>| employees6 | P_LT_2022-05-01 |          2 |
>| employees6 | P_LT_2022-06-01 |          2 |
>| employees6 | P_LT_2022-07-01 |          0 |
>| employees6 | P_LT_2022-08-01 |          0 |
>| employees6 | P_LT_2022-09-01 |          0 |
>| employees6 | P_LT_2022-10-01 |          2 |
>| employees6 | P_LT_2022-11-01 |          1 |
>| employees6 | P_LT_2022-12-01 |          0 |
>| employees6 | P_LT_2023-01-01 |          0 |
>| employees6 | P_LT_2023-02-01 |          0 |
>| employees6 | P_LT_2023-03-01 |          0 |
>| employees6 | P_LT_2023-04-01 |          0 |
>| employees6 | P_LT_2023-05-01 |          0 |
>| employees6 | P_LT_2023-06-01 |          0 |
>| employees6 | P_LT_2023-07-01 |          1 |
>| employees6 | P_LT_2023-08-01 |          1 |
>| employees6 | P_LT_2023-09-01 |          1 |
>| employees6 | P_LT_2023-10-01 |          3 |
>| employees6 | P_LT_2023-11-01 |          0 |
>| employees6 | P_LT_2023-12-01 |          0 |
>| employees6 | P_LT_2024-01-01 |          1 |
>| employees6 | P_LT_2024-02-01 |          0 |
>| employees6 | P_LT_2024-03-01 |          0 |
>| employees6 | P_LT_2024-04-01 |          0 |
>| employees6 | P_LT_2024-05-01 |          0 |
>| employees6 | P_LT_2024-06-01 |          0 |
>| employees6 | P_LT_2024-07-01 |          2 |
>| employees6 | P_LT_2024-08-01 |          1 |
>| employees6 | P_LT_2024-09-01 |          0 |
>| employees6 | P_LT_2024-10-01 |          0 |
>| employees6 | P_LT_2024-11-01 |          0 |
>| employees6 | P_LT_2024-12-01 |          0 |
>| employees6 | P_LT_2025-01-01 |          0 |
>| employees6 | P_LT_2025-02-01 |          2 |
>| employees6 | P_LT_2025-03-01 |          0 |
>| employees6 | P_LT_2025-04-01 |          0 |
>| employees6 | P_LT_2025-05-01 |          1 |
>| employees6 | P_LT_2025-06-01 |          1 |
>| employees6 | P_LT_2025-07-01 |          0 |
>| employees6 | P_LT_2025-08-01 |          0 |
>| employees6 | P_LT_2025-09-01 |          1 |
>| employees6 | P_LT_2025-10-01 |          0 |
>| employees6 | P_LT_2025-11-01 |          0 |
>| employees6 | P_LT_2025-12-01 |          0 |
>| employees6 | P_LT_2026-01-01 |          0 |
>| employees6 | P_MAXVALUE      |          0 |
>+------------+-----------------+------------+
>95 rows in set (0.01 sec)
> ```

- You see that Range INTERVAL partitioning supports two methods: PARTITION BY RANGE and PARTITION BY RANGE COLUMNS. It can also automatically partition according to the interval set.

### Step 10. Clean up the tables

```
DROP TABLE emp;
DROP TABLE employees1;
DROP TABLE employees2;
DROP TABLE employees3;
DROP TABLE employees4;
DROP TABLE employees5;
DROP TABLE employees6;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.81 sec)
> ```

## Module 2: Exchanging Partitions
- In this exercise, we will create a List COLUMNS partitioned table, insert data into the table, and then use EXCHANGE PARTITION to exchange the data between the partitions and a non-partitioned table.

### Step 1. Create a non-partitioned table students

```
CREATE TABLE students (
   id INT PRIMARY KEY,
   fname VARCHAR(30),
   lname VARCHAR(30),
   city VARCHAR(30),
   DOB DATE NOT NULL,
   dept INT NOT NULL
);
```
> **sample output**
> ```
>tidb>CREATE TABLE students (
>   ->     id INT PRIMARY KEY,
>   ->     fname VARCHAR(30),
>   ->     lname VARCHAR(30),
>   ->     city VARCHAR(30),
>   ->     DOB DATE NOT NULL,
>   ->     dept INT NOT NULL
>   -> );
>Query OK, 0 rows affected (0.53 sec)
> ```

### Step 2. Check the value of the variables tidb_enable_list_partition
```
SHOW VARIABLES LIKE 'tidb_enable_list_partition';
```

> **sample output**
> ```
>tidb>SHOW VARIABLES LIKE 'tidb_enable_list_partition';
>+----------------------------+-------+
>| Variable_name              | Value |
>+----------------------------+-------+
>| tidb_enable_list_partition | ON    |
>+----------------------------+-------+
>1 row in set (0.00 sec)
> ```

- Before creating a List partitioned table, you need to set the value of the session variable tidb_enable_list_partition to ON.

### Step 3. Create a new LIST COLUMNS partitioned table students1

```
CREATE TABLE students1 (
   id INT PRIMARY KEY,
   fname VARCHAR(30),
   lname VARCHAR(30),
   city VARCHAR(30),
   DOB DATE NOT NULL,
   dept INT NOT NULL
)
PARTITION BY LIST ( city )(
PARTITION p0 VALUES IN ('NewYork','BeiJing','Tokyo'),
PARTITION p1 VALUES IN ('Atlanta', 'ShangHai', 'Sydney'),
PARTITION p2 VALUES IN ('Boston', 'NanJing', 'Chicago'));
```

> **sample output**
> ```
>ERROR 1697 (HY000): VALUES value for partition 'p0' must have type INT
> ```

- The reason for the error ERROR 1697 (HY000): VALUES value for partition 'p0' must have type INT is that the KEY following PARTITION BY LIST must be of INT type.

```
CREATE TABLE students1 (
   id INT PRIMARY KEY,
   fname VARCHAR(30),
   lname VARCHAR(30),
   city VARCHAR(30),
   DOB DATE NOT NULL,
   dept INT NOT NULL
)
PARTITION BY LIST COLUMNS ( city )(
PARTITION p0 VALUES IN ('NewYork','BeiJing','Tokyo'),
PARTITION p1 VALUES IN ('Atlanta', 'ShangHai', 'Sydney'),
PARTITION p2 VALUES IN ('Boston', 'NanJing', 'Chicago'));
```
> **sample output**
> ```
>ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function
> ```

- The reason for the error ERROR 1503 (HY000): A CLUSTERED INDEX must include all columns in the table's partitioning function is that, for tables with unique keys, all columns in the partition expression must be included in the unique key. Primary key is also a unique key.

```
CREATE TABLE students1 (
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   city VARCHAR(30),
   DOB DATE NOT NULL,
   dept INT NOT NULL,
   PRIMARY KEY(id, city)
)
PARTITION BY LIST COLUMNS ( city )(
PARTITION p0 VALUES IN ('NewYork','BeiJing','Tokyo'),
PARTITION p1 VALUES IN ('Atlanta', 'ShangHai', 'Sydney'),
PARTITION p2 VALUES IN ('Boston', 'NanJing', 'Chicago'));
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.54 sec)
> ```

### Step 4. Insert some values into the table students1

```
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='students1' ORDER BY partition_name;
INSERT INTO students1 VALUES(1, 'Chars', 'Smith', 'NewYork', '2005-01-02', 1);
INSERT INTO students1 VALUES(2, 'Tom', 'Garcia', 'NewYork', '2005-02-01', 6);
INSERT INTO students1 VALUES(3, 'Crabb', 'Ford', 'NewYork', '2004-11-19', 7);
ANALYZE TABLE students1;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='students1' ORDER BY partition_name;
```
> **sample output**
> ```
>+------------+----------------+-------------------------------+------------+
>| table_name | partition_name | partition_description         | table_rows |
>+------------+----------------+-------------------------------+------------+
>| students1  | p0             | 'NewYork','BeiJing','Tokyo'   |          0 |
>| students1  | p1             | 'Atlanta','ShangHai','Sydney' |          0 |
>| students1  | p2             | 'Boston','NanJing','Chicago'  |          0 |
>+------------+----------------+-------------------------------+------------+
>3 rows in set (0.01 sec)
>
>+------------+----------------+-------------------------------+------------+
>| table_name | partition_name | partition_description         | table_rows |
>+------------+----------------+-------------------------------+------------+
>| students1  | p0             | 'NewYork','BeiJing','Tokyo'   |          3 |
>| students1  | p1             | 'Atlanta','ShangHai','Sydney' |          0 |
>| students1  | p2             | 'Boston','NanJing','Chicago'  |          0 |
>+------------+----------------+-------------------------------+------------+
>3 rows in set (0.00 sec)
> ```
- We can see that all the data is on the p0 partition.

### Step 5. Exchange the partitions

- Step 5. Exchange the partitions
```
ALTER TABLE students1 EXCHANGE PARTITION `p0` WITH TABLE `students`;
```
> **sample output**
> ```
>ERROR 1736 (HY000): Tables have different definitions
> ```

- The reason for the error ERROR 1736 (HY000): Tables have different definitions is that the primary key in table students1 is column (id, city), while in table students it is only column id

```
DROP TABLE students;
CREATE TABLE students (
 id INT NOT NULL,
 fname VARCHAR(30),
 lname VARCHAR(30),
 city VARCHAR(30),
 DOB DATE NOT NULL,
 dept INT NOT NULL,
 PRIMARY KEY(id, city)
);
ALTER TABLE students1 EXCHANGE PARTITION `p0` WITH TABLE `students`;
ANALYZE TABLE students1;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='students1' ORDER BY partition_name;
SELECT * FROM students;
```

> **sample output**
> ```
>+------------+----------------+-------------------------------+------------+
>| table_name | partition_name | partition_description         | table_rows |
>+------------+----------------+-------------------------------+------------+
>| students1  | p0             | 'NewYork','BeiJing','Tokyo'   |          0 |
>| students1  | p1             | 'Atlanta','ShangHai','Sydney' |          0 |
>| students1  | p2             | 'Boston','NanJing','Chicago'  |          0 |
>+------------+----------------+-------------------------------+------------+
>3 rows in set (0.00 sec)
>
>+----+-------+--------+---------+------------+------+
>| id | fname | lname  | city    | DOB        | dept |
>+----+-------+--------+---------+------------+------+
>|  1 | Chars | Smith  | NewYork | 2005-01-02 |    1 |
>|  2 | Tom   | Garcia | NewYork | 2005-02-01 |    6 |
>|  3 | Crabb | Ford   | NewYork | 2004-11-19 |    7 |
>+----+-------+--------+---------+------------+------+
>3 rows in set (0.00 sec)
> ```

### Step 6. Clean up the tables

```
DROP TABLE students;
DROP TABLE students1;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.52 sec)
> ```

## Module 3: Hash and Key Partitioned Tables

- In this exercise, we will create a Hash partitioned table and a Key partitioned table. After inserting the same data into both tables, we will compare the distribution of the data.

### Step 1. Create a new Hash partitioned table employees1

```
CREATE TABLE employees1(
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '1970-01-01',
   job_code INT
)
PARTITION BY HASH(YEAR(hired))
PARTITIONS 5;
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.54 sec)
> ```

### Step 2. Insert some data into employees1
```
INSERT INTO employees1 VALUES(1, 'Jim', 'Lee', '2023-04-04', 2);
INSERT INTO employees1 VALUES(2, 'Tim', 'Anderson', '2023-05-10', 2);
INSERT INTO employees1 VALUES(3, 'Mike', 'Jackson', '2023-05-10', 3);
INSERT INTO employees1 VALUES(4, 'Benjamin', 'Thompson', '2023-09-24', 1);
INSERT INTO employees1 VALUES(5, 'Charlotte', 'Clark', '2023-09-24', 1);
```
> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```

- We found that all rows inserted have same YEAR(hired) value. Let's check the distribution of the data.

```
ANALYZE TABLE employees1;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='employees1'
ORDER BY partition_name;
```

> **sample output**
> ```
>+------------+----------------+-----------------------+------------+
>| table_name | partition_name | partition_description | table_rows |
>+------------+----------------+-----------------------+------------+
>| employees1 | p0             |                       |          0 |
>| employees1 | p1             |                       |          0 |
>| employees1 | p2             |                       |          0 |
>| employees1 | p3             |                       |          5 |
>| employees1 | p4             |                       |          0 |
>+------------+----------------+-----------------------+------------+
>5 rows in set (0.01 sec)
> ```

- You can see that all rows are stored in partition p3. Why? Let's now truncate the table employees1 and insert 5 new rows again.

```
TRUNCATE TABLE employees1;
INSERT INTO employees1 VALUES(1, 'Jim', 'Lee', '2020-04-04', 2);
INSERT INTO employees1 VALUES(2, 'Tim', 'Anderson', '2021-05-10', 2);
INSERT INTO employees1 VALUES(3, 'Mike', 'Jackson', '2022-05-10', 3);
INSERT INTO employees1 VALUES(4, 'Benjamin', 'Thompson', '2023-09-24', 1);
INSERT INTO employees1 VALUES(5, 'Charlotte', 'Clark', '2024-09-24', 1);
```

> **sample output**
> ```
>Query OK, 1 row affected (0.01 sec)
> ```

- This time all rows inserted have different YEAR(hired) value. Let's check the distribution of the data.

```
ANALYZE TABLE employees1;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='employees1'
ORDER BY partition_name;
```

> **sample output**
> ```
>+------------+----------------+-----------------------+------------+
>| table_name | partition_name | partition_description | table_rows |
>+------------+----------------+-----------------------+------------+
>| employees1 | p0             |                       |          1 |
>| employees1 | p1             |                       |          1 |
>| employees1 | p2             |                       |          1 |
>| employees1 | p3             |                       |          1 |
>| employees1 | p4             |                       |          1 |
>+------------+----------------+-----------------------+------------+
>5 rows in set (0.01 sec)
> ```
- This time there is one row each partition.
### Step 3. Create a new Key partitioned table employees2

```
CREATE TABLE employees2(
   id INT NOT NULL,
   fname VARCHAR(30),
   lname VARCHAR(30),
   hired DATE NOT NULL DEFAULT '1970-01-01',
   job_code INT
)
PARTITION BY KEY(hired)
PARTITIONS 5;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.54 sec)
> ```

### Step 4. Insert some data into employees2
```
INSERT INTO employees2 VALUES(1, 'Jim', 'Lee', '2023-04-04', 2);
INSERT INTO employees2 VALUES(2, 'Tim', 'Anderson', '2023-05-10', 2);
INSERT INTO employees2 VALUES(3, 'Mike', 'Jackson', '2023-05-10', 3);
INSERT INTO employees2 VALUES(4, 'Benjamin', 'Thompson', '2023-09-24', 1);
INSERT INTO employees2 VALUES(5, 'Charlotte', 'Clark', '2023-09-24', 1);
```
> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```
- All rows inserted into employees2 have different hired value, but the VALUES of YEAR(hired) are same. Let's check the distribution of the data.

```
ANALYZE TABLE employees2;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='employees2'
ORDER BY partition_name;
```

> **sample output**
> ```
>+------------+----------------+-----------------------+------------+
>| table_name | partition_name | partition_description | table_rows |
>+------------+----------------+-----------------------+------------+
>| employees2 | p0             |                       |          2 |
>| employees2 | p1             |                       |          2 |
>| employees2 | p2             |                       |          0 |
>| employees2 | p3             |                       |          1 |
>| employees2 | p4             |                       |          0 |
>+------------+----------------+-----------------------+------------+
>5 rows in set (0.02 sec)
> ```

- The rows are distribute in 3 partitions, even the values of YEAR(hired) are same. Let's truncate the table employees2 and insert 5 new rows again

```
TRUNCATE TABLE employees2;
INSERT INTO employees2 VALUES(1, 'Jim', 'Lee', '2021-05-10', 2);
INSERT INTO employees2 VALUES(2, 'Tim', 'Anderson', '2021-05-10', 2);
INSERT INTO employees2 VALUES(3, 'Mike', 'Jackson', '2021-05-10', 3);
INSERT INTO employees2 VALUES(4, 'Benjamin', 'Thompson', '2021-05-10', 1);
INSERT INTO employees2 VALUES(5, 'Charlotte', 'Clark', '2021-05-10', 1);
```

> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```

- All rows inserted into employees2 have the same hired value. Let's check the distribution of the data.


> **sample output**
> ```
>+------------+----------------+-----------------------+------------+
>| table_name | partition_name | partition_description | table_rows |
>+------------+----------------+-----------------------+------------+
>| employees2 | p0             |                       |          0 |
>| employees2 | p1             |                       |          0 |
>| employees2 | p2             |                       |          0 |
>| employees2 | p3             |                       |          0 |
>| employees2 | p4             |                       |          5 |
>+------------+----------------+-----------------------+------------+
>5 rows in set (0.01 sec)
> ```

- All rows are stored in partition p4. Why? Let's truncate the table employees2 and insert 5 new rows again.

```
TRUNCATE TABLE employees2;
INSERT INTO employees2 VALUES(1, 'Jim', 'Lee', '2020-04-04', 2);
INSERT INTO employees2 VALUES(2, 'Tim', 'Anderson', '2021-05-10', 2);
INSERT INTO employees2 VALUES(3, 'Mike', 'Jackson', '2022-05-10', 3);
INSERT INTO employees2 VALUES(4, 'Benjamin', 'Thompson', '2023-09-24', 1);
INSERT INTO employees2 VALUES(5, 'Charlotte', 'Clark', '2024-09-24', 1);
```

> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```

- All rows inserted into employees2 have different hired value. Let's check the distribution of the data.

```
ANALYZE TABLE employees2;
SELECT table_name, partition_name, partition_description, table_rows FROM information_schema.partitions WHERE table_schema = 'test' AND table_name='employees2' ORDER BY partition_name;
```
> **sample output**
> ```
>+------------+----------------+-----------------------+------------+
>| table_name | partition_name | partition_description | table_rows |
>+------------+----------------+-----------------------+------------+
>| employees2 | p0             |                       |          1 |
>| employees2 | p1             |                       |          2 |
>| employees2 | p2             |                       |          1 |
>| employees2 | p3             |                       |          0 |
>| employees2 | p4             |                       |          1 |
>+------------+----------------+-----------------------+------------+
>5 rows in set (0.01 sec)
> ```

### Step 5. Clean up the tables

```
DROP TABLE employees1;
DROP TABLE employees2;
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.53 sec)
> ```

## Module 4: Performance Comparison
- In this exercise, we evaluate the effect of partitioned tables on performance.
    - Partition an existing table by using LIST COLUMNS partitioning.
    - Execute queries against both the original and partitioned tables to compare performance.

### Step 1. Created a non-partitioned table orders_no_partition

```
CREATE TABLE orders_no_partition(
   ID int(11) NOT NULL AUTO_INCREMENT,
   Name char(35) NOT NULL DEFAULT '',
   StoreCode char(3) NOT NULL DEFAULT '',
   City char(35) NOT NULL DEFAULT '',
   Mount int(11) NOT NULL DEFAULT '0',
   Type int(11) NOT NULL DEFAULT '0',
   PRIMARY KEY(ID),
   Key idx_StoreCode(StoreCode)
);
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.14 sec)
> ```

```
EXIT
```
```
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 1);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 2);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 3);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 4);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 5);"; done;
for i in `seq 9`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO  test.orders_no_partition SELECT NULL, Name, StoreCode, City,Mount, Type FROM test.orders_no_partition;"; done;
```

> **sample output**
> ```
>Qmysql: [Warning] Using a password on the command line interface can be insecure.
>... ...
>mysql: [Warning] Using a password on the command line interface can be insecure.
> ```

### Step 2. Create a LIST COLUMNS partition table orders_partition

```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

```
use test;
```

```
CREATE TABLE orders_partition(
   ID int(11) NOT NULL AUTO_INCREMENT,
   Name char(35) NOT NULL DEFAULT '',
   StoreCode char(3) NOT NULL DEFAULT '',
   City char(35) NOT NULL DEFAULT '',
   Mount int(11) NOT NULL DEFAULT '0',
   Type int(11) NOT NULL DEFAULT '0',
   PRIMARY KEY(ID, Type),
   Key idx_StoreCode(StoreCode)
   )
   PARTITION BY LIST COLUMNS (Type)(
      PARTITION p0 VALUES IN (1),
      PARTITION p1 VALUES IN (2),
      PARTITION p2 VALUES IN (3),
      PARTITION p3 VALUES IN (4),
      PARTITION p4 VALUES IN (5)
);
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.15 sec)
> ```

```
EXIT
```
```
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 1);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 2);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 3);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 4);"; done;
for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 5);"; done;
for i in `seq 9`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "INSERT INTO test.orders_partition SELECT NULL, Name, StoreCode, City,Mount, Type from test.orders_partition;"; done;
```

> **sample output**
> ```
>for i in `seq 1000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP}  -e "INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 1);"; done;
>mysql: [Warning] Using a password on the command line interface can be insecure.
>... ...
>mysql: [Warning] Using a password on the command line interface can be insecure.
> ```

### Step 3. Insert data into the orders_no_partition and orders_partition

```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

```
use test;
```

> **sample output**
> ```
>Database changed
> ```

```
INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
INSERT INTO test.orders_no_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
```

> **sample output**
> ```
>tidb>INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
>ERROR 1526 (HY000): Table has no partition for value from column_list
> ```

- The reason for the error ERROR 1526 (HY000): Table has no partition for value from column_list is that the table orders_partition does not have a partition for Type 6, so a partition needs to be added.

```
ALTER TABLE test.orders_partition ADD PARTITION (PARTITION p5 VALUES IN (6));
INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
INSERT INTO test.orders_partition VALUES(NULL,SUBSTRING(MD5(RAND()),1,20), SUBSTRING(MD5(RAND()),1,3), SUBSTRING(MD5(RAND()),1,20), FLOOR(RAND()*10000), 6);
```

> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```

### Step 4. Check the status of the orders_no_partition table
- What value does the Create_options column in the output contain?

```
SHOW TABLE STATUS LIKE 'orders_no_partition'\G
```
> **sample output**
> ```
>tidb>SHOW TABLE STATUS LIKE 'orders_no_partition'\G
>*************************** 1. row ***************************
>           Name: orders_no_partition
>         Engine: InnoDB
>        Version: 10
>     Row_format: Compact
>           Rows: 2560000
> Avg_row_length: 70
>    Data_length: 179200000
>Max_data_length: 0
>   Index_length: 10240000
>      Data_free: 0
> Auto_increment: 2560002
>    Create_time: 2023-11-16 15:10:53
>    Update_time: NULL
>     Check_time: NULL
>      Collation: utf8mb4_bin
>       Checksum: 
> Create_options: 
>        Comment: 
>1 row in set (0.01 sec)
> ```

- Answer: The Create_options column in the SHOW TABLE STATUS output for the orders_no_partition table is empty.

### Step 5. Check the status of the orders_partition table
- What value does the Create_options column in the output contain?

```
SHOW TABLE STATUS LIKE 'orders_partition'\G
```

> **sample output**
> ```
>tidb>SHOW TABLE STATUS LIKE 'orders_partition'\G
>*************************** 1. row ***************************
>           Name: orders_partition
>         Engine: InnoDB
>        Version: 10
>     Row_format: Compact
>           Rows: 2560002
> Avg_row_length: 70
>    Data_length: 179200140
>Max_data_length: 0
>   Index_length: 51200040
>      Data_free: 0
> Auto_increment: 2560004
>    Create_time: 2023-11-16 15:48:21
>    Update_time: NULL
>     Check_time: NULL
>      Collation: utf8mb4_bin
>       Checksum: 
> Create_options: partitioned
>        Comment: 
>1 row in set (0.01 sec)
> ```

- Answer: The Create_options column in the SHOW TABLE STATUS output for the orders_partition table is partitioned.

### Step 6. Compare the CREATE TABLE statements of the two tables

```
SHOW CREATE TABLE orders_no_partition\G
SHOW CREATE TABLE orders_partition\G
```

> **sample output**
> ```
>*************************** 1. row ***************************
>      Table: orders_no_partition
>Create Table: CREATE TABLE `orders_no_partition` (
>`ID` int(11) NOT NULL AUTO_INCREMENT,
>`Name` char(35) NOT NULL DEFAULT '',
>`StoreCode` char(3) NOT NULL DEFAULT '',
>`City` char(35) NOT NULL DEFAULT '',
>`Mount` int(11) NOT NULL DEFAULT '0',
>`Type` int(11) NOT NULL DEFAULT '0',
>PRIMARY KEY (`ID`) /*T![clustered_index] CLUSTERED */,
>KEY `idx_StoreCode` (`StoreCode`)
>) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin AUTO_INCREMENT=3182134
>1 row in set (0.01 sec)
>
>*************************** 1. row ***************************
>      Table: orders_partition
>Create Table: CREATE TABLE `orders_partition` (
>`ID` int(11) NOT NULL AUTO_INCREMENT,
>`Name` char(35) NOT NULL DEFAULT '',
>`StoreCode` char(3) NOT NULL DEFAULT '',
>`City` char(35) NOT NULL DEFAULT '',
>`Mount` int(11) NOT NULL DEFAULT '0',
>`Type` int(11) NOT NULL DEFAULT '0',
>PRIMARY KEY (`ID`,`Type`) /*T![clustered_index] CLUSTERED */,
>KEY `idx_StoreCode` (`StoreCode`)
>) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin AUTO_INCREMENT=2677458
>PARTITION BY LIST COLUMNS(`Type`)
>(PARTITION `p0` VALUES IN (1),
>PARTITION `p1` VALUES IN (2),
>PARTITION `p2` VALUES IN (3),
>PARTITION `p3` VALUES IN (4),
>PARTITION `p4` VALUES IN (5),
>PARTITION `p5` VALUES IN (6))
>1 row in set (0.00 sec)
> ```

- Answer: The orders_partition table uses LIST COLUMNS partitioning on the value of the Type column. To support partitioning on the Type column, the Type column must be part of the primary key.

### Step 7. Check the number of rows in each partition for orders_partition
- Query the PARTITIONS table in the INFORMATION_SCHEMA database to retrieve the number of rows and the data length for each partition in the orders_partition` table.
```
ANALYZE TABLE orders_partition;
SELECT PARTITION_NAME, TABLE_ROWS, DATA_LENGTH FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_NAME='orders_partition';
```
> **sample output**
> ```
>+----------------+------------+-------------+
>| PARTITION_NAME | TABLE_ROWS | DATA_LENGTH |
>+----------------+------------+-------------+
>| p0             |     512000 |    35840000 |
>| p1             |     512000 |    35840000 |
>| p2             |     512000 |    35840000 |
>| p3             |     512000 |    35840000 |
>| p4             |     512000 |    35840000 |
>| p5             |          2 |         140 |
>+----------------+------------+-------------+
>6 rows in set (0.01 sec)
> ```

### Step 8. Compare the performance
```
SELECT * FROM test.orders_no_partition WHERE Type = 6;
SELECT * FROM test.orders_partition WHERE Type = 6;
```

> **sample output**
> ```
>+---------+----------------------+-----------+----------------------+-------+------+
>| ID      | Name                 | StoreCode | City                 | Mount | Type |
>+---------+----------------------+-----------+----------------------+-------+------+
>| 2560001 | 71add0d7809cbcbcbe70 | 5a7       | ed366dcc516557dc9dd1 |  9718 |    6 |
>| 2560002 | 729981de40b9d6518610 | ce2       | ec29e10a4e200a85f39d |  2252 |    6 |
>+---------+----------------------+-----------+----------------------+-------+------+
>2 rows in set (0.74 sec)
>
>+---------+----------------------+-----------+----------------------+-------+------+
>| ID      | Name                 | StoreCode | City                 | Mount | Type |
>+---------+----------------------+-----------+----------------------+-------+------+
>| 2560002 | 74daf6785e90d21e49fa | d0d       | 7264824fb92dff278fc8 |  6453 |    6 |
>| 2560003 | 34558dcfaf8c36730107 | d79       | 712aa60533e92f925232 |    14 |    6 |
>+---------+----------------------+-----------+----------------------+-------+------+
>2 rows in set (0.01 sec)
> ```

- Note：If execute the queries again, you will found there are no different between them in performance, because the data that we just selected is cached

> **sample output**
> ```
>tidb>SELECT * FROM test.orders_no_partition WHERE Type = 6;
>+---------+----------------------+-----------+----------------------+-------+------+
>| ID      | Name                 | StoreCode | City                 | Mount | Type |
>+---------+----------------------+-----------+----------------------+-------+------+
>| 2560001 | 71add0d7809cbcbcbe70 | 5a7       | ed366dcc516557dc9dd1 |  9718 |    6 |
>| 2560002 | 729981de40b9d6518610 | ce2       | ec29e10a4e200a85f39d |  2252 |    6 |
>+---------+----------------------+-----------+----------------------+-------+------+
>1 row in set (0.00 sec)
>
>tidb>SELECT * FROM test.orders_partition WHERE Type = 6;
>+---------+----------------------+-----------+----------------------+-------+------+
>| ID      | Name                 | StoreCode | City                 | Mount | Type |
>+---------+----------------------+-----------+----------------------+-------+------+
>| 2560002 | 74daf6785e90d21e49fa | d0d       | 7264824fb92dff278fc8 |  6453 |    6 |
>| 2560003 | 34558dcfaf8c36730107 | d79       | 712aa60533e92f925232 |    14 |    6 |
>+---------+----------------------+-----------+----------------------+-------+------+
>2 rows in set (0.00 sec)
> ```

- Based on your results, what is the effect of partitioning on query performance?
    - Answer: For queries that use data from a single partition, the performance is better than accessing the same data from a non-partitioned table. But,for queries that must access all partitions to retrieve data, the performance is often worse than accessing the same data from a non-partitioned table    .

### Step 9. Drop the orders_no_partition and orders_partition tables
```
DROP TABLE test.orders_no_partition;
DROP TABLE test.orders_partition;
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.34 sec)
> ```
====================================================================================================================
# TiDB SQL Tuning Lab 3: Query Optimizer
## Module 1: Data Preparation
- In this exercise, you will import more experimental data into TiDB database for the subsequent experiments. We will create a database called universe.

### Step 1. Log in to the remote Linux
### Step 2. Connect to TiDB and use test database
```
cd universe/
```

```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

```
use test;
```

> **sample output**
> ```
>Database changed
> ```

### Step 3. Import the data
NOTE : Script universeV2.sql
```
/* 
        Sample schema for planets and stars in the universe. No warranty of any data correctness. It should not be used in any circumstances outside PingCAP training courses. 
        Data source: Space research open data set all over the world.
        Author: guanglei.bao@pingcap.com
        Note: The initial(this) size is small, so I turn on the autocommit.
*/

SET AUTOCOMMIT=1;

DROP DATABASE IF EXISTS `universe`;
CREATE DATABASE `universe` DEFAULT CHARACTER SET utf8mb4;

USE `universe`;

DROP TABLE IF EXISTS `stars`;
CREATE TABLE `stars` (
  `id` bigint AUTO_RANDOM,
  `name` char(20) NOT NULL DEFAULT '',
  `mass` float NOT NULL DEFAULT 0.0 COMMENT '10**24 kg',
        `density` int NOT NULL DEFAULT 0 COMMENT 'kg/m**3',
        `gravity` decimal(20,4) NOT NULL DEFAULT 0.0 COMMENT 'm/s**2',
        `escape_velocity` decimal(8,1) DEFAULT NULL COMMENT 'km/s',
        `mass_conversion_rate` int DEFAULT NULL COMMENT '10**6 kg/s',
        `spectral_type` char(8) NOT NULL DEFAULT '',
        `distance_from_earth` float COMMENT 'light year',
        `discover_date` datetime DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`)
);

DROP TABLE IF EXISTS `planet_categories`;
CREATE TABLE `planet_categories` (
        `id` int AUTO_INCREMENT,
        `name` char(20) NOT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`)
);

DROP TABLE IF EXISTS `planets`;
CREATE TABLE `planets` (
  `id` bigint AUTO_RANDOM,
  `name` char(20) NOT NULL DEFAULT '',
  `mass` float NOT NULL DEFAULT 0.0 COMMENT '10**24 kg',
        `diameter` decimal(12,2) NOT NULL DEFAULT 0 COMMENT 'km',
  `density` int NOT NULL DEFAULT 0 COMMENT 'kg/m**3',
  `gravity` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'm/s**2',
        `escape_velocity` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'km/s',
        `rotation_period` decimal(5,1) NOT NULL DEFAULT 0.0 COMMENT 'hours',
        `length_of_day` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'hours',
        `distance_from_sun` float NOT NULL DEFAULT 0.0 COMMENT '10**6 km',
        `perihelion` float DEFAULT NULL COMMENT '10**6 km',
        `aphelion` float DEFAULT NULL COMMENT '10**6 km',
        `orbital_period` decimal(12,1) NOT NULL DEFAULT 0.0 COMMENT 'days',
        `orbital_velocity` decimal(12,1) DEFAULT NULL COMMENT 'km/s',
        `orbital_inclination` decimal(8,1) DEFAULT NULL COMMENT 'degrees', 
        `orbital_eccentricity` decimal(7,0) NOT NULL DEFAULT 0.0,
        `obliquity_to_orbit` decimal(10,4) DEFAULT NULL COMMENT 'degrees',
        `mean_temperature` int NOT NULL DEFAULT 0 COMMENT 'C',
        `surface_pressure` float DEFAULT NULL COMMENT 'bars',
        `ring_systems` boolean NOT NULL DEFAULT 0,
        `global_magnetic_field` boolean DEFAULT 0,
        `sun_id` bigint NOT NULL,
        `category_id` int NOT NULL,
        `discover_date` datetime DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`),
        CONSTRAINT `planet_sun_fk` FOREIGN KEY (`sun_id`) REFERENCES `stars` (`id`),
        CONSTRAINT `planet_cat_fk` FOREIGN KEY (`category_id`) REFERENCES `planet_categories` (`id`)
);

DROP TABLE IF EXISTS `moons`;
CREATE TABLE `moons` (
        `id` bigint AUTO_RANDOM,
        `name` char(20) NOT NULL,
        `alias` char(20) NOT NULL,
        `mass` float COMMENT '10**24 kg',
        `diameter` decimal(12,2) NOT NULL DEFAULT 0 COMMENT 'km',
        `planet_id` bigint NOT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`),
        KEY (`alias`)
);

DROP TABLE IF EXISTS `parameterReference`;
CREATE TABLE `parameterReference` (
    `id` int AUTO_INCREMENT,
    `ref_name` varchar(64) COMMENT 'Planetary Parameter Reference Name',
        `ref_link` varchar(256) COMMENT 'Planetary Parameter Reference Link',
    PRIMARY KEY(`id`),
        KEY (`ref_name`)
);

DROP TABLE IF EXISTS `discoveryFacility`;
CREATE TABLE `discoveryFacility` (
    `id` int AUTO_INCREMENT,
    `name` varchar(64) COMMENT 'Discovery Facility',
    PRIMARY KEY(`id`),
        KEY (`name`)
);

CREATE TABLE `planetarySystem` (
    `id` bigint AUTO_RANDOM,
    `pl_name` varchar(32) NOT NULL COMMENT 'Planet Name',
    `hostname` varchar(32) NOT NULL COMMENT 'Host Name',
    `default_flag`  tinyint NOT NULL  DEFAULT 0 COMMENT 'Default Parameter Set',
    `sy_snum`int NOT NULL DEFAULT 0 COMMENT 'Number of Stars',
    `sy_pnum` int NOT NULL DEFAULT 0 COMMENT 'Number of Planets',
    `discoverymethod` varchar(64) COMMENT 'Discovery Method',
    `disc_year` int COMMENT 'Discovery Year',
    `df_id` int NOT NULL COMMENT 'Discovery Facility ID',
    `soltype` varchar(64) COMMENT 'Solution Type',
    `ref_id` int NOT NULL COMMENT 'Planetary Parameter Reference ID',
    `pl_orbper` float DEFAULT NULL COMMENT 'Orbital Period [days]',
    `pl_bmasse` float DEFAULT NULL COMMENT 'Planet Mass or Mass*sin(i) [Earth Mass]', 
    `st_teff` float DEFAULT NULL COMMENT 'Stellar Effective Temperature [K]',
    `sy_dist` float DEFAULT NULL COMMENT 'Distance [pc]',
    `sy_gaiamag` float DEFAULT NULL COMMENT 'Gaia Magnitude',
    `rowupdate` datetime DEFAULT NULL COMMENT 'Planetary Parameter Reference Publication Date',
    `releasedate` datetime DEFAULT NULL COMMENT 'Release Date',
    PRIMARY KEY(`id`)
);

INSERT INTO `stars` (
        name, mass, density, gravity, escape_velocity, mass_conversion_rate, spectral_type, distance_from_earth, discover_date
) VALUES
('Sun', 1988500, 1408, 274.0, 617.6, 4260, 'G2 V', 0.0000158, NULL);
SET @SUN_ID=LAST_INSERT_ID();
INSERT INTO `stars` (
        name, mass, density, gravity, escape_velocity, mass_conversion_rate, spectral_type, distance_from_earth, discover_date
) VALUES
('Proxima Centauri', 244600, 56800, 0.052, NULL, NULL, 'M5.5 Ve', 4.426, '1915-01-01');
SET @PC_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Terrestrial');
SET @CAT_TER_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Jovian');
SET @CAT_JOV_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Dwarf');
SET @CAT_DWA_ID=LAST_INSERT_ID();

INSERT INTO `planets` (
        name, 
        mass, 
        diameter, 
        density, 
        gravity, 
        escape_velocity, 
        rotation_period, 
        length_of_day,
        distance_from_sun,
        perihelion,
        aphelion,
        orbital_period,
        orbital_velocity,
        orbital_inclination,
        orbital_eccentricity,
        obliquity_to_orbit,
        mean_temperature,
        surface_pressure,
        ring_systems,
        global_magnetic_field,
        sun_id,
        category_id,
        discover_date) VALUES 
        ('Mercury', 0.33, 4879, 5429, 3.7, 4.3, 1407.6, 4222.6, 57.9, 46.0, 69.8, 88.0, 47.4, 7.0, 0.206, 0.034, 167, 0, 0, 1, @SUN_ID, @CAT_TER_ID, NULL),
        ('Venus', 4.87, 12104, 5234, 8.9, 10.4, -5832.5, 2802, 108.2, 107.5, 108.9, 224.7, 35.0, 3.4, 0.007, 177.4, 464, 92, 0, 0, @SUN_ID, @CAT_TER_ID, NULL),
        ('Earth', 5.97, 12756, 5514, 9.8, 11.2, 23.9, 24.0, 149.6, 147.1, 152.1, 365.2, 29.8, 0, 0.017, 23.4, 15, 1, 0, 1, @SUN_ID, @CAT_TER_ID, NULL),
        ('Mars', 0.642, 6792, 3934, 3.7, 5.0, 24.6, 24.7, 228.0, 206.7, 249.3, 687.0, 24.1, 1.8, 0.094, 25.2, -65, 0.01, 0, 0, @SUN_ID, @CAT_TER_ID, NULL),
        ('Jupiter', 1898, 142984, 1326, 23.1, 59.5, 9.9, 9.9, 778.5, 740.6, 816.4, 4331, 13.1, 1.3, 0.049, 3.1, -110, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Saturn', 568, 120536, 687, 9.0, 35.5, 10.7, 10.7, 1432.0, 1357.6, 1506.5, 10747, 9.7, 2.5, 0.052, 26.7, -140, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Uranus', 86.8, 51118, 1270, 8.7, 21.3, -17.2, 17.2, 2867.0, 2732.7, 3001.4, 30589, 6.8, 0.8, 0.047, 97.8, -195, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Neptune', 102, 49528, 1638, 11.0, 23.5, 16.1, 16.1, 4515.0, 4471.1, 4558.9, 59800, 5.4, 1.8, 0.010, 28.3, -200, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Pluto', 0.0130, 2376, 1850, 0.7, 1.3, -153.3, 153.3, 5906.4, 4436.8, 7375.9, 90560, 4.7, 17.2, 0.244, 122.5, -225, 0.00001, 0, null, @SUN_ID, @CAT_DWA_ID, NULL),
        ('Proxima Centauri b', 7.5819, 16582.8, 5970, 11.32194, 9.3, 267.68, 267.7, 2992, NULL, NULL, 11.21, NULL, NULL, 0.1, NULL, -57.15, NULL, 0, NULL, @PC_ID, @CAT_TER_ID, '2016-08-24')
;

SET @P_ID=(select id from planets where name='Earth');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Moon', 'Lunar', 0.07342, 3474.2, @P_ID);

SET @P_ID=(select id from planets where name='Jupiter');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Io', 'Jupiter 1', 0.089319, 3642, @P_ID),
('Europa', 'Jupiter 2', 0.048, 3138, @P_ID),
('Ganymede', 'Jupiter 3', NULL, 5262, @P_ID),
('Callisto', 'Jupiter 4', 0.108, 4800, @P_ID);

SET @P_ID=(select id from planets where name='Saturn');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Mimas', 'Saturn 1', 0.000384, 397.2, @P_ID),
('Enceladus', 'Saturn 2', 0.011, 505, @P_ID),
('Tethys', 'Saturn 3', 0.000617449, 1062, @P_ID),
('Dione', 'Saturn 4', 0.001096, 1118, @P_ID),
('Rhea', 'Saturn 5', 0.002306518, 1528, @P_ID),
('Titan', 'Saturn 6', 0.1345, 4828, @P_ID),
('Hyperion', 'Saturn 7', 0.0000056119, 270, @P_ID),
('Iapetus', 'Saturn 8', 0.001805635, 1469, @P_ID);

SET @P_ID=(select id from planets where name='Uranus');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Ariel', 'Uranus 1', 0.001251, 1157.8, @P_ID),
('Umbriel', 'Uranus 2', 0.001275, 1169.4, @P_ID),
('Titania', 'Uranus 3', 0.0034, 1576.8, @P_ID),
('Oberon', 'Uranus 4', 0.003076, 1522.8, @P_ID),
('Miranda', 'Uranus 5', 0.0000659, 473, @P_ID);

SET @P_ID=(select id from planets where name='Neptune');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Triton', 'Neptune 1', 0.02139, 2706.8, @P_ID),
('Nereid', 'Neptune 2', 0.00002, 340, @P_ID),
('Naiad', 'Neptune 3', 0.00000019, 58, @P_ID),
('Thalassa', 'Neptune 4', 0.00000037, 82, @P_ID),
('Despina', 'Neptune 5', 0.0000008, 150, @P_ID),
('Galatea', 'Neptune 6', 0.00000212, 184, @P_ID),
('Larissa', 'Neptune 7', NULL, 193, @P_ID),
('Proteus', 'Neptune 8', NULL, 416, @P_ID);

SET @P_ID=(select id from planets where name='Pluto');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Charon', 'Pluto 1', 0.001586, 1212, @P_ID),
('Nix', 'Pluto 2', 0.000000045, 49.8, @P_ID);

LOAD DATA LOCAL INFILE './discoveryFacility.csv' INTO TABLE `discoveryFacility` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (`id`,`name`);

LOAD DATA LOCAL INFILE './parameterReference.csv' INTO TABLE `parameterReference` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (`id`, `ref_name`, `ref_link`);

LOAD DATA LOCAL INFILE './planetarySystem.csv' INTO TABLE `planetarySystem` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (pl_name, hostname, default_flag, sy_snum, sy_pnum, discoverymethod, disc_year, df_id, soltype, ref_id, pl_orbper, pl_bmasse, st_teff, sy_dist, sy_gaiamag, rowupdate, releasedate);
```
```
source universeV2.sql
```

> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (2.03 sec)
> ```

### Step 4. Make universe the current database

```
use universe;
```
> **sample output**
> ```
>Database changed
> ```

### Step 5. Check the imported data
```
SHOW TABLES;
SELECT COUNT(*) FROM stars;
SELECT COUNT(*) FROM planets;
SELECT COUNT(*) FROM moons;
SELECT COUNT(*) FROM planet_categories;
SELECT COUNT(*) FROM discoveryFacility;
SELECT COUNT(*) FROM parameterReference;
SELECT COUNT(*) FROM planetarySystem;
```

> **sample output**
> ```
>tidb>show tables;
>+--------------------+
>| Tables_in_universe |
>+--------------------+
>| discoveryFacility  |
>| moons              |
>| parameterReference |
>| planet_categories  |
>| planetarySystem    |
>| planets            |
>| stars              |
>+--------------------+
>7 rows in set (0.00 sec)
>
>tidb>select count(*) from stars;
>+----------+
>| count(*) |
>+----------+
>|        2 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planets;
>+----------+
>| count(*) |
>+----------+
>|       10 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from moons;
>+----------+
>| count(*) |
>+----------+
>|       28 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planet_categories;
>+----------+
>| count(*) |
>+----------+
>|        3 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from discoveryFacility;
>+----------+
>| count(*) |
>+----------+
>|       70 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from parameterReference;
>+----------+
>| count(*) |
>+----------+
>|     2055 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planetarySystem;
>+----------+
>| count(*) |
>+----------+
>|    34713 |
>+----------+
>1 row in set (0.02 sec)
> ```

### Module 2: Using EXPLAIN Command
- In this exercise, you will you examine the TiDB optimizer's execution plan for several different SELECT queries by using the EXPLAIN command.

### Step 1. Execute the following statement to view the EXPLAIN output

> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM parameterReference WHERE id=100;
>+-------------+---------+------+--------------------------+---------------+
>| id          | estRows | task | access object            | operator info |
>+-------------+---------+------+--------------------------+---------------+
>| Point_Get_1 | 1.00    | root | table:parameterReference | handle:100    |
>+-------------+---------+------+--------------------------+---------------+
>1 row in set (0.00 sec)
> ```

- What is the access method for this query?
    - Answer: Point_Get
- How many rows must TiDB read to perform this query?
    - Answer: 1
- Does this access strategy result in good performance? Why?
    - Answer: Yes. The Point_Get type applies when you compare all parts of a PRIMARY KEY or UNIQUE Index to constant values. In this example, the table has at the most one matching row. The optimizer treats this row's column values as constants and reads them only once. In TiDB, Point_Get does not require compiler to generate an execution plan. As data is stored in TiKV in key-value format, we can directly retrieve data using Point_Get when the WHERE condition is a primary key or unique index.

### Step 2. Execute the following statement to view the EXPLAIN output

```
EXPLAIN SELECT * FROM parameterReference WHERE id=100+5;
```

> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM parameterReference WHERE id=100+5;
>+-------------+---------+------+--------------------------+---------------+
>| id          | estRows | task | access object            | operator info |
>+-------------+---------+------+--------------------------+---------------+
>| Point_Get_5 | 1.00    | root | table:parameterReference | handle:105    |
>+-------------+---------+------+--------------------------+---------------+
>1 row in set (0.00 sec)
> ```
- Is there any difference in the execution plan for this query, when compared to the query that you looked at in step 1? Why?
    - Answer: No. The optimizer evaluates the expression 100+5 before executing the query.

### Step 3. Execute the following statement to view the EXPLAIN output
```
EXPLAIN SELECT * FROM parameterReference WHERE id-5=100;
```

> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM parameterReference WHERE id-5=100;
>+-------------------------+---------+-----------+--------------------------+---------------------------------------------------+
>| id                      | estRows | task      | access object            | operator info                                     |
>+-------------------------+---------+-----------+--------------------------+---------------------------------------------------+
>| TableReader_7           | 1644.00 | root      |                          | data:Selection_6                                  |
>| └─Selection_6           | 1644.00 | cop[tikv] |                          | eq(minus(universe.parameterreference.id, 5), 100) |
>|   └─TableFullScan_5     | 2055.00 | cop[tikv] | table:parameterReference | keep order:false                                  |
>+-------------------------+---------+-----------+--------------------------+---------------------------------------------------+
>3 rows in set (0.00 sec)
> ```

- Is there any difference in the execution plan for this query, when compared to the query that you looked at in step 1 and 2? Why?
    - Answer: Yes. The optimizer cannot reduce the expression ID-5=100 to a single value before executing the query. Thus, it must scan the full table (type: TableFullScan). The number of rows is an estimate based on statistics and may not equal to the exact number of rows in the parameterReference table.

### Step 4. Display the optimizer cost for the query in JSON format

```
CREATE INDEX idx_host_name_ref ON planetarySystem(hostname, pl_name, ref_id);
EXPLAIN FORMAT = "tidb_json" SELECT pl_name, sy_snum, sy_pnum FROM planetarySystem WHERE pl_name='11 Com b' AND hostname='11 Com b' AND ref_id=71\G
DROP INDEX idx_host_name_ref on planetarySystem;
```
> **sample output**
> ```
>*************************** 1. row ***************************
>TiDB_JSON: [
>   {
>      "id": "Projection_4",
>      "estRows": "1.00",
>      "taskType": "root",
>      "operatorInfo": "universe.planetarysystem.pl_name, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum",
>      "subOperators": [
>            {
>               "id": "IndexLookUp_10",
>               "estRows": "1.00",
>               "taskType": "root",
>               "subOperators": [
>                  {
>                        "id": "IndexRangeScan_8(Build)",
>                        "estRows": "1.00",
>                        "taskType": "cop[tikv]",
>                        "accessObject": "table:planetarySystem, index:idx_host_name_ref(hostname, pl_name, ref_id)",
>                        "operatorInfo": "range:[\"11 Com b\" \"11 Com b\" 71,\"11 Com b\" \"11 Com b\" 71], keep order:false"
>                  },
>                  {
>                        "id": "TableRowIDScan_9(Probe)",
>                        "estRows": "1.00",
>                        "taskType": "cop[tikv]",
>                        "accessObject": "table:planetarySystem",
>                        "operatorInfo": "keep order:false"
>                  }
>               ]
>            }
>      ]
>   }
>]
>
>1 row in set (0.01 sec)
> ```

- What cost did the optimizer calculate for this query?
    - Answer: 1.0 (the value of the "estRows" field in the JSON output. The cost info JSON object displays a more detailed breakdown of the cost.)
- Which columns must TiDB access to resolve the query?
    - Answer: The universe.planetarysystem.pl_name, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum columns (the elements in the JSON "operatorInfo" array field)
- Which operations are carried out on TiKV?
    - Answer: The IndexRangeScan and TableRowIDScan are carried out on TiKV (the value of the taskType field in the JSON output is cop[tikv]).
- Which operations are carried out on the TiDB Server?
    - Answer: The IndexLookUp is carried out on TiDB Server (the value of the "taskType" field in the JSON output is "root").

## Module 3: Using EXPLAIN ANALYZE Command

- In this exercise, you will you examine the TiDB optimizer's execution plan for several different SELECT queries by using the EXPLAIN ANALYZE command.

### Step 1. Create tables

- Create a clustered primary key table T1 and non-clustered primary key table T2.

```
DROP TABLE IF EXISTS `T1`;
CREATE TABLE `T1` (
  `id` bigint AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '',
   PRIMARY KEY (`id`),
   KEY (`name`)
);
DROP TABLE IF EXISTS `T2`;
CREATE TABLE `T2` (
  `id` bigint AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '',
   PRIMARY KEY (`id`) NONCLUSTERED,
   KEY (`name`)
);
```


> **sample output**
> ```
>Query OK, 0 rows affected (0.53 sec)
> ```
### Step 2. Insert data into the tables

```
INSERT INTO T1 VALUES(NULL, 'Tom');
INSERT INTO T1 VALUES(NULL, 'Jack');
INSERT INTO T1 VALUES(NULL, 'Frank');
INSERT INTO T2 VALUES(NULL, 'Tom');
INSERT INTO T2 VALUES(NULL, 'Jack');
INSERT INTO T2 VALUES(NULL, 'Frank');
```

> **sample output**
> ```
>Query OK, 1 row affected (0.00 sec)
> ```

### Step 3. View the output of EXPLAIN ANALYZE for table T1

```
SELECT id, name from T1;
EXPLAIN ANALYZE SELECT * FROM T1 WHERE id=1\G
```

> **sample output**
> ```
>+----+-------+
>| id | name  |
>+----+-------+
>|  1 | Tom   |
>|  2 | Jack  |
>|  3 | Frank |
>+----+-------+
>3 rows in set (0.00 sec)
>
>*************************** 1. row ***************************
>            id: Point_Get_1
>       estRows: 1.00
>       actRows: 1
>          task: root
> access object: table:T1
>execution info: time:474.1µs, loops:2, RU:0.273086, Get:{num_rpc:1, total_time:443.5µs}, total_process_time: 67.5µs, total_wait_time: 56µs, tikv_wall_time: 149.5µs, scan_detail: {total_process_keys: 1, total_process_keys_size: 39, total_keys: 1, get_snapshot_time: 18.2µs, rocksdb: {block: {cache_hit_count: 2}}}
> operator info: handle:1
>        memory: N/A
>          disk: N/A
>1 row in set (0.00 sec)
> ```

- What is the access method for this query?
    - Answer: Point_Get
- What is the actual execution time of this query?
    - Answer: time:474.1µs
- How many times the operator(access method) was executed?
    - Answer: loops:2 times

### Step 4. View the output of EXPLAIN ANALYZE for table T1
```
SELECT id, name from T1;
EXPLAIN ANALYZE SELECT * FROM T1 WHERE id=4\G
```

> **sample output**
> ```
>+----+-------+
>| id | name  |
>+----+-------+
>|  1 | Tom   |
>|  2 | Jack  |
>|  3 | Frank |
>+----+-------+
>3 rows in set (0.00 sec)
>
>*************************** 1. row ***************************
>            id: Point_Get_1
>       estRows: 1.00
>       actRows: 0
>          task: root
> access object: table:T1
>execution info: time:473.8µs, loops:1, RU:0.272975, Get:{num_rpc:1, total_time:446.6µs}, total_process_time: 68.9µs, total_wait_time: 51.7µs, tikv_wall_time: 151.9µs, scan_detail: {total_keys: 1, get_snapshot_time: 17.3µs, rocksdb: {block: {cache_hit_count: 2}}}
> operator info: handle:4
>        memory: N/A
>          disk: N/A
>1 row in set (0.00 sec)
> ```

- Is there any difference in the execution plan for this query, when compared to the query that in the previously step? Why?
    - Answer: The actual execution time of this query is 473.8µs, and the times the operator(access method) was executed is just once. This is because once we confirm that there is no row with id=4 in the clustered primary key table, we know that the data we need is not stored in the table. However, in the previous step, even after we find the row with id=3, we still need to confirm again whether there are any rows that meet the condition, even though the primary key is id.

### Step 5. View the output of EXPLAIN ANALYZE for table T2

```
SELECT _tidb_rowid, id, name from T2;
EXPLAIN ANALYZE SELECT * FROM T2 WHERE id=1\G
```

> **sample output**
> ```
>+-------------+----+-------+
>| _tidb_rowid | id | name  |
>+-------------+----+-------+
>|           2 |  1 | Tom   |
>|           4 |  3 | Jack  |
>|           6 |  5 | Frank |
>+-------------+----+-------+
>3 rows in set (0.00 sec)
>
>*************************** 1. row ***************************
>            id: Point_Get_1
>       estRows: 1.00
>       actRows: 1
>          task: root
> access object: table:T2, index:PRIMARY(id)
>execution info: time:592.9µs, loops:2, RU:0.520601, Get:{num_rpc:2, total_time:565.6µs}, total_process_time: 57.8µs, total_wait_time: 53.1µs, tikv_wall_time: 147.1µs, scan_detail: {total_process_keys: 2, total_process_keys_size: 87, total_keys: 2, get_snapshot_time: 12.6µs, rocksdb: {block: {cache_hit_count: 8}}}
> operator info: 
>        memory: N/A
>          disk: N/A
>1 row in set (0.00 sec)
> ```

- Is there any difference in the execution plan for this query, when compared to the query that you looked at in step 3 and 4? Why?
    - Answer: From the Get:{num_rpc:2, total_time:565.6µs} field, we find that the number of the Get RPC requests (num_rpc) sent to TiKV is 2. This is because T2 is a non-clustered primary key table, and we have to use Point_Get to obtain the _tidb_rowid from the primary key id first, and then use the _tidb_rowid to obtain the required row from the table. Therefore, there needs to be two RPC requests.

### Step 6. View the output of EXPLAIN ANALYZE for table T2
```
SELECT _tidb_rowid, id, name from T2;
EXPLAIN ANALYZE SELECT * FROM T2 WHERE _tidb_rowid=2\G
```

> **sample output**
> ```
>+-------------+----+-------+
>| _tidb_rowid | id | name  |
>+-------------+----+-------+
>|           2 |  1 | Tom   |
>|           4 |  3 | Jack  |
>|           6 |  5 | Frank |
>+-------------+----+-------+
>3 rows in set (0.00 sec)
>
>*************************** 1. row ***************************
>            id: Point_Get_1
>       estRows: 1.00
>       actRows: 1
>          task: root
> access object: table:T2
>execution info: time:518.2µs, loops:2, RU:0.274932, Get:{num_rpc:1, total_time:487.5µs}, total_process_time: 72.8µs, total_wait_time: 53.5µs, tikv_wall_time: 158.9µs, scan_detail: {total_process_keys: 1, total_process_keys_size: 43, total_keys: 1, get_snapshot_time: 16.6µs, rocksdb: {block: {cache_hit_count: 4}}}
> operator info: handle:2
>        memory: N/A
>          disk: N/A
>1 row in set (0.00 sec) 
> ```

- Is there any difference in the execution plan for this query, when compared to the query that you looked at in step 5? Why? Answer: From the Get:{num_rpc:1, total_time:487.5µs} field, we can conclude that the number of the Get RPC requests (num_rpc) sent to TiKV is 1. Although T2 table is a non-clustered primary key table, if we use _tidb_rowid directly as the query condition, only one RPC request is needed.

### Step 7. Clean up the table T1 and T2

```
DROP TABLE T1;
DROP TABLE T2;
```

> **sample output**
> ```
Query OK, 0 rows affected (0.53 sec)
> ```

====================================================================================================================

# TiDB SQL Tuning Lab 4: Optimizing Queries
## Module 1: Data Preparation
- In this exercise, you will import more experimental data into TiDB database for the subsequent experiments. We will create a database called universe.

### Step 1. Log in to the remote Linux
### Step 2. Connect to TiDB and use test database
```
cd universe/
```
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
use test;
```

> **sample output**
> ```
> Database changed
> ```

NOTE : Script universeV2.sql 
```
/* 
        Sample schema for planets and stars in the universe. No warranty of any data correctness. It should not be used in any circumstances outside PingCAP training courses. 
        Data source: Space research open data set all over the world.
        Author: guanglei.bao@pingcap.com
        Note: The initial(this) size is small, so I turn on the autocommit.
*/

SET AUTOCOMMIT=1;

DROP DATABASE IF EXISTS `universe`;
CREATE DATABASE `universe` DEFAULT CHARACTER SET utf8mb4;

USE `universe`;

DROP TABLE IF EXISTS `stars`;
CREATE TABLE `stars` (
  `id` bigint AUTO_RANDOM,
  `name` char(20) NOT NULL DEFAULT '',
  `mass` float NOT NULL DEFAULT 0.0 COMMENT '10**24 kg',
        `density` int NOT NULL DEFAULT 0 COMMENT 'kg/m**3',
        `gravity` decimal(20,4) NOT NULL DEFAULT 0.0 COMMENT 'm/s**2',
        `escape_velocity` decimal(8,1) DEFAULT NULL COMMENT 'km/s',
        `mass_conversion_rate` int DEFAULT NULL COMMENT '10**6 kg/s',
        `spectral_type` char(8) NOT NULL DEFAULT '',
        `distance_from_earth` float COMMENT 'light year',
        `discover_date` datetime DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`)
);

DROP TABLE IF EXISTS `planet_categories`;
CREATE TABLE `planet_categories` (
        `id` int AUTO_INCREMENT,
        `name` char(20) NOT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`)
);

DROP TABLE IF EXISTS `planets`;
CREATE TABLE `planets` (
  `id` bigint AUTO_RANDOM,
  `name` char(20) NOT NULL DEFAULT '',
  `mass` float NOT NULL DEFAULT 0.0 COMMENT '10**24 kg',
        `diameter` decimal(12,2) NOT NULL DEFAULT 0 COMMENT 'km',
  `density` int NOT NULL DEFAULT 0 COMMENT 'kg/m**3',
  `gravity` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'm/s**2',
        `escape_velocity` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'km/s',
        `rotation_period` decimal(5,1) NOT NULL DEFAULT 0.0 COMMENT 'hours',
        `length_of_day` decimal(8,1) NOT NULL DEFAULT 0.0 COMMENT 'hours',
        `distance_from_sun` float NOT NULL DEFAULT 0.0 COMMENT '10**6 km',
        `perihelion` float DEFAULT NULL COMMENT '10**6 km',
        `aphelion` float DEFAULT NULL COMMENT '10**6 km',
        `orbital_period` decimal(12,1) NOT NULL DEFAULT 0.0 COMMENT 'days',
        `orbital_velocity` decimal(12,1) DEFAULT NULL COMMENT 'km/s',
        `orbital_inclination` decimal(8,1) DEFAULT NULL COMMENT 'degrees', 
        `orbital_eccentricity` decimal(7,0) NOT NULL DEFAULT 0.0,
        `obliquity_to_orbit` decimal(10,4) DEFAULT NULL COMMENT 'degrees',
        `mean_temperature` int NOT NULL DEFAULT 0 COMMENT 'C',
        `surface_pressure` float DEFAULT NULL COMMENT 'bars',
        `ring_systems` boolean NOT NULL DEFAULT 0,
        `global_magnetic_field` boolean DEFAULT 0,
        `sun_id` bigint NOT NULL,
        `category_id` int NOT NULL,
        `discover_date` datetime DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`),
        CONSTRAINT `planet_sun_fk` FOREIGN KEY (`sun_id`) REFERENCES `stars` (`id`),
        CONSTRAINT `planet_cat_fk` FOREIGN KEY (`category_id`) REFERENCES `planet_categories` (`id`)
);

DROP TABLE IF EXISTS `moons`;
CREATE TABLE `moons` (
        `id` bigint AUTO_RANDOM,
        `name` char(20) NOT NULL,
        `alias` char(20) NOT NULL,
        `mass` float COMMENT '10**24 kg',
        `diameter` decimal(12,2) NOT NULL DEFAULT 0 COMMENT 'km',
        `planet_id` bigint NOT NULL,
        PRIMARY KEY (`id`),
        KEY (`name`),
        KEY (`alias`)
);

DROP TABLE IF EXISTS `parameterReference`;
CREATE TABLE `parameterReference` (
    `id` int AUTO_INCREMENT,
    `ref_name` varchar(64) COMMENT 'Planetary Parameter Reference Name',
        `ref_link` varchar(256) COMMENT 'Planetary Parameter Reference Link',
    PRIMARY KEY(`id`),
        KEY (`ref_name`)
);

DROP TABLE IF EXISTS `discoveryFacility`;
CREATE TABLE `discoveryFacility` (
    `id` int AUTO_INCREMENT,
    `name` varchar(64) COMMENT 'Discovery Facility',
    PRIMARY KEY(`id`),
        KEY (`name`)
);

CREATE TABLE `planetarySystem` (
    `id` bigint AUTO_RANDOM,
    `pl_name` varchar(32) NOT NULL COMMENT 'Planet Name',
    `hostname` varchar(32) NOT NULL COMMENT 'Host Name',
    `default_flag`  tinyint NOT NULL  DEFAULT 0 COMMENT 'Default Parameter Set',
    `sy_snum`int NOT NULL DEFAULT 0 COMMENT 'Number of Stars',
    `sy_pnum` int NOT NULL DEFAULT 0 COMMENT 'Number of Planets',
    `discoverymethod` varchar(64) COMMENT 'Discovery Method',
    `disc_year` int COMMENT 'Discovery Year',
    `df_id` int NOT NULL COMMENT 'Discovery Facility ID',
    `soltype` varchar(64) COMMENT 'Solution Type',
    `ref_id` int NOT NULL COMMENT 'Planetary Parameter Reference ID',
    `pl_orbper` float DEFAULT NULL COMMENT 'Orbital Period [days]',
    `pl_bmasse` float DEFAULT NULL COMMENT 'Planet Mass or Mass*sin(i) [Earth Mass]', 
    `st_teff` float DEFAULT NULL COMMENT 'Stellar Effective Temperature [K]',
    `sy_dist` float DEFAULT NULL COMMENT 'Distance [pc]',
    `sy_gaiamag` float DEFAULT NULL COMMENT 'Gaia Magnitude',
    `rowupdate` datetime DEFAULT NULL COMMENT 'Planetary Parameter Reference Publication Date',
    `releasedate` datetime DEFAULT NULL COMMENT 'Release Date',
    PRIMARY KEY(`id`)
);

INSERT INTO `stars` (
        name, mass, density, gravity, escape_velocity, mass_conversion_rate, spectral_type, distance_from_earth, discover_date
) VALUES
('Sun', 1988500, 1408, 274.0, 617.6, 4260, 'G2 V', 0.0000158, NULL);
SET @SUN_ID=LAST_INSERT_ID();
INSERT INTO `stars` (
        name, mass, density, gravity, escape_velocity, mass_conversion_rate, spectral_type, distance_from_earth, discover_date
) VALUES
('Proxima Centauri', 244600, 56800, 0.052, NULL, NULL, 'M5.5 Ve', 4.426, '1915-01-01');
SET @PC_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Terrestrial');
SET @CAT_TER_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Jovian');
SET @CAT_JOV_ID=LAST_INSERT_ID();

INSERT INTO `planet_categories` (
        name
) VALUES
('Dwarf');
SET @CAT_DWA_ID=LAST_INSERT_ID();

INSERT INTO `planets` (
        name, 
        mass, 
        diameter, 
        density, 
        gravity, 
        escape_velocity, 
        rotation_period, 
        length_of_day,
        distance_from_sun,
        perihelion,
        aphelion,
        orbital_period,
        orbital_velocity,
        orbital_inclination,
        orbital_eccentricity,
        obliquity_to_orbit,
        mean_temperature,
        surface_pressure,
        ring_systems,
        global_magnetic_field,
        sun_id,
        category_id,
        discover_date) VALUES 
        ('Mercury', 0.33, 4879, 5429, 3.7, 4.3, 1407.6, 4222.6, 57.9, 46.0, 69.8, 88.0, 47.4, 7.0, 0.206, 0.034, 167, 0, 0, 1, @SUN_ID, @CAT_TER_ID, NULL),
        ('Venus', 4.87, 12104, 5234, 8.9, 10.4, -5832.5, 2802, 108.2, 107.5, 108.9, 224.7, 35.0, 3.4, 0.007, 177.4, 464, 92, 0, 0, @SUN_ID, @CAT_TER_ID, NULL),
        ('Earth', 5.97, 12756, 5514, 9.8, 11.2, 23.9, 24.0, 149.6, 147.1, 152.1, 365.2, 29.8, 0, 0.017, 23.4, 15, 1, 0, 1, @SUN_ID, @CAT_TER_ID, NULL),
        ('Mars', 0.642, 6792, 3934, 3.7, 5.0, 24.6, 24.7, 228.0, 206.7, 249.3, 687.0, 24.1, 1.8, 0.094, 25.2, -65, 0.01, 0, 0, @SUN_ID, @CAT_TER_ID, NULL),
        ('Jupiter', 1898, 142984, 1326, 23.1, 59.5, 9.9, 9.9, 778.5, 740.6, 816.4, 4331, 13.1, 1.3, 0.049, 3.1, -110, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Saturn', 568, 120536, 687, 9.0, 35.5, 10.7, 10.7, 1432.0, 1357.6, 1506.5, 10747, 9.7, 2.5, 0.052, 26.7, -140, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Uranus', 86.8, 51118, 1270, 8.7, 21.3, -17.2, 17.2, 2867.0, 2732.7, 3001.4, 30589, 6.8, 0.8, 0.047, 97.8, -195, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Neptune', 102, 49528, 1638, 11.0, 23.5, 16.1, 16.1, 4515.0, 4471.1, 4558.9, 59800, 5.4, 1.8, 0.010, 28.3, -200, null, 1, 1, @SUN_ID, @CAT_JOV_ID, NULL),
        ('Pluto', 0.0130, 2376, 1850, 0.7, 1.3, -153.3, 153.3, 5906.4, 4436.8, 7375.9, 90560, 4.7, 17.2, 0.244, 122.5, -225, 0.00001, 0, null, @SUN_ID, @CAT_DWA_ID, NULL),
        ('Proxima Centauri b', 7.5819, 16582.8, 5970, 11.32194, 9.3, 267.68, 267.7, 2992, NULL, NULL, 11.21, NULL, NULL, 0.1, NULL, -57.15, NULL, 0, NULL, @PC_ID, @CAT_TER_ID, '2016-08-24')
;

SET @P_ID=(select id from planets where name='Earth');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Moon', 'Lunar', 0.07342, 3474.2, @P_ID);

SET @P_ID=(select id from planets where name='Jupiter');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Io', 'Jupiter 1', 0.089319, 3642, @P_ID),
('Europa', 'Jupiter 2', 0.048, 3138, @P_ID),
('Ganymede', 'Jupiter 3', NULL, 5262, @P_ID),
('Callisto', 'Jupiter 4', 0.108, 4800, @P_ID);

SET @P_ID=(select id from planets where name='Saturn');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Mimas', 'Saturn 1', 0.000384, 397.2, @P_ID),
('Enceladus', 'Saturn 2', 0.011, 505, @P_ID),
('Tethys', 'Saturn 3', 0.000617449, 1062, @P_ID),
('Dione', 'Saturn 4', 0.001096, 1118, @P_ID),
('Rhea', 'Saturn 5', 0.002306518, 1528, @P_ID),
('Titan', 'Saturn 6', 0.1345, 4828, @P_ID),
('Hyperion', 'Saturn 7', 0.0000056119, 270, @P_ID),
('Iapetus', 'Saturn 8', 0.001805635, 1469, @P_ID);

SET @P_ID=(select id from planets where name='Uranus');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Ariel', 'Uranus 1', 0.001251, 1157.8, @P_ID),
('Umbriel', 'Uranus 2', 0.001275, 1169.4, @P_ID),
('Titania', 'Uranus 3', 0.0034, 1576.8, @P_ID),
('Oberon', 'Uranus 4', 0.003076, 1522.8, @P_ID),
('Miranda', 'Uranus 5', 0.0000659, 473, @P_ID);

SET @P_ID=(select id from planets where name='Neptune');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Triton', 'Neptune 1', 0.02139, 2706.8, @P_ID),
('Nereid', 'Neptune 2', 0.00002, 340, @P_ID),
('Naiad', 'Neptune 3', 0.00000019, 58, @P_ID),
('Thalassa', 'Neptune 4', 0.00000037, 82, @P_ID),
('Despina', 'Neptune 5', 0.0000008, 150, @P_ID),
('Galatea', 'Neptune 6', 0.00000212, 184, @P_ID),
('Larissa', 'Neptune 7', NULL, 193, @P_ID),
('Proteus', 'Neptune 8', NULL, 416, @P_ID);

SET @P_ID=(select id from planets where name='Pluto');
INSERT INTO `moons` (
        name, alias, mass, diameter, planet_id
) VALUES
('Charon', 'Pluto 1', 0.001586, 1212, @P_ID),
('Nix', 'Pluto 2', 0.000000045, 49.8, @P_ID);

LOAD DATA LOCAL INFILE './discoveryFacility.csv' INTO TABLE `discoveryFacility` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (`id`,`name`);

LOAD DATA LOCAL INFILE './parameterReference.csv' INTO TABLE `parameterReference` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (`id`, `ref_name`, `ref_link`);

LOAD DATA LOCAL INFILE './planetarySystem.csv' INTO TABLE `planetarySystem` FIELDS TERMINATED BY ',' ENCLOSED BY '\"' LINES TERMINATED BY '\r\n' IGNORE 1 LINES (pl_name, hostname, default_flag, sy_snum, sy_pnum, discoverymethod, disc_year, df_id, soltype, ref_id, pl_orbper, pl_bmasse, st_teff, sy_dist, sy_gaiamag, rowupdate, releasedate);
```
### Step 3. Import the data
```
source universeV2.sql
```
> **sample output**
> ```
> Query OK, 0 rows affected, 1 warning (2.03 sec)
> ```
### Step 4. Make universe the current database
```
use universe;
```
> **sample output**
> ```
> Database changed
> ```
### Step 5. Check the imported data
````
SHOW TABLES;
SELECT COUNT(*) FROM stars;
SELECT COUNT(*) FROM planets;
SELECT COUNT(*) FROM moons;
SELECT COUNT(*) FROM planet_categories;
SELECT COUNT(*) FROM discoveryFacility;
SELECT COUNT(*) FROM parameterReference;
SELECT COUNT(*) FROM planetarySystem;
````
> **sample output**
> ```
> tidb>show tables;
>+--------------------+
>| Tables_in_universe |
>+--------------------+
>| discoveryFacility  |
>| moons              |
>| parameterReference |
>| planet_categories  |
>| planetarySystem    |
>| planets            |
>| stars              |
>+--------------------+
>7 rows in set (0.00 sec)
>
>tidb>select count(*) from stars;
>+----------+
>| count(*) |
>+----------+
>|        2 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planets;
>+----------+
>| count(*) |
>+----------+
>|       10 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from moons;
>+----------+
>| count(*) |
>+----------+
>|       28 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planet_categories;
>+----------+
>| count(*) |
>+----------+
>|        3 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from discoveryFacility;
>+----------+
>| count(*) |
>+----------+
>|       70 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from parameterReference;
>+----------+
>| count(*) |
>+----------+
>|     2055 |
>+----------+
>1 row in set (0.00 sec)
>
>tidb>select count(*) from planetarySystem;
>+----------+
>| count(*) |
>+----------+
>|    34713 |
>+----------+
>1 row in set (0.02 sec)
> ```

## Module 2: Indexing Strategies
- In this module, we optimize and compare the execution plan for the following queries:
```
SELECT pl_name, hostname FROM planetarySystem WHERE pl_name='HAT-P-42 b' and hostname='HAT-P-42';
```
```
SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where hostname='11 Com' AND ref_id=2048;
```
```
SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND ref_id=2048;
```
```
SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND hostname='11 Com' AND ref_id=2048;
```
```
SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND hostname like '11%' AND ref_id=2048;
```
```
SELECT id, pl_name, hostname FROM planetarySystem WHERE hostname = '11 Com' AND ref_id=2048;
```
```
SELECT id, pl_name, hostname FROM planetarySystem WHERE id=835868090839969012 OR ref_id=2048;
```

### Step 1. Add an index and check the execution plan for the following query
```
CREATE INDEX idx_pl_name on planetarySystem(pl_name);
ANALYZE TABLE planetarySystem;
EXPLAIN SELECT pl_name, hostname FROM planetarySystem WHERE pl_name='HAT-P-42 b' and hostname='HAT-P-42';
```
> **sample output**
> ```
>+-------------------------------+---------+-----------+---------------------------------------------------+-----------------------------------------------------+
>| id                            | estRows | task      | access object                                     | operator info                                       |
>+-------------------------------+---------+-----------+---------------------------------------------------+-----------------------------------------------------+
>| IndexLookUp_11                | 0.00    | root      |                                                   |                                                     |
>| ├─IndexRangeScan_8(Build)     | 5.62    | cop[tikv] | table:planetarySystem, index:idx_pl_name(pl_name) | range:["HAT-P-42 b","HAT-P-42 b"], keep order:false |
>| └─Selection_10(Probe)         | 0.00    | cop[tikv] |                                                   | eq(universe.planetarysystem.hostname, "HAT-P-42")   |
>|   └─TableRowIDScan_9          | 5.62    | cop[tikv] | table:planetarySystem                             | keep order:false                                    |
>+-------------------------------+---------+-----------+---------------------------------------------------+-----------------------------------------------------+
>4 rows in set (0.00 sec)
> ```
- What is the access method for this query?
    - Answer: IndexLookUp and IndexRangeScan.

- What cost did the optimizer calculate for this query?
    - Answer: 5.62 rows.

- Which columns must TiDB access to resolve the query?
    - Answer: pl_name and hostname columns.

- Does this access strategy result in good performance? Why?
    - Answer: Maybe. The IndexLookUp and IndexRangeScan access method indicates that TiDB cannot select a single row based on the key value. Performance is good only if the key used by the optimizer matches only a few rows.

```
DROP INDEX idx_pl_name ON planetarySystem;
```
> **sample output**
> ```
> Query OK, 0 rows affected (0.36 sec)
> ```
### Step 2. Add another index and check the execution plan for the same query
- Create a covering index called idx_pl_name_hostname that indexes the appropriate columns in the planetarySystem table so that the query that you examined in the previous step executes without accessing the table rows.
```
CREATE INDEX idx_pl_name_hostname on planetarySystem(pl_name, hostname);
ANALYZE TABLE planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (5.23 sec)
> ```
```
EXPLAIN SELECT pl_name, hostname FROM planetarySystem WHERE pl_name='HAT-P-42 b' and hostname='HAT-P-42';
```
> **sample output**
> ```
>+------------------------+---------+-----------+----------------------------------------------------------------------+---------------------------------------------------------------------------+
>| id                     | estRows | task      | access object                                                        | operator info                                                             |
>+------------------------+---------+-----------+----------------------------------------------------------------------+---------------------------------------------------------------------------+
>| IndexReader_6          | 5.62    | root      |                                                                      | index:IndexRangeScan_5                                                    |
>| └─IndexRangeScan_5     | 5.62    | cop[tikv] | table:planetarySystem, index:idx_pl_name_hostname(pl_name, hostname) | range:["HAT-P-42 b" "HAT-P-42","HAT-P-42 b" "HAT-P-42"], keep order:false |
>+------------------------+---------+-----------+----------------------------------------------------------------------+---------------------------------------------------------------------------+
>2 rows in set (0.01 sec)>
> ```
- Explain the difference in the output.
    - Answer: The query now uses the idx_pl_name_hostname index to resolve the query, without having to access the table rows. The access method for this query is IndexReader and IndexRangeScan.

```
DROP INDEX idx_pl_name_hostname ON planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.37 sec)
> ```
### Step 3. Add an index and check the execution plan for the following query
```
CREATE INDEX idx_pl_name_hostname_ref ON planetarySystem(pl_name, hostname, ref_id);
ANALYZE TABLE planetarySystem;
```

> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (2.88 sec)
> ```

```
EXPLAIN SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where hostname='11 Com' AND ref_id=2048;
```
> **sample output**
> ```
>+---------------------------+----------+-----------+-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| id                        | estRows  | task      | access object         | operator info                                                                                                                           |
>+---------------------------+----------+-----------+-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_4              | 0.00     | root      |                       | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum |
>| └─TableReader_7           | 0.00     | root      |                       | data:Selection_6                                                                                                                       |
>|   └─Selection_6           | 0.00     | cop[tikv] |                       | eq(universe.planetarysystem.hostname, "11 Com"), eq(universe.planetarysystem.ref_id, 2048)                                              |
>|     └─TableFullScan_5     | 34713.00 | cop[tikv] | table:planetarySystem | keep order:false                                                                                                                       |
>+---------------------------+----------+-----------+-----------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>4 rows in set (0.00 sec)
> ``
- What is the access method for this query?
    - Answer: TableReader and TableFullScan.
- Is the index idx_pl_name_hostname_ref helpful for improving the performance of this query? Why?
    - Answer: No. The index idx_pl_name_hostname_ref is composed in the order of pl_name, hostname, and ref_id. If the filtering condition does not include pl_name, then the index can not be used.

### Step 4. Check the execution plan for the following query
```
EXPLAIN SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND ref_id=2048;
```
> **sample output**
> ```
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows | task      | access object                                                                    | operator info                                                                                                                          |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_4                    | 0.00    | root      |                                                                                 | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum |
>| └─IndexLookUp_11                | 0.00    | root      |                                                                                 |                                                                                                                                        |
>|   ├─Selection_10(Build)         | 0.00    | cop[tikv] |                                                                                 | eq(universe.planetarysystem.ref_id, 2048)                                                                                              |
>|   │ └─IndexRangeScan_8          | 5.62    | cop[tikv] | table:planetarySystem, index:idx_pl_name_hostname_ref(pl_name, hostname, ref_id) | range:["11 Com b","11 Com b"], keep order:false                                                                                        |
>|   └─TableRowIDScan_9(Probe)     | 0.00    | cop[tikv] | table:planetarySystem                                                           | keep order:false                                                                                                                       |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>5 rows in set (0.00 sec)
> ```
- What is the access method for this query?
    - Answer: IndexLookUp and IndexRangeScan.
- What cost did the optimizer calculate for this query?
    - Answer: 5.62.
- Which columns must TiDB access to resolve the query?
    - Answer: pl_name, hostname, sy_snum, sy_pnum, and ref_id.
- Is the index idx_pl_name_hostname_ref helpful for improving the performance of this query? Why?
    - Answer: Yes. The query involves conditions related to the column pl_name, the index idx_pl_name_hostname_ref will be used, but the conditions related to the column ref_id will not use an index, because the conditions related to column hostname did not appear in the query. We can see from the third line of the EXPLAIN output, in the Selection section, that the condition related to ref_id is completed separately after the IndexRangeScan.

### Step 5. Check the execution plan for the following query
```
EXPLAIN SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND hostname='11 Com' AND ref_id=2048;
```

> **sample output**
> ```
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows | task      | access object                                                                    | operator info                                                                                                                           |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_4                    | 1.00    | root      |                                                                                  | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum |
>| └─IndexLookUp_10                | 1.00    | root      |                                                                                  |                                                                                                                                         |
>|   ├─IndexRangeScan_8(Build)     | 1.00    | cop[tikv] | table:planetarySystem, index:idx_pl_name_hostname_ref(pl_name, hostname, ref_id) | range:["11 Com b" "11 Com" 2048,"11 Com b" "11 Com" 2048], keep order:false                                                             |
>|   └─TableRowIDScan_9(Probe)     | 1.00    | cop[tikv] | table:planetarySystem                                                            | keep order:false                                                                                                                        |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>4 rows in set (0.00 sec)
> ```

- What is the access method for this query?
    - Answer: IndexLookUp and IndexRangeScan.
- What cost did the optimizer calculate for this query?
    - Answer: 1.00.
- Which columns must TiDB access to resolve the query?
    - Answer: pl_name, hostname, sy_snum, sy_pnum and ref_id columns.
- Is the index idx_pl_name_hostname_ref helpful for improving the performance of this query? Why?
    - Answer: Yes. The columns pl_name, hostname, and ref_id in the index idx_pl_name_hostname_ref all appear in the query.

### Step 6. Check the execution plan for the following query
```
EXPLAIN SELECT pl_name, hostname, sy_snum, sy_pnum FROM planetarySystem where pl_name='11 Com b' AND hostname like '11%' AND ref_id=2048;
```
> **sample output**
> ```
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows | task      | access object                                                                    | operator info                                                                                                                           |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_4                    | 0.00    | root      |                                                                                  | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.sy_snum, universe.planetarysystem.sy_pnum |
>| └─IndexLookUp_11                | 0.00    | root      |                                                                                  |                                                                                                                                         |
>|   ├─Selection_10(Build)         | 0.00    | cop[tikv] |                                                                                  | eq(universe.planetarysystem.ref_id, 2048)                                                                                               |
>|   │ └─IndexRangeScan_8          | 0.00    | cop[tikv] | table:planetarySystem, index:idx_pl_name_hostname_ref(pl_name, hostname, ref_id) | range:["11 Com b" "11","11 Com b" "12"), keep order:false                                                                               |
>|   └─TableRowIDScan_9(Probe)     | 0.00    | cop[tikv] | table:planetarySystem                                                            | keep order:false                                                                                                                        |
>+---------------------------------+---------+-----------+----------------------------------------------------------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
>5 rows in set (0.00 sec)
> ```
- What is the access method for this query?
    - Answer: IndexLookUp and IndexRangeScan.
- Which columns must TiDB access to resolve the query?
    - Answer: pl_name, hostname, sy_snum, sy_pnum, and ref_id.
- Is the index idx_pl_name_hostname_ref helpful for improving the performance of this query? Why?
    - Answer: Yes. Although the columns pl_name, hostname, and ref_id in the index idx_pl_name_hostname_ref all appear in the query, we still cannot use the column ref_id in the index because the condition on the hostname column is not an equality query, but rather a range query. We can see from the third line of the EXPLAIN output, in the Selection section, that the condition related to ref_id is completed separately after the IndexRangeScan.

### Step 7. Check the execution plan for the following query
```
EXPLAIN SELECT id, pl_name, hostname FROM planetarySystem WHERE hostname = '11 Com' AND ref_id=2048;
```
> **sample output**
> ```
>+---------------------------+----------+-----------+----------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------+
>| id                        | estRows  | task      | access object                                                                    | operator info                                                                                    |
>+---------------------------+----------+-----------+----------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------+
>| Projection_4              | 0.00     | root      |                                                                                  | universe.planetarysystem.id, universe.planetarysystem.pl_name, universe.planetarysystem.hostname |
>| └─IndexReader_10          | 0.00     | root      |                                                                                  | index:Selection_9                                                                                |
>|   └─Selection_9           | 0.00     | cop[tikv] |                                                                                  | eq(universe.planetarysystem.hostname, "11 Com"), eq(universe.planetarysystem.ref_id, 2048)       |
>|     └─IndexFullScan_8     | 34713.00 | cop[tikv] | table:planetarySystem, index:idx_pl_name_hostname_ref(pl_name, hostname, ref_id) | keep order:false                                                                                 |
>+---------------------------+----------+-----------+----------------------------------------------------------------------------------+--------------------------------------------------------------------------------------------------+
>4 rows in set (0.01 sec)
> ```
- What is the access method for this query?
    - Answer: IndexReader and IndexFullScan.
- Is the index idx_pl_name_hostname_ref helpful for improving the performance of this query? Why?
    - Answer: Yes. Although the column pl_name in the index idx_pl_name_hostname_ref is not used in the query condition, since the SELECT statement includes id, pl_name, and hostname columns, all of which are included in the index idx_pl_name_hostname_ref, we can use IndexFullScan.
- Does this access strategy result in good performance? Why?
    - Answer: Probably not. Because as the data volume increases, the amount of index data that IndexFullScan needs to scan will also increase, which will result in a decrease in performance.

```
DROP INDEX idx_pl_name_hostname_ref ON planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.36 sec)
> ```

### Step 8. Add an index and check the execution plan for the following query
- Add index idx_ref_id on the ref_id columns of the planetarySystem table.
```
CREATE INDEX idx_ref_id ON planetarySystem(ref_id);
ANALYZE TABLE planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (3.26 sec)
> ```
```
EXPLAIN SELECT id, pl_name, hostname FROM planetarySystem WHERE id=835868090839969012 OR ref_id=2048;
```
> **sample output**
> ```
>+----------------------------------+---------+-----------+-------------------------------------------------+--------------------------------------------------------------------------------------------------+
>| id                               | estRows | task      | access object                                   | operator info                                                                                    |
>+----------------------------------+---------+-----------+-------------------------------------------------+--------------------------------------------------------------------------------------------------+
>| Projection_4                     | 17.00   | root      |                                                 | universe.planetarysystem.id, universe.planetarysystem.pl_name, universe.planetarysystem.hostname |
>| └─IndexMerge_11                  | 17.00   | root      |                                                 | type: union                                                                                      |
>|   ├─TableRangeScan_8(Build)      | 1.00    | cop[tikv] | table:planetarySystem                           | range:[835868090839969012,835868090839969012], keep order:false                                  |
>|   ├─IndexRangeScan_9(Build)      | 16.00   | cop[tikv] | table:planetarySystem, index:idx_ref_id(ref_id) | range:[2048,2048], keep order:false                                                              |
>|   └─TableRowIDScan_10(Probe)     | 17.00   | cop[tikv] | table:planetarySystem                           | keep order:false                                                                                 |
>+----------------------------------+---------+-----------+-------------------------------------------------+--------------------------------------------------------------------------------------------------+
>5 rows in set (0.00 sec)
> ```
- What is the access method for this query?
    - Answer: IndexMerge.
- Does this access strategy result in good performance? Why?
    - Answer: Maybe. The index merge access strategy tells you that TiDB has found multiple indexes that it can use for the query. In this case, it uses the primary key and the ref_id index. The index merge method retrieves rows with multiple range scans and merges the results performance is good if there are high selectivity indexes on the table. The operator info using union entry shows how TiDB merges index scans for the index merge join type.
```
DROP INDEX idx_ref_id ON planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.36 sec)
> ```

## Module 3: Indexing Strategies with ORDER BY
- In this module, we optimize the following query
```
EXPLAIN SELECT pl_name, hostname, disc_year FROM planetarySystem WHERE hostname='14 Her' ORDER BY disc_year DESC;
```
### Step 1. Check the execution plan for the following que
```
EXPLAIN SELECT pl_name, hostname, disc_year FROM planetarySystem WHERE hostname='14 Her' ORDER BY disc_year DESC;
```
> **sample output**
> ```
>+---------------------------+----------+-----------+-----------------------+-------------------------------------------------+
>| id                        | estRows  | task      | access object         | operator info                                   |
>+---------------------------+----------+-----------+-----------------------+-------------------------------------------------+
>| Sort_5                    | 5.81     | root      |                       | universe.planetarysystem.disc_year:desc         |
>| └─TableReader_10          | 5.81     | root      |                       | data:Selection_9                                |
>|   └─Selection_9           | 5.81     | cop[tikv] |                       | eq(universe.planetarysystem.hostname, "14 Her") |
>|     └─TableFullScan_8     | 34713.00 | cop[tikv] | table:planetarySystem | keep order:false                                |
>+---------------------------+----------+-----------+-----------------------+-------------------------------------------------+
>4 rows in set (0.00 sec)
> ```

- What is the access method for this query?
    - Answer: TableFullScan, Selection, TableReader and Sort.
- Which indexes can the optimizer use for this query?
    - Answer: None
- What information does the operator info column contain?
    - Answer: eq(universe.planetarysystem.hostname, "14 Her"): The query uses a WHERE clause to filter the output rows; universe.planetarysystem.disc_year:desc: TiDB must perform an extra pass through the table to work out how to retrieve the rows in sorted order. The sort is performed by iterating through all the rows, and storing the sort keys together with the pointers to the row for all rows that match the WHERE clause. TiDB Server sorts the keys, and then retrieves the rows in the required order.
- Does this query perform well? Why?
    - Answer: No. This query requires a full table scan(type: TableFullScan), cannot use any keys, and requires a sort on the disc_year column. Sort requires all data to be loaded into one TiDB Server to finish the tasks.
- What change must you make to the table to prevent the query from performing a full table scan?
    - Answer: You must add an index to the hostname column of the planetarySystem table.

### Step 2. Add an index and check the execution plan for the following query
```
ALTER TABLE planetarySystem ADD INDEX idx_hostname(hostname);
ANALYZE TABLE planetarySystem;
```

> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (3.41 sec)
> ```
```
EXPLAIN SELECT pl_name, hostname, disc_year FROM planetarySystem WHERE hostname='14 Her' ORDER BY disc_year DESC;
```
> **sample output**
> ```
>+----------------------------------+---------+-----------+-----------------------------------------------------+---------------------------------------------+
>| id                               | estRows | task      | access object                                       | operator info                               |
>+----------------------------------+---------+-----------+-----------------------------------------------------+---------------------------------------------+
>| Sort_5                           | 5.81    | root      |                                                     | universe.planetarysystem.disc_year:desc     |
>| └─IndexLookUp_13                 | 5.81    | root      |                                                     |                                             |
>|   ├─IndexRangeScan_11(Build)     | 5.81    | cop[tikv] | table:planetarySystem, index:idx_hostname(hostname) | range:["14 Her","14 Her"], keep order:false |
>|   └─TableRowIDScan_12(Probe)     | 5.81    | cop[tikv] | table:planetarySystem                               | keep order:false                            |
>+----------------------------------+---------+-----------+-----------------------------------------------------+---------------------------------------------+
>4 rows in set (0.00 sec)
> ```

- What is the access method for this query?
    - Answer: IndexRangeScan, IndexLookUp and Sort.
- What key does the optimizer use?
    - Answer: hostname
- What information does the operator info column contain?
    - Answer: range:["14 Her","14 Her"], keep order:false; universe.planetarysystem.disc_year:desc.
- Does the change that you implemented in the preceding step improve the performance of the query? Why?
    - Answer: Yes. Now the query does not require a full table scan. Instead, it uses the IndexRangeScan and IndexLookUp to retrieve the rows from the index on the hostname column. In addition, task: cop[tikv] column shows all index filtering operations are completed in parallel in TiKVs.
- What must you do to prevent this query from having to perform a sort？
    - Answer: Add a composite index to the planetarySystem table that includes the hostname and disc_year columns. The index on the hostname column is no longer required in this case, because TiDB can use the index prefix.

### Step 3. Add an index and check the execution plan for the following query
- Remove the sort operation by making the changes identified in the previous step.
```
ALTER TABLE planetarySystem DROP INDEX idx_hostname;
ALTER TABLE planetarySystem ADD INDEX idx_hostname_disc_year(hostname, disc_year);
ANALYZE TABLE planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected (3.06 sec)
> ```
```
EXPLAIN SELECT pl_name, hostname, disc_year FROM planetarySystem WHERE hostname='14 Her' ORDER BY disc_year DESC;
```

> **sample output**
> ```
>+----------------------------------+---------+-----------+--------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------+
>| id                               | estRows | task      | access object                                                            | operator info                                                                                           |
>+----------------------------------+---------+-----------+--------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------+
>| Projection_18                    | 5.81    | root      |                                                                          | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.disc_year |
>| └─IndexLookUp_17                 | 5.81    | root      |                                                                          |                                                                                                         |
>|   ├─IndexRangeScan_15(Build)     | 5.81    | cop[tikv] | table:planetarySystem, index:idx_hostname_disc_year(hostname, disc_year) | range:["14 Her","14 Her"], keep order:true, desc                                                        |
>|   └─TableRowIDScan_16(Probe)     | 5.81    | cop[tikv] | table:planetarySystem                                                    | keep order:false                                                                                        |
>+----------------------------------+---------+-----------+--------------------------------------------------------------------------+---------------------------------------------------------------------------------------------------------+
>4 rows in set (0.00 sec)
> ```
- What key does the optimizer use?
    - Answer: The key that you added in the preceding step. (idx_hostname_disc_year in the example solution)
- What information does the operator info column contain?
    - Answer: range:["14 Her","14 Her"], keep order:true, desc.
- Does this change improve the performance of the query? Why?
    - Answer: Yes. Adding a composite key on the hostname and disc_year columns removes the requirement to perform a sort in TiDB Server.
- What must you do to prevent this query from requiring a primary key lookup after finding the indexed row?
    - Answer: Add a composite index to the planetarySystem table that includes the hostname, disc_year, and pl_name columns. Drop the composite index on the hostname and disc_year columns, because TiDB can use the index prefix.

### Step 4. Add an index and check the execution plan for the following query
```
ALTER TABLE planetarySystem DROP INDEX idx_hostname_disc_year;
ALTER TABLE planetarySystem ADD INDEX idx_hostname_disc_year_pl_name(hostname, disc_year, pl_name); 
ANALYZE TABLE planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.35 sec)
> ```
```
EXPLAIN SELECT pl_name, hostname, disc_year FROM planetarySystem WHERE hostname='14 Her' ORDER BY disc_year DESC;
```
> **sample output**
> ```
>+-------------------------+---------+-----------+-------------------------------------------------------------------------------------------+--------------------------------------------------+
>| id                      | estRows | task      | access object                                                                             | operator info                                    |
>+-------------------------+---------+-----------+-------------------------------------------------------------------------------------------+--------------------------------------------------+
>| IndexReader_12          | 5.81    | root      |                                                                                           | index:IndexRangeScan_11                          |
>| └─IndexRangeScan_11     | 5.81    | cop[tikv] | table:planetarySystem, index:idx_hostname_disc_year_pl_name(hostname, disc_year, pl_name) | range:["14 Her","14 Her"], keep order:true, desc |
>+-------------------------+---------+-----------+-------------------------------------------------------------------------------------------+--------------------------------------------------+
>2 rows in set (0.00 sec)
> ```
- What key does the optimizer use?
    - Answer: idx_hostname_disc_year_pl_name
- What information does the operator info column contain?
    - Answer: range:["14 Her","14 Her"], keep order:true, desc; index:IndexRangeScan_11
- Does this change improve the performance of the query? Why?
    - Answer: Yes. The query now retrieves all the data that it needs from the index, without accessing the base table (operator info: index:IndexRangeScan_11). This type of index is known as a covering index. Performance will be very good for reads, but there might be a negative impact on performance for table updates because TiDB must update the index too.

### Step 5. Drop the index
- Drop any indexes that you added in this practice to return the planetarySystem table to its original state.
```
ALTER TABLE planetarySystem DROP INDEX idx_hostname_disc_year_pl_name;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.35 sec)
> ```
## Module 4: Indexing Strategies with JOIN
- In this module, we optimize the following two queries:
```
SELECT A.pl_name, A.hostname, A.discoverymethod, A.disc_year,B.name as DiscoveryFacility, C.ref_name as ParameterReference 
FROM planetarySystem A, discoveryFacility B, parameterReference C 
WHERE A.df_id = B.id AND A.ref_id = C.id AND C.ref_name LIKE 'Rosenthal%' AND A.hostname LIKE '14 Her' AND A.disc_year > 2020;
```
```
SELECT * FROM planetarySystem WHERE ref_id IN ( SELECT ID FROM parameterReference WHERE ref_name LIKE 'Rosenthal%');
```
### Step 1. Check the execution plan for the following query
```
EXPLAIN SELECT A.pl_name, A.hostname, A.discoverymethod, A.disc_year,B.name as DiscoveryFacility, C.ref_name as ParameterReference 
FROM planetarySystem A, discoveryFacility B, parameterReference C 
WHERE A.df_id = B.id AND A.ref_id = C.id AND C.ref_name LIKE 'Rosenthal%' AND A.hostname LIKE '14 Her' AND A.disc_year > 2020;
```
> **sample output**
> ```
>+------------------------------------+----------+-----------+---------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                                 | estRows  | task      | access object | operator info                                                                                                                                                                                                            |
>+------------------------------------+----------+-----------+---------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_13                      | 1.98     | root      |               | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.discoverymethod, universe.planetarysystem.disc_year, universe.discoveryfacility.name, universe.parameterreference.ref_name |
>| └─IndexJoin_18                     | 1.98     | root      |               | inner join, inner:TableReader_15, outer key:universe.planetarysystem.df_id, inner key:universe.discoveryfacility.id, equal cond:eq(universe.planetarysystem.df_id, universe.discoveryfacility.id)                        |
>|   ├─IndexJoin_31(Build)            | 1.98     | root      |               | inner join, inner:TableReader_27, outer key:universe.planetarysystem.ref_id, inner key:universe.parameterreference.id, equal cond:eq(universe.planetarysystem.ref_id, universe.parameterreference.id)                    |
>|   │ ├─TableReader_40(Build)        | 1.98     | root      |               | data:Selection_39                                                                                                                                                                                                        |
>|   │ │ └─Selection_39               | 1.98     | cop[tikv] |               | gt(universe.planetarysystem.disc_year, 2020), like(universe.planetarysystem.hostname, "14 Her", 92)                                                                                                                      |
>|   │ │   └─TableFullScan_38         | 34713.00 | cop[tikv] | table:A       | keep order:false                                                                                                                                                                                                         |
>|   │ └─TableReader_27(Probe)        | 0.00     | root      |               | data:Selection_26                                                                                                                                                                                                        |
>|   │   └─Selection_26               | 0.00     | cop[tikv] |               | like(universe.parameterreference.ref_name, "Rosenthal%", 92)                                                                                                                                                             |
>|   │     └─TableRangeScan_25        | 1.98     | cop[tikv] | table:C       | range: decided by [universe.planetarysystem.ref_id], keep order:false                                                                                                                                                    |
>|   └─TableReader_15(Probe)          | 1.98     | root      |               | data:TableRangeScan_14                                                                                                                                                                                                   |
>|     └─TableRangeScan_14            | 1.98     | cop[tikv] | table:B       | range: decided by [universe.planetarysystem.df_id], keep order:false                                                                                                                                                     |
>+------------------------------------+----------+-----------+---------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>11 rows in set (0.01 sec)
> ```
- Which two tables are joined first? What type of table join is used?
    - Answer: Table planetarySystem (A) and parameterReference (C) are joined first, the join type is IndexJoin.
- Which table only participates in the second table join? What type of table join is used?
    - Answer: Table discoveryFacility (B), the join type is IndexJoin.
- What is the access method for each table query?
    - Answer: TableFullScan, Selection and TableReader for table planetarySystem (A); TableRangeScan, Selection and TableReader for table parameterReference (C); TableRangeScan and TableReader for table discoveryFacility (B).
- Does this access strategy result in good performance? Why?
    - Answer: No. This is because in the WHERE clause, you see filter conditions like C.ref_name LIKE 'Rosenthal%' AND A.hostname LIKE '14 Her' AND A.disc_year > 2020. We can create indexes to accelerate these filter operations in TiKV nodes and avoid TableFullScan or other filtering actions such as SELECT, in order to reduce IO pressure.

### Step 2. Add indexes and check the execution plan for the following query
```
CREATE INDEX idx_hostname_disc_year ON planetarySystem(hostname, disc_year);
CREATE INDEX idx_ref_name ON parameterReference(ref_name);
ANALYZE TABLE planetarySystem;
ANALYZE TABLE parameterReference;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (0.63 sec)
> ```

```
EXPLAIN SELECT A.pl_name, A.hostname, A.discoverymethod, A.disc_year,B.name as DiscoveryFacility, C.ref_name as ParameterReference 
FROM planetarySystem A, discoveryFacility B, parameterReference C 
WHERE A.df_id = B.id AND A.ref_id = C.id AND C.ref_name LIKE 'Rosenthal%' AND A.hostname LIKE '14 Her' AND A.disc_year > 2020;
```
> **sample output**
> ```
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                                       | estRows | task      | access object                                              | operator info                                                                                                                                                                                                            |
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_13                            | 1.98    | root      |                                                            | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.discoverymethod, universe.planetarysystem.disc_year, universe.discoveryfacility.name, universe.parameterreference.ref_name |
>| └─IndexJoin_18                           | 1.98    | root      |                                                            | inner join, inner:TableReader_15, outer key:universe.planetarysystem.df_id, inner key:universe.discoveryfacility.id, equal cond:eq(universe.planetarysystem.df_id, universe.discoveryfacility.id)                        |
>|   ├─HashJoin_37(Build)                   | 1.98    | root      |                                                            | inner join, equal:[eq(universe.planetarysystem.ref_id, universe.parameterreference.id)]                                                                                                                                  |
>|   │ ├─IndexLookUp_43(Build)              | 1.98    | root      |                                                            |                                                                                                                                                                                                                          |
>|   │ │ ├─IndexRangeScan_41(Build)         | 1.98    | cop[tikv] | table:A, index:idx_hostname_disc_year(hostname, disc_year) | range:("14 Her" 2020,"14 Her" +inf], keep order:false                                                                                                                                                                    |
>|   │ │ └─TableRowIDScan_42(Probe)         | 1.98    | cop[tikv] | table:A                                                    | keep order:false                                                                                                                                                                                                         |
>|   │ └─IndexReader_45(Probe)              | 2.72    | root      |                                                            | index:IndexRangeScan_44                                                                                                                                                                                                  |
>|   │   └─IndexRangeScan_44                | 2.72    | cop[tikv] | table:C, index:idx_ref_name(ref_name)                      | range:["Rosenthal","Rosentham"), keep order:false                                                                                                                                                                        |
>|   └─TableReader_15(Probe)                | 1.98    | root      |                                                            | data:TableRangeScan_14                                                                                                                                                                                                   |
>|     └─TableRangeScan_14                  | 1.98    | cop[tikv] | table:B                                                    | range: decided by [universe.planetarysystem.df_id], keep order:false                                                                                                                                                     |
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>10 rows in set (0.01 sec)
> ```

- Which two tables are joined first? What type of table join is used?
    - Answer: Table planetarySystem (A) and parameterReference (C) are joined first, the join type is HashJoin.
- Which table only participates in the second table join? What type of table join is used?
    - Answer: Table discoveryFacility (B), the join type is IndexJoin.
- What is the access method for each table query?
    - Answer: IndexRangeScan and IndexLookUp for table planetarySystem (A); IndexRangeScan and IndexReader for table parameterReference (C); TableRangeScan and TableReader for table discoveryFacility (B).
- Does this access strategy result in good performance? Why?
    - Answer: Yes. This is because TableFullScan or other filtering actions such as SELECT before table joining are avoided by using indexes.

### Step 3. Add indexes and check the execution plan for the following query
```
CREATE INDEX idx_ref_id ON planetarySystem(ref_id);
CREATE INDEX idx_df_id ON planetarySystem(df_id);
ANALYZE TABLE planetarySystem;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (4.00 sec)
> ```
```
EXPLAIN SELECT A.pl_name, A.hostname, A.discoverymethod, A.disc_year,B.name as DiscoveryFacility, C.ref_name as ParameterReference 
FROM planetarySystem A, discoveryFacility B, parameterReference C 
WHERE A.df_id = B.id AND A.ref_id = C.id AND C.ref_name LIKE 'Rosenthal%' AND A.hostname LIKE '14 Her' AND A.disc_year > 2020;
```
> **sample output**
> ```
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                                       | estRows | task      | access object                                              | operator info                                                                                                                                                                                                            |
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Projection_13                            | 1.98    | root      |                                                            | universe.planetarysystem.pl_name, universe.planetarysystem.hostname, universe.planetarysystem.discoverymethod, universe.planetarysystem.disc_year, universe.discoveryfacility.name, universe.parameterreference.ref_name |
>| └─IndexJoin_19                           | 1.98    | root      |                                                            | inner join, inner:TableReader_16, outer key:universe.planetarysystem.df_id, inner key:universe.discoveryfacility.id, equal cond:eq(universe.planetarysystem.df_id, universe.discoveryfacility.id)                        |
>|   ├─HashJoin_71(Build)                   | 1.98    | root      |                                                            | inner join, equal:[eq(universe.planetarysystem.ref_id, universe.parameterreference.id)]                                                                                                                                  |
>|   │ ├─IndexLookUp_85(Build)              | 1.98    | root      |                                                            |                                                                                                                                                                                                                          |
>|   │ │ ├─IndexRangeScan_83(Build)         | 1.98    | cop[tikv] | table:A, index:idx_hostname_disc_year(hostname, disc_year) | range:("14 Her" 2020,"14 Her" +inf], keep order:false                                                                                                                                                                    |
>|   │ │ └─TableRowIDScan_84(Probe)         | 1.98    | cop[tikv] | table:A                                                    | keep order:false                                                                                                                                                                                                         |
>|   │ └─IndexReader_87(Probe)              | 2.72    | root      |                                                            | index:IndexRangeScan_86                                                                                                                                                                                                  |
>|   │   └─IndexRangeScan_86                | 2.72    | cop[tikv] | table:C, index:idx_ref_name(ref_name)                      | range:["Rosenthal","Rosentham"), keep order:false                                                                                                                                                                        |
>|   └─TableReader_16(Probe)                | 1.98    | root      |                                                            | data:TableRangeScan_15                                                                                                                                                                                                   |
>|     └─TableRangeScan_15                  | 1.98    | cop[tikv] | table:B                                                    | range: decided by [universe.planetarysystem.df_id], keep order:false                                                                                                                                                     |
>+------------------------------------------+---------+-----------+------------------------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>10 rows in set (0.01 sec)
> ```

- Has the join order of three tables been changed? Why?
    - Answer: No. Because the optimizer often determines the order of the join based on the amount of data involved in each table, with the table having the smallest amount of data being the first to join. It determine the number of rows in each table that participate in the join based on the filtering conditions in the query, as follows:
```
SELECT COUNT(*) FROM planetarysystem WHERE hostname LIKE '14 Her' AND disc_year > 2020;
SELECT COUNT(*) FROM parameterReference WHERE ref_name LIKE 'Rosenthal%';
SELECT COUNT(*) FROM discoveryFacility;
```
> **sample output**
> ```
>+----------+
>| COUNT(*) |
>+----------+
>|        2 |
>+----------+
>1 row in set (0.01 sec)
>
>+----------+
>| COUNT(*) |
>+----------+
>|        2 |
>+----------+
>1 row in set (0.00 sec)
>
>+----------+
>| COUNT(*) |
>+----------+
>|       70 |
>+----------+
>1 row in set (0.00 sec)

> ```
- The conclusion is that the planetarySystem and parameterReference tables participates in the join with the fewest rows, while the discoveryFacility table participates in the connection with the most rows.
### Step 4. Drop the indexes
```
ALTER TABLE planetarySystem DROP INDEX idx_ref_id;
ALTER TABLE planetarySystem DROP INDEX idx_df_id;
ALTER TABLE planetarySystem DROP INDEX idx_hostname_disc_year;
ALTER TABLE ParameterReference DROP INDEX idx_ref_name;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.52 sec)
> ```
### Step 5. Check the execution plan for the following query
```
EXPLAIN SELECT * FROM planetarySystem WHERE ref_id IN ( SELECT ID FROM parameterReference WHERE ref_name LIKE 'Rosenthal%');
```
> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM planetarySystem WHERE ref_id IN ( SELECT ID FROM parameterReference WHERE ref_name LIKE 'Rosenthal%');
>+------------------------------+----------+-----------+--------------------------+-----------------------------------------------------------------------------------------+
>| id                           | estRows  | task      | access object            | operator info                                                                           |
>+------------------------------+----------+-----------+--------------------------+-----------------------------------------------------------------------------------------+
>| HashJoin_26                  | 45.97    | root      |                          | inner join, equal:[eq(universe.parameterreference.id, universe.planetarysystem.ref_id)] |
>| ├─TableReader_31(Build)      | 2.72     | root      |                          | data:Selection_30                                                                       |
>| │ └─Selection_30             | 2.72     | cop[tikv] |                          | like(universe.parameterreference.ref_name, "Rosenthal%", 92)                            |
>| │   └─TableFullScan_29       | 2055.00  | cop[tikv] | table:parameterReference | keep order:false                                                                        |
>| └─TableReader_28(Probe)      | 34713.00 | root      |                          | data:TableFullScan_27                                                                   |
>|   └─TableFullScan_27         | 34713.00 | cop[tikv] | table:planetarySystem    | keep order:false                                                                        |
>+------------------------------+----------+-----------+--------------------------+-----------------------------------------------------------------------------------------+
>6 rows in set (0.00 sec)
> ```
- What is the join type in the join?
    - Answer: HashJoin
- What is the select type for the tables in the join?
    - Answer: TableFullScan, Selection and TableReader for table parameterReference; TableFullScan and TableReader for table parameterReference.
- Does this access strategy result in good performance? Why?
    - Answer: No. First of all, for filtering with the WHERE clause, we can create indexes of table parameterReference. Secondly, we can also create indexes on the join columns of table planetarySystem.
### Step 6. Add indexes and check the execution plan for the following query
```
CREATE INDEX idx_ref_id ON planetarySystem(ref_id);
CREATE INDEX idx_ref_name ON ParameterReference(ref_name); 
ANALYZE TABLE planetarySystem;
ANALYZE TABLE ParameterReference;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (0.83 sec)
> ```
```
EXPLAIN SELECT * FROM planetarySystem WHERE ref_id IN ( SELECT ID FROM parameterReference WHERE ref_name LIKE 'Rosenthal%');
```
> **sample output**
> ```
>+----------------------------------+---------+-----------+--------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                               | estRows | task      | access object                                          | operator info                                                                                                                                                                                         |
>+----------------------------------+---------+-----------+--------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>| IndexHashJoin_20                 | 45.97   | root      |                                                        | inner join, inner:IndexLookUp_17, outer key:universe.parameterreference.id, inner key:universe.planetarysystem.ref_id, equal cond:eq(universe.parameterreference.id, universe.planetarysystem.ref_id) |
>| ├─IndexReader_46(Build)          | 2.72    | root      |                                                        | index:IndexRangeScan_45                                                                                                                                                                               |
>| │ └─IndexRangeScan_45            | 2.72    | cop[tikv] | table:parameterReference, index:idx_ref_name(ref_name) | range:["Rosenthal","Rosentham"), keep order:false                                                                                                                                                     |
>| └─IndexLookUp_17(Probe)          | 45.97   | root      |                                                        |                                                                                                                                                                                                       |
>|   ├─IndexRangeScan_15(Build)     | 45.97   | cop[tikv] | table:planetarySystem, index:idx_ref_id(ref_id)        | range: decided by [eq(universe.planetarysystem.ref_id, universe.parameterreference.id)], keep order:false                                                                                             |
>|   └─TableRowIDScan_16(Probe)     | 45.97   | cop[tikv] | table:planetarySystem                                  | keep order:false                                                                                                                                                                                      |
>+----------------------------------+---------+-----------+--------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
>6 rows in set (0.00 sec)
> ```
- What is the join type in the join?
    - Answer: IndexHashJoin
- What is the select type for the tables in the join?
    - Answer: IndexRangeScan and IndexReader for table parameterReference; IndexRangeScan and IndexLookUp for table planetarySystem.
- Does this access strategy result in good performance? Why?
    - Answer: Yes. The index idx_ref_id ensure that there is an index on the join columns of the planetarySystem table, the HashJoin can be turned into an IndexHashJoin, which avoids full table scans. At the same time, by ensuring that there is an index idx_ref_name on the ParameterReference table, filtering can be performed first to avoid full table scans.

### Step 7. Drop the indexes
```
DROP INDEX idx_ref_id ON planetarysystem;
DROP INDEX idx_ref_name ON parameterreference;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.52 sec)
> ```

====================================================================================================================
# TiDB SQL Tuning Lab 5: Statistics Management
## Module 1: Obsolete TiDB Statistics
- In this exercise, you resolve the query performance problem caused by changes in execution plans.
### Step 1. Log in to the remote Linux
### Step 2. Connect to TiDB and change to test database
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```
### Step 3. Check the parameters
- Check the parameters related to automatic collection of statistics, and ensure that tidb_enable_auto_analyze, tidb_auto_analyze_ratio, tidb_auto_analyze_start_time, and tidb_auto_analyze_end_time are set to the default values.
```
show variables like 'tidb_enable_auto_analyze';
show variables like 'tidb_auto_analyze_ratio';
show variables like 'tidb_auto_analyze_start_time';
show variables like 'tidb_auto_analyze_end_time';
```

> **sample output**
> ```
>tidb>show variables like 'tidb_enable_auto_analyze';
>+--------------------------+-------+
>| Variable_name            | Value |
>+--------------------------+-------+
>| tidb_enable_auto_analyze | ON    |
>+--------------------------+-------+
>1 row in set (0.00 sec)
>
>tidb>show variables like 'tidb_auto_analyze_ratio';
>+-------------------------+-------+
>| Variable_name           | Value |
>+-------------------------+-------+
>| tidb_auto_analyze_ratio | 0.5   |
>+-------------------------+-------+
>1 row in set (0.00 sec)
>
>tidb>show variables like 'tidb_auto_analyze_start_time';
>+------------------------------+-------------+
>| Variable_name                | Value       |
>+------------------------------+-------------+
>| tidb_auto_analyze_start_time | 00:00 +0000 |
>+------------------------------+-------------+
>1 row in set (0.00 sec)
>
>tidb>show variables like 'tidb_auto_analyze_end_time';
>+----------------------------+-------------+
>| Variable_name              | Value       |
>+----------------------------+-------------+
>| tidb_auto_analyze_end_time | 23:59 +0000 |
>+----------------------------+-------------+
>1 row in set (0.00 sec)
> ```

### Step 4. Create a table T1
```
CREATE TABLE test.T1(
   id int(11) AUTO_INCREMENT PRIMARY KEY, 
   month VARCHAR(32), 
   cnt int(11), 
   pname VARCHAR(32), 
   pid VARCHAR(32), 
   KEY idx_month(month)
);
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.53 sec)
> ```
```
EXIT
```
> **sample output**
> ```
>Bye
> ```
### Step 5. Insert the data
```
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('1',21,'BeiJing','10')"; done;
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('2',22,'ShangHai','10')"; done;
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('3',23,'NewYork','10')"; done;
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('4',24,'Tokyo','10')"; done;
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('5',25,'Los Angeles','10')"; done;
```
> **sample output**
> ```
>for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('5',25,'Los Angeles','10')"; done;  
> ```

### Step 6. Connect to TiDB again
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```
### Step 7. Collect the statistic of table T1
```
ANALYZE TABLE T1;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (0.29 sec)
> ```
### Step 8. Check the data distribution of table T1
```
SELECT COUNT(*) FROM T1;
SELECT month, COUNT(*) FROM T1 GROUP BY month;
```
> **sample output**
> ```
>tidb>SELECT COUNT(*) FROM T1;
>+----------+
>| COUNT(*) |
>+----------+
>|    10000 |
>+----------+
>1 row in set (0.01 sec)
>
>tidb>SELECT month, COUNT(*) FROM T1 GROUP BY month;
>+-------+----------+
>| month | COUNT(*) |
>+-------+----------+
>| 3     |     2000 |
>| 1     |     2000 |
>| 2     |     2000 |
>| 4     |     2000 |
>| 5     |     2000 |
>+-------+----------+
>5 rows in set (0.01 sec)
> ```
### Step 9. Check the statistics of table T1
```
SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month')\G
SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name = 'idx_month';
```
> **sample output**
> ```
>tidb>SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month')\G
>*************************** 1. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: month
>       Is_index: 0
>    Update_time: 2023-12-04 14:36:23
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 2
>    Correlation: 1
>    Load_status: allEvicted
>Total_mem_usage: 0
> Hist_mem_usage: 0
> Topn_mem_usage: 0
>  Cms_mem_usage: 0
>*************************** 2. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_month
>       Is_index: 1
>    Update_time: 2023-12-04 14:36:23
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 242
> Hist_mem_usage: 0
> Topn_mem_usage: 242
>  Cms_mem_usage: 0
>2 rows in set (0.00 sec)
>
>tidb>SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name = 'idx_month';
>+---------+------------+----------------+-------------+----------+-------+-------+
>| Db_name | Table_name | Partition_name | Column_name | Is_index | Value | Count |
>+---------+------------+----------------+-------------+----------+-------+-------+
>| test    | T1         |                | idx_month   |        1 | 1     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 2     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 3     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 4     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 5     |  2000 |
>+---------+------------+----------------+-------------+----------+-------+-------+
>5 rows in set (0.00 sec)
> ```
- What can you find on the month column and the index idx_month recorded in the statistics?
    - Answer: In STATS_HISTOGRAMS, you found there are 5 different values in the table T1. In STATS_TOPN, you found the 5 different values of month column is 1,2,3,4 and 5, each of them has 2,000 rows.

### Step 10. Update the values of the pid column and create an index
```
UPDATE test.T1 set pid=id;
ALTER TABLE test.T1 ADD INDEX idx_pid(pid);
ANALYZE TABLE test.T1;
```
> ```
>tidb>UPDATE test.T1 set pid=id;
>Query OK, 9999 rows affected (0.12 sec)
>Rows matched: 10000  Changed: 9999  Warnings: 0
>
>tidb>ALTER TABLE test.T1 ADD INDEX idx_pid(pid);
>Query OK, 0 rows affected (3.04 sec)
>
>tidb>ANALYZE TABLE test.T1;
>Query OK, 0 rows affected, 1 warning (0.67 sec)
> ```
### Step 11. Check the execution plans for the following queries
```
EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
```
> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.20    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.20    | cop[tikv] |                              | eq(test.t1.month, "1")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.01 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.20    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.20    | cop[tikv] |                              | eq(test.t1.month, "2")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.20    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.20    | cop[tikv] |                              | eq(test.t1.month, "3")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.01 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.20    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.20    | cop[tikv] |                              | eq(test.t1.month, "4")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.20    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.20    | cop[tikv] |                              | eq(test.t1.month, "5")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
> ```
- Which index does the query use? Why?
    - Answer: idx_pid, This is because the selectivity of idx_pid (1:1) is higher than the selectivity of idx_month (10000:5).
Selectivity is a ratio between the number of values in a column and the number of values that are distinct or unique. High selectivity means that the ratio is close to 1:1.

### Step 12. Check the statistics of table T1 again
```
SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G
SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
```
> **sample output**
> ```
>tidb>SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G
>*************************** 1. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: month
>       Is_index: 0
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 2
>    Correlation: 1
>    Load_status: allEvicted
>Total_mem_usage: 0
> Hist_mem_usage: 0
> Topn_mem_usage: 0
>  Cms_mem_usage: 0
>*************************** 2. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: pid
>       Is_index: 0
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 10000
>     Null_count: 0
>   Avg_col_size: 4.89
>    Correlation: 0.8186540409145404
>    Load_status: allEvicted
>Total_mem_usage: 0
> Hist_mem_usage: 0
> Topn_mem_usage: 0
>  Cms_mem_usage: 0
>*************************** 3. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_month
>       Is_index: 1
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 242
> Hist_mem_usage: 0
> Topn_mem_usage: 242
>  Cms_mem_usage: 0
>*************************** 4. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_pid
>       Is_index: 1
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 10000
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 42703
> Hist_mem_usage: 21671
> Topn_mem_usage: 21032
>  Cms_mem_usage: 0
>4 rows in set (0.00 sec)
>tidb>SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
>+---------+------------+----------------+-------------+----------+-------+-------+
>| Db_name | Table_name | Partition_name | Column_name | Is_index | Value | Count |
>+---------+------------+----------------+-------------+----------+-------+-------+
>| test    | T1         |                | idx_month   |        1 | 1     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 2     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 3     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 4     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 5     |  2000 |
>| test    | T1         |                | idx_pid     |        1 | 1     |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10    |     1 |
>| test    | T1         |                | idx_pid     |        1 | 100   |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1000  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10000 |     1 |
>...
>| test    | T1         |                | idx_pid     |        1 | 1445  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1446  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1447  |     1 |
>+---------+------------+----------------+-------------+----------+-------+-------+
>505 rows in set (0.00 sec)
> ```
- How is the information on the month and pid columns and the index idx_month and idx_pid recorded in the statistics?
    - Answer: In STATS_HISTOGRAMS, you found there are 5 different values in the month column of table T1, and 10,000 different values in the pid column of table T1. In STATS_TOPN, you found the 5 different values of month column is 1,2,3,4 and 5, each of them has 2,000 rows, you found the top 500 different values of pid column is 1...1447, each of them has 1 row.(Please note that TopN only records the top 500 values by default.)

### Step 13. Check the execution plan for the following query
```
EXPLAIN SELECT * FROM T1 WHERE month='6' and pid='1000';
```
> **sample output**
> ```
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>| id                            | estRows | task      | access object                    | operator info                     |
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>| IndexLookUp_11                | 0.00    | root      |                                  |                                   |
>| ├─IndexRangeScan_8(Build)     | 0.00    | cop[tikv] | table:T1, index:idx_month(month) | range:["6","6"], keep order:false |
>| └─Selection_10(Probe)         | 0.00    | cop[tikv] |                                  | eq(test.t1.pid, "1000")           |
>|   └─TableRowIDScan_9          | 0.00    | cop[tikv] | table:T1                         | keep order:false                  |
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>4 rows in set (0.00 sec)
> ```
- Which index does the query use? Why?
    - Answer: idx_month. This is because the optimizer can deduce from the TopN statistics (which was check during Step 12) that there are no rows with a value of 6 in the month column.

### Step 14. Exit the TiDB Session and insert more data into T1
```
EXIT
```
> **sample output**
> ```
>Bye
> ```
```
for i in `seq 2000`; do mysql -uroot -P4000 -h${HOST_DB1_PRIVATE_IP} -e "insert into test.T1(month,cnt,pname,pid)values('6',29,'Berlin','10001')"; done;
```
### Step 15. Connect to TiDB again
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```

### Step 16. Check the data distribution of table T1
```
SELECT COUNT(*) FROM T1;
SELECT month, COUNT(*) FROM T1 GROUP BY month;
```
> **sample output**
> ```
>tidb>SELECT COUNT(*) FROM T1;
>+----------+
>| COUNT(*) |
>+----------+
>|    12000 |
>+----------+
>1 row in set (0.01 sec)
>
>tidb>SELECT month, COUNT(*) FROM T1 GROUP BY month;
>+-------+----------+
>| month | COUNT(*) |
>+-------+----------+
>| 1     |     2000 |
>| 3     |     2000 |
>| 4     |     2000 |
>| 2     |     2000 |
>| 6     |     2000 |
>| 5     |     2000 |
>+-------+----------+
>6 rows in set (0.01 sec)
> ```
### Step 17. Check the execution plan of the following queries
```
EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='6' and pid='1000';
```
> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.24    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.20    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.24    | cop[tikv] |                              | eq(test.t1.month, "1")                  |
>|   └─TableRowIDScan_13          | 1.20    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.24    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.20    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.24    | cop[tikv] |                              | eq(test.t1.month, "2")                  |
>|   └─TableRowIDScan_13          | 1.20    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.24    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.20    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.24    | cop[tikv] |                              | eq(test.t1.month, "3")                  |
>|   └─TableRowIDScan_13          | 1.20    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.24    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.20    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.24    | cop[tikv] |                              | eq(test.t1.month, "4")                  |
>|   └─TableRowIDScan_13          | 1.20    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.01 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.24    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.20    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.24    | cop[tikv] |                              | eq(test.t1.month, "5")                  |
>|   └─TableRowIDScan_13          | 1.20    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='6' and pid='1000';
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>| id                            | estRows | task      | access object                    | operator info                     |
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>| IndexLookUp_11                | 0.00    | root      |                                  |                                   |
>| ├─IndexRangeScan_8(Build)     | 0.00    | cop[tikv] | table:T1, index:idx_month(month) | range:["6","6"], keep order:false |
>| └─Selection_10(Probe)         | 0.00    | cop[tikv] |                                  | eq(test.t1.pid, "1000")           |
>|   └─TableRowIDScan_9          | 0.00    | cop[tikv] | table:T1                         | keep order:false                  |
>+-------------------------------+---------+-----------+----------------------------------+-----------------------------------+
>4 rows in set (0.01 sec)
> ```
### Step 18. Check the statistics of table T1 again
```
SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G
SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
```
> **sample output**
> ```
>tidb>SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G
>*************************** 1. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: month
>       Is_index: 0
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 2
>    Correlation: 1
>    Load_status: allEvicted
>Total_mem_usage: 0
> Hist_mem_usage: 0
> Topn_mem_usage: 0
>  Cms_mem_usage: 0
>*************************** 2. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: pid
>       Is_index: 0
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 10000
>     Null_count: 0
>   Avg_col_size: 4.89
>    Correlation: 0.8186540409145404
>    Load_status: allEvicted
>Total_mem_usage: 0
> Hist_mem_usage: 0
> Topn_mem_usage: 0
>  Cms_mem_usage: 0
>*************************** 3. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_month
>       Is_index: 1
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 5
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 242
> Hist_mem_usage: 0
> Topn_mem_usage: 242
>  Cms_mem_usage: 0
>*************************** 4. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_pid
>       Is_index: 1
>    Update_time: 2023-12-04 15:07:28
> Distinct_count: 10000
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 42703
> Hist_mem_usage: 21671
> Topn_mem_usage: 21032
>  Cms_mem_usage: 0
>4 rows in set (0.00 sec)
>
>tidb>SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
>+---------+------------+----------------+-------------+----------+-------+-------+
>| Db_name | Table_name | Partition_name | Column_name | Is_index | Value | Count |
>+---------+------------+----------------+-------------+----------+-------+-------+
>| test    | T1         |                | idx_month   |        1 | 1     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 2     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 3     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 4     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 5     |  2000 |
>| test    | T1         |                | idx_pid     |        1 | 1     |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10    |     1 |
>| test    | T1         |                | idx_pid     |        1 | 100   |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1000  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10000 |     1 |
>...
>| test    | T1         |                | idx_pid     |        1 | 1445  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1446  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1447  |     1 |
>+---------+------------+----------------+-------------+----------+-------+-------+
>505 rows in set (0.00 sec)
> ```
- How is the information on the month and pid columns and the index idx_month and idx_pid recorded in the statistics?
    - Answer: In STATS_HISTOGRAMS, you found there are 5 different values in the month column of table T1, and 10,000 different values in the pid column of table T1. In STATS_TOPN, you found the 5 different values of month column is 1,2,3,4 and 5, each of them has 2,000 rows, you found the top 500 different values of pid column is 1...1447, each of them has 1 row.(Please note that TopN only records the top 500 values by default.)
- Are the statistics expired?
    - Answer: Yes, the statistics is the same as before(Step 12) inserting 2000 rows with value 6 of month column. Therefore, the optimizer assumes that there is no row in table T1 with month=6 and uses the idx_month index as before.
- Why would the statistics be expired?
    - Answer: This is because the system variable tidb_auto_analyze_ratio is 0.5, which means that the statistics is only automatically updated when the modified data reaches 50%. Obviously, updating 2000 rows of data does not reach 50%. You can check STATS_META to confirm.

```
SHOW STATS_META WHERE table_name='T1'\G
```
> **sample output**
> ```
>tidb>SHOW STATS_META WHERE table_name='T1'\G
>*************************** 1. row ***************************
>   Db_name: test
>   Table_name: T1
>Partition_name: 
>Update_time: 2023-12-04 15:44:29
>Modify_count: 2000
>   Row_count: 12000
>1 row in set (0.00 sec)
> ```
- How should you solve the performance issue here?
    - Answer: You need to manually collect statistics of table T1 to solve the performance problem.

### Step 19. Collect the statistic of table T1
```
ANALYZE TABLE T1;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (0.29 sec)
> ```

### Step 20. Check the execution plan for the following queries again
```
EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
EXPLAIN SELECT * FROM T1 WHERE month='6' and pid='1000';
```
> **sample output**
> ```
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='1' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "1")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='2' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "2")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='3' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "3")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='4' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "4")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='5' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "5")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
>
>tidb>EXPLAIN SELECT * FROM T1 WHERE month='6' and pid='1000';
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| id                             | estRows | task      | access object                | operator info                           |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>| IndexLookUp_15                 | 0.17    | root      |                              |                                         |
>| ├─IndexRangeScan_12(Build)     | 1.00    | cop[tikv] | table:T1, index:idx_pid(pid) | range:["1000","1000"], keep order:false |
>| └─Selection_14(Probe)          | 0.17    | cop[tikv] |                              | eq(test.t1.month, "6")                  |
>|   └─TableRowIDScan_13          | 1.00    | cop[tikv] | table:T1                     | keep order:false                        |
>+--------------------------------+---------+-----------+------------------------------+-----------------------------------------+
>4 rows in set (0.00 sec)
> ```
- Has the previous performance issue been resolved? Answer: Yes. All queries have utilized the correct indexes.
### Step 21. Check the statistics of table T1 again
```
SHOW STATS_META WHERE table_name='T1'\G
SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G
SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
```
> **sample output**
> ```
>tidb>SHOW STATS_META WHERE table_name='T1'\G;
>*************************** 1. row ***************************
>       Db_name: test
>    Table_name: T1
>Partition_name: 
>   Update_time: 2023-12-04 16:21:17
>  Modify_count: 0
>     Row_count: 12000
>1 row in set (0.00 sec)
>
>tidb>SHOW STATS_HISTOGRAMS WHERE Table_name='T1' AND Db_name='test' AND Column_name IN ('month', 'idx_month','pid','idx_pid')\G;
>*************************** 1. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: month
>       Is_index: 0
>    Update_time: 2023-12-04 16:21:17
> Distinct_count: 6
>     Null_count: 0
>   Avg_col_size: 2
>    Correlation: 1
>    Load_status: allLoaded
>Total_mem_usage: 284
> Hist_mem_usage: 0
> Topn_mem_usage: 284
>  Cms_mem_usage: 0
>*************************** 2. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: pid
>       Is_index: 0
>    Update_time: 2023-12-04 16:21:17
> Distinct_count: 9982
>     Null_count: 0
>   Avg_col_size: 5.07
>    Correlation: 0.06233068719674088
>    Load_status: allLoaded
>Total_mem_usage: 41407
> Hist_mem_usage: 20375
> Topn_mem_usage: 21032
>  Cms_mem_usage: 0
>*************************** 3. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_month
>       Is_index: 1
>    Update_time: 2023-12-04 16:21:17
> Distinct_count: 6
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 284
> Hist_mem_usage: 0
> Topn_mem_usage: 284
>  Cms_mem_usage: 0
>*************************** 4. row ***************************
>        Db_name: test
>     Table_name: T1
> Partition_name: 
>    Column_name: idx_pid
>       Is_index: 1
>    Update_time: 2023-12-04 16:21:17
> Distinct_count: 9982
>     Null_count: 0
>   Avg_col_size: 0
>    Correlation: 0
>    Load_status: allLoaded
>Total_mem_usage: 42767
> Hist_mem_usage: 21735
> Topn_mem_usage: 21032
>  Cms_mem_usage: 0
>4 rows in set (0.00 sec)
>
>tidb>SHOW STATS_TOPN WHERE Db_name = 'test' AND Table_name = 'T1' AND Column_name IN ('idx_month', 'idx_pid');
>+---------+------------+----------------+-------------+----------+-------+-------+
>| Db_name | Table_name | Partition_name | Column_name | Is_index | Value | Count |
>+---------+------------+----------------+-------------+----------+-------+-------+
>| test    | T1         |                | idx_month   |        1 | 1     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 2     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 3     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 4     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 5     |  2000 |
>| test    | T1         |                | idx_month   |        1 | 6     |  2000 |
>| test    | T1         |                | idx_pid     |        1 | 1     |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10    |     1 |
>| test    | T1         |                | idx_pid     |        1 | 100   |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1000  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10000 |     1 |
>| test    | T1         |                | idx_pid     |        1 | 10001 |  2000 |
>| test    | T1         |                | idx_pid     |        1 | 1001  |     1 |
>...
>| test    | T1         |                | idx_pid     |        1 | 1444  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1445  |     1 |
>| test    | T1         |                | idx_pid     |        1 | 1446  |     1 |
>+---------+------------+----------------+-------------+----------+-------+-------+
>506 rows in set (0.00 sec)
> ```
### Step 22. Clean up the table T1
```
DROP TABLE T1;
```
> **sample output**
> ```
>Query OK, 0 rows affected (0.53 sec)
> ```
====================================================================================================================
# TiDB SQL Tuning Lab 6: Optimizer Hints and SQL Plan Management (SPM)
## Module 1: Optimizer Hints and SQL Plan Management (SPM)
- In this exercise, you use SQL hints and SQL Plan Management (SPM) to selection a specific execution plan
### Step 1. Log in to the remote Linux
### Step 2. Log into TiDB and change to test database
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```
### Step 3. Create tables and build indexes
```
CREATE TABLE t1(a int, b int, c varchar(10), d varchar(20));
CREATE TABLE t2(a int, b int, c varchar(10), d varchar(20));
CREATE INDEX t1_idx_1 ON t1(a);
CREATE INDEX t1_idx_2 ON t1(a,b,c);
CREATE INDEX t2_idx_1 ON t2(a);
CREATE INDEX t2_idx_2 ON t2(a,b,c);
```
> **sample output**
> ```
>Query OK, 0 rows affected (1.03 sec)
> ```
```
exit
```
> **sample output**
> ```
>Bye
> ```
### Step 4. Insert the data
```
for i in `seq 10000`; do mysql -h${HOST_DB1_PRIVATE_IP} -P4000 -uroot -e "INSERT INTO test.t1 VALUES($i, FLOOR(RAND()*10000000), SUBSTRING(MD5(RAND()), 1, 5), SUBSTRING(MD5(RAND()), 1, 10))"; done;
for i in `seq 10000`; do mysql -h${HOST_DB1_PRIVATE_IP} -P4000 -uroot -e "INSERT INTO test.t2 VALUES($i, FLOOR(RAND()*10000000), SUBSTRING(MD5(RAND()), 1, 5), SUBSTRING(MD5(RAND()), 1, 10))"; done;
```

### Step 5. Connect to TiDB
```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```
### Step 6. Collect the statistic for table t1 and t2
```
ANALYZE TABLE t1;
ANALYZE TABLE t2;
```
> **sample output**
> ```
>Query OK, 0 rows affected, 1 warning (1.41 sec)
> ```

### Step 7. Check the execution plan for the following query
```
EXPLAIN SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```
> **sample output**
> ```
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>| id                             | estRows  | task      | access object                     | operator info                                                                            |
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>| Sort_8                         | 10000.00 | root      |                                   | test.t1.b, test.t2.b                                                                     |
>| └─MergeJoin_11                 | 10000.00 | root      |                                   | inner join, left key:test.t1.a, right key:test.t2.a, other cond:gt(test.t1.b, test.t2.b) |
>|   ├─IndexReader_42(Build)      | 10000.00 | root      |                                   | index:Selection_41                                                                       |
>|   │ └─Selection_41             | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.b))                                                                   |
>|   │   └─IndexFullScan_40       | 10000.00 | cop[tikv] | table:t2, index:t2_idx_2(a, b, c) | keep order:true                                                                          |
>|   └─IndexReader_39(Probe)      | 10000.00 | root      |                                   | index:Selection_38                                                                       |
>|     └─Selection_38             | 10000.00 | cop[tikv] |                                   | not(isnull(test.t1.b))                                                                   |
>|       └─IndexFullScan_37       | 10000.00 | cop[tikv] | table:t1, index:t1_idx_2(a, b, c) | keep order:true                                                                          |
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>8 rows in set (0.02 sec)
> ```
### Step 8. Check the execution plan for the following query with Hint
- Use SQL Hint, so that index t2_idx_1 and IndexJoin can be used, instead of t2_idx_2 and MergeJoin.
```
EXPLAIN SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 USE INDEX(t2_idx_1) WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```

> **sample output**
> ```
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows  | task      | access object                     | operator info                                                                                                                                        |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Sort_8                          | 10000.00 | root      |                                   | test.t1.b, test.t2.b                                                                                                                                 |
>| └─IndexJoin_18                  | 10000.00 | root      |                                   | inner join, inner:IndexLookUp_17, outer key:test.t1.a, inner key:test.t2.a, equal cond:eq(test.t1.a, test.t2.a), other cond:gt(test.t1.b, test.t2.b) |
>|   ├─IndexReader_52(Build)       | 10000.00 | root      |                                   | index:Selection_51                                                                                                                                   |
>|   │ └─Selection_51              | 10000.00 | cop[tikv] |                                   | not(isnull(test.t1.b))                                                                                                                               |
>|   │   └─IndexFullScan_50        | 10000.00 | cop[tikv] | table:t1, index:t1_idx_2(a, b, c) | keep order:false                                                                                                                                     |
>|   └─IndexLookUp_17(Probe)       | 10000.00 | root      |                                   |                                                                                                                                                      |
>|     ├─Selection_15(Build)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.a))                                                                                                                               |
>|     │ └─IndexRangeScan_13       | 10000.00 | cop[tikv] | table:t2, index:t2_idx_1(a)       | range: decided by [eq(test.t2.a, test.t1.a)], keep order:false                                                                                       |
>|     └─Selection_16(Probe)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.b))                                                                                                                               |
>|       └─TableRowIDScan_14       | 10000.00 | cop[tikv] | table:t2                          | keep order:false                                                                                                                                     |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>10 rows in set (0.00 sec)
> ```
### Step 9. Use SQL Plan Management (SPM)
- Use SQL Plan Management (SPM) to let all SQL to use index t2_idx_1 and use HashJoin all queries similar to the previous one without changing the SQL statements in the running program.

```
CREATE GLOBAL BINDING FOR 
SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b
USING
SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 USE INDEX(t2_idx_1) WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```

> **sample output**
> ```
>Query OK, 0 rows affected (0.01 sec)
> ```

### Step 10. Check the execution plan for the query in Step 7 again

```
EXPLAIN SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```
> **sample output**
> ```
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows  | task      | access object                     | operator info                                                                                                                                        |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Sort_8                          | 10000.00 | root      |                                   | test.t1.b, test.t2.b                                                                                                                                 |
>| └─IndexJoin_18                  | 10000.00 | root      |                                   | inner join, inner:IndexLookUp_17, outer key:test.t1.a, inner key:test.t2.a, equal cond:eq(test.t1.a, test.t2.a), other cond:gt(test.t1.b, test.t2.b) |
>|   ├─IndexReader_52(Build)       | 10000.00 | root      |                                   | index:Selection_51                                                                                                                                   |
>|   │ └─Selection_51              | 10000.00 | cop[tikv] |                                   | not(isnull(test.t1.b))                                                                                                                               |
>|   │   └─IndexFullScan_50        | 10000.00 | cop[tikv] | table:t1, index:t1_idx_2(a, b, c) | keep order:false                                                                                                                                     |
>|   └─IndexLookUp_17(Probe)       | 10000.00 | root      |                                   |                                                                                                                                                      |
>|     ├─Selection_15(Build)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.a))                                                                                                                               |
>|     │ └─IndexRangeScan_13       | 10000.00 | cop[tikv] | table:t2, index:t2_idx_1(a)       | range: decided by [eq(test.t2.a, test.t1.a)], keep order:false                                                                                       |
>|     └─Selection_16(Probe)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.b))                                                                                                                               |
>|       └─TableRowIDScan_14       | 10000.00 | cop[tikv] | table:t2                          | keep order:false                                                                                                                                     |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>10 rows in set (0.00 sec) 
> ```

### Step 11. Open Terminal 2 and connect to TiDB

```
mysql -h ${HOST_DB1_PRIVATE_IP} --port 4000 -u root
```

### Step 12. In Terminal 2, make test the current database
```
USE test;
```
> **sample output**
> ```
>Database changed
> ```

### Step 13. In Terminal 2, check the execution plan for the query in Step 7 again
```
EXPLAIN SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```

> **sample output**
> ```
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| id                              | estRows  | task      | access object                     | operator info                                                                                                                                        |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>| Sort_8                          | 10000.00 | root      |                                   | test.t1.b, test.t2.b                                                                                                                                 |
>| └─IndexJoin_18                  | 10000.00 | root      |                                   | inner join, inner:IndexLookUp_17, outer key:test.t1.a, inner key:test.t2.a, equal cond:eq(test.t1.a, test.t2.a), other cond:gt(test.t1.b, test.t2.b) |
>|   ├─IndexReader_52(Build)       | 10000.00 | root      |                                   | index:Selection_51                                                                                                                                   |
>|   │ └─Selection_51              | 10000.00 | cop[tikv] |                                   | not(isnull(test.t1.b))                                                                                                                               |
>|   │   └─IndexFullScan_50        | 10000.00 | cop[tikv] | table:t1, index:t1_idx_2(a, b, c) | keep order:false                                                                                                                                     |
>|   └─IndexLookUp_17(Probe)       | 10000.00 | root      |                                   |                                                                                                                                                      |
>|     ├─Selection_15(Build)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.a))                                                                                                                               |
>|     │ └─IndexRangeScan_13       | 10000.00 | cop[tikv] | table:t2, index:t2_idx_1(a)       | range: decided by [eq(test.t2.a, test.t1.a)], keep order:false                                                                                       |
>|     └─Selection_16(Probe)       | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.b))                                                                                                                               |
>|       └─TableRowIDScan_14       | 10000.00 | cop[tikv] | table:t2                          | keep order:false                                                                                                                                     |
>+---------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------------------------------------------------------------------+
>10 rows in set (0.00 sec)
> ```
### Step 14. In any linux terminal, check the global bindings
```
SHOW GLOBAL BINDINGS WHERE ORIGINAL_SQL LIKE '%t1%t2%'\G
```
> **sample output**
> ```
>*************************** 1. row ***************************
>Original_sql: select `t1` . `a` , `t1` . `b` , `t1` . `c` , `t2` . `b` from ( `test` . `t1` ) join `test` . `t2` where `t1` . `a` = `t2` . `a` and `t1` . `b` > `t2` . `b` order by `t1` . `b` , `t2` . `b`
>    Bind_sql: SELECT `t1`.`a`,`t1`.`b`,`t1`.`c`,`t2`.`b` FROM (`test`.`t1`) JOIN `test`.`t2` USE INDEX (`t2_idx_1`) WHERE `t1`.`a` = `t2`.`a` AND `t1`.`b` > `t2`.`b` ORDER BY `t1`.`b`,`t2`.`b`
>  Default_db: test
>      Status: enabled
> Create_time: 2023-12-04 12:22:50.927
> Update_time: 2023-12-04 12:22:50.927
>     Charset: utf8
>   Collation: utf8_general_ci
>      Source: manual
>  Sql_digest: fc14be07f408211ab8bcba115f45f2c602250e16e3d540960fceb7238c4e9d87
> Plan_digest: 
>1 row in set (0.00 sec)
> ```

### Step 15. In any linux terminal, drop the global bindings
```
DROP GLOBAL BINDING FOR SQL DIGEST 'fc14be07f408211ab8bcba115f45f2c602250e16e3d540960fceb7238c4e9d87';
DROP SESSION BINDING FOR SQL DIGEST 'fc14be07f408211ab8bcba115f45f2c602250e16e3d540960fceb7238c4e9d87';
```
> **sample output**
> ```
>Query OK, 1 row affected (0.01 sec)
> ```

### Step 16. In any linux terminal, check the execution plan for the query in Step 7 again
```
EXPLAIN SELECT t1.a, t1.b, t1.c, t2.b FROM t1, t2 WHERE t1.a = t2.a AND t1.b > t2.b ORDER BY t1.b, t2.b;
```
> **sample output**
> ```
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>| id                             | estRows  | task      | access object                     | operator info                                                                            |
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>| Sort_8                         | 10000.00 | root      |                                   | test.t1.b, test.t2.b                                                                     |
>| └─MergeJoin_11                 | 10000.00 | root      |                                   | inner join, left key:test.t1.a, right key:test.t2.a, other cond:gt(test.t1.b, test.t2.b) |
>|   ├─IndexReader_42(Build)      | 10000.00 | root      |                                   | index:Selection_41                                                                       |
>|   │ └─Selection_41             | 10000.00 | cop[tikv] |                                   | not(isnull(test.t2.b))                                                                   |
>|   │   └─IndexFullScan_40       | 10000.00 | cop[tikv] | table:t2, index:t2_idx_2(a, b, c) | keep order:true                                                                          |
>|   └─IndexReader_39(Probe)      | 10000.00 | root      |                                   | index:Selection_38                                                                       |
>|     └─Selection_38             | 10000.00 | cop[tikv] |                                   | not(isnull(test.t1.b))                                                                   |
>|       └─IndexFullScan_37       | 10000.00 | cop[tikv] | table:t1, index:t1_idx_2(a, b, c) | keep order:true                                                                          |
>+--------------------------------+----------+-----------+-----------------------------------+------------------------------------------------------------------------------------------+
>8 rows in set (0.00 sec)
> ```
### Step 17. Clean up the tables
```
DROP TABLE t1,t2;
```
> **sample output**
> ```
>Query OK, 0 rows affected (1.06 sec)
> ```