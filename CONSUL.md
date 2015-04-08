# consul

## vagrant

```
$ vagrant up
```

## start consul

```
$ vagrant ssh manager
[vagrant@manager ~]$ $ ./consul agent -server -bootstrap -client=192.168.44.10 -dc=vagrantdc -data-dir=/tmp/manager -bind=192.168.44.10
```

## add node

```
$ curl -s -X PUT -d '{ "Node": "master-db", "Address": "192.168.44.20" }' http://192.168.44.10:8500/v1/catalog/register
```

digで名前を引いてみる

```
[vagrant@app ~]$ dig master-db.node.consul

; <<>> DiG 9.3.6-P1-RedHat-9.3.6-25.P1.el5_11.2 <<>> master-db.node.consul
;; global options:  printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12505
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;master-db.node.consul.         IN      A

;; ANSWER SECTION:
master-db.node.consul.  0       IN      A       192.168.44.20

;; Query time: 3 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Apr  8 10:08:57 2015
;; MSG SIZE  rcvd: 76
```

ping

```
[vagrant@app ~]$ ping master-db.node.consul
PING master-db.node.consul (192.168.44.20) 56(84) bytes of data.
64 bytes from 192.168.44.20: icmp_seq=1 ttl=64 time=0.200 ms
64 bytes from 192.168.44.20: icmp_seq=2 ttl=64 time=0.315 ms
64 bytes from 192.168.44.20: icmp_seq=3 ttl=64 time=0.249 ms
...
```
