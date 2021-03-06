# Transport layer protocol  
- IPによってインターネットの世界で差出人と宛先位の間でエンドツーエンドのデータのやりとりができる
- より実用的に扱うにはまだ課題が
  - コンピュータが扱う通信の内容にはいろいろな種類がある
    - web, メール, その他アプリケーション
    - 端末では様々なアプリケーションが通信を行っている
  - パケットによってどの目的で行われている通信なのかを正しく判別する必要がある
    - これはIP Headerの情報だけではわからない


- 場合によってはパケットが経路の途中で破棄されている場合もある．
- このような問題はIPのさらに上位のレイヤーのプロトコルで解決される
  - IPのペイロードとなるプロトコル
  - OSIモデルであれば，トランスポート層に相当
    - 代表的なプロトコル
      - TCP: Transmission Control Protocol
      - UDP: User Datagram Protocol


# UDP: User Datagram Protocol
- TCPと比較するとシンプル
- 通信しているアプリケーションを区別する

- UDPとIPを組み合わせるときのフォーマットの概略
  - Ethernet Header
  - IP Header
  - UDP Header + Data

- UDPはTransport layerのプロトコル，つまりIPの上位に位置する
- パケット内では，IPヘッダの直後にペイロードとして連結される
- UDPヘッダの中にアプリケーションの情報が含まれる

## 具体的な解決  
- ヘッダにはポートという16ビットの正の整数で表現されるフィールドがある
- これはポート番号と呼ばれる
- アプリはポート番号を指定して通信を初め，端末に送られてきたパケットはポート番号を参照して端末内の所定のポートに情報を届ける
- あるポートが同時に複数のアプリケーションの通信に使われることは基本的には無いようになっている

UDPヘッダのフォーマット
- Source Port | Destination Port
- Length | Checksum
- Data

- Source Port：送信元のポート
- Destination Port：送信先のポート
- Data：アプリケーションでやりとりされるデータ（UDPではこの単位をデータグラムと呼ぶ）

ポート番号の決定

- 相手から通信を待ち受ける立場のアプリケーション
  - 例えばサーバー
  - サーバーとなるアプリケーションが通信しようとしているプロトコルによってポート番号が決定される
  - **Transport layerで通信を待っている**時は，さらに上位のプロトコルによってポート番号があらかじめ決まっている．
    - 例：DNS(Domain Name Service)というUDPを使う上位層のプロトコルでは，ポート番号に53を使うことが決まっている
- 自分から通信を始める立場のアプリケーション
  - 例えばクライアント
  - アプリケーションから指定がない場合はOSが自動的にポート番号を割り当てる
    - その基準は，特定のポート番号の範囲かつ他のアプリケーションが使ってないもの，となる．
    - このようなポート番号を Ephemeral Port と呼ぶ

UDPヘッダを組み立てる際，クライアント側で送信先のポートに指定する番号を決めなければいけない．
- 送信元のポートを下に考えることができる
- クライアント側で，通信に対応したサーバーが使っているであろうポート番号をあらかじめ入れておく．
- サーバー側は受け取った時点でクライアントのポート番号はUDPヘッダに含まれているので問題はない

# 実験1

`nc` というコマンドを利用する．今回はNetwork Namespaceは使わない．
- 通信を待ち受けるポートには54321番を使う

```shell
nc -ulnv 127.0.0.1 54321
```

この状態は，ncコマンドというアプリケーションがUDPの54321番ポートを占有している状態である．
ncコマンドのオプション
- `-u`: UDPでの通信を指定
- `-l`: サーバーとして動作させる
- `-n`: IPアドレスをDNSで名前解決させない
- `-v`: コマンドの表示を詳細にするため指定するオプション

続いてクライアントを用意してサーバーに接続する．別ターミナルを開いて，ncコマンドでクライアントを立てる.

```shell
nc -u 127.0.0.1 54321
```

何も表示はされないがサーバーとクライアントの接続はできた．この時点では接続しただけで通信は生じていない．

さらに別のターミナルを開いて次のコマンドを入力

```shell
sudo tcpdump -i lo -tnlA "udp and port 54321"
```

これで，ループバックアドレスを使ったUDPの54321番ポートが関係した通信をキャプチャできる．

最後に，ncコマンドのクライアントを実行しているターミナルで適当な文字列をキーボードで入力してEnterを押す．  

```shell
nc -u 127.0.0.1 54321
Heelo, World!
```

以上がUDPによってアプリケーションを区別する仕組みの確認．ただし，これではパケットのロスの問題は解決できていない．このように，送りっぱなしのプロトコルをコネクションレス型と呼ぶ．
パケットの破棄に関しては，UDPは一切関与しない，UDPを使って信頼性のある通信を行いたい場合はさらに上位層のプロトコルを使って対処をする必要がある．

# TCP  
- TCPはUDPと同じようにアプリケーションをポートで区別できる
- 加えて，信頼性のある通信を実現する
  - パケット破棄の問題を解決する
  - パケットがロスなく到達したかどうかを確認しながら通信を進める．

## TCPヘッダフォーマット

- Source Port | Destination
- Sequence Number
- Acknowledgement Number
- Data Offset | Reserved | Flags | Window
- Checksum | Urgent Pointer
- Data...

次の２点がポイント
1. UDPと同じように送信元と送信先のポートが含まれるフィールドがある
2. UDPよりもヘッダのフィールドが多い

## 実際の通信  
- TCPでの通信を始めるときに３つのセグメントをやり取りする
  - three hand handshake
- コントロールビット
  - TCPのヘッダに含まれるフィールドで6ビット分のフラグから構成されている（Flags）
- シーケンス番号
  - TCPでのやりとりにおいて，データの順番を管理するフィールド
- 送信先がデータを受け取ったことをACKの立ったセグメントを送り返すことでデータの授受の信頼性を担保している
  - もしACKのフラグがたったセグメントが帰ってこない場合は，何度か送り直すようになっている

# まとめ
- Transport層のプロトコル，UDPとTCPについて概観した
- ポート番号によってアプリケーションによる通信は整理されている
- UDPは送りっぱなし
- TCPは授受の確認の通信も発生する