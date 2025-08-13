# TRAINING TIDB SQL TUNING
================================================================================================

# TiDB SQL Tuning Lab 1: Clustered and Non-Clustered Indexes
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