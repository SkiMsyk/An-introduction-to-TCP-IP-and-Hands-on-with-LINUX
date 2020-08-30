# TCP/IPとは

- インターネットを構成するプロトコルの総称

# たくさんあるプロトコル

- TCP   : Transmission Control Protocol
- IP    : Internet Protocol

## プロトコルが複数あるわけ

- 役割分担
- 目的に合わせてプロトコルを組み合わせる
- 自由度が上がる

# プロトコルの階層構造  
- プロトコルは役割ごとに階層構造で分類できる
- OSIモデル : Open Systems Interconnection Model
    - Application layer
    - Presentation layer
    - Session layer
    - Transport layer
    - Network layer
    - Data link layer
    - Physical layer

注意
- 厳密にはOSIモデルとTCP/IPのプロトコルの間に対応関係はない
- [RFC 122](https://tools.ietf.org/html/rfc1122.html)での定義
    - Application layer
    - Transport layer
    - Internet layer
    - Link layer
- TCP/IPはOSIモデルから独立した階層構造を自ら定義している
- とはいえ，OSIモデルを例に説明されることも多いのが実情

# 準備
```shell
sudo apt-get update
sudo apt-get -y install \
> iproute2 \
> iputils-ping \
> traceroute \
> tcpdump \
> dnsmasq \
> netcat-openbsd \
> python3 \
> curl \
> wget \
> gawk \
> dnsutils \
> procps
```

# ping
```shell
ping -c 3 8.8.8.8
```

result
```shell
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=37 time=6.51 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=37 time=7.42 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=37 time=13.2 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
rtt min/avg/max/mdev = 6.518/9.067/13.262/2.989 ms
```

- `-c 3`というオプションにより，応答を要求するメッセージを３回送った．
- `statistics`には通信に関する統計情報が表示される．
- 3つのパケットを送り，3の応答があり，さらに情報のロスは0%であったことが書かれている

# IP Address
- 8.8.8.8 はGoogleの公開DNSサーバー
- IPでは送信する情報の塊を**Packet**または**Datagram**と呼ぶ
- Packet一つ一つに差出人と宛先のIPアドレスが含まれる
    - **Header** : 差出人，宛先のIPアドレスなど送受信の宛先に関する情報
    - Headerには通信を成立させるために必要な情報が含まれる
- IPヘッダの構造  
    1. Version, IHL, Type of Service, Total Length
    2. Identification, Flags, Fragment Offset
    3. Time to Live, Protocol, Header Checksum
    4. Source Address
    5. Destination Address
    6. Options, Padding
    7. Payload

差出人と宛先の表示
```shell
ip address show
```

```shell
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1000
    link/tunnel6 :: brd ::
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
