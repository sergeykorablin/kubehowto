# Настройка Teaming


```console
$ sudo apt install libteam-utils
```

```console
$ teamnl team0 getoption mode
*NOMODE*
```
## Типы режимов

Есть пять режимов работы, в основном используются первыке три
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


## Настройка activebackup

/etc/teamd.d/team0.conf
```json
{
    "device": "team0",
    "ports": {
        "eth0": {
            "link_watch": {
                "name": "ethtool"
            }
        },
        "eth1": {
            "link_watch": {
                "name": "ethtool"
            }
        }
    },
    "runner": {
        "name": "activebackup",
        "hwaddr_policy": "by_active"
    }
}
```


```console
$ sudo teamd -U -D -o -t team0 -f /etc/teamd.d/team0.conf
```

### Настройка systemd
/etc/systemd/system/teamd@.service
```ini
[Unit]
Description=Team Daemon for device %I
Before=network-pre.target
Wants=network-pre.target

[Service]
BusName=org.libteam.teamd.%i
ExecStart=/usr/bin/teamd -U -D -o -t %i -f /run/teamd/%i.conf
Restart=on-failure
RestartPreventExitStatus=1
```

```console
systemctl enable teamd@team0
```

## В ручном режиме


```console
$ sudo ip link add type team team0
$ sudo teamnl team0 setoption mode activebackup

$ sudo ip link set dev eth0 master team0
$ sudo ip link set dev eth1 master team0
```
