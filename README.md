# An-introduction-to-TCP-IP-and-Hands-on-with-LINUX
LINUXで動かしながら学ぶTCP/IP入門の備忘録

# 環境構築  
[参考ページ](https://qiita.com/yasuoka_dev/items/073f7e8c7dba75993323)  
dockerのインストール
```shell
$ brew install docker
$ brew cask install doker
```

ubuntu 18.04のイメージを`pull`する
```shell
$ docker pull ubuntu:18.04
```

ubuntu:18.04コンテナの起動
```shell
$ docker run -it -d --name ubuntu ubuntu:18.04
```

起動したコンテナに入る
```shell
$ docker exec -it ubuntu /bin/bash
```

コンテナから抜ける
```shell
$ exit
```

コンテナの停止
```shell
$ docker stop ubuntu
```

# ubuntuの初期設定

```shell
$ apt-get update
$ adduser <user_name>
$ password <password>
```

いろいろ聞かれるが空白で進める．  
`sudo`が無いと怒られたのでインストール

[参考にしたページ](https://shingo-sasaki-0529.hatenablog.com/entry/sudo_not_found)  
```shell
$ sudo
bash: sudo: command not found
```

となった．

```shell
$ cd /
$ find . -name sudo
$ 
```

という結果で，そもそも`sudo`が無いようなのでインストールした．

```shell
$ apt-get install -y sudo
```

終わってから一応確認

```shell
$ which sudo
$ /usr/bin/sudo
```

と表示されたのでOK.