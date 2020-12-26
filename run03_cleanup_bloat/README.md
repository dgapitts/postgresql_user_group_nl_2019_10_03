## Run 03 - cleanup bloat - bit of a gotcha !?

### Review previous step (run02 moderate bloat) 17MB table and 30MB index


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

### postgres_bloat_demo.py vacuum  

```
~/projects/postgresql_user_group_nl_2019_10_03 $ time python postgres_bloat_demo.py vacuum  'port=5432 dbname=pgbench user=bench1 password=changeme'

real	0m0.475s
user	0m0.089s
sys	0m0.024s
```

### Re-run postgres_bloat_demo.py check - moderate bloat - 17MB table and 30MB index)


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

### Switch back to psql for 'vacuum verbose' - 2172 frozen pages (out of 2233)


```
pgbench=# vacuum verbose bloated;
INFO:  vacuuming "public.bloated"
INFO:  "bloated": found 0 removable, 33 nonremovable row versions in 1 out of 2233 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 449861
There were 108 unused item pointers.
Skipped 0 pages due to buffer pins, 2172 frozen pages.
0 pages are entirely empty.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s.
VACUUM
```

### Check for blocking transactions



```
SELECT md5(query),
left(query,75) AS query_first_75,
left(now()::text,22) AS CURRENT_TIME,
left(query_start::text,22) AS query_start,
state,
client_addr,
pid||':'||client_port AS pid_clntport
FROM pg_stat_activity
ORDER BY 4;

pgbench-# 

               md5                |          query_first_75           |      current_time      |      query_start       | state  | client_addr | pid_clntport
----------------------------------+-----------------------------------+------------------------+------------------------+--------+-------------+--------------
 bb3ac9c964ef59b1b6df28ba6037d09f | SELECT md5(query),               +| 2019-10-14 15:43:26.83 | 2019-10-14 15:43:26.83 | active |             | 54684:-1
                                  | left(query,75) AS query_first_75,+|                        |                        |        |             |
                                  | left(now()::text,22) A            |                        |                        |        |             |
 d41d8cd98f00b204e9800998ecf8427e |                                   | 2019-10-14 15:43:26.83 |                        |        |             |
 d41d8cd98f00b204e9800998ecf8427e |                                   | 2019-10-14 15:43:26.83 |                        |        |             |
 d41d8cd98f00b204e9800998ecf8427e |                                   | 2019-10-14 15:43:26.83 |                        |        |             |
 d41d8cd98f00b204e9800998ecf8427e |                                   | 2019-10-14 15:43:26.83 |                        |        |             |
 d41d8cd98f00b204e9800998ecf8427e |                                   | 2019-10-14 15:43:26.83 |                        |        |             |
(6 rows)
```

### Still in psql try 'vacuum verbose full' - 2172 frozen pages (out of 2233)


```
pgbench=# vacuum verbose bloated;
INFO:  vacuuming "public.bloated"
INFO:  "bloated": found 0 removable, 33 nonremovable row versions in 1 out of 2233 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 449861
There were 108 unused item pointers.
Skipped 0 pages due to buffer pins, 2172 frozen pages.
0 pages are entirely empty.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.04 s.
VACUUM
pgbench=# \timing on
Timing is on.
pgbench=# vacuum full verbose bloated;
INFO:  vacuuming "public.bloated"
INFO:  "bloated": found 0 removable, 1000 nonremovable row versions in 2233 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.04 s.
VACUUM
Time: 374.652 ms
pgbench=# vacuum verbose bloated;
INFO:  vacuuming "public.bloated"
INFO:  index "idx_bloated_id_value" now contains 1000 row versions in 6 pages
DETAIL:  0 index row versions were removed.
0 index pages have been deleted, 0 are currently reusable.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.04 s.
INFO:  "bloated": found 0 removable, 1000 nonremovable row versions in 6 out of 6 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 449862
There were 0 unused item pointers.
Skipped 0 pages due to buffer pins, 0 frozen pages.
0 pages are entirely empty.
CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.04 s.
VACUUM
Time: 66.678 ms
```

### Re-run postgres_bloat_demo.py check -  bloat gone  - 17MB table and 30MB index)


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

