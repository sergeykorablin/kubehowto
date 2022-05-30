# Настройка VXLAN
Настроим overlay сеть между мостами нескольких linux хостов.

Два хоста находятся в одной сети:
* host-1 - eth0 - 192.168.0.101
* host-2 - eth0 - 192.168.0.102

Третий хост в другой сети и подключен к первой по vpn:
* host-3 - eth0 - 10.0.0.103

host-1
```console
$ ip link add vxlan1000 type vxlan id 1000 group 225.0.0.1 dev eth0 dstport 4789
$ ip link set up vxlan1000

$ brctl addbr brvx0
$ brctl addif brvx0 vxlan1000
$ ip addr add 10.1.0.1/24 dev brvx0
$ ip link set up brvx0
```

host-2
```console
$ ip link add vxlan1000 type vxlan id 1000 group 225.0.0.1 dev eth0 dstport 4789
$ ip link set up vxlan1000
$ brctl addbr brvx0
$ brctl addif brvx0 vxlan1000
$ ip addr add 10.1.0.2/24 dev brvx0
```

...

host-3
```console
$ ip link add vxlan1000 type vxlan id 1000 group 225.0.0.1 dev eth0 dstport 4789
$ ip link set up vxlan1000
$ brctl addbr brvx0
$ brctl addif brvx0 vxlan1000
$ ip addr add 10.1.0.3/24 dev brvx0

$ bridge fdb append to 00:00:00:00:00:00 dst 192.168.0.101 dev vxlan1000
$ bridge fdb append to 00:00:00:00:00:00 dst 192.168.0.102 dev vxlan1000
```
