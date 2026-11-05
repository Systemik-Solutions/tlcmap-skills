# A1 API v2: EXPLAIN ANALYZE plans

Appendix to the [acceptance record](A1-API-V2.md). Measured 2026-11-05 on `temp_tlcmap` (PostgreSQL
14.5, `shared_buffers` 128MB, PostGIS), application branch `a1-api-v2` at b1aa4c6, with the three
`2026_11_10_*` migrations applied.

Each request was run through the application with a query listener (a throwaway script outside the
repository) that recorded every SQL statement and its bindings, in order. Each statement was then
run in `psql` as `EXPLAIN (ANALYZE, BUFFERS)` with the bindings written in as literals. Every
statement is a `SELECT`; nothing was written to `temp_tlcmap`. The "reported" time is what the
application measured for the statement when the request ran; the plan's execution time is a separate
run, so the two differ (cache state, and `EXPLAIN` instrumentation).


## 1. The unfiltered catalogue

Request: `GET /api/v2/layers`

### Statement 0 (reported 2.48 ms; plan execution 0.025 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.012..0.013 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.765 ms
 Execution Time: 0.025 ms
(6 rows)
```

### Statement 1 (reported 3.86 ms; plan execution 0.414 ms)

```sql
select "d".*, d.id as _k, d.id as _id from (select d.id, d.name, d.description, d.creator, rt.type as record_type, CAST(NULL AS bigint) as matching_count, (nullif(btrim(d.license, E' \t\r\n'), '') IS NOT NULL) as has_license, (nullif(btrim(d.warning, E' \t\r\n'), '') IS NOT NULL) as has_warning, count(*) over () as _total from "tlcmap"."dataset" as "d" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "d"."recordtype_id" where d.public = true) as "d" order by (d.id) ASC, (d.id) ASC limit 101;
```

```text
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=23.08..23.33 rows=101 width=423) (actual time=0.318..0.326 rows=101 loops=1)
   Buffers: shared hit=13
   ->  Sort  (cost=23.08..23.46 rows=152 width=423) (actual time=0.317..0.320 rows=101 loops=1)
         Sort Key: d.id
         Sort Method: quicksort  Memory: 76kB
         Buffers: shared hit=13
         ->  Subquery Scan on d  (cost=1.20..17.57 rows=152 width=423) (actual time=0.186..0.253 rows=155 loops=1)
               Buffers: shared hit=10
               ->  WindowAgg  (cost=1.20..16.05 rows=152 width=407) (actual time=0.185..0.243 rows=155 loops=1)
                     Buffers: shared hit=10
                     ->  Hash Left Join  (cost=1.20..12.63 rows=152 width=630) (actual time=0.040..0.128 rows=155 loops=1)
                           Hash Cond: (d_1.recordtype_id = rt.id)
                           Buffers: shared hit=10
                           ->  Seq Scan on dataset d_1  (cost=0.00..10.84 rows=152 width=420) (actual time=0.017..0.069 rows=155 loops=1)
                                 Filter: public
                                 Rows Removed by Filter: 32
                                 Buffers: shared hit=9
                           ->  Hash  (cost=1.09..1.09 rows=9 width=226) (actual time=0.013..0.014 rows=9 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 Buffers: shared hit=1
                                 ->  Seq Scan on recordtype rt  (cost=0.00..1.09 rows=9 width=226) (actual time=0.008..0.008 rows=9 loops=1)
                                       Buffers: shared hit=1
 Planning:
   Buffers: shared hit=136
 Planning Time: 2.413 ms
 Execution Time: 0.414 ms
(26 rows)
```

### Statement 2 (reported 0.22 ms; plan execution 0.012 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.006..0.007 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.043 ms
 Execution Time: 0.012 ms
(4 rows)
```

### Statement 3 (reported 268.95 ms; plan execution 319.310 ms)

```sql
select r.dataset_id, count(*) as record_count, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date from (select "i"."dataset_id", "i"."geom", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and "i"."dataset_id" in (78, 80, 81, 84, 87, 89, 90, 99, 102, 112, 113, 118, 127, 132, 135, 140, 141, 150, 151, 152, 153, 154, 158, 159, 163, 164, 165, 167, 170, 172, 176, 192, 194, 198, 206, 208, 209, 210, 211, 213, 218, 219, 222, 223, 230, 234, 235, 236, 237, 238, 239, 240, 241, 242, 247, 256, 257, 258, 259, 260, 277, 278, 284, 287, 288, 292, 295, 299, 301, 302, 303, 304, 306, 307, 314, 315, 324, 325, 326, 327, 328, 329, 330, 361, 378, 379, 380, 383, 384, 385, 386, 387, 388, 390, 391, 392, 394, 395, 398, 399) offset 0) as "r" group by "r"."dataset_id";
```

```text
                                                                                                                                                                                                                                    QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=103304.78..103358.96 rows=84 width=88) (actual time=304.074..313.583 rows=99 loops=1)
   Group Key: i.dataset_id
   Batches: 1  Memory Usage: 48kB
   Buffers: shared hit=10758
   ->  Gather  (cost=2100.95..97503.86 rows=35698 width=74) (actual time=4.399..293.212 rows=34373 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=10758
         ->  Hash Left Join  (cost=1100.95..92934.06 rows=14874 width=74) (actual time=2.767..228.892 rows=11458 loops=3)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=10758
               ->  Parallel Bitmap Heap Scan on dataitem i  (cost=1087.81..77858.45 rows=15105 width=67) (actual time=1.112..19.882 rows=11458 loops=3)
                     Recheck Cond: (dataset_id = ANY ('{78,80,81,84,87,89,90,99,102,112,113,118,127,132,135,140,141,150,151,152,153,154,158,159,163,164,165,167,170,172,176,192,194,198,206,208,209,210,211,213,218,219,222,223,230,234,235,236,237,238,239,240,241,242,247,256,257,258,259,260,277,278,284,287,288,292,295,299,301,302,303,304,306,307,314,315,324,325,326,327,328,329,330,361,378,379,380,383,384,385,386,387,388,390,391,392,394,395,398,399}'::bigint[]))
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=2499
                     Buffers: shared hit=9499
                     ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1078.75 rows=36252 width=0) (actual time=2.279..2.280 rows=49867 loops=1)
                           Index Cond: (dataset_id = ANY ('{78,80,81,84,87,89,90,99,102,112,113,118,127,132,135,140,141,150,151,152,153,154,158,159,163,164,165,167,170,172,176,192,194,198,206,208,209,210,211,213,218,219,222,223,230,234,235,236,237,238,239,240,241,242,247,256,257,258,259,260,277,278,284,287,288,292,295,299,301,302,303,304,306,307,314,315,324,325,326,327,328,329,330,361,378,379,380,383,384,385,386,387,388,390,391,392,394,395,398,399}'::bigint[]))
                           Buffers: shared hit=380
               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.380..0.381 rows=187 loops=3)
                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                     Buffers: shared hit=27
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.314..0.359 rows=187 loops=3)
                           Buffers: shared hit=27
 Planning:
   Buffers: shared hit=231
 Planning Time: 5.340 ms
 Execution Time: 319.310 ms
(29 rows)
```


## 2. The catalogue, Hunter Valley bounding box

Request: `GET /api/v2/layers?bbox=150.8,-33.1,151.4,-32.6`

### Statement 0 (reported 4.71 ms; plan execution 0.027 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.013..0.014 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.428 ms
 Execution Time: 0.027 ms
(6 rows)
```

### Statement 1 (reported 44.28 ms; plan execution 7.207 ms)

```sql
select "d".*, d.id as _k, d.id as _id from (select d.id, d.name, d.description, d.creator, rt.type as record_type, m.matching_count as matching_count, (nullif(btrim(d.license, E' \t\r\n'), '') IS NOT NULL) as has_license, (nullif(btrim(d.warning, E' \t\r\n'), '') IS NOT NULL) as has_warning, count(*) over () as _total from "tlcmap"."dataset" as "d" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "d"."recordtype_id" inner join (select i.dataset_id, count(*) as matching_count from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and i.geom && ST_MakeEnvelope(CAST('150.8' AS double precision), CAST('-33.1' AS double precision), CAST('151.4' AS double precision), CAST('-32.6' AS double precision), 4326) and "i"."dataset_id" is not null group by "i"."dataset_id") as "m" on "m"."dataset_id" = "d"."id" where d.public = true) as "d" order by (d.id) ASC, (d.id) ASC limit 101;
```

```text
                                                                                                                                                                QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=1827.38..1827.52 rows=56 width=423) (actual time=5.988..5.993 rows=8 loops=1)
   Buffers: shared hit=415 read=88
   ->  Sort  (cost=1827.38..1827.52 rows=56 width=423) (actual time=5.987..5.990 rows=8 loops=1)
         Sort Key: d.id
         Sort Method: quicksort  Memory: 28kB
         Buffers: shared hit=415 read=88
         ->  Subquery Scan on d  (cost=1812.47..1825.75 rows=56 width=423) (actual time=5.943..5.970 rows=8 loops=1)
               Buffers: shared hit=412 read=88
               ->  WindowAgg  (cost=1812.47..1825.19 rows=56 width=407) (actual time=5.942..5.968 rows=8 loops=1)
                     Buffers: shared hit=412 read=88
                     ->  Hash Left Join  (cost=1812.47..1823.93 rows=56 width=638) (actual time=5.883..5.925 rows=8 loops=1)
                           Hash Cond: (d_1.recordtype_id = rt.id)
                           Buffers: shared hit=412 read=88
                           ->  Hash Join  (cost=1811.26..1822.51 rows=56 width=428) (actual time=5.850..5.891 rows=8 loops=1)
                                 Hash Cond: (d_1.id = m.dataset_id)
                                 Buffers: shared hit=411 read=88
                                 ->  Seq Scan on dataset d_1  (cost=0.00..10.84 rows=152 width=420) (actual time=0.008..0.037 rows=155 loops=1)
                                       Filter: public
                                       Rows Removed by Filter: 32
                                       Buffers: shared hit=9
                                 ->  Hash  (cost=1810.41..1810.41 rows=68 width=16) (actual time=5.826..5.828 rows=8 loops=1)
                                       Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                       Buffers: shared hit=402 read=88
                                       ->  Subquery Scan on m  (cost=1808.02..1810.41 rows=68 width=16) (actual time=5.808..5.823 rows=8 loops=1)
                                             Buffers: shared hit=402 read=88
                                             ->  GroupAggregate  (cost=1808.02..1809.73 rows=68 width=16) (actual time=5.808..5.822 rows=8 loops=1)
                                                   Group Key: i.dataset_id
                                                   Buffers: shared hit=402 read=88
                                                   ->  Sort  (cost=1808.02..1808.36 rows=138 width=8) (actual time=5.801..5.807 rows=104 loops=1)
                                                         Sort Key: i.dataset_id
                                                         Sort Method: quicksort  Memory: 29kB
                                                         Buffers: shared hit=402 read=88
                                                         ->  Hash Left Join  (cost=1255.32..1803.11 rows=138 width=8) (actual time=5.323..5.747 rows=104 loops=1)
                                                               Hash Cond: (i.dataset_id = d_2.id)
                                                               Filter: ((i.dataset_id IS NULL) OR d_2.public)
                                                               Rows Removed by Filter: 114
                                                               Buffers: shared hit=402 read=88
                                                               ->  Bitmap Heap Scan on dataitem i  (cost=1242.18..1789.61 rows=140 width=16) (actual time=4.840..5.550 rows=218 loops=1)
                                                                     Recheck Cond: ((geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry) AND (dataset_id IS NOT NULL))
                                                                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                                                                     Heap Blocks: exact=199
                                                                     Buffers: shared hit=393 read=88
                                                                     ->  BitmapAnd  (cost=1242.18..1242.18 rows=140 width=0) (actual time=4.793..4.793 rows=0 loops=1)
                                                                           Buffers: shared hit=194 read=88
                                                                           ->  Bitmap Index Scan on v2_dataitem_geom_gist  (cost=0.00..60.20 rows=1589 width=0) (actual time=1.411..1.411 rows=1432 loops=1)
                                                                                 Index Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                                                                                 Buffers: shared read=88
                                                                           ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1181.66 rows=60431 width=0) (actual time=3.237..3.237 rows=82162 loops=1)
                                                                                 Index Cond: (dataset_id IS NOT NULL)
                                                                                 Buffers: shared hit=194
                                                               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.135..0.135 rows=187 loops=1)
                                                                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                                                     Buffers: shared hit=9
                                                                     ->  Seq Scan on dataset d_2  (cost=0.00..10.84 rows=184 width=9) (actual time=0.036..0.106 rows=187 loops=1)
                                                                           Buffers: shared hit=9
                           ->  Hash  (cost=1.09..1.09 rows=9 width=226) (actual time=0.021..0.021 rows=9 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 Buffers: shared hit=1
                                 ->  Seq Scan on recordtype rt  (cost=0.00..1.09 rows=9 width=226) (actual time=0.014..0.015 rows=9 loops=1)
                                       Buffers: shared hit=1
 Planning:
   Buffers: shared hit=321 read=1
 Planning Time: 9.899 ms
 Execution Time: 7.207 ms
(64 rows)
```

### Statement 2 (reported 0.72 ms; plan execution 0.014 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.006..0.007 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.042 ms
 Execution Time: 0.014 ms
(4 rows)
```

### Statement 3 (reported 512.32 ms; plan execution 297.249 ms)

```sql
select r.dataset_id, count(*) as record_count, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date from (select "i"."dataset_id", "i"."geom", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and "i"."dataset_id" in (84, 102, 140, 170, 206, 379, 461, 462) offset 0) as "r" group by "r"."dataset_id";
```

```text
                                                                       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=108395.89..108450.07 rows=84 width=88) (actual time=289.098..295.455 rows=8 loops=1)
   Group Key: i.dataset_id
   Batches: 1  Memory Usage: 24kB
   Buffers: shared hit=8306
   ->  Gather  (cost=1827.97..101965.76 rows=39570 width=74) (actual time=4.117..280.333 rows=40469 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=8306
         ->  Hash Left Join  (cost=827.97..97008.76 rows=16488 width=74) (actual time=3.217..252.164 rows=13490 loops=3)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=8306
               ->  Parallel Bitmap Heap Scan on dataitem i  (cost=814.83..80298.70 rows=16744 width=67) (actual time=0.839..12.717 rows=13490 loops=3)
                     Recheck Cond: (dataset_id = ANY ('{84,102,140,170,206,379,461,462}'::bigint[]))
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=2431
                     Buffers: shared hit=7071
                     ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..804.78 rows=40185 width=0) (actual time=1.680..1.680 rows=52281 loops=1)
                           Index Cond: (dataset_id = ANY ('{84,102,140,170,206,379,461,462}'::bigint[]))
                           Buffers: shared hit=120
               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.488..0.488 rows=187 loops=3)
                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                     Buffers: shared hit=27
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.413..0.462 rows=187 loops=3)
                           Buffers: shared hit=27
 Planning:
   Buffers: shared hit=65
 Planning Time: 0.380 ms
 Execution Time: 297.249 ms
(29 rows)
```


## 3. Place search, Hunter Valley bounding box

Request: `GET /api/v2/places?bbox=150.8,-33.1,151.4,-32.6`

### Statement 0 (reported 3.29 ms; plan execution 0.022 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.011..0.012 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.344 ms
 Execution Time: 0.022 ms
(6 rows)
```

### Statement 1 (reported 0.53 ms; plan execution 0.008 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.004..0.005 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.022 ms
 Execution Time: 0.008 ms
(4 rows)
```

### Statement 2 (reported 29.44 ms; plan execution 15.048 ms)

```sql
select count(*) as total, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, count(*) filter (where r.geom is null) as no_coordinates, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where ((NOT r.sb AND r.s IS NULL) OR (NOT r.eb AND r.e IS NULL))) as unparseable_dates, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date, count(*) filter (where r.datasource_id = 1) as source_1, count(*) filter (where r.datasource_id = 2) as source_2, count(*) filter (where r.datasource_id = 3) as source_3, count(*) filter (where r.datasource_id = 4) as source_4 from (select "i"."geom", "i"."datasource_id", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and i.geom && ST_MakeEnvelope(CAST('150.8' AS double precision), CAST('-33.1' AS double precision), CAST('151.4' AS double precision), CAST('-32.6' AS double precision), 4326) offset 0) as "r" limit 1;
```

```text
                                                                                                                           QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=7719.49..7720.01 rows=1 width=128) (actual time=12.122..12.125 rows=1 loops=1)
   Buffers: shared hit=723 read=749
   ->  Aggregate  (cost=7719.49..7720.01 rows=1 width=128) (actual time=12.121..12.123 rows=1 loops=1)
         Buffers: shared hit=723 read=749
         ->  Hash Left Join  (cost=73.74..7429.84 rows=1565 width=74) (actual time=0.888..11.645 rows=1318 loops=1)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Rows Removed by Filter: 114
               Buffers: shared hit=723 read=749
               ->  Bitmap Heap Scan on dataitem i  (cost=60.60..5831.87 rows=1589 width=75) (actual time=0.810..8.309 rows=1432 loops=1)
                     Recheck Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=1106
                     Buffers: shared hit=445 read=749
                     ->  Bitmap Index Scan on v2_dataitem_geom_gist  (cost=0.00..60.20 rows=1589 width=0) (actual time=0.676..0.676 rows=1432 loops=1)
                           Index Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                           Buffers: shared hit=88
               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.069..0.071 rows=187 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                     Buffers: shared hit=9
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.009..0.055 rows=187 loops=1)
                           Buffers: shared hit=9
 Planning:
   Buffers: shared hit=421
 Planning Time: 10.208 ms
 Execution Time: 15.048 ms
(26 rows)
```

### Statement 3 (reported 5.69 ms; plan execution 4.024 ms)

```sql
select i.dataset_id as layer_id, d.name as name, count(*) as count, CASE WHEN btrim(d.warning, E' ') = '' THEN NULL ELSE d.warning END as warning from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and i.geom && ST_MakeEnvelope(CAST('150.8' AS double precision), CAST('-33.1' AS double precision), CAST('151.4' AS double precision), CAST('-32.6' AS double precision), 4326) and "i"."dataset_id" is not null group by "i"."dataset_id", "d"."name", "d"."warning" order by "i"."dataset_id" asc;
```

```text
                                                                                                                                        QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=1808.02..1811.81 rows=138 width=299) (actual time=3.941..3.961 rows=8 loops=1)
   Group Key: i.dataset_id, d.name, d.warning
   Buffers: shared hit=499
   ->  Sort  (cost=1808.02..1808.36 rows=138 width=259) (actual time=3.938..3.941 rows=104 loops=1)
         Sort Key: i.dataset_id, d.name, d.warning
         Sort Method: quicksort  Memory: 52kB
         Buffers: shared hit=499
         ->  Hash Left Join  (cost=1255.32..1803.11 rows=138 width=259) (actual time=3.795..3.893 rows=104 loops=1)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Rows Removed by Filter: 114
               Buffers: shared hit=490
               ->  Bitmap Heap Scan on dataitem i  (cost=1242.18..1789.61 rows=140 width=16) (actual time=3.630..3.783 rows=218 loops=1)
                     Recheck Cond: ((geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry) AND (dataset_id IS NOT NULL))
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=199
                     Buffers: shared hit=481
                     ->  BitmapAnd  (cost=1242.18..1242.18 rows=140 width=0) (actual time=3.609..3.610 rows=0 loops=1)
                           Buffers: shared hit=282
                           ->  Bitmap Index Scan on v2_dataitem_geom_gist  (cost=0.00..60.20 rows=1589 width=0) (actual time=0.452..0.453 rows=1432 loops=1)
                                 Index Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                                 Buffers: shared hit=88
                           ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1181.66 rows=60431 width=0) (actual time=3.048..3.049 rows=82162 loops=1)
                                 Index Cond: (dataset_id IS NOT NULL)
                                 Buffers: shared hit=194
               ->  Hash  (cost=10.84..10.84 rows=184 width=260) (actual time=0.074..0.075 rows=187 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 30kB
                     Buffers: shared hit=9
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=260) (actual time=0.010..0.049 rows=187 loops=1)
                           Buffers: shared hit=9
 Planning:
   Buffers: shared hit=31
 Planning Time: 0.313 ms
 Execution Time: 4.024 ms
(34 rows)
```

### Statement 4 (reported 6.3 ms; plan execution 3.661 ms)

```sql
select i.id, i.uid, i.title, i.placename, i.description, i.datasource_id, i.dataset_id, rt.type as record_type, i.state, i.lga, i.parish, i.feature_term, i.datestart, i.dateend, tlcmap.v2_udate(i.datestart) as udatestart, tlcmap.v2_udate(i.dateend) as udateend, i.source, i.external_url, i.image_path, i.glycerine_url, i.created_at, i.updated_at, i.extended_data, i.latitude, i.longitude, i.id as _k, i.id as _id from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where ((i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and i.geom && ST_MakeEnvelope(CAST('150.8' AS double precision), CAST('-33.1' AS double precision), CAST('151.4' AS double precision), CAST('-32.6' AS double precision), 4326)) order by (i.id) ASC, (i.id) ASC limit 501;
```

```text
                                                                                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=5964.70..6221.46 rows=501 width=1932) (actual time=2.607..3.529 rows=501 loops=1)
   Buffers: shared hit=1207
   ->  Result  (cost=5964.70..6766.76 rows=1565 width=1932) (actual time=2.606..3.508 rows=501 loops=1)
         Buffers: shared hit=1207
         ->  Sort  (cost=5964.70..5968.61 rows=1565 width=1916) (actual time=2.533..2.556 rows=501 loops=1)
               Sort Key: i.id
               Sort Method: top-N heapsort  Memory: 319kB
               Buffers: shared hit=1207
               ->  Nested Loop Left Join  (cost=73.88..5886.70 rows=1565 width=1916) (actual time=0.616..2.025 rows=1318 loops=1)
                     Buffers: shared hit=1207
                     ->  Hash Left Join  (cost=73.74..5849.19 rows=1565 width=1690) (actual time=0.607..1.692 rows=1318 loops=1)
                           Hash Cond: (i.dataset_id = d.id)
                           Filter: ((i.dataset_id IS NULL) OR d.public)
                           Rows Removed by Filter: 114
                           Buffers: shared hit=1203
                           ->  Bitmap Heap Scan on dataitem i  (cost=60.60..5831.87 rows=1589 width=1690) (actual time=0.561..1.466 rows=1432 loops=1)
                                 Recheck Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                                 Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                                 Heap Blocks: exact=1106
                                 Buffers: shared hit=1194
                                 ->  Bitmap Index Scan on v2_dataitem_geom_gist  (cost=0.00..60.20 rows=1589 width=0) (actual time=0.467..0.467 rows=1432 loops=1)
                                       Index Cond: (geom && '0103000020E610000001000000050000009A99999999D96240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC4C40C0CDCCCCCCCCEC6240CDCCCCCCCC8C40C09A99999999D96240CDCCCCCCCC8C40C0'::geometry)
                                       Buffers: shared hit=88
                           ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.038..0.038 rows=187 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                 Buffers: shared hit=9
                                 ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.005..0.026 rows=187 loops=1)
                                       Buffers: shared hit=9
                     ->  Memoize  (cost=0.15..0.17 rows=1 width=226) (actual time=0.000..0.000 rows=1 loops=1318)
                           Cache Key: i.recordtype_id
                           Cache Mode: logical
                           Hits: 1316  Misses: 2  Evictions: 0  Overflows: 0  Memory Usage: 1kB
                           Buffers: shared hit=4
                           ->  Index Scan using idx_18878_primary on recordtype rt  (cost=0.14..0.16 rows=1 width=226) (actual time=0.004..0.004 rows=1 loops=2)
                                 Index Cond: (id = i.recordtype_id)
                                 Buffers: shared hit=4
 Planning:
   Buffers: shared hit=92
 Planning Time: 0.463 ms
 Execution Time: 3.661 ms
(40 rows)
```

### Statement 5 (reported 0.35 ms; plan execution 0.010 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.005..0.006 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.037 ms
 Execution Time: 0.010 ms
(4 rows)
```


## 4. A gazetteer-wide fuzzy search

Request: `GET /api/v2/places?name=Wollombi&match=fuzzy`

### Statement 0 (reported 4.87 ms; plan execution 0.025 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.012..0.014 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.515 ms
 Execution Time: 0.025 ms
(6 rows)
```

### Statement 1 (reported 2.75 ms; plan execution 0.060 ms)

```sql
select set_config('pg_trgm.similarity_threshold', '0.3', true);
```

```text
                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=32) (actual time=0.055..0.055 rows=1 loops=1)
 Planning Time: 0.012 ms
 Execution Time: 0.060 ms
(3 rows)
```

### Statement 2 (reported 0.53 ms; plan execution 0.010 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.005..0.006 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.024 ms
 Execution Time: 0.010 ms
(4 rows)
```

### Statement 3 (reported 49.82 ms; plan execution 25.222 ms)

```sql
select count(*) as total, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, count(*) filter (where r.geom is null) as no_coordinates, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where ((NOT r.sb AND r.s IS NULL) OR (NOT r.eb AND r.e IS NULL))) as unparseable_dates, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date, count(*) filter (where r.datasource_id = 1) as source_1, count(*) filter (where r.datasource_id = 2) as source_2, count(*) filter (where r.datasource_id = 3) as source_3, count(*) filter (where r.datasource_id = 4) as source_4 from (select "i"."geom", "i"."datasource_id", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and (i.title % CAST('Wollombi' AS text) OR i.placename % CAST('Wollombi' AS text)) offset 0) as "r" limit 1;
```

```text
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=916.76..917.28 rows=1 width=128) (actual time=20.207..20.210 rows=1 loops=1)
   Buffers: shared hit=623 read=571
   ->  Aggregate  (cost=916.76..917.28 rows=1 width=128) (actual time=20.205..20.208 rows=1 loops=1)
         Buffers: shared hit=623 read=571
         ->  Hash Left Join  (cost=254.18..892.96 rows=128 width=74) (actual time=7.730..20.098 rows=177 loops=1)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Rows Removed by Filter: 1
               Buffers: shared hit=623 read=571
               ->  Bitmap Heap Scan on dataitem i  (cost=241.04..750.20 rows=130 width=75) (actual time=6.096..17.357 rows=178 loops=1)
                     Recheck Cond: ((title % 'Wollombi'::text) OR (placename % 'Wollombi'::text))
                     Rows Removed by Index Recheck: 1513
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=777
                     Buffers: shared hit=348 read=571
                     ->  BitmapOr  (cost=241.04..241.04 rows=130 width=0) (actual time=5.761..5.763 rows=0 loops=1)
                           Buffers: shared hit=73 read=69
                           ->  Bitmap Index Scan on v2_dataitem_title_trgm  (cost=0.00..120.50 rows=66 width=0) (actual time=3.078..3.078 rows=1665 loops=1)
                                 Index Cond: (title % 'Wollombi'::text)
                                 Buffers: shared hit=37 read=35
                           ->  Bitmap Index Scan on v2_dataitem_placename_trgm  (cost=0.00..120.48 rows=64 width=0) (actual time=2.680..2.680 rows=1560 loops=1)
                                 Index Cond: (placename % 'Wollombi'::text)
                                 Buffers: shared hit=36 read=34
               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.073..0.073 rows=187 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                     Buffers: shared hit=9
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.012..0.058 rows=187 loops=1)
                           Buffers: shared hit=9
 Planning:
   Buffers: shared hit=436
 Planning Time: 9.338 ms
 Execution Time: 25.222 ms
(32 rows)
```

### Statement 4 (reported 17.09 ms; plan execution 10.852 ms)

```sql
select i.dataset_id as layer_id, d.name as name, count(*) as count, CASE WHEN btrim(d.warning, E' ') = '' THEN NULL ELSE d.warning END as warning from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and (i.title % CAST('Wollombi' AS text) OR i.placename % CAST('Wollombi' AS text)) and "i"."dataset_id" is not null group by "i"."dataset_id", "d"."name", "d"."warning" order by "i"."dataset_id" asc;
```

```text
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=763.50..763.80 rows=11 width=299) (actual time=10.737..10.750 rows=6 loops=1)
   Group Key: i.dataset_id, d.name, d.warning
   Buffers: shared hit=937
   ->  Sort  (cost=763.50..763.52 rows=11 width=259) (actual time=10.732..10.735 rows=49 loops=1)
         Sort Key: i.dataset_id, d.name, d.warning
         Sort Method: quicksort  Memory: 49kB
         Buffers: shared hit=937
         ->  Hash Left Join  (cost=254.12..763.31 rows=11 width=259) (actual time=4.802..10.695 rows=49 loops=1)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Rows Removed by Filter: 1
               Buffers: shared hit=928
               ->  Bitmap Heap Scan on dataitem i  (cost=240.98..750.14 rows=11 width=16) (actual time=4.718..10.600 rows=50 loops=1)
                     Recheck Cond: ((title % 'Wollombi'::text) OR (placename % 'Wollombi'::text))
                     Rows Removed by Index Recheck: 1513
                     Filter: ((dataset_id IS NOT NULL) AND (datasource_id = ANY ('{1,2,3,4}'::bigint[])))
                     Rows Removed by Filter: 128
                     Heap Blocks: exact=777
                     Buffers: shared hit=919
                     ->  BitmapOr  (cost=240.98..240.98 rows=130 width=0) (actual time=4.553..4.553 rows=0 loops=1)
                           Buffers: shared hit=142
                           ->  Bitmap Index Scan on v2_dataitem_title_trgm  (cost=0.00..120.50 rows=66 width=0) (actual time=2.400..2.400 rows=1665 loops=1)
                                 Index Cond: (title % 'Wollombi'::text)
                                 Buffers: shared hit=72
                           ->  Bitmap Index Scan on v2_dataitem_placename_trgm  (cost=0.00..120.48 rows=64 width=0) (actual time=2.152..2.152 rows=1560 loops=1)
                                 Index Cond: (placename % 'Wollombi'::text)
                                 Buffers: shared hit=70
               ->  Hash  (cost=10.84..10.84 rows=184 width=260) (actual time=0.076..0.077 rows=187 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 30kB
                     Buffers: shared hit=9
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=260) (actual time=0.015..0.052 rows=187 loops=1)
                           Buffers: shared hit=9
 Planning:
   Buffers: shared hit=33
 Planning Time: 0.876 ms
 Execution Time: 10.852 ms
(36 rows)
```

### Statement 5 (reported 19.11 ms; plan execution 16.011 ms)

```sql
select i.id, i.uid, i.title, i.placename, i.description, i.datasource_id, i.dataset_id, rt.type as record_type, i.state, i.lga, i.parish, i.feature_term, i.datestart, i.dateend, tlcmap.v2_udate(i.datestart) as udatestart, tlcmap.v2_udate(i.dateend) as udateend, i.source, i.external_url, i.image_path, i.glycerine_url, i.created_at, i.updated_at, i.extended_data, i.latitude, i.longitude, i.id as _k, i.id as _id from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where ((i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and (i.title % CAST('Wollombi' AS text) OR i.placename % CAST('Wollombi' AS text))) order by (i.id) ASC, (i.id) ASC limit 501;
```

```text
                                                                                 QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=771.90..837.50 rows=128 width=1932) (actual time=14.994..15.794 rows=177 loops=1)
   Buffers: shared hit=934
   ->  Result  (cost=771.90..837.50 rows=128 width=1932) (actual time=14.994..15.782 rows=177 loops=1)
         Buffers: shared hit=934
         ->  Sort  (cost=771.90..772.22 rows=128 width=1916) (actual time=14.991..15.004 rows=177 loops=1)
               Sort Key: i.id
               Sort Method: quicksort  Memory: 128kB
               Buffers: shared hit=934
               ->  Nested Loop Left Join  (cost=254.32..767.42 rows=128 width=1916) (actual time=5.320..14.829 rows=177 loops=1)
                     Buffers: shared hit=934
                     ->  Hash Left Join  (cost=254.18..763.68 rows=128 width=1690) (actual time=5.293..14.662 rows=177 loops=1)
                           Hash Cond: (i.dataset_id = d.id)
                           Filter: ((i.dataset_id IS NULL) OR d.public)
                           Rows Removed by Filter: 1
                           Buffers: shared hit=928
                           ->  Bitmap Heap Scan on dataitem i  (cost=241.04..750.20 rows=130 width=1690) (actual time=5.227..14.509 rows=178 loops=1)
                                 Recheck Cond: ((title % 'Wollombi'::text) OR (placename % 'Wollombi'::text))
                                 Rows Removed by Index Recheck: 1513
                                 Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                                 Heap Blocks: exact=777
                                 Buffers: shared hit=919
                                 ->  BitmapOr  (cost=241.04..241.04 rows=130 width=0) (actual time=5.047..5.048 rows=0 loops=1)
                                       Buffers: shared hit=142
                                       ->  Bitmap Index Scan on v2_dataitem_title_trgm  (cost=0.00..120.50 rows=66 width=0) (actual time=2.620..2.620 rows=1665 loops=1)
                                             Index Cond: (title % 'Wollombi'::text)
                                             Buffers: shared hit=72
                                       ->  Bitmap Index Scan on v2_dataitem_placename_trgm  (cost=0.00..120.48 rows=64 width=0) (actual time=2.425..2.425 rows=1560 loops=1)
                                             Index Cond: (placename % 'Wollombi'::text)
                                             Buffers: shared hit=70
                           ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.056..0.057 rows=187 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 16kB
                                 Buffers: shared hit=9
                                 ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.019..0.043 rows=187 loops=1)
                                       Buffers: shared hit=9
                     ->  Memoize  (cost=0.15..0.25 rows=1 width=226) (actual time=0.000..0.000 rows=1 loops=177)
                           Cache Key: i.recordtype_id
                           Cache Mode: logical
                           Hits: 174  Misses: 3  Evictions: 0  Overflows: 0  Memory Usage: 1kB
                           Buffers: shared hit=6
                           ->  Index Scan using idx_18878_primary on recordtype rt  (cost=0.14..0.24 rows=1 width=226) (actual time=0.012..0.012 rows=1 loops=3)
                                 Index Cond: (id = i.recordtype_id)
                                 Buffers: shared hit=6
 Planning:
   Buffers: shared hit=84
 Planning Time: 1.135 ms
 Execution Time: 16.011 ms
(46 rows)
```

### Statement 6 (reported 0.73 ms; plan execution 0.015 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.008..0.009 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.045 ms
 Execution Time: 0.015 ms
(4 rows)
```


## 5. A deep page of the unfiltered search

Request: `GET /api/v2/places?cursor=<cursor 100 pages of 5,000 records in>` (the cursor is the `next_cursor` after walking 100 pages at `per_page=5000`; it resumes at id 500,500)

### Statement 0 (reported 3.38 ms; plan execution 0.055 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.024..0.026 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.732 ms
 Execution Time: 0.055 ms
(6 rows)
```

### Statement 1 (reported 0.37 ms; plan execution 0.014 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.006..0.007 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.036 ms
 Execution Time: 0.014 ms
(4 rows)
```

### Statement 2 (reported 1115.41 ms; plan execution 856.314 ms)

```sql
select count(*) as total, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, count(*) filter (where r.geom is null) as no_coordinates, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where ((NOT r.sb AND r.s IS NULL) OR (NOT r.eb AND r.e IS NULL))) as unparseable_dates, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date, count(*) filter (where r.datasource_id = 1) as source_1, count(*) filter (where r.datasource_id = 2) as source_2, count(*) filter (where r.datasource_id = 3) as source_3, count(*) filter (where r.datasource_id = 4) as source_4 from (select "i"."geom", "i"."datasource_id", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) offset 0) as "r" limit 1;
```

```text
                                                                      QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=593122.99..593123.51 rows=1 width=128) (actual time=841.758..850.279 rows=1 loops=1)
   Buffers: shared hit=16767 read=93952
   ->  Aggregate  (cost=593122.99..593123.51 rows=1 width=128) (actual time=841.756..850.277 rows=1 loops=1)
         Buffers: shared hit=16767 read=93952
         ->  Gather  (cost=1013.14..467928.74 rows=676725 width=74) (actual time=1.801..597.276 rows=681827 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               Buffers: shared hit=16767 read=93952
               ->  Hash Left Join  (cost=13.14..399256.24 rows=281969 width=74) (actual time=2.524..637.856 rows=227276 loops=3)
                     Hash Cond: (i.dataset_id = d.id)
                     Filter: ((i.dataset_id IS NULL) OR d.public)
                     Rows Removed by Filter: 1803
                     Buffers: shared hit=16767 read=93952
                     ->  Parallel Seq Scan on dataitem i  (cost=0.00..113701.22 rows=286348 width=75) (actual time=0.019..273.377 rows=229078 loops=3)
                           Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                           Buffers: shared hit=15454 read=93952
                     ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.503..0.508 rows=187 loops=3)
                           Buckets: 1024  Batches: 1  Memory Usage: 16kB
                           Buffers: shared hit=27
                           ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.411..0.476 rows=187 loops=3)
                                 Buffers: shared hit=27
 Planning:
   Buffers: shared hit=415
 Planning Time: 7.094 ms
 Execution Time: 856.314 ms
(25 rows)
```

### Statement 3 (reported 149.27 ms; plan execution 125.344 ms)

```sql
select i.dataset_id as layer_id, d.name as name, count(*) as count, CASE WHEN btrim(d.warning, E' ') = '' THEN NULL ELSE d.warning END as warning from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and "i"."dataset_id" is not null group by "i"."dataset_id", "d"."name", "d"."warning" order by "i"."dataset_id" asc;
```

```text
                                                                                 QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=100408.12..106849.29 rows=14784 width=299) (actual time=56.919..124.794 rows=146 loops=1)
   Group Key: i.dataset_id, d.name, d.warning
   Buffers: shared hit=11677 read=3262, temp read=806 written=808
   ->  Gather Merge  (cost=100408.12..106331.85 rows=29568 width=267) (actual time=56.772..124.668 rows=277 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=11677 read=3262, temp read=806 written=808
         ->  Partial GroupAggregate  (cost=99408.10..101918.94 rows=14784 width=267) (actual time=31.918..54.917 rows=92 loops=3)
               Group Key: i.dataset_id, d.name, d.warning
               Buffers: shared hit=11677 read=3262, temp read=806 written=808
               ->  Incremental Sort  (cost=99408.10..101523.15 rows=24795 width=259) (actual time=31.848..53.109 rows=18661 loops=3)
                     Sort Key: i.dataset_id, d.name, d.warning
                     Presorted Key: i.dataset_id
                     Full-sort Groups: 61  Sort Method: quicksort  Average Memory: 41kB  Peak Memory: 45kB
                     Pre-sorted Groups: 78  Sort Methods: quicksort, external merge  Average Memory: 27kB  Peak Memory: 168kB  Average Disk: 2899kB  Peak Disk: 4112kB
                     Buffers: shared hit=11677 read=3262, temp read=806 written=808
                     Worker 0:  Full-sort Groups: 22  Sort Method: quicksort  Average Memory: 30kB  Peak Memory: 30kB
                       Pre-sorted Groups: 22  Sort Method: quicksort  Average Memory: 43kB  Peak Memory: 43kB
                     Worker 1:  Full-sort Groups: 22  Sort Method: quicksort  Average Memory: 30kB  Peak Memory: 30kB
                       Pre-sorted Groups: 40  Sort Method: quicksort  Average Memory: 89kB  Peak Memory: 113kB
                     ->  Merge Left Join  (cost=99386.74..99545.85 rows=24795 width=259) (actual time=31.736..34.575 rows=18661 loops=3)
                           Merge Cond: (i.dataset_id = d.id)
                           Filter: ((i.dataset_id IS NULL) OR d.public)
                           Rows Removed by Filter: 1803
                           Buffers: shared hit=11597 read=3262
                           ->  Sort  (cost=99368.98..99431.93 rows=25180 width=16) (actual time=30.858..31.536 rows=20464 loops=3)
                                 Sort Key: i.dataset_id
                                 Sort Method: quicksort  Memory: 4008kB
                                 Buffers: shared hit=11570 read=3262
                                 Worker 0:  Sort Method: quicksort  Memory: 131kB
                                 Worker 1:  Sort Method: quicksort  Memory: 250kB
                                 ->  Parallel Bitmap Heap Scan on dataitem i  (cost=1196.77..97528.33 rows=25180 width=16) (actual time=2.001..28.651 rows=20464 loops=3)
                                       Recheck Cond: (dataset_id IS NOT NULL)
                                       Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                                       Heap Blocks: exact=7253
                                       Buffers: shared hit=11570 read=3262
                                       ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1181.66 rows=60431 width=0) (actual time=4.191..4.191 rows=82162 loops=1)
                                             Index Cond: (dataset_id IS NOT NULL)
                                             Buffers: shared hit=194
                           ->  Sort  (cost=17.76..18.22 rows=184 width=260) (actual time=0.869..0.878 rows=186 loops=3)
                                 Sort Key: d.id
                                 Sort Method: quicksort  Memory: 53kB
                                 Buffers: shared hit=27
                                 Worker 0:  Sort Method: quicksort  Memory: 53kB
                                 Worker 1:  Sort Method: quicksort  Memory: 53kB
                                 ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=260) (actual time=0.760..0.813 rows=187 loops=3)
                                       Buffers: shared hit=27
 Planning:
   Buffers: shared hit=31
 Planning Time: 0.286 ms
 Execution Time: 125.344 ms
(51 rows)
```

### Statement 4 (reported 8.87 ms; plan execution 1.552 ms)

```sql
select i.id, i.uid, i.title, i.placename, i.description, i.datasource_id, i.dataset_id, rt.type as record_type, i.state, i.lga, i.parish, i.feature_term, i.datestart, i.dateend, tlcmap.v2_udate(i.datestart) as udatestart, tlcmap.v2_udate(i.dateend) as udateend, i.source, i.external_url, i.image_path, i.glycerine_url, i.created_at, i.updated_at, i.extended_data, i.latitude, i.longitude, i.id as _k, i.id as _id from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where ((i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4)) and (((i.id), (i.id)) > (CAST(559213 AS bigint), CAST(559213 AS bigint))) order by (i.id) ASC, (i.id) ASC limit 501;
```

```text
                                                                          QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.71..1193.60 rows=501 width=1932) (actual time=0.145..1.437 rows=501 loops=1)
   Buffers: shared hit=6 read=42
   ->  Nested Loop Left Join  (cost=0.71..427699.50 rows=179630 width=1932) (actual time=0.144..1.406 rows=501 loops=1)
         Buffers: shared hit=6 read=42
         ->  Nested Loop Left Join  (cost=0.57..333632.80 rows=179630 width=1690) (actual time=0.125..1.153 rows=501 loops=1)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=4 read=42
               ->  Index Scan using idx_18848_primary on dataitem i  (cost=0.42..302123.74 rows=182420 width=1690) (actual time=0.114..0.936 rows=501 loops=1)
                     Index Cond: (ROW(id, id) > ROW('559213'::bigint, '559213'::bigint))
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Buffers: shared hit=4 read=42
               ->  Index Scan using idx_18858_primary on dataset d  (cost=0.14..0.16 rows=1 width=9) (actual time=0.000..0.000 rows=0 loops=501)
                     Index Cond: (id = i.dataset_id)
         ->  Memoize  (cost=0.15..0.16 rows=1 width=226) (actual time=0.000..0.000 rows=1 loops=501)
               Cache Key: i.recordtype_id
               Cache Mode: logical
               Hits: 500  Misses: 1  Evictions: 0  Overflows: 0  Memory Usage: 1kB
               Buffers: shared hit=2
               ->  Index Scan using idx_18878_primary on recordtype rt  (cost=0.14..0.15 rows=1 width=226) (actual time=0.015..0.015 rows=1 loops=1)
                     Index Cond: (id = i.recordtype_id)
                     Buffers: shared hit=2
 Planning:
   Buffers: shared hit=89 read=3
 Planning Time: 3.025 ms
 Execution Time: 1.552 ms
(25 rows)
```

### Statement 5 (reported 0.71 ms; plan execution 0.027 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.013..0.014 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.065 ms
 Execution Time: 0.027 ms
(4 rows)
```


## 6. The catalogue with a date range

Request: `GET /api/v2/layers?date_from=1820&date_to=1860`

### Statement 0 (reported 2.89 ms; plan execution 0.027 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.011..0.013 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 1.538 ms
 Execution Time: 0.027 ms
(6 rows)
```

### Statement 1 (reported 3.38 ms; plan execution 0.003 ms)

```sql
select tlcmap.v2_udate('1820') as s, tlcmap.v2_udate_end('1820') as e;
```

```text
                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=16) (actual time=0.000..0.000 rows=1 loops=1)
 Planning:
   Buffers: shared hit=335
 Planning Time: 2.670 ms
 Execution Time: 0.003 ms
(5 rows)
```

### Statement 2 (reported 0.28 ms; plan execution 0.002 ms)

```sql
select tlcmap.v2_udate('1860') as s, tlcmap.v2_udate_end('1860') as e;
```

```text
                                     QUERY PLAN
-------------------------------------------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=16) (actual time=0.000..0.000 rows=1 loops=1)
 Planning Time: 0.070 ms
 Execution Time: 0.002 ms
(3 rows)
```

### Statement 3 (reported 463.24 ms; plan execution 554.366 ms)

```sql
select "d".*, d.id as _k, d.id as _id from (select d.id, d.name, d.description, d.creator, rt.type as record_type, m.matching_count as matching_count, (nullif(btrim(d.license, E' \t\r\n'), '') IS NOT NULL) as has_license, (nullif(btrim(d.warning, E' \t\r\n'), '') IS NOT NULL) as has_warning, count(*) over () as _total from "tlcmap"."dataset" as "d" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "d"."recordtype_id" inner join (select i.dataset_id, count(*) as matching_count from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and (least(coalesce(tlcmap.v2_udate(i.datestart), tlcmap.v2_udate(i.dateend)), coalesce(tlcmap.v2_udate(i.dateend), tlcmap.v2_udate(i.datestart))) IS NULL OR (least(coalesce(tlcmap.v2_udate(i.datestart), tlcmap.v2_udate(i.dateend)), coalesce(tlcmap.v2_udate(i.dateend), tlcmap.v2_udate(i.datestart))) <= CAST(-3439670400001 AS bigint) AND greatest(coalesce(tlcmap.v2_udate_end(i.dateend), tlcmap.v2_udate_end(i.datestart)), coalesce(tlcmap.v2_udate_end(i.datestart), tlcmap.v2_udate_end(i.dateend))) >= CAST(-4733596800000 AS bigint))) and "i"."dataset_id" is not null group by "i"."dataset_id") as "m" on "m"."dataset_id" = "d"."id" where d.public = true) as "d" order by (d.id) ASC, (d.id) ASC limit 101;
```

```text
                                                                                                                                                                                                                                                                                                                                                                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=174627.26..174627.43 rows=69 width=423) (actual time=547.155..553.927 rows=101 loops=1)
   Buffers: shared hit=15748
   ->  Sort  (cost=174627.26..174627.43 rows=69 width=423) (actual time=547.153..553.919 rows=101 loops=1)
         Sort Key: d.id
         Sort Method: quicksort  Memory: 62kB
         Buffers: shared hit=15748
         ->  Subquery Scan on d  (cost=174611.39..174625.15 rows=69 width=423) (actual time=546.987..553.862 rows=108 loops=1)
               Buffers: shared hit=15748
               ->  WindowAgg  (cost=174611.39..174624.46 rows=69 width=407) (actual time=546.986..553.848 rows=108 loops=1)
                     Buffers: shared hit=15748
                     ->  Hash Left Join  (cost=174611.39..174622.91 rows=69 width=638) (actual time=546.803..553.671 rows=108 loops=1)
                           Hash Cond: (d_1.recordtype_id = rt.id)
                           Buffers: shared hit=15748
                           ->  Hash Join  (cost=174610.18..174621.43 rows=69 width=428) (actual time=546.723..553.567 rows=108 loops=1)
                                 Hash Cond: (d_1.id = m.dataset_id)
                                 Buffers: shared hit=15747
                                 ->  Seq Scan on dataset d_1  (cost=0.00..10.84 rows=152 width=420) (actual time=0.015..0.058 rows=155 loops=1)
                                       Filter: public
                                       Rows Removed by Filter: 32
                                       Buffers: shared hit=9
                                 ->  Hash  (cost=174609.13..174609.13 rows=84 width=16) (actual time=546.695..553.451 rows=108 loops=1)
                                       Buckets: 1024  Batches: 1  Memory Usage: 14kB
                                       Buffers: shared hit=15738
                                       ->  Subquery Scan on m  (cost=174553.67..174609.13 rows=84 width=16) (actual time=545.058..553.423 rows=108 loops=1)
                                             Buffers: shared hit=15738
                                             ->  Finalize GroupAggregate  (cost=174553.67..174608.29 rows=84 width=16) (actual time=545.058..553.411 rows=108 loops=1)
                                                   Group Key: i.dataset_id
                                                   Buffers: shared hit=15738
                                                   ->  Gather Merge  (cost=174553.67..174606.61 rows=168 width=16) (actual time=544.983..553.359 rows=255 loops=1)
                                                         Workers Planned: 2
                                                         Workers Launched: 2
                                                         Buffers: shared hit=15738
                                                         ->  Partial GroupAggregate  (cost=173553.64..173587.20 rows=84 width=16) (actual time=517.130..518.108 rows=85 loops=3)
                                                               Group Key: i.dataset_id
                                                               Buffers: shared hit=15738
                                                               ->  Merge Left Join  (cost=173553.64..173572.03 rows=2865 width=8) (actual time=517.088..517.871 rows=3932 loops=3)
                                                                     Merge Cond: (i.dataset_id = d_2.id)
                                                                     Filter: ((i.dataset_id IS NULL) OR d_2.public)
                                                                     Rows Removed by Filter: 1339
                                                                     Buffers: shared hit=15738
                                                                     ->  Sort  (cost=173535.88..173543.16 rows=2910 width=16) (actual time=516.260..516.462 rows=5271 loops=3)
                                                                           Sort Key: i.dataset_id
                                                                           Sort Method: quicksort  Memory: 517kB
                                                                           Buffers: shared hit=15711
                                                                           Worker 0:  Sort Method: quicksort  Memory: 418kB
                                                                           Worker 1:  Sort Method: quicksort  Memory: 288kB
                                                                           ->  Parallel Bitmap Heap Scan on dataitem i  (cost=1183.40..173368.46 rows=2910 width=16) (actual time=2.772..514.798 rows=5271 loops=3)
                                                                                 Recheck Cond: (dataset_id IS NOT NULL)
                                                                                 Filter: ((datasource_id = ANY ('{1,2,3,4}'::bigint[])) AND ((LEAST(COALESCE(tlcmap.v2_udate_parts(datestart, false), tlcmap.v2_udate_parts(dateend, false)), COALESCE(tlcmap.v2_udate_parts(dateend, false), tlcmap.v2_udate_parts(datestart, false))) IS NULL) OR ((LEAST(COALESCE(tlcmap.v2_udate_parts(datestart, false), tlcmap.v2_udate_parts(dateend, false)), COALESCE(tlcmap.v2_udate_parts(dateend, false), tlcmap.v2_udate_parts(datestart, false))) <= '-3439670400001'::bigint) AND (GREATEST(COALESCE(tlcmap.v2_udate_parts(dateend, true), tlcmap.v2_udate_parts(datestart, true)), COALESCE(tlcmap.v2_udate_parts(datestart, true), tlcmap.v2_udate_parts(dateend, true))) >= '-4733596800000'::bigint))))
                                                                                 Rows Removed by Filter: 15193
                                                                                 Heap Blocks: exact=3267
                                                                                 Buffers: shared hit=15701
                                                                                 ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1181.66 rows=60431 width=0) (actual time=4.531..4.531 rows=82162 loops=1)
                                                                                       Index Cond: (dataset_id IS NOT NULL)
                                                                                       Buffers: shared hit=194
                                                                     ->  Sort  (cost=17.76..18.22 rows=184 width=9) (actual time=0.818..0.825 rows=185 loops=3)
                                                                           Sort Key: d_2.id
                                                                           Sort Method: quicksort  Memory: 33kB
                                                                           Buffers: shared hit=27
                                                                           Worker 0:  Sort Method: quicksort  Memory: 33kB
                                                                           Worker 1:  Sort Method: quicksort  Memory: 33kB
                                                                           ->  Seq Scan on dataset d_2  (cost=0.00..10.84 rows=184 width=9) (actual time=0.755..0.792 rows=187 loops=3)
                                                                                 Buffers: shared hit=27
                           ->  Hash  (cost=1.09..1.09 rows=9 width=226) (actual time=0.038..0.038 rows=9 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 Buffers: shared hit=1
                                 ->  Seq Scan on recordtype rt  (cost=0.00..1.09 rows=9 width=226) (actual time=0.029..0.031 rows=9 loops=1)
                                       Buffers: shared hit=1
 Planning:
   Buffers: shared hit=299
 Planning Time: 6.398 ms
 Execution Time: 554.366 ms
(72 rows)
```

### Statement 4 (reported 0.82 ms; plan execution 0.016 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.008..0.008 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.044 ms
 Execution Time: 0.016 ms
(4 rows)
```

### Statement 5 (reported 481.42 ms; plan execution 453.092 ms)

```sql
select r.dataset_id, count(*) as record_count, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date from (select "i"."dataset_id", "i"."geom", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and "i"."dataset_id" in (80, 81, 84, 87, 89, 90, 99, 112, 113, 118, 135, 140, 141, 150, 152, 153, 158, 159, 163, 164, 165, 167, 170, 172, 176, 192, 194, 198, 206, 208, 209, 211, 213, 218, 219, 223, 230, 234, 235, 236, 237, 238, 240, 241, 242, 247, 256, 258, 259, 260, 278, 284, 287, 288, 292, 295, 314, 315, 361, 378, 379, 380, 383, 384, 390, 391, 392, 394, 395, 398, 399, 402, 405, 407, 412, 415, 417, 421, 422, 423, 427, 429, 431, 434, 435, 436, 437, 438, 439, 440, 441, 442, 443, 448, 449, 454, 456, 459, 460, 461) offset 0) as "r" group by "r"."dataset_id";
```

```text
                                                                                                                                                                                                                                    QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=133454.57..133508.75 rows=84 width=88) (actual time=439.767..448.834 rows=100 loops=1)
   Group Key: i.dataset_id
   Batches: 1  Memory Usage: 48kB
   Buffers: shared hit=12690
   ->  Gather  (cost=2238.92..124794.62 rows=53292 width=74) (actual time=4.109..415.381 rows=52367 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=12690
         ->  Hash Left Join  (cost=1238.92..118465.42 rows=22205 width=74) (actual time=2.999..393.824 rows=17456 loops=3)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=12690
               ->  Parallel Bitmap Heap Scan on dataitem i  (cost=1225.78..95965.91 rows=22550 width=67) (actual time=1.216..24.492 rows=17456 loops=3)
                     Recheck Cond: (dataset_id = ANY ('{80,81,84,87,89,90,99,112,113,118,135,140,141,150,152,153,158,159,163,164,165,167,170,172,176,192,194,198,206,208,209,211,213,218,219,223,230,234,235,236,237,238,240,241,242,247,256,258,259,260,278,284,287,288,292,295,314,315,361,378,379,380,383,384,390,391,392,394,395,398,399,402,405,407,412,415,417,421,422,423,427,429,431,434,435,436,437,438,439,440,441,442,443,448,449,454,456,459,460,461}'::bigint[]))
                     Filter: (datasource_id = ANY ('{1,2,3,4}'::bigint[]))
                     Heap Blocks: exact=2477
                     Buffers: shared hit=11715
                     ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1212.25 rows=54120 width=0) (actual time=2.353..2.353 rows=69291 loops=1)
                           Index Cond: (dataset_id = ANY ('{80,81,84,87,89,90,99,112,113,118,135,140,141,150,152,153,158,159,163,164,165,167,170,172,176,192,194,198,206,208,209,211,213,218,219,223,230,234,235,236,237,238,240,241,242,247,256,258,259,260,278,284,287,288,292,295,314,315,361,378,379,380,383,384,390,391,392,394,395,398,399,402,405,407,412,415,417,421,422,423,427,429,431,434,435,436,437,438,439,440,441,442,443,448,449,454,456,459,460,461}'::bigint[]))
                           Buffers: shared hit=421
               ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.633..0.633 rows=187 loops=3)
                     Buckets: 1024  Batches: 1  Memory Usage: 16kB
                     Buffers: shared hit=27
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.535..0.594 rows=187 loops=3)
                           Buffers: shared hit=27
 Planning:
   Buffers: shared hit=41
 Planning Time: 0.487 ms
 Execution Time: 453.092 ms
(29 rows)
```


## 7. An extended_data filter without layers

Request: `GET /api/v2/places?extended_data=State = "NSW"&per_page=500` (URL-encoded)

### Statement 0 (reported 2.77 ms; plan execution 0.033 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.017..0.019 rows=4 loops=1)
   Buffers: shared hit=1
 Planning:
   Buffers: shared hit=88
 Planning Time: 2.329 ms
 Execution Time: 0.033 ms
(6 rows)
```

### Statement 1 (reported 0.67 ms; plan execution 0.011 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.005..0.006 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.032 ms
 Execution Time: 0.011 ms
(4 rows)
```

### Statement 2 (reported 44819.39 ms; plan execution 46536.450 ms)

```sql
select count(*) as total, ST_XMin(ST_Extent(r.geom)) as min_lon, ST_YMin(ST_Extent(r.geom)) as min_lat, ST_XMax(ST_Extent(r.geom)) as max_lon, ST_YMax(ST_Extent(r.geom)) as max_lat, count(*) filter (where r.geom is null) as no_coordinates, min(least(coalesce(r.s, r.e), coalesce(r.e, r.s))) as date_start, max(greatest(coalesce(r.ee, r.se), coalesce(r.se, r.ee))) as date_end, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null) as undated_total, count(*) filter (where (r.sb AND r.eb)) as no_date, count(*) filter (where ((NOT r.sb AND r.s IS NULL) OR (NOT r.eb AND r.e IS NULL))) as unparseable_dates, count(*) filter (where least(coalesce(r.s, r.e), coalesce(r.e, r.s)) is null and not (r.sb AND r.eb)) as unparseable_date, count(*) filter (where r.datasource_id = 1) as source_1, count(*) filter (where r.datasource_id = 2) as source_2, count(*) filter (where r.datasource_id = 3) as source_3, count(*) filter (where r.datasource_id = 4) as source_4 from (select "i"."geom", "i"."datasource_id", tlcmap.v2_udate(i.datestart) as s, tlcmap.v2_udate(i.dateend) as e, tlcmap.v2_udate_end(i.datestart) as se, tlcmap.v2_udate_end(i.dateend) as ee, nullif(btrim(i.datestart, E' \t\r\n'), '') IS NULL as sb, nullif(btrim(i.dateend, E' \t\r\n'), '') IS NULL as eb from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and tlcmap.v2_extended_value(i.extended_data, CAST('State' AS text)) = CAST('NSW' AS text) offset 0) as "r" limit 1;
```

```text
                                                                          QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=189409.36..189409.88 rows=1 width=128) (actual time=46512.107..46530.369 rows=1 loops=1)
   Buffers: shared hit=17669 read=93884
   ->  Aggregate  (cost=189409.36..189409.88 rows=1 width=128) (actual time=46512.105..46530.365 rows=1 loops=1)
         Buffers: shared hit=17669 read=93884
         ->  Gather  (cost=1013.14..188783.38 rows=3383 width=74) (actual time=3.741..46523.980 rows=3308 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               Buffers: shared hit=17669 read=93884
               ->  Hash Left Join  (cost=13.14..187445.08 rows=1410 width=74) (actual time=6.053..46468.591 rows=1103 loops=3)
                     Hash Cond: (i.dataset_id = d.id)
                     Filter: ((i.dataset_id IS NULL) OR d.public)
                     Buffers: shared hit=17669 read=93884
                     ->  Parallel Seq Scan on dataitem i  (cost=0.00..186004.07 rows=1432 width=75) (actual time=3.128..46409.380 rows=1103 loops=3)
                           Filter: ((datasource_id = ANY ('{1,2,3,4}'::bigint[])) AND (tlcmap.v2_extended_value(extended_data, 'State'::text) = 'NSW'::text))
                           Rows Removed by Filter: 227976
                           Buffers: shared hit=16788 read=93884
                     ->  Hash  (cost=10.84..10.84 rows=184 width=9) (actual time=0.936..0.942 rows=187 loops=3)
                           Buckets: 1024  Batches: 1  Memory Usage: 16kB
                           Buffers: shared hit=27
                           ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=9) (actual time=0.823..0.905 rows=187 loops=3)
                                 Buffers: shared hit=27
 Planning:
   Buffers: shared hit=474
 Planning Time: 9.163 ms
 Execution Time: 46536.450 ms
(25 rows)
```

### Statement 3 (reported 2835.68 ms; plan execution 2622.897 ms)

```sql
select i.dataset_id as layer_id, d.name as name, count(*) as count, CASE WHEN btrim(d.warning, E' ') = '' THEN NULL ELSE d.warning END as warning from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where (i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and tlcmap.v2_extended_value(i.extended_data, CAST('State' AS text)) = CAST('NSW' AS text) and "i"."dataset_id" is not null group by "i"."dataset_id", "d"."name", "d"."warning" order by "i"."dataset_id" asc;
```

```text
                                                                            QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=104927.47..104935.64 rows=297 width=299) (actual time=2616.785..2622.551 rows=7 loops=1)
   Group Key: i.dataset_id, d.name, d.warning
   Buffers: shared hit=16133
   ->  Sort  (cost=104927.47..104928.21 rows=297 width=259) (actual time=2616.774..2621.171 rows=3308 loops=1)
         Sort Key: i.dataset_id, d.name, d.warning
         Sort Method: quicksort  Memory: 3038kB
         Buffers: shared hit=16133
         ->  Hash Left Join  (cost=2194.87..104915.27 rows=297 width=259) (actual time=210.089..2616.976 rows=3308 loops=1)
               Hash Cond: (i.dataset_id = d.id)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=16124
               ->  Gather  (cost=2181.73..104901.34 rows=302 width=16) (actual time=209.800..2613.983 rows=3308 loops=1)
                     Workers Planned: 2
                     Workers Launched: 2
                     Buffers: shared hit=16115
                     ->  Parallel Bitmap Heap Scan on dataitem i  (cost=1181.73..103871.14 rows=126 width=16) (actual time=178.732..2584.040 rows=1103 loops=3)
                           Recheck Cond: (dataset_id IS NOT NULL)
                           Filter: ((datasource_id = ANY ('{1,2,3,4}'::bigint[])) AND (tlcmap.v2_extended_value(extended_data, 'State'::text) = 'NSW'::text))
                           Rows Removed by Filter: 19361
                           Heap Blocks: exact=1055
                           Buffers: shared hit=16115
                           ->  Bitmap Index Scan on idx_18848_dataset_id  (cost=0.00..1181.66 rows=60431 width=0) (actual time=12.731..12.731 rows=82162 loops=1)
                                 Index Cond: (dataset_id IS NOT NULL)
                                 Buffers: shared hit=194
               ->  Hash  (cost=10.84..10.84 rows=184 width=260) (actual time=0.276..0.277 rows=187 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 30kB
                     Buffers: shared hit=9
                     ->  Seq Scan on dataset d  (cost=0.00..10.84 rows=184 width=260) (actual time=0.050..0.184 rows=187 loops=1)
                           Buffers: shared hit=9
 Planning:
   Buffers: shared hit=31
 Planning Time: 0.447 ms
 Execution Time: 2622.897 ms
(33 rows)
```

### Statement 4 (reported 730.76 ms; plan execution 795.457 ms)

```sql
select i.id, i.uid, i.title, i.placename, i.description, i.datasource_id, i.dataset_id, rt.type as record_type, i.state, i.lga, i.parish, i.feature_term, i.datestart, i.dateend, tlcmap.v2_udate(i.datestart) as udatestart, tlcmap.v2_udate(i.dateend) as udateend, i.source, i.external_url, i.image_path, i.glycerine_url, i.created_at, i.updated_at, i.extended_data, i.latitude, i.longitude, i.id as _k, i.id as _id from "tlcmap"."dataitem" as "i" left join "tlcmap"."dataset" as "d" on "d"."id" = "i"."dataset_id" left join "tlcmap"."recordtype" as "rt" on "rt"."id" = "i"."recordtype_id" where ((i.dataset_id IS NULL OR d.public = true) and "i"."datasource_id" in (1, 2, 3, 4) and tlcmap.v2_extended_value(i.extended_data, CAST('State' AS text)) = CAST('NSW' AS text)) order by (i.id) ASC, (i.id) ASC limit 501;
```

```text
                                                                                  QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=1000.59..67558.34 rows=501 width=1932) (actual time=138.486..795.092 rows=501 loops=1)
   Buffers: shared hit=5943 read=66
   ->  Nested Loop Left Join  (cost=1000.59..450431.45 rows=3383 width=1932) (actual time=138.484..794.913 rows=501 loops=1)
         Join Filter: (rt.id = i.recordtype_id)
         Buffers: shared hit=5943 read=66
         ->  Nested Loop Left Join  (cost=1000.59..448322.74 rows=3383 width=1690) (actual time=138.422..770.459 rows=501 loops=1)
               Filter: ((i.dataset_id IS NULL) OR d.public)
               Buffers: shared hit=5942 read=66
               ->  Gather Merge  (cost=1000.45..447686.11 rows=3436 width=1690) (actual time=138.374..765.083 rows=501 loops=1)
                     Workers Planned: 2
                     Workers Launched: 2
                     Buffers: shared hit=4940 read=66
                     ->  Parallel Index Scan using idx_18848_primary on dataitem i  (cost=0.42..446289.49 rows=1432 width=1690) (actual time=99.386..462.970 rows=197 loops=3)
                           Filter: ((datasource_id = ANY ('{1,2,3,4}'::bigint[])) AND (tlcmap.v2_extended_value(extended_data, 'State'::text) = 'NSW'::text))
                           Rows Removed by Filter: 2433
                           Buffers: shared hit=4940 read=66
               ->  Index Scan using idx_18858_primary on dataset d  (cost=0.14..0.18 rows=1 width=9) (actual time=0.006..0.006 rows=1 loops=501)
                     Index Cond: (id = i.dataset_id)
                     Buffers: shared hit=1002
         ->  Materialize  (cost=0.00..1.14 rows=9 width=226) (actual time=0.000..0.000 rows=1 loops=501)
               Buffers: shared hit=1
               ->  Seq Scan on recordtype rt  (cost=0.00..1.09 rows=9 width=226) (actual time=0.051..0.051 rows=1 loops=1)
                     Buffers: shared hit=1
 Planning:
   Buffers: shared hit=92
 Planning Time: 0.474 ms
 Execution Time: 795.457 ms
(27 rows)
```

### Statement 5 (reported 0.43 ms; plan execution 0.020 ms)

```sql
select "id", "search_param_name" from "tlcmap"."datasource";
```

```text
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Seq Scan on datasource  (cost=0.00..14.50 rows=450 width=40) (actual time=0.009..0.011 rows=4 loops=1)
   Buffers: shared hit=1
 Planning Time: 0.058 ms
 Execution Time: 0.020 ms
(4 rows)
```
