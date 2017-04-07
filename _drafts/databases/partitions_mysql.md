## Introduction

We started to have performances issues in our warehouse. Filter the orders by date could take 30 seconds!.

## Facts

We have a database of 417,773 order, nearly a .5 Mega rows, and an total of 712,932 order items.


## Partitions



```sql
# Migration down
ALTER TABLE orders REMOVE PARTITIONING;
```


* Mysql 5.6 don't support partitioning by date intervals, however other database systems whether.