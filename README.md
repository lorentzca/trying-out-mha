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
[vagrant@db1 ~]$ mysql -u root -e "FLUSH TABLES WITH READ LOCK;"
[vagrant@db1 ~]$ mysql -u root -e "SHOW MASTER STATUS;" # ポジションとログファイルのパスメモ
[vagrant@db1 ~]$ mysqldump -u root --all-databases --lock-all-tables > dbdump.db
[vagrant@db1 ~]$ mysql -u root -e "UNLOCK TABLES;"
[vagrant@db1 ~]$ scp dbdump.db vagrant@192.168.44.30:
```

### slave(db2)側

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

mysql5.0のdump食わせると何かのデータ読み込んだタイミングでクラッシュするので調査中

レプリケーション自体は正常に動いた(レプリケーション開始前のデータ捨てることになるけど)

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

フェイルオーバーするとmha-managerがフェイルオーバー実行する

その後mha-managerは終了する？

```
$ vagrant ssh db1
[vagrant@db1 ~]$ sudo killall -9 mysqld mysqld_safe
```
