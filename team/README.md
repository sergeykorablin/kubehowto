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

Basically there are three types of runner in use broadly in 5 that mention in bold.
* activebackup
* roundrobin
* loadbalance
* broadcast
* lacp

              broadcast — Simple runner which directs the team device to transmit packets via all ports.

              roundrobin — Simple runner which directs the team device to transmits packets in a round-robin fashion.

              activebackup — Watches for link changes and selects active port to be used for data transfers.

              loadbalance  —  To  do  passive load balancing, runner only sets up BPF hash function which will determine port for packet transmit. 
              To do active load balancing, runner moves hashes among available ports trying to reach perfect balance.

              lacp — Implements 802.3ad LACP protocol. Can use same Tx port selection possibilities as loadbalance runner.


### activebackup
Один порт находится в активном состоянии, остальные в запасе. При пропадании линка на активном порту активный порт меняется на ддругой доступный.

### roundrobin
В случае с Round robin все интерфейсы нгаходятся в активном состоянии, запросы отправляються поочередно через все активные порты. Если линк на порту пропадает - запросы начинают идти через оставшиеся.

### loadbalance
Все порты активны одновременно. Система пытается загружать их равномерно.

In industries point of view we use active backup and round robin. Company use load-balancer on application level.

### broadcast
Пакеты отправляются со всех портов

### Lacp
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
