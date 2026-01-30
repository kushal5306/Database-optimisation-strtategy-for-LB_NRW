# 🚀 Spatial Partitioning of Lb_nrw_2023 with PostGIS using `tile_id`

**Project: Partition ****\`\`**** Table by Spatial Tiles for Fast Querying**

---

## 📌 Objective

Improve query performance on the `lb_nrw_2023` table  by **partitioning spatial data** using a computed `tile_id` based on each geometry’s bounding box. This enables **efficient spatial filtering** and **partition pruning** in PostGIS.

---

## 📁 Table of Contents

1. [Overview](#1-overview)
2. [Assumptions](#2-assumptions)
3. [Step-by-Step Instructions](#3-step-by-step-instructions)
   - 3.1 [Add ](#31-add-tile_id-to-original-table)[`tile_id`](#31-add-tile_id-to-original-table)
   - 3.2 [Create Partitioned Parent Table](#32-create-partitioned-parent-table)
   - 3.3 [Create Default Partition](#33-create-default-partition)
   - 3.4 [Create Child Partitions Automatically](#34-auto-create-child-partitions)
   - 3.5 [Insert Data Into Partitioned Table](#35-insert-data-into-partitions)
4. [How to Query Efficiently](#4-how-to-query-efficiently)
5. [Tips and Best Practices](#5-tips-and-best-practices)

---

## 1. 📖 Overview

The original table `lb_nrw_2023` stores spatial data with a `geom` column. To accelerate spatial queries, especially those with bounding polygons or points, we:

- Compute  per geometry using its bounding box.
- Partition the table using `tile_id` (LIST partitioning).
- Route queries to only the relevant tile(s) based on user-provided coordinates.

---

## 2. 🧠 Assumptions

| Parameter       | Value                       |
| --------------- | --------------------------- |
| Original Table  | `lb_nrw_2023`               |
| Geometry Column | `geom`                      |
| Geometry Type   | `MultiPolygon` (adjustable) |
| SRID            | `25832` (projected UTM)     |
| Partition Key   | `tile_id` (Text)            |

---

## 3. 🛡️ Step-by-Step Instructions

### 3.1 ➕ Add `tile_id` to Original Table

```sql
ALTER TABLE public.lb_nrw_2023 ADD COLUMN IF NOT EXISTS tile_id TEXT;

UPDATE public.lb_nrw_2023
SET tile_id = CONCAT(
    FLOOR(ST_XMin(geom) / 50000)::int, '_',
    FLOOR(ST_YMin(geom) / 50000)::int
);
```

- This creates a **50km × 50km tile grid**.
- Each geometry is assigned a `tile_id` like `8_112`.

---

### 3.2 🏧 Create Partitioned Parent Table

> **Note:** Matching column names and types from  original table.

```sql
CREATE TABLE public.lb_nrw_2023_partitioned (
    fid      integer,
    id       bigint,
    klasse   bigint,
    lbklasse character varying,
    datum    character varying,
    "1mean"  double precision,
    "2mean"  double precision,
    "3mean"  double precision,
    "4mean"  double precision,
    "5mean"  double precision,
    "6mean"  double precision,
    "7mean"  double precision,
    "8mean"  double precision,
    "9mean"  double precision,
    "10mean" double precision,
    oid_     character varying,
    geom     geometry(MultiPolygon, 25832),
    tile_id  text NOT NULL
) PARTITION BY LIST (tile_id);
```

---

### 3.3 🧯 Create Default Partition  as partitioned cannot be created from original table
```sql
CREATE TABLE public.lb_nrw_2023_partition_default
PARTITION OF public.lb_nrw_2023_partitioned
DEFAULT;
```

Catches records with missing or unexpected `tile_id`.

---

### 3.4 🧬 Auto-Create Child Partitions

Run this PL/pgSQL block in pgAdmin Query Tool:

```sql
DO $$
DECLARE
    r record;
    part_name text;
BEGIN
    FOR r IN
        SELECT DISTINCT tile_id FROM public.lb_nrw_2023 WHERE tile_id IS NOT NULL
    LOOP
        part_name := 'lb_nrw_2023_p_' || regexp_replace(r.tile_id, '[^a-zA-Z0-9_]', '_', 'g');

        EXECUTE format(
            'CREATE TABLE IF NOT EXISTS public.%I PARTITION OF public.lb_nrw_2023_partitioned FOR VALUES IN (%L);',
            part_name, r.tile_id
        );

        EXECUTE format(
            'CREATE INDEX IF NOT EXISTS %I ON public.%I USING GIST (geom);',
            part_name || '_geom_gist', part_name
        );
    END LOOP;
END$$;
```

- Each tile gets its own partition table.
- A `GIST` spatial index is created on each child.

---

### 3.5 📤 Insert Data into Partitioned Table

```sql
INSERT INTO public.lb_nrw_2023_partitioned (
    fid, id, klasse, lbklasse, datum,
    "1mean", "2mean", "3mean", "4mean", "5mean",
    "6mean", "7mean", "8mean", "9mean", "10mean",
    oid_, geom, tile_id
)
SELECT
    fid, id, klasse, lbklasse, datum,
    "1mean", "2mean", "3mean", "4mean", "5mean",
    "6mean", "7mean", "8mean", "9mean", "10mean",
    oid_, geom, tile_id
FROM public.lb_nrw_2023;
```

---

## 4. 🔍 How to Query Efficiently

### ✅ Simple Intersection Query

```sql
WITH input AS (
  SELECT ST_Transform(
           ST_GeomFromText('POLYGON((7.18 50.78, 7.18 50.79, 7.19 50.79, 7.18 50.78))', 4326),
           25832
         ) AS geom
),
tiles AS (
  SELECT CONCAT(
           FLOOR(ST_XMin(geom)/50000)::int, '_',
           FLOOR(ST_YMin(geom)/50000)::int
         ) AS tile_id,
         geom
  FROM input
)
SELECT l.*
FROM public.lb_nrw_2023_partitioned l
JOIN tiles t ON l.tile_id = t.tile_id
WHERE ST_Intersects(l.geom, t.geom);
```

---

## 5. 💡 Tips and Best Practices

| Tip                                       | Description                                     |
| ----------------------------------------- | ----------------------------------------------- |
| Use `tile_id` filtering first             | Helps PostgreSQL prune partitions early         |
| Use `GIST` index on `geom`                | Enables fast spatial filtering inside each tile |
| Add a `DEFAULT` partition                 | Avoid insert errors                             |
| Use bounding boxes if polygon spans tiles | Pre-compute `tile_id`s that intersect           |

---

## ✅ Final Notes

- This setup scales well with **hundreds of partitions**.
- You can automate tile generation and indexing.
- Query performance improves dramatically when filtering by `tile_id`.

---
<!-- Sometimes the pgAdmin4 crashes so please run db_improv.py -->
<!-- Also the implementation doesnt run on docker, after lot of attempts we concluded docker slows it down -->
