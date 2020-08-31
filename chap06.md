# Application layer Protocol
- TCPのような信頼性のあるプロトコルを使うことで，インターネット上のパケットのやりとりが実用的になった
- ここではOSIモデルでいうアプリケーション層のプロトコルを概観する
- アプリケーション層のプロトコルは無数に存在．その中でも代表的なものを見ていく
  - HTTP: Hypertext Transfer Protocol
  - DNS: Domain Name System
  - DHCP: Dynamic Host Configuration Protocol

# HTTP
- webページを閲覧するための通信には，HTTP, HTTPSが主に使われている
- HTTPはトランスポート層にTCPを用いるテキストベースのプロトコル（ここで扱うのはHTTP/1.0, 1.1）

# 実験

## 準備  
ディレクトリを用意し移動する
```shell
mkdir -p /var/tmp/http-home
cd /var/tmp/http-home/
```

シンプルなHTMLを作成する

```shell
cat << 'EOF' > index.html
<!doctype html>
<html>
  <head>
    <title>Hello, World</title>
  </head>
  <body>
    <h1>Hello, World</h1>
  </body>
</html>

EOF
```

PythonでHTTPサーバーを起動する．最後のポート番号は空いているものを使う．
起動したディレクトリが仮想sever上に存在しているような状態になる．

```shell
sudo python3 -m http.server -b 127.0.0.1 80
```

別のターミナルを開き次のコマンドでこのサーバーと通信をしてみる

```shell
echo -en "GET / HTTP/1.0\r\n\r\n" | nc 127.0.0.1 80
```

`"GET / HTTP/1.0\r\n\r\n"` という文字列を ncコマンドでサーバーに送信する．`\r\n` は改行コード．

curlでの実行 

```shell
curl -X GET -D - http://127.0.0.1/
```

w3mでの実行

```shell
sudo apt-get -y install w3m
w3m http://127.0.0.1/
```

# DNS
- DNSはIPアドレスを解決するためのシステム
- DNSを利用することで，URLとIPアドレスを１対１に対応させる
- DNSを利用するのはブラウザだけでなく，localhostというドメイン名について考える
    - これは，ループバックアドレスの127.0.0.1 を表したドメイン名．
    - pingを使って確かめてみる

```shell
ping -c 3 localhost  
PING localhost (127.0.0.1) 56(84) bytes of data.
64 bytes from localhost (127.0.0.1): icmp_seq=1 ttl=64 time=0.129 ms
64 bytes from localhost (127.0.0.1): icmp_seq=2 ttl=64 time=0.075 ms
64 bytes from localhost (127.0.0.1): icmp_seq=3 ttl=64 time=0.046 ms

--- localhost ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2041ms
rtt min/avg/max/mdev = 0.046/0.083/0.129/0.035 ms
```

## どのように解決されるのか  
- ドメイン名の解決はOSの中のresolverというプログラムが担当します．
- resolverはコンピューターの中にあるhostsと呼ばれるファイルを参照したり，外部サーバに問い合わせることで名前解決をする
  - 具体的には，テキスト形式でドメイン名とIPアドレスの対応が書かれているので，それを読む
  - localhost についても例外ではない

```shell
grep 127.0.0.1 /etc/hosts
127.0.0.1	localhost
```

- resolverが問い合わせるDNSのサーバーをネームサーバーとも呼ぶ
- ネームサーバーは加入しているISPが提供しているものや，公開されているDNSサービスが提供していたりする

## ネームサーバーを使った名前解決

Terminal-1
```shell
sudo tcpdump -tnl -i any "udp and port 53"
```

Terminal-2
```shell
dig +short @8.8.8.8 example.org A
93.184.216.34
```

このときTerminal-1で通信内容を見てみると
```shell
IP 172.17.0.2.60547 > 8.8.8.8.53: 27072+ [1au] A? example.org. (52)
IP 8.8.8.8.53 > 172.17.0.2.60547: 27072$ 1/0/1 A 93.184.216.34 (56)
```

となっており，最終的にexample.org のIPアドレスが返ってきていることがわかる．

# DHCP
- 端末がTCP/IPでの通信をするために少なくとも必要になる設定
  - ネットワークインターフェースに対してIPアドレスの付与
  - デフォルトルートのルーティングエントリをルーティングテーブルに追加
  - 名前解決のためのネームサーバーを指定

これらを自動的に設定するのがDHCP．

- DHCPは
  - サーバー・クライアント方式のプロトコル
  - ネットワークの設定を配布するコンピュータがDHCPサーバー
  - 設定を受ける側をDHCPクライアント
  - DHCPクライアントはDHCPサーバーに問合せ，配布された設定内容を自らに反映します


## 実際に使ってみる

|Network Namespace|veth interface name|
|:-|:-|
|client|c-veth0|
|server|s-veth0|

```shell
for ns in $(ip netns list | awk '{print $1}'); do sudo ip netns delete $ns; done
```

```shell
sudo ip netns add server
sudo ip netns add client

sudo ip link add s-veth0 type veth peer name c-veth0

sudo ip link set s-veth0 netns server 
sudo ip link set c-veth0 netns client

sudo ip netns exec server ip link set s-veth0 up
sudo ip netns exec server ip link set c-veth0 up

sudo ip netns exec server ip address add 192.0.2.254/24 dev s-veth0
```

ここまではこれまでの実験準備と同じ流れ．ただし，clientのIPアドレスについては，DHCPを使って自動的に設定するためここでは設定していない．

次に，serverでdnsmasqコマンドを実行しておく．

```shell
sudo ip netns exec server dnsmasq \
--dhcp-range=192.0.2.100,192.0.2.200,255.255.255.0 \
--interface=s-veth0 \
--no-daemon
```

ここでは，DHCPクライアントに配布するIPアドレスとして，192.0.2.100 - 192.0.2.200の範囲内と指定している．
DHCPサーバーを立てたので，次はclientでDHCPクライアントに問い合わせてみる

別のターミナルを開き次のコマンドを実行

```shell
sudo ip netns exec client dhclient c-veth0
```

と思ったら `dhclient: command not found `という悲しいエラーが．

```shell
sudo apt-get -y install dhclient
Reading package lists... Done
Building dependency tree       
Reading state information... Done
E: Unable to locate package dhclient
```

このコマンドは`isc-dhcp-client`というパッケージに含まれるのでインストール

```shell
sudo apt-get -y install isc-dhcp-client
```

改めて

```shell
sudo ip netns exec client dhclient c-veth0
```

clientのインターフェースにIPアドレスとデフォルトルートが設定されているかを確認

```shell
sudo ip netns exec client ip address show | grep "inet"
    inet 192.0.2.147/24 brd 192.0.2.255 scope global c-veth0
```

```shell
sudo ip netns exec client ip route show
default via 192.0.2.254 dev c-veth0 
192.0.2.0/24 dev c-veth0 proto kernel scope link src 192.0.2.147 
```

無事，設定されていた．

- DCHPでの設定では，DNSのネームサーバーや時刻同期に使うNTPサーバーも設定できる！

# まとめ
- 代表的なアプリケーション層のプロトコルについて概観した
- HTTP, DNS, DHCPについて簡単に通信を行ってみた
- 特にDHCPはTCP/IPで通信を始めるための設定を支援するプロトコルであった