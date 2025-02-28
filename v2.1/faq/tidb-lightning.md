---
title: TiDB Lightning FAQ
summary: Learn about the frequently asked questions (FAQs) and answers about TiDB Lightning.
category: faq
---

# TiDB Lightning FAQ

## What is the minimum TiDB/TiKV/PD cluster version supported by TiDB Lightning?

The minimum version is 2.0.9.

## Does TiDB Lightning support importing multiple schemas (databases)?

Yes.

## What is the privilege requirements for the target database?

TiDB Lightning requires the following privileges:

* SELECT
* UPDATE
* ALTER
* CREATE
* DROP

If the target database is used to store checkpoints, it additionally requires these privileges:

* INSERT
* DELETE

If the `checksum` configuration item of TiDB Lightning is set to `true`, then the admin user privileges in the downstream TiDB need to be granted to TiDB Lightning.

## TiDB Lightning encountered an error when importing one table. Will it affect other tables? Will the process be terminated?

If only one table has an error encountered, the rest will still be processed normally.

## How to ensure the integrity of the imported data?

TiDB Lightning by default performs checksum on the local data source and the imported tables. If there is checksum mismatch, the process would be aborted. These checksum information can be read from the log.

You could also execute the `ADMIN CHECKSUM TABLE` SQL command on the target table to recompute the checksum of the imported data.

```text
mysql> ADMIN CHECKSUM TABLE `schema`.`table`;
+---------+------------+---------------------+-----------+-------------+
| Db_name | Table_name | Checksum_crc64_xor  | Total_kvs | Total_bytes |
+---------+------------+---------------------+-----------+-------------+
| schema  | table      | 5505282386844578743 |         3 |          96 |
+---------+------------+---------------------+-----------+-------------+
1 row in set (0.01 sec)
```

## What kind of data source format is supported by TiDB Lightning?

In version 2.1.6, TiDB Lightning only supports the SQL dump generated by
[Mydumper](/reference/tools/mydumper.md)
or [CSV files](/reference/tools/tidb-lightning/csv.md) stored in the local filesystem.

## Could TiDB Lightning skip creating schema and tables?

Yes. If you have already created the tables in the target database, you could set `no-schema = true` in the `[mydumper]` section in `tidb-lightning.toml`. This makes Lightning skip the
`CREATE TABLE` invocations and fetch the metadata directly from the target database. Lightning will exit with error if a table is actually missing.

## Can the Strict SQL Mode be disabled to allow importing invalid data?

Yes. By default, the [`sql_mode`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html) used by Lightning is `"STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION"`, which disallows invalid data such as the date `1970-00-00`. The mode can be changed by modifying the `sql-mode` setting in the `[tidb]` section in `tidb-lightning.toml`.

```toml
...
[tidb]
sql-mode = ""
...
```

## Can one `tikv-importer` serve multiple `tidb-lightning` instances?

Yes, as long as every `tidb-lightning` instance operates on different tables.

## How to stop `tikv-importer`?

If it is deployed using TiDB Ansible, run `scripts/stop_importer.sh` under the deployed folder.

Otherwise, obtain the process ID with `ps aux | grep tikv-importer`, and then run `kill «pid»`.

## How to stop `tidb-lightning`?

If it is deployed using TiDB Ansible, run `scripts/stop_lightning.sh` under the deployed folder.

If `tidb-lightning` is running in foreground, simply press <kbd>Ctrl</kbd>+<kbd>C</kbd> to stop it.

Otherwise, obtain the process ID with `ps aux | grep tidb-importer`, then run `kill -2 «pid»`.

## Why `tidb-lightning` suddenly quits while running in background?

It is potentially caused by starting `tidb-lightning` incorrectly, which causes the system to send a SIGHUP signal to stop it. If this is the case, there should be a log entry like:

```
2018/08/10 07:29:08.310 main.go:47: [info] Got signal hangup to exit.
```

We do not recommend using `nohup` directly in the command line. Rather, put the `nohup` inside a script file and execute the script.

## Why my TiDB cluster is using lots of CPU resources and running very slowly after using TiDB Lightning?

If `tidb-lightning` abnormally exited, the cluster might be stuck in the "import mode", which is not suitable for production. You can force the cluster back to "normal mode" using the following command:

```sh
tidb-lightning-ctl --switch-mode=normal
```

## Can TiDB Lightning be used with 1-Gigabit network card?

The TiDB Lightning toolset requires a 10-Gigabit network card. The 1-Gigabit network card is *not acceptable*, especially for `tikv-importer`.
A 1-Gigabit network card can only provide a total bandwidth of 120 MB/s, which has to be shared among all target TiKV stores.
TiDB Lightning can easily saturate all bandwidth of the 1-Gigabit network, and brings down the cluster because PD is unable to contact it anymore.

## Why TiDB Lightning requires so much free space in the target TiKV cluster?

With the default settings of 3 replicas, the space requirement of the target TiKV cluster is 6 times the size of data source. The extra multiple of “2” is a conservative estimation because the following factors are not reflected in the data source:

- The space occupied by indices
- Space amplification in RocksDB

## Can TiKV-Importer be restarted while TiDB Lightning is running?

No. Importer stores some information of engines in memory. If Importer is restarted, Lightning must be restarted.
