# consul

## vagrant

```
$ vagrant up
```

## start consul

```
[vagrant@manager ~]$ sudo /sbin/service consul start
```

## add node

```
[vagrant@manager ~]$ $ curl -s -X PUT -d '{ "Node": "masterdb", "Address": "192.168.44.20" }' http://192.168.44.10:8500/v1/catalog/register
```

## name resolusioin

```
[vagrant@app ~]$ dig dbmaster.node.consul

; <<>> DiG 9.3.6-P1-RedHat-9.3.6-25.P1.el5_11.5 <<>> dbmaster.node.consul
;; global options:  printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51065
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;dbmaster.node.consul.          IN      A

;; ANSWER SECTION:
dbmaster.node.consul.   0       IN      A       192.168.44.30

;; Query time: 2 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Jan 20 09:26:29 2016
;; MSG SIZE  rcvd: 74
```