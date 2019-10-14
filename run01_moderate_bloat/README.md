## Run 01 - moderate bloat -  'Index Only Scan' a bit more expensive - 'Seq Scan' significantly more expensive 

### Review previous step (setup02_dsn_and_prepare)

In previous step we setup a database/name and prepared a bloat table with 1000 rows

```
pgbench=# \d bloated
                                 Table "public.bloated"
 Column |       Type       | Collation | Nullable |               Default
--------+------------------+-----------+----------+-------------------------------------
 id     | bigint           |           | not null | nextval('bloated_id_seq'::regclass)
 value  | double precision |           | not null |
Indexes:
    "idx_bloated_id_value" btree (id, value) WITH (fillfactor='90')

pgbench=# select count(*) from bloated;
 count
-------
  1000
(1 row)
```

### Review postgres_bloat_demo.py check - 48kB table and 48kB

there is a 'postgres_bloat_demo.py check' operation:

```
~/projects/postgresql_user_group_nl_2019_10_03 $ python postgres_bloat_demo.py check 'port=5432 dbname=pgbench user=bench1 password=changeme'
Table bloat:

tablename    disk_size    actual_size    bloat_size    bloat_pct
-----------  -----------  -------------  ------------  -----------
bloated      48 kB        48 kB          0 bytes       0 %


Index bloat:

index_name            disk_size    actual_size    bloat_size    bloat_pct
--------------------  -----------  -------------  ------------  -----------
idx_bloated_id_value  48 kB        40 kB          8192 bytes    20 %
```


### Review execution plans for simple queries - both 'Index Only Scan' and 'Seq Scan' are very cheap

at this stage, simple queries on bloated are faster irrespective of wnether they'ew  using
* Index Only Scan using idx_bloated_id_value - typically < 10 blocks (shared buffer hits)
* Seq Scan on bloated - typically < 10 blocks (shared buffer hits)


```
pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3;
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=4.31..4.32 rows=1 width=8) (actual time=0.014..0.014 rows=1 loops=1)
   Buffers: shared hit=3
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.28..4.31 rows=2 width=0) (actual time=0.006..0.007 rows=2 loops=1)
         Index Cond: (id < 3)
         Heap Fetches: 0
         Buffers: shared hit=3
 Planning Time: 0.124 ms
 Execution Time: 0.048 ms
(8 rows)

pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 300;
                                                                  QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=14.26..14.27 rows=1 width=8) (actual time=1.038..1.039 rows=1 loops=1)
   Buffers: shared hit=4
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.28..13.51 rows=299 width=0) (actual time=0.028..0.972 rows=299 loops=1)
         Index Cond: (id < 300)
         Heap Fetches: 0
         Buffers: shared hit=4
 Planning Time: 0.103 ms
 Execution Time: 1.367 ms
(8 rows)


pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3000;
                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=21.00..21.01 rows=1 width=8) (actual time=0.562..0.562 rows=1 loops=1)
   Buffers: shared hit=6
   ->  Seq Scan on bloated  (cost=0.00..18.50 rows=1000 width=0) (actual time=0.028..0.372 rows=1000 loops=1)
         Filter: (id < 3000)
         Buffers: shared hit=6
 Planning Time: 3.997 ms
 Execution Time: 0.603 ms
(7 rows)
```


### Review execution plans for simple queries - both 'Index Only Scan' and 'Seq Scan' are very cheap

Now let's a 'postgres_bloat_demo.py bloat' operation:

```
~/projects/postgresql_user_group_nl_2019_10_03 $ time python postgres_bloat_demo.py create-bloat 'port=5432 dbname=pgbench user=bench1 password=changeme'
Number of updates [10]:
Block size [1000]:
Creating index bloat... Iteration 0 is done
Creating index bloat... Iteration 1 is done
Creating index bloat... Iteration 2 is done
Creating index bloat... Iteration 3 is done
Done.

real	0m8.969s
user	0m0.116s
sys	0m0.051s
```


### Review postgres_bloat_demo.py check - 440kB table and 1024kB

and then rerunning the  'postgres_bloat_demo.py check' operation:

```
~/projects/postgresql_user_group_nl_2019_10_03 $ python postgres_bloat_demo.py check 'port=5432 dbname=pgbench user=bench1 password=changeme'
Table bloat:

tablename    disk_size    actual_size    bloat_size    bloat_pct
-----------  -----------  -------------  ------------  -----------
bloated      440 kB       48 kB          392 kB        817 %


Index bloat:

index_name            disk_size    actual_size    bloat_size    bloat_pct
--------------------  -----------  -------------  ------------  -----------
idx_bloated_id_value  1024 kB      40 kB          984 kB        2460 %
```


### Review execution plans for simple queries - both 'Index Only Scan' and 'Seq Scan' becoming more expensive

and now our "simple queries on the bloated table" are becoming more expensive 
* Index Only Scan using idx_bloated_id_value - this is still fast for low cardinality queries (i.e. not that many matching rows)
* Seq Scan on bloated - this is now roughly x10 more expensive


```
pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3;
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=4.30..4.31 rows=1 width=8) (actual time=0.014..0.014 rows=1 loops=1)
   Buffers: shared hit=3
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.28..4.29 rows=1 width=0) (actual time=0.006..0.007 rows=1 loops=1)
         Index Cond: (id < 3)
         Heap Fetches: 0
         Buffers: shared hit=3
 Planning Time: 0.127 ms
 Execution Time: 0.048 ms
(8 rows)

pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 300;
                                                                 QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=41.70..41.70 rows=1 width=8) (actual time=0.130..0.131 rows=1 loops=1)
   Buffers: shared hit=20
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.28..41.52 rows=71 width=0) (actual time=0.014..0.109 rows=71 loops=1)
         Index Cond: (id < 300)
         Heap Fetches: 0
         Buffers: shared hit=20
 Planning Time: 0.186 ms
 Execution Time: 0.171 ms
(8 rows)

pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3000;
                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=70.00..70.01 rows=1 width=8) (actual time=0.538..0.539 rows=1 loops=1)
   Buffers: shared hit=55
   ->  Seq Scan on bloated  (cost=0.00..67.50 rows=1000 width=0) (actual time=0.015..0.349 rows=1000 loops=1)
         Filter: (id < 3000)
         Buffers: shared hit=55
 Planning Time: 0.236 ms
 Execution Time: 0.591 ms
(7 rows)

```

