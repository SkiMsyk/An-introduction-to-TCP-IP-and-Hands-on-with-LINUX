# Network Namespace

- Linux の Network Namespace という機能を使って実験環境を作る

# Namespaceを使う   

まず次を実行する

```shell
$ sudo ip netns add helloworld
```

しかし，
```shell
$ mount --make-shared /var/run/netns failed: Operation not permitted
```
と怒られた．dockerではrootユーザーであってもip系のコマンドにはpermissionがない場合があるらしい．`docker run`の際にオプションをつけておけばOK

※ ということで一旦コンテナを削除してから作り直した
```shell
$ docker exec --privileged -it -d --name ubuntu ubuntu:18.04
```

無事`sudo ip netns add helloworld`が通る．  
コードが通ると，`helloworld`という名前の`Network Namespace`が作られる．  
作られたかどうかを次のように確認できる．

```shell
$ ip netns list
helloworld
```

作成されたNetwork Namespaceではコマンドを実行できる．

```shell
$ ip netns exec [hogehoge]
```

という形式を取る．

- ip address showと同等のコマンド

```shell
$ sudo ip netns exec helloworld ip address show
```

- `sudo ip address show`と`sudo ip netns exec helloworld ip address show`では全く異なる結果が返される
- これは，Network Namespaceがネットワークにおいてシステムから独立した領域を作っていることによる
  - ネットワークインターフェース, ルーティングテーブルなども独立している

Network Namespaceの削除

```shell
$ sudo ip netns delete helloworld
```

ちなみに，

```shell
$ sudo ip  netns exec helloworld bash
```

でNetwork Namespaceの環境のシェルを立ち上げることができる．実際にはこっちの方がタイプが楽．

# 繋ぐ  

- 2つのスペースを作る
- 次のような構成のネットワークを作る
  - ns1: ns1-veth0
  - ns2: ns2-veth0
- この二つのネットワークを繋げる

```shell
$ sudo ip netns add ns1
$ sudo ip netns add ns2
```

このコードを実行しただけでは，`ns1,ns2`の二つのネットワークは独立している．  
次に，両者を繋ぐために`veth: Virtual Ethernet Device`という仮想的なネットワークインターフェースを使う．

`veth`を作るコマンドを実行する

```shell
$ sudo ip link add ns1-veth0 type veth peer name ns2-veth0
```

これで仮想的なネットワークインターフェース（veth）が作成された．  
- `ns1-veth0, ns2-veth0`は任意の名前

作られたネットワークインターフェースの確認

```shell
$ ip link show | grep veth
```

このvethインターフェースのペアをNetwork Namespaceで使えるようにする．  
- 今の段階では，作成されたvethインターフェースはシステム領域に属している．
- これをNetwork Namespaceの領域に移す
- ip link setサブコマンドを使う

```shell
$ sudo ip link set ns1-veth0 netns ns1
$ sudo ip link set ns2-veth0 netns ns2
```

- vethインターフェースがNetwork Namespaceで使えるように

この状態では

```shell
$ ip link show | grep veth
$ 
```

となるように，システム領域からvethインターフェースは見えなくなっている．  
一方，`ns1, ns2`それぞれの領域でvethインターフェースが稼働していることを確認できる．

```shell
$ sudo ip netns exec <name> ip link show | grep veth
```

これで，ns1とns2のNetwork Namespaceがvethインターフェースによって繋がった．
ただし，**IPを使った通信を行うにはIPアドレスが必要になる**が，IPアドレスがないので，通信はできない．

- IPを使った通信を実現させるために，vethインターフェースにIPアドレスを付与する必要がある
- 以下のコマンドでIPアドレスを付与する

|veth|ip address|
|:-|:-|
|ns1-veth0|192.0.2.1|
|ns2-veth0|192.0.2.2|


```shell
$ sudo ip netns exec ns1 ip address add 192.0.2.1/24 dev ns1-veth0
$ sudo ip netns exec ns2 ip address add 192.0.2.2/24 dev ns2-veth0
```

## ネットワークインターフェースにおけるUPとDOWN  

- ネットワークインターフェースには UP, DOWN という二つの状態がある
- それぞれ，UP/active, DOWN/non-active を意味する

初期状態ではDOWNになっていることがあるので確かめておく必要がある．
また，明示的にUPを設定するのが良い．

```shell
$ sudo ip netns exec ns1 ip link set ns1-veth0 up
$ sudo ip netns exec ns2 ip link set ns2-veth0 up
```

## Network Namespace同士の通信  
今，ns1, ns2というNetwork Namespaceはvethインターフェースのペアで繋がれ，それぞれIPを持っていて，ネットワークインターフェースも有効の状態になっているので，IPを介した通信ができるようになった．

- pingコマンドを実行してみよう

```shell
$ sudo ip netns exec ns1 ping -c 3 192.0.2.2
PING 192.0.2.2 (192.0.2.2) 56(84) bytes of data.
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.284 ms
64 bytes from 192.0.2.2: icmp_seq=2 ttl=64 time=0.212 ms
64 bytes from 192.0.2.2: icmp_seq=3 ttl=64 time=0.174 ms

--- 192.0.2.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2090ms
rtt min/avg/max/mdev = 0.174/0.223/0.284/0.047 ms
```

これが成功していれば，Network Namespaceという機能を使って，TCP/IPの実験環境を構築できた頃になる．
これで，現実のサービスと通信することなく，閉じた環境で実験ができるようになった．

## メモ  
- ネットワークはただ物理的に敗戦しただけでは使えない
- 物理的な敗戦をどのように扱うかを論理的に設定する必要がある
- ネットワークの構成については物理と論理を分けて扱うことに慣れよう

# ルータ  
先ほどの実験のまとめ
- Network Namespaceを二つ用意
- これらをvethインターフェースで繋ぐ
- それぞれのvethインターフェースにIPアドレスを付与
- pingでこの二つを通信させる

この実験の中にはルータが登場しない．IPによるネットワークではルータによるパケットのリレーが行われることで成り立っているが，今の実験におけるルータの役割とは何だったのか．

## IPで構築されたネットワークのセグメント  
IPで構築されたネットワークには**セグメント**という概念があり，実は，同じネットワークのセグメントに所属するIPアドレス同士は，基本的にルータがいなくても通信が可能になっている．

- 実験では，vethインターフェイスのペアに同じネットワークのセグメントのIPアドレスが付与されていたのでルータがなくても通信ができた
- セグメントが異なる場合はルータを用意しなければならない

**ネットワークセグメント**とは？

セグメントを理解するための前提知識
- IPv4におけるIPアドレスは32ビットの空間を持った正の整数
- 4つの8ビット領域に分けて10進数で表現したものが，普段目にしているIPアドレス
  - 192.0.2.1 みたいな
- ネットワーク部とホスト部
  - 24ビット目まで：ネットワーク部 → ネットワークアドレス
  - 24ビット目より後ろ：ホスト部 → ホストアドレス

Example
|Network part|Host part|
|:--|:--|
|192.0.2|.1|

ネットワークセグメントの識別は，ネットワークアドレスによって行われる．  
ところで，IPアドレスのネットワーク部とホスト部の境目はどのように識別されているのか
- 192.0.2.1/24 と指定いたのを思い出そう
- `/24`がネットワーク部とホスト部の境目を指定している
- この書き方はCIDR: Classless Inter-Domain Routing と呼ばれる

CIDRとは別の表現 **サブネットマスク**  
- ネットワーク部だけを1にして，ホスト部を0にした32ビットの整数．
  - 11111111 111111111 11111111 00000000
- IPアドレスとサブネットマスクをアンド演算する

|||
|:-|:-|
|`192.0.2.1`|`11000000 00000000 00000010 00000001`
|サブネットマスク: `255.255.255.0`|`11111111 11111111 11111111 00000000`|
|アンド演算の結果: `192.0.2.0`|`11000000 00000000 00000010 00000000`|
  

---
# ルータを入れる  
- セグメントが異なるネットワークを作る
- Network Namespaceは全部で３つ
  - ns1
  - router
  - ns2
- vethインターフェースも ns1 - router, router - ns2 を繋ぐので２つに増える
  - ns1 - router : ns1-veth0 - gw-veth0
  - ns2 - router : ns2-veth0 - gw-veth1

|Network namespace|veth name|IP address|
|:--|:--|:--|
|ns1|ns1-veth0||192.0.2.1|
|router|gw-veth0|192.0.2.254|
|router|gw-veth1|198.51.100.254|
|ns2|ns2-veth0|198.51.100.1|

## 実験の目標  
**セグメントを跨いだ通信を成功させる**
- 192.0.2.0と198.51.100.0との通信をする
- ルーターを介す

## 準備  
先ほど作ったNetwork Namespaceは削除する

```shell
$ for ns in $(ip netns list | awk '{print $1}'): do sudo ip netns delete $ns: done
```

- Network namespaceの作成
```shell
$ sudo ip netns add ns1
$ sudo ip netns add router
$ sudo ip netns add ns2
```

- vethインターフェースの作成
```shell
$ sudo ip link add ns1-veth0 type veth peer name gw-veth0
$ sudo ip link add ns2-veth0 type veth peer name gw veth1
```

- vethインターフェースの所属の変更

```shell
$ sudo ip link set ns1-veth0 netns ns1
$ sudo ip link set gw-veth0 netns router
$ sudo ip link set gw-veth1 netns router
$ sudo ip link set ns2-veth0 netns ns2
```

- vethインターフェースの状態を有効にする

```shell
$ sudo ip netns exec ns1 ip link set ns1-veth0 up
$ sudo ip netns exec router ip link set gw-veth0 up
$ sudo ip netns exec router ip link set gw-veth1 up
$ sudo ip netns exec ns2 ip link set ns2-veth0 up
```

ここまでで物理的な配線が完了．

- IPアドレスの設定

ns1側(`192.0.2.0`)
```shell
$ sudo ip netns exec ns1 ip address add 192.0.2.1/24 dev ns1-veth0
$ sudo ip netns exec router ip address add 192.0.2.254/24 dev gw-veth0
```

ns2側(`198.51.100.0`)
```shell
$ sudo ip netns exec ns2 ip address add 198.51.100.1/24 dev ns2-veth0
$ sudo ip netns exec ns2 ip address add 198.51.100.254/24 dev gw-veth1
```

## 実験  

### ns1 → routerへの通信  

```shell
$ sudo ip netns exec ns1 ping -c 3 192.0.2.254 -I 192.0.2.1
PING 192.0.2.254 (192.0.2.254) from 192.0.2.1 : 56(84) bytes of data.
64 bytes from 192.0.2.254: icmp_seq=1 ttl=64 time=0.233 ms
64 bytes from 192.0.2.254: icmp_seq=2 ttl=64 time=0.288 ms
64 bytes from 192.0.2.254: icmp_seq=3 ttl=64 time=0.165 ms

--- 192.0.2.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2031ms
rtt min/avg/max/mdev = 0.165/0.228/0.288/0.053 ms
```

### ns2 → routerへの通信
```shell
$ sudo ip netns exec ns2 ping -c 3 198.51.100.254 -I 198.51.100.1
PING 198.51.100.254 (198.51.100.254) from 198.51.100.1 : 56(84) bytes of data.
64 bytes from 198.51.100.254: icmp_seq=1 ttl=64 time=0.175 ms
64 bytes from 198.51.100.254: icmp_seq=2 ttl=64 time=0.097 ms
64 bytes from 198.51.100.254: icmp_seq=3 ttl=64 time=0.588 ms

--- 198.51.100.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2087ms
rtt min/avg/max/mdev = 0.097/0.286/0.588/0.216 ms
```

両方成功．

### routerを介して ns1 - ns2での通信  
この状態で実行してみる．

```shell
$ sudo ip netns exec ns1 ping -c 3 198.51.100.1 -I 192.0.2.1
PING 198.51.100.1 (198.51.100.1) from 192.0.2.1 : 56(84) bytes of data.
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable
ping: sendmsg: Network is unreachable

--- 198.51.100.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2043ms
```

`Network is unreachable` はネットワークに到達できない場合のエラーを指す．  
今の状態では，ルーティングテーブルの情報が足りないため，どのルートを辿れば相手にたどり着くのかを解決できない．

- ルーティングテーブルを確認する

```shell
$ sudo ip netns exec ns1 ip route show
192.0.2.0/24 dev ns1-veth0 proto kernel scope link src 192.0.2.1
```

```shell
$ sudo ip netns exec ns2 ip route show
198.51.100.0/24 dev ns2-veth0 proto kernel scope link src 198.51.100.1 
```

現状のルーティングテーブルには，`192.0.2.0/24`に宛てた通信は `ns1-veth0`というネットワークインターフェースで通信する，という情報しかない．今必要なものは，
- `ns1`からの通信の際，宛先に一致しない時に探しに行くデフォルトルートの追加
- `ns2`からの通信の際，宛先に一致しない時に探しに行くデフォルトルートの追加

```shell
$ sudo ip netns exec ns1 ip route add default via 192.0.2.254
$ sudo ip netns exec ns2 ip route add default via 198.51.100.254
```

再度，ns1 から ns2にpingで通信テストを行なってみる．

```shell
$ sudo ip netns exec ns1 ping -c 3 198.51.100.1 -I 192.0.2.1
PING 198.51.100.1 (198.51.100.1) from 192.0.2.1 : 56(84) bytes of data.
64 bytes from 198.51.100.1: icmp_seq=1 ttl=63 time=0.137 ms
64 bytes from 198.51.100.1: icmp_seq=2 ttl=63 time=0.194 ms
64 bytes from 198.51.100.1: icmp_seq=3 ttl=63 time=0.264 ms

--- 198.51.100.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2069ms
rtt min/avg/max/mdev = 0.137/0.198/0.264/0.053 ms
```

成功！

環境によってはこの状態でもまだ失敗する場合がある．その理由は，カーネルのパラメータの値による．想定している失敗を再現するには以下を実行しておくと再現する．

```shell
$ sudo ip netns exec router sysctl net.ipv4.ip_forward=0
```

すると，次のような結果を得る．

```shell
$ sudo ip netns exec ns1 ping -c 3 198.51.100.1 -I 192.0.2.1
PING 198.51.100.1 (198.51.100.1) from 192.0.2.1 : 56(84) bytes of data.

--- 198.51.100.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2038ms
```

返答がなく，パケットが100%と失われている．これを修正するために，先ほどとは逆にパラメータの値を変更し通信を有効にする．

```shell
$ sudo ip netns exec router sysctl net.ipv4.ip_forward=1
```

## カーネル  
ここでいうカーネルとは，OS(Operating System)の心臓部のことを指している．パケットを処理するプロトコルスタックと呼ばれる領域は，一般的にはこのカーネルの中に存在する．  

`net.ipv4.ip_forward`というパラメータは，LinuxにおいてIPv4のルータとして動作するのかどうかを決めるパラメータである．つまり，ここが1であればこのルータはIPv4のルータとして動作し，0であればそうではない．

この実験環境はIPv4で行なっているわけなので，ルータもIPv4環境で動くように設定をしなければいけないのである．

# ルータをふやす

次のような構成で再度環境を作ってみよう

|Network namespace|veth name|IP address|
|:--|:--|:--|
|ns1|ns1-veth0|192.0.2.1|
|router1|gw1-veth0|192.0.2.254|
|router1|gw1-veth1|203.0.113.1|
|router2|gw2-veth0|203.0.113.2|
|router2|gw2-veth1|198.51.100.254|
|ns2|ns2-veth0|198.51.100.1|

## ポイント  
- route1とrouter2の間のルートの設定
  - 直接IPアドレスを指定してもよし→ Static routing
  - 今回の場合はデフォルトルートとして追加してもよし