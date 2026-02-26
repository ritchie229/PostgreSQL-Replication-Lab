# PATRONI

## 🧭 Архитектура


Минимальный прод-вариант:
```pgsql
        etcd (DCS)
         |
 -----------------
 |       |       |
pg1     pg2     pg3
(master)(replica)(replica)
   |
 logical → analytics / backup / BI
```


## Patroni-кластер

Сначала:
```nginx
etcd
  |
patroni1 ─ postgres
patroni2 ─ postgres
```
Потом:

```nginx
logical-replica (отдельный контейнер)
```

## Patroni + etcd в Docker

Цель:
```pgsql
patronictl list

+ Cluster: labcluster ----+
| Member | Role   | State |
+--------+--------+-------+
| pg1    | Leader | running
| pg2    | Replica| running
```




