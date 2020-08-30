# Ethernet  

# Ethernetの役割  
- 身近な表現をすると，近隣までの荷物の配達
- データを送る単位を「フレーム」と呼ぶ
  - IPのパケットは異なるフレームに積み替えられながら目的地に運ばれる
  - フレームについて，送信元と送信先を管理する必要が出てくる
  - このとき使われるのが**MACアドレス** (Media Access Control address)

## MACアドレス  
- MACアドレスはEthernetのフレームを送受信するネットワーク機器ごとに付与される
- 48ビットの空間を持った正の整数
- 原則的に全世界において一意な識別子として扱われる
- 上位24ビットがベンダー識別子，下位24ビットが機器の識別子

# フレームの観察  
３章で作ったのと同じ構成の環境を作る
- 2つのNetwork Namespaceを1つのvethインターフェースのペアで繋ぐ

まず，既に作られているNetwork Namespaceを削除する

```shell
$ $ $ for ns in $(ip netns list | awk '{print $1}'); dsudo ip netns delete $ns; done
```

## 構成  
|Network namespace|veth name|IP address|
|:--|:--|:--|
|ns1|ns1-veth0|192.0.2.1|
|ns2|ns2-veth0|192.0.2.2|

```shell
$ $ sudo ip netns add ns1
$ $ sudo ip netns add ns2
$ $ sudo ip link add ns1-veth0 type veth peer name ns2-veth0
$ $ sudo ip link set ns1-veth0 netns ns1
$ $ sudo ip link set ns2-veth0 netns ns2
$ $ sudo ip netns exec ns1 ip link set ns1-veth0 up
$ $ sudo ip netns exec ns2 ip link set ns2-veth0 up
$ $ sudo ip netns exec ns1 ip address add 192.0.2.1/24 dev ns1-veth0
$ $ sudo ip netns exec ns2 ip address add 192.0.2.2/24 dev ns2-veth0
```

MACアドレスの変更

```shell
$ $ sudo ip netns exec ns1 ip link set dev ns1-veth0 address 00:00:5E:00:53:01
$ $ sudo ip netns exec ns2 ip link set dev ns2-veth0 address 00:00:5E:00:53:02
```

変更の確認

```shell
$ $ $sudo ip netns exec ns1 ip link show | grep link/ether
link/ether 00:00:5e:00:53:01 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

```shell
$ $ sudo ip netns exec ns2 ip link show | grep link/ether
link/ether 00:00:5e:00:53:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

パケットキャプチャの準備  
- 別のターミナルを開いて，次のコマンドを入力

```shell
$ $ sudo ip netns exec ns1 tcpdump -tnel -i ns1-veth0 icmp
```

- 元のターミナルでpingで通信をする．
```shell
$ $ sudo ip netns exec ns1 ping -c 3 192.0.2.2 -I 192.0.2.1
PING 192.0.2.2 (192.0.2.2) from 192.0.2.1 : 56(84) bytes of data.
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.128 ms
64 bytes from 192.0.2.2: icmp_seq=2 ttl=64 time=0.060 ms
64 bytes from 192.0.2.2: icmp_seq=3 ttl=64 time=0.066 ms

--- 192.0.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2030ms
rtt min/avg/max/mdev = 0.060/0.084/0.128/0.032 ms
```

tcpdumpを実行しているターミナルの表示を見てみる．
```shell
$ $ sudo ip netns exec ns1 tcpdump -tnel -i ns1-veth0 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ns1-veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3840, seq 1, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3840, seq 1, length 64
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3840, seq 2, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3840, seq 2, length 64
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3840, seq 3, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3840, seq 3, length 64
```

`-e`オプションの指定によって，最初に表示される列がIPではなくEthernetに関する情報になっている．

- Ethernet HeaderはIP Headerの直前に連結されている．

# MACアドレスを知る
- IPアドレスを持った相手とEthernetで通信するには，相手のMACアドレスを知る必要がある

## ARP : Address Resolution Protocol
このプロトコルを使ってIPアドレスに対応するMACアドレスを解決できる．

# ARPの観察

MACアドレスのキャッシュを全て削除するコマンド．
```shell
$ $ sudo ip netns exec ns1 ip neigh flush all
```

一般的なプロトコルスタックでは，既知のMACアドレスをキャッシュしていて，キャッシュが残った状態ではこの内容が優先して使われる．

ARPを観察するためのオプションを追加して再度tcpdumpコマンドを実行

```shell
$ $ sudo ip netns exec ns1 tcpdump -tnel -i ns1-veth0 icmp or arp
```

```shell
$ $ sudo ip netns exec ns1 ping -c 3 192.0.2.2 -I 192.0.2.1
PING 192.0.2.2 (192.0.2.2) from 192.0.2.1 : 56(84) bytes of data.
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.781 ms
64 bytes from 192.0.2.2: icmp_seq=2 ttl=64 time=0.193 ms
64 bytes from 192.0.2.2: icmp_seq=3 ttl=64 time=0.679 ms

--- 192.0.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2056ms
rtt min/avg/max/mdev = 0.193/0.551/0.781/0.256 ms
```

すると次のようにキャプチャされる

```shell
$ $ sudo ip netns exec ns1 tcpdump -tnel -i ns1-veth0 icmp or arp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ns1-veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
00:00:5e:00:53:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.0.2.2 tell 192.0.2.1, length 28
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype ARP (0x0806), length 42: Reply 192.0.2.2 is-at 00:00:5e:00:53:02, length 28
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3846, seq 1, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3846, seq 1, length 64
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3846, seq 2, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3846, seq 2, length 64
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype IPv4 (0x0800), length 98: 192.0.2.1 > 192.0.2.2: ICMP echo request, id 3846, seq 3, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype IPv4 (0x0800), length 98: 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 3846, seq 3, length 64
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype ARP (0x0806), length 42: Request who-has 192.0.2.1 tell 192.0.2.2, length 28
00:00:5e:00:53:01 > 00:00:5e:00:53:02, ethertype ARP (0x0806), length 42: Reply 192.0.2.1 is-at 00:00:5e:00:53:01, length 28
```

最初は送信先のMACアドレスがわからないので

```shell
00:00:5e:00:53:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.0.2.2 tell 192.0.2.1, length 28
```
というところで，`192.0.2.2`のIPアドレスを持ったMACアドレスを解決しようとしている．

次にARPによって解決されたMACアドレスが返ってきているのが次の部分

```shell
00:00:5e:00:53:02 > 00:00:5e:00:53:01, ethertype ARP (0x0806), length 42: Reply 192.0.2.2 is-at 00:00:5e:00:53:02, length 28
```

このやりとりを，
- ARPリクエスト
- ARPリプライ

と呼ぶ．

また，ARPもIPと同じようにEthernetのフレームによって運ばれていることがわかる．
- Ethernet Header
- ARP Header
がパケットに含まれる．

ここで，最初の送信先の`ff:ff:ff:ff:ff:ff`というMACアドレスはブロードキャストアドレスと呼ばれる特別なMACアドレスである．
- このフレームが届く限りすべての範囲の聞きにリクエストを送るという意味で使われる

要するに，
「このIPアドレスのMACアドレスを誰か教えて」
と知っているすべての人に聞いているようなものである．

- IPv6ではMACアドレスの解決にARPは使われていない．

# パケットの積み替え

この構成のネットワークを再度作る．

|Network namespace|veth name|IP address|
|:--|:--|:--|
|ns1|ns1-veth0||192.0.2.1|
|router|gw-veth0|192.0.2.254|
|router|gw-veth1|198.51.100.254|
|ns2|ns2-veth0|198.51.100.1|

```shell
$ $ for ns in $(ip netns list | awk '{print $1}'); do sudo ip netns delete $ns; done
```

```shell
$ sudo ip netns add ns1
$ sudo ip netns add ns2
$ sudo ip netns add router

$ sudo ip link add ns1-veth0 type veth peer name gw-veth0
$ sudo ip link add ns2-veth0 type veth peer name gw-veth1

$ sudo ip link set ns1-veth0 netns ns1
$ sudo ip link set gw-veth0 netns router
$ sudo ip link set gw-veth1 netns router
$ sudo ip link set ns2-veth0 netns ns2

$ sudo ip netns exec ns1 ip link set ns1-veth0 up
$ sudo ip netns exec ns2 ip link set ns2-veth0 up
$ sudo ip netns exec router ip link set gw-veth0 up
$ sudo ip netns exec router ip link set gw-veth1 up

$ sudo ip netns exec ns1 ip address add 192.0.2.1/24 dev ns1-veth0
$ sudo ip netns exec router ip address add 192.0.2.254/24 dev gw-veth0
$ sudo ip netns exec router ip address add 198.51.100.254/24 dev gw-veth1
$ sudo ip netns exec ns2 ip address add 198.51.100.1/24 dev ns2-veth0

$ sudo ip netns exec ns1 ip route add default via 192.0.2.254
$ sudo ip netns exec ns2 ip route add default via 198.51.100.254

$ sudo ip netns exec router sysctl net.ipv4.ip_forward=1

$ sudo ip netns exec ns1 ip link set dev ns1-veth0 address 00:00:5E:00:53:11
$ sudo ip netns exec ns2 ip link set dev ns2-veth0 address 00:00:5E:00:53:22

$ sudo ip netns exec router ip link set dev gw-veth0 address 00:00:5E:00:53:12
$ sudo ip netns exec router ip link set dev gw-veth1 address 00:00:5E:00:53:21

```

## 観察

gw-veth0側
```shell
$ sudo ip netns exec router tcpdump -tnel -i gw-veth0 icmp or arp
```

gw-veth1側
```shell
$ sudo ip netns exec router tcpdump -tnel -i gw-veth1 icmp or arp
```

ping
```shell
$ sudo ip netns exec ns1 ping -c 3 198.51.100.1 -I 192.0.2.1
```

# ブリッジ  
- これまでの実験では一つのブロードキャストドメインにネットワーク機器は２台まで
- 一般的な家庭においてさえも，インターネットに接続する機器はもっと多い

改めて，vethインターフェースとは
- ２つのネットワークインターフェースがペアになって動作する仮想的なネットワークインターフェースカード

通常，ルータと端末の間にハブがあることが多い．これはスイッチングハブなどと呼ばれる．これのおかげで複数の端末が同じブロードキャストドメインでインターネットに接続できるようになる．

スイッチングハブという言葉をより一般的に言い直したものがブリッジである．ここでルータの説明を再掲

- ルータ：ネットワーク層でパケットを転送する機器

これと同じように
- ブリッジ：データリンク層でフレームを転送する機器

**ブリッジの仕事は，自身のどのポートにどのMACアドレスの機器が繋がっているのかを管理すること**  

フレームがやってくると，MACアドレスを下に適切に振り分けるということをしている．そのためにMACアドレスを管理する必要があり，これを**MACアドレステーブルという**

# 実験：ブリッジを導入した環境の構築  

- Network Namespaceは3つ
  1. ns1
  2. ns2
  3. ns3

これまでは，同じセグメントには２つのNetwork Namespaceしかつながらなかったが，Linuxのネットワークブリッジを使って複数つなげるようにする．

- Network Namespaceはブリッジを介してお互いにつながる．
- 結果的に3つのNetwork Namespaceは同じセグメントに繋ぐことができる

## 構成  

- ネットワークセグメント: 192.0.2.0
  - ns1
    - ns1-veth0: 192.0.2.1 / 00:00:5E:00:53:01
  - ns2
    - ns2-veth0: 192.0.2.2 / 00:00:5E:00:53:02
  - ns3
    - ns3-veth0: 192.0.2.3 / 00:00:5E:00:53:03
- br0
  - ns1-br0
  - ns2-br0
  - ns3-br0

既存のNetwork Namespaceを削除
```shell
$ for ns in $(ip netns list | awk '{print $1}'); do sudo ip netns delete $ns; done
```


```shell
$ sudo ip netns add ns1
$ sudo ip netns add ns2
$ sudo ip netns add ns3

$ sudo ip link add ns1-veth0 type veth peer name ns1-br0
$ sudo ip link add ns2-veth0 type veth peer name ns2-br0
$ sudo ip link add ns3-veth0 type veth peer name ns3-br0

$ sudo ip link set ns1-veth0 netns ns1
$ sudo ip link set ns2-veth0 netns ns2
$ sudo ip link set ns3-veth0 netns ns3

$ sudo ip netns exec ns1 ip link set ns1-veth0 up
$ sudo ip netns exec ns2 ip link set ns2-veth0 up
$ sudo ip netns exec ns3 ip link set ns3-veth0 up

$ sudo ip netns exec ns1 ip address add 192.0.2.1/24 dev ns1-veth0
$ sudo ip netns exec ns2 ip address add 192.0.2.2/24 dev ns2-veth0
$ sudo ip netns exec ns3 ip address add 192.0.2.3/24 dev ns3-veth0

$ sudo ip netns exec ns1 ip link set dev ns1-veth0 address 00:00:5E:00:53:01
$ sudo ip netns exec ns2 ip link set dev ns2-veth0 address 00:00:5E:00:53:02
$ sudo ip netns exec ns3 ip link set dev ns3-veth0 address 00:00:5E:00:53:03

```

次にネットワークブリッジを追加する

```shell
$ sudo ip link add dev br0 type bridge
$ sudo ip link set br0 up

```

用意したvethインターフェースの片方をネットワークブリッジに追加する

```shell
$ sudo ip link set ns1-br0 master br0
$ sudo ip link set ns2-br0 master br0
$ sudo ip link set ns3-br0 master br0
$ sudo ip link set ns1-br0 up
$ sudo ip link set ns2-br0 up
$ sudo ip link set ns3-br0 up

```

以上で，ns1-ns2, ns1-ns3, ns2-ns3の間での通信が可能になる．
vethインターフェースだけではこのような構成を実現することは難しい．（vethはペアのネットワークインターフェースし）

もう一つ，ブリッジ（スイッチングハブ）の重要な機能に
- 自身に繋がっているMACアドレスを学習する
- 学習ずみのアドレスについては無駄なフレーム送信をしないようになっている

という機能がある．ただし，MACアドレステーブルのエントリには有効期限がある

# まとめ
- Ethernetというプロトコルについて概観した
- 実験用のネットワークを使って，パケットがEthernetのフレームで運ばれる様子を観察した
- ブリッジを使うことで１つのブロードキャストドメインに複数のネットワーク機器をつなげる方法を見た
- 