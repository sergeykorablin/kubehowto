# Агрегация линков с помощью Teaming

```console
$ sudo apt install libteam-utils
```

```console
$ sudo ip link add type team team0
$ sudo ip link set dev eth0 master team0
$ sudo ip link set dev eth1 master team0
```

```console
$ teamnl team0 getoption mode
*NOMODE*

$ teamnl team0 setoption mode <MODE>
```
## Типы режимов

Есть пять режимов работы, в соновном используются первыке три
* **activebackup**
* **roundrobin**
* **loadbalance**
* broadcast
* lacp

### activebackup
Один порт находится в активном состоянии, остальные в запасе. При пропадании линка на активном порту активный порт меняется на ддругой доступный.

### roundrobin
В случае с Round robin все интерфейсы нгаходятся в активном состоянии, запросы отправляються поочередно через все активные порты. Если линк на порту пропадает - запросы начинают идти через оставшиеся.

### loadbalance
Все порты активны одновременно. Система пытается загружать их равномерно.

In industries point of view we use active backup and round robin. Company use load-balancer on application level.

### broadcast
Пакеты отправляются со всех портов

### lacp
Реализует 802.3ad Link Aggregation Control Protocol. Can use the same transmit port selection possibilities as the load-balancer runner. 

```console
$ ip link add team0 type team
$ teamnl team0 getoption mode
*NOMODE*
$ teamnl team0 setoption mode activebackup
$ teamnl team0 getoption mode
activebackup
$ ip link set dev eth1 master team0
$ ip link set dev eth2 master team0
$ teamnl team0 ports
6: eth2: up 100 fullduplex
5: eth1: up 100 fullduplex
$ ip addr add 192.168.252.2/24 dev team0
$ ip link set team0 up
$ teamnl team0 getoption activeport
0
$ teamnl team0 setoption activeport 5
```
