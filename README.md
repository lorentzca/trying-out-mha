# trying out mha

## 準備

鍵作成

```
$ ssh-keygen -f ./id_rsa
```

## Vagrant up

```
$ vagrant up
```

## mha-manager → mha-node ssh接続確認

`All SSH connection tests passed successfullya`.となればok

```
$ vagrant ssh manager
[vagrant@manager ~]$ sudo su -
[root@manager ~]# masterha_check_ssh --conf=/etc/mha.cnf
```

## レプリケーション準備

### master(db1)側

```
[vagrant@db1 ~]$ mysql -u root
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SHOW MASTER STATUS; # ポジションとログファイルのパスメモ
[別窓で実行] [vagrant@db1 ~]$ mysqldump -u root --all-databases --lock-all-tables > dbdump.db
mysql> UNLOCK TABLES;
[vagrant@db1 ~]$ scp dbdump.db vagrant@192.168.44.30:
[vagrant@db1 ~]$ scp dbdump.db vagrant@192.168.44.40:
```

### slave(db2)側

dumpしたデータ読み込みとレプリケーションの開始

```
[vagrant@db2 ~]$ mysql -u root < dbdump.db
[vagrant@db2 ~]$ mysql -u root
mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.44.20',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='slavepass',
    -> MASTER_LOG_FILE='mysql-bin.000001', # メモした値に
    -> MASTER_LOG_POS=189;                 # メモした値に
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G                 # Slave_SQL_Running, Slave_IO_Running がYesになっていればok
```

### slave(db3)側

dumpしたデータ読み込み

```
[vagrant@db3 ~]$ mysql -u root < dbdump.db
```

mysql5.0との互換性の問題を解消するためmysqlのチェックと修正を行う

```
[vagrant@db3 ~]$ sudo mysql_upgrade
```

レプリケーションの開始

```
[vagrant@db3 ~]$ mysql -u root
mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.44.20',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='slavepass',
    -> MASTER_LOG_FILE='mysql-bin.000001', # メモした値に
    -> MASTER_LOG_POS=189;                 # メモした値に
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G                 # Slave_SQL_Running, Slave_IO_Running がYesになっていればok
```

## レプリケーション確認

`MySQL Replication Health is OK.`となればok

```
[root@manager ~]# masterha_check_repl --conf=/etc/mha.cnf
```

## mha-manager起動

```
[root@manager ~]# masterha_manager --conf=/etc/mha.cnf
```

## mha-managerステータス

`mha (pid:8925) is running(0:PING_OK), master:192.168.44.20`となればok

```
[root@manager ~]# masterha_check_status --conf=/etc/mha.cnf
```

## フェイルオーバーさせる

マスターを落とすとmha-managerがフェイルオーバーを実行する

その後mha-managerは終了する

```
$ vagrant ssh db1
[vagrant@db1 ~]$ sudo killall -9 mysqld mysqld_safe
```

フェイルオーバーが成功するとmanagerの/var/log以下にに`mha.failover.complete`という空ファイルが作成される

## 旧マスター(db1)をmysqlのクラスタにslaveとして戻す

db1のmysqlを起動

```
[vagrant@db1 ~]$ sudo /etc/init.d/mysqld start
```

db01はmasterで起動しているのでslaveとしてクラスタに戻す

managerの/var/log/mha.logにslave昇格時のバイナリログのポジションが出力されているので確認(passwordがxxxになっている)

```
[root@manager ~]# cat /var/log/mha.log
...
Mon Mar 30 06:54:23 2015 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='192.168.44.30', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=440184, MASTER_USER='repl', MASTER_PASSWORD='xxx';
```

このログを元にして旧マスターをスレーブとして追加

```
[vagrant@db1 ~]$ mysql -u root
mysql> CHANGE MASTER TO MASTER_HOST='192.168.44.30', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=440184, MASTER_USER='repl', MASTER_PASSWORD='slavepass';
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
```

mha-manager起動

```
[root@manager ~]# masterha_manager --conf=/etc/mha.cnf
```

## 旧マスター(db1)を再びmasterにする

`masterha_master_switch`コマンドを使い手動でフェイルオーバー実行する

masterha_managerをストップさせる

```
[root@manager ~]# masterha_stop --conf=/etc/mha.cnf
```

手動フェイルオーバー実行

```
[root@manager ~]# masterha_master_switch --master_state=alive --conf=/etc/mha.cnf --new_master_host=192.168.44.20 --orig_master_is_new_slave --interactive=0
```

旧マスターがマスターに戻り、現マスターがスレーブに降格する
