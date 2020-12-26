## Run 02 - moderate bloat -  'Index Only Scan' a bit more expensive - 'Seq Scan' significantly more expensive 

### Review previous step (setup02_dsn_and_prepare)

In previous step we had moderate bloat with 'Index Only Scan' a bit more expensive and 'Seq Scan' significantly more expensive:

* Index Only Scan using idx_bloated_id_value - this is still fast for low cardinality queries (i.e. not that many matching rows)
* Seq Scan on bloated - this is now roughly x10 more expensive


### Much bigger (x100) bloat in run02 

In run updates was 10, this time we have x100 bigger i.e. 1000

```
~/projects/postgresql_user_group_nl_2019_10_03 $ time python postgres_bloat_demo.py create-bloat 'port=5432 dbname=pgbench user=bench1 password=changeme'
Number of updates [10]: 1000
Block size [1000]:
Creating index bloat... Iteration 0 is done
Creating index bloat... Iteration 1 is done
Creating index bloat... Iteration 2 is done
Creating index bloat... Iteration 3 is done
Creating index bloat... Iteration 4 is done
Creating index bloat... Iteration 5 is done
Done.

real	4m2.132s
user	0m2.806s
sys	0m1.945s
```

### Review postgres_bloat_demo.py check - 17Mb table and 30Mb index


```
~/projects/postgresql_user_group_nl_2019_10_03 $ python postgres_bloat_demo.py check 'port=5432 dbname=pgbench user=bench1 password=changeme'
Table bloat:

tablename    disk_size    actual_size    bloat_size    bloat_pct
-----------  -----------  -------------  ------------  -----------
bloated      17 MB        48 kB          17 MB         37117 %


Index bloat:

index_name            disk_size    actual_size    bloat_size    bloat_pct
--------------------  -----------  -------------  ------------  -----------
idx_bloated_id_value  30 MB        40 kB          30 MB         77820 %
```



### Review execution plans for simple queries - 'Seq Scan' seriously more expensive

In previous step we had moderate bloat with 'Index Only Scan' a bit more expensive and 'Seq Scan' significantly more expensive:

* Index Only Scan using idx_bloated_id_value - this is still fast for low cardinality queries (i.e. not that many matching rows)
* Seq Scan on bloated - this is now roughly x1000 more expensive



```
pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3;
                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=16.42..16.43 rows=1 width=8) (actual time=0.022..0.023 rows=1 loops=1)
   Buffers: shared hit=3
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.40..16.42 rows=1 width=0) (actual time=0.015..0.015 rows=0 loops=1)
         Index Cond: (id < 3)
         Heap Fetches: 0
         Buffers: shared hit=3
 Planning Time: 95.704 ms
 Execution Time: 1.933 ms
(8 rows)

pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 300;
                                                                 QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=236.70..236.71 rows=1 width=8) (actual time=0.202..0.203 rows=1 loops=1)
   Buffers: shared hit=22
   ->  Index Only Scan using idx_bloated_id_value on bloated  (cost=0.40..236.66 rows=15 width=0) (actual time=0.016..0.192 rows=17 loops=1)
         Index Cond: (id < 300)
         Heap Fetches: 0
         Buffers: shared hit=22
 Planning Time: 0.129 ms
 Execution Time: 2.435 ms
(8 rows)

pgbench=# explain (analyze,buffers) select count(*) from bloated where id < 3000;
                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=2246.12..2246.13 rows=1 width=8) (actual time=5.493..5.493 rows=1 loops=1)
   Buffers: shared hit=2233
   ->  Seq Scan on bloated  (cost=0.00..2245.50 rows=249 width=0) (actual time=1.867..5.450 rows=249 loops=1)
         Filter: (id < 3000)
         Rows Removed by Filter: 751
         Buffers: shared hit=2233
 Planning Time: 0.312 ms
 Execution Time: 5.520 ms
(8 rows)
``` 
