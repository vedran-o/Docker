# Docker

Cassandra-stress using Docker container 

---
title: UMS - Eksperimentalni rad
author: Vedran Orešković
---

# Eksperimentalni rad

## 1. Preuzimanje Cassandra slike

Naredba:

    docker pull cassandra:latest
    
Ispis:

    latest: Pulling from library/cassandra
    Digest: sha256:566489b615eec3d43427f73d986aaa9568ff88f3a83ac6bea2175f5bfcd2469d
    Status: Image is up to date for cassandra:latest
    docker.io/library/cassandra:latest


## 2. Stvaranje i pokretanje kontejnera cassandraTest:
Naredba:

    docker run --name cassandraTest -e CASSANDRA_PASSWORD=1234 -p 5432:5432 cassandra

Kontejner ostavljamo pokrenutim.

## 3. Konfiguracija YAML datoteke

    keyspace: example
    
    # Would almost always be network topology unless running something locally
    keyspace_definition: |
      CREATE KEYSPACE example WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
    
    table: staff_activities
    
    # The table under test. Start with a partition per staff member
    # Is this a good idea?
    table_definition: |
      CREATE TABLE staff_activities (
            name text,
            when timeuuid,
            what text,
            PRIMARY KEY(name, when)
      )

    columnspec:
      - name: name
        size: uniform(5..10) # The names of the staff members are between 5-10 characters
        population: uniform(1..10) # 10 possible staff members to pick from
      - name: when
        cluster: uniform(20..500) # Staff members do between 20 and 500 events
      - name: what
        size: normal(10..100,50)
    
    insert:
      # we only update a single partition in any given insert
      partitions: fixed(1)
      # we want to insert a single row per partition and we have between 20 and 500
      # rows per partition
      select: fixed(1)/500
      batchtype: UNLOGGED             # Single partition unlogged batches are essentially noops

    queries:
       events:
          cql: select *  from staff_activities where name = ?
          fields: samerow
       latest_event:
          cql: select * from staff_activities where name = ?  LIMIT 1
          fields: samerow

  
## 4. Kako bi saznali adresu upisujemo sljedeću naredbu:
Naredba:
    
    docker inspect cassandraTest

Te u ispisu pronalazimo adresu:

     "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "5ba59e704ce6e2ce9a898a0eb37277d372bcc55acb406730590ea9839df43714",
                    "EndpointID": "f3f7856fdd49bf38e4c2b6a17d51d9324f1156b5aab14063aeef049925e30cd1",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }

Adresa na koju ćemo se spojiti cassandra-stress naredbom u sljedećem koraku je

    "IPAddress": "172.17.0.2",

## 5. U drugom terminalu pokrećemo cassandra-stress alat
Naredba:

    cassandra-stress user profile=./example.yaml duration=30s "ops(insert=1,latest_event=1,events=1)" truncate=once -graph file=graph.html -node 172.17.0.2
    
Alat kreće sa 4 threada te izvodi pisanje i čitanje podataka, a rezultate dobivamo u ispisu:
![](https://i.imgur.com/YuMOrKe.png)

Te za 8 threadova: 
![](https://i.imgur.com/YA0uGvE.png)

## 6. Naredbom docker status pratimo metrike kontejnera
Naredba:
 
    docker stats 

Ispis:

    CONTAINER ID   NAME            CPU %     MEM USAGE / LIMIT     MEM %     NET I/O          BLOCK I/O        PIDS
    d3423eeb6a22   cassandraTest   208.52%   2.156GiB / 7.733GiB   27.89%    342MB / 27.3GB   172MB / 64.9MB   211
 
Zauzeće procesora kreće se od 200-300%, memorija je stabilna oko 30%, opterećenje mreže je oko 300-500 MB
Usporedbu po različitim threadovima dobivamo na kraju cassandra-test alata:

                id, type                                               total ops,    op/s,    pk/s,   row/s,    mean,     med,     .95,     .99,    .999,     max,   time,   stderr, errors,  gc: #,  max ms,  sum ms,  sdv ms,      mb
    4 threadCount, events,                                                24435,    1568,    1566,  612342,     1,4,     1,3,     2,4,     5,6,    11,1,    17,3,   15,6,  0,09297,      0,      0,       0,       0,       0,       0
    4 threadCount, insert,                                                24183,    1551,    1551,    1758,     0,5,     0,3,     1,2,     3,2,     9,1,    19,9,   15,6,  0,09297,      0,      0,       0,       0,       0,       0
    4 threadCount, latest_event,                                          23987,    1539,    1537,    1537,     0,4,     0,3,     1,0,     2,6,     8,0,    16,6,   15,6,  0,09297,      0,      0,       0,       0,       0,       0
    4 threadCount, total,                                                 72605,    4658,    4655,  615636,     0,8,     0,5,     1,9,     3,9,     9,8,    19,9,   15,6,  0,09297,      0,      0,       0,       0,       0,       0
    8 threadCount, events,                                                22086,    1414,    1414,  701342,     3,0,     2,4,     6,6,    12,0,    21,4,    33,5,   15,6,  0,04948,      0,      0,       0,       0,       0,       0
    8 threadCount, insert,                                                22745,    1456,    1456,    1666,     1,2,     0,8,     3,7,     7,3,    15,4,    27,6,   15,6,  0,04948,      0,      0,       0,       0,       0,       0
    8 threadCount, latest_event,                                          22208,    1422,    1422,    1422,     1,1,     0,7,     3,2,     6,2,    13,6,    24,1,   15,6,  0,04948,      0,      0,       0,       0,       0,       0
    8 threadCount, total,                                                 67039,    4292,    4292,  704430,     1,8,     1,3,     4,9,     9,2,    18,2,    33,5,   15,6,  0,04948,      0,      0,       0,       0,       0,       0
    16 threadCount, events,                                                28076,    1790,    1790,  894041,     5,3,     3,8,    14,1,    23,5,    35,5,    52,3,   15,7,  0,04897,      0,      0,       0,       0,       0,       0
    16 threadCount, insert,                                                27925,    1780,    1780,    2025,     1,7,     1,1,     4,9,    10,4,    22,9,    46,8,   15,7,  0,04897,      0,      0,       0,       0,       0,       0
    16 threadCount, latest_event,                                          28066,    1789,    1789,    1789,     1,5,     1,0,     4,3,     9,8,    23,9,    43,8,   15,7,  0,04897,      0,      0,       0,       0,       0,       0
    16 threadCount, total,                                                 84067,    5358,    5358,  897855,     2,8,     1,7,     9,5,    18,1,    31,4,    52,3,   15,7,  0,04897,      0,      0,       0,       0,       0,       0
    24 threadCount, events,                                                25267,    1603,    1603,  801368,     8,2,     5,4,    24,2,    40,0,    60,2,    87,9,   15,8,  0,05375,      0,      0,       0,       0,       0,       0
    24 threadCount, insert,                                                24597,    1561,    1561,    1774,     3,1,     2,0,     9,4,    20,3,    35,1,    71,3,   15,8,  0,05375,      0,      0,       0,       0,       0,       0
    24 threadCount, latest_event,                                          25822,    1638,    1638,    1638,     2,9,     1,9,     8,2,    18,8,    36,5,    83,8,   15,8,  0,05375,      0,      0,       0,       0,       0,       0
    24 threadCount, total,                                                 75686,    4802,    4802,  804780,     4,7,     2,8,    16,2,    30,3,    53,1,    87,9,   15,8,  0,05375,      0,      0,       0,       0,       0,       0
    36 threadCount, events,                                                27699,    1747,    1747,  873404,    12,0,     7,1,    39,9,    64,4,   100,5,   201,6,   15,9,  0,05724,      0,      0,       0,       0,       0,       0
    36 threadCount, insert,                                                28265,    1783,    1783,    2028,     3,8,     2,1,    12,3,    32,8,    62,7,   156,9,   15,9,  0,05724,      0,      0,       0,       0,       0,       0
    36 threadCount, latest_event,                                          27788,    1752,    1752,    1752,     3,5,     2,0,    10,6,    29,6,    60,5,   102,4,   15,9,  0,05724,      0,      0,       0,       0,       0,       0
    36 threadCount, total,                                                 83752,    5282,    5282,  877184,     6,4,     3,0,    25,0,    50,9,    84,1,   201,6,   15,9,  0,05724,      0,      0,       0,       0,       0,       0
    54 threadCount, events,                                                28034,    1751,    1751,  875624,    17,8,     9,6,    62,3,   105,3,   163,2,   218,8,   16,0,  0,09391,      0,      0,       0,       0,       0,       0
    54 threadCount, insert,                                                27837,    1739,    1739,    1977,     5,8,     3,1,    19,7,    53,6,    99,9,   208,3,   16,0,  0,09391,      0,      0,       0,       0,       0,       0
    54 threadCount, latest_event,                                          28526,    1782,    1782,    1782,     5,3,     3,0,    16,4,    47,7,    96,0,   183,5,   16,0,  0,09391,      0,      0,       0,       0,       0,       0
    54 threadCount, total,                                                 84397,    5272,    5272,  879384,     9,6,     4,2,    38,7,    80,4,   138,3,   218,8,   16,0,  0,09391,      0,      0,       0,       0,       0,       0
    END
    
Rezultati za 4 threada:

	Results:
	Op rate                   :    4.658 op/s  [events: 1.568 op/s, insert: 1.551 op/s, latest_event: 1.539 	op/s]
	Partition rate            :    4.655 pk/s  [events: 1.566 pk/s, insert: 1.551 pk/s, latest_event: 1.537 	pk/s]
	Row rate                  :  615.636 row/s [events: 612.342 row/s, insert: 1.758 row/s, latest_event: 	1.537 row/s]
	Latency mean              :    0,8 ms [events: 1,4 ms, insert: 0,5 ms, latest_event: 0,4 ms]
	Latency median            :    0,5 ms [events: 1,3 ms, insert: 0,3 ms, latest_event: 0,3 ms]
	Latency 95th percentile   :    1,9 ms [events: 2,4 ms, insert: 1,2 ms, latest_event: 1,0 ms]
	Latency 99th percentile   :    3,9 ms [events: 5,6 ms, insert: 3,2 ms, latest_event: 2,6 ms]
	Latency 99.9th percentile :    9,8 ms [events: 11,1 ms, insert: 9,1 ms, latest_event: 8,0 ms]
	Latency max               :   19,9 ms [events: 17,3 ms, insert: 19,9 ms, latest_event: 16,6 ms]
	Total partitions          :     72.556 [events: 24.418, insert: 24.183, latest_event: 23.955]
	Total errors              :          0 [events: 0, insert: 0, latest_event: 0]
	Total GC count            : 0
	Total GC memory           : 0,000 KiB
	Total GC time             :    0,0 seconds
	Avg GC time               :    NaN ms
	StdDev GC time            :    0,0 ms
	Total operation time      : 00:00:15
    
Rezultati za 8 threadova:

	Results:
	Op rate                   :    4.292 op/s  [events: 1.414 op/s, insert: 1.456 op/s, latest_event: 1.422 	op/s]
	Partition rate            :    4.292 pk/s  [events: 1.414 pk/s, insert: 1.456 pk/s, latest_event: 1.422 	pk/s]
	Row rate                  :  704.430 row/s [events: 701.342 row/s, insert: 1.666 row/s, latest_event: 	1.422 row/s]
	Latency mean              :    1,8 ms [events: 3,0 ms, insert: 1,2 ms, latest_event: 1,1 ms]
	Latency median            :    1,3 ms [events: 2,4 ms, insert: 0,8 ms, latest_event: 0,7 ms]
	Latency 95th percentile   :    4,9 ms [events: 6,6 ms, insert: 3,7 ms, latest_event: 3,2 ms]
	Latency 99th percentile   :    9,2 ms [events: 12,0 ms, insert: 7,3 ms, latest_event: 6,2 ms]
	Latency 99.9th percentile :   18,2 ms [events: 21,4 ms, insert: 15,4 ms, latest_event: 13,6 ms]
	Latency max               :   33,5 ms [events: 33,5 ms, insert: 27,6 ms, latest_event: 24,1 ms]
	Total partitions          :     67.039 [events: 22.086, insert: 22.745, latest_event: 22.208]
	Total errors              :          0 [events: 0, insert: 0, latest_event: 0]
	Total GC count            : 0
	Total GC memory           : 0,000 KiB
	Total GC time             :    0,0 seconds
	Avg GC time               :    NaN ms
	StdDev GC time            :    0,0 ms
	Total operation time      : 00:00:15
	Improvement over 4 threadCount: -8%
 
Rezultati za 16 threadova:

	Results:
	Op rate                   :    5.358 op/s  [events: 1.790 op/s, insert: 1.780 op/s, latest_event: 1.789 	op/s]
	Partition rate            :    5.358 pk/s  [events: 1.790 pk/s, insert: 1.780 pk/s, latest_event: 1.789 	pk/s]
	Row rate                  :  897.855 row/s [events: 894.041 row/s, insert: 2.025 row/s, latest_event: 	1.789 row/s]
	Latency mean              :    2,8 ms [events: 5,3 ms, insert: 1,7 ms, latest_event: 1,5 ms]
	Latency median            :    1,7 ms [events: 3,8 ms, insert: 1,1 ms, latest_event: 1,0 ms]
	Latency 95th percentile   :    9,5 ms [events: 14,1 ms, insert: 4,9 ms, latest_event: 4,3 ms]
	Latency 99th percentile   :   18,1 ms [events: 23,5 ms, insert: 10,4 ms, latest_event: 9,8 ms]
	Latency 99.9th percentile :   31,4 ms [events: 35,5 ms, insert: 22,9 ms, latest_event: 23,9 ms]
	Latency max               :   52,3 ms [events: 52,3 ms, insert: 46,8 ms, latest_event: 43,8 ms]
	Total partitions          :     84.067 [events: 28.076, insert: 27.925, latest_event: 28.066]
	Total errors              :          0 [events: 0, insert: 0, latest_event: 0]
	Total GC count            : 0
	Total GC memory           : 0,000 KiB
	Total GC time             :    0,0 seconds
	Avg GC time               :    NaN ms
	StdDev GC time            :    0,0 ms
	Total operation time      : 00:00:15
	
	Improvement over 8 threadCount: 25%
    
![]()
Grafičkim rezultatima testiranja može se pristupiti putem html datoteke koje cassandra stress daje kao ispis. Replication factor postavljen je na 3 u YAML datoteci.

Slika na 1 querry:
![](https://i.imgur.com/zgvWc8K.png)

Slika na 2 querry-ja:
![](https://i.imgur.com/d5vq8Lv.png)

## 7. Literatura

- https://hub.docker.com/_/cassandra
- https://cassandra.apache.org/doc/latest/cassandra/tools/cassandra_stress.html
- https://gaseri.org/hr/nastava/izvedbeni/2019-2020/UMS/#raspored-nastave-zimski-iii-semestar-ak-god-20192020
- https://thelastpickle.com/blog/2020/04/06/comparing-stress-tools.html
- https://stackoverflow.com/questions/55708768/how-to-calibrate-cassandra-stress-graph-option
- https://docs.datastax.com/en/dse/5.1/dse-dev/datastax_enterprise/tools/toolsCStress.html
- https://wp.huangshiyang.com/the-cassandra-stress-tool
- https://www.youtube.com/watch?v=fKV_j7i8KCI

## 8. Autor

- Vedran Orešković
