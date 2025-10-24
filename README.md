# Docker-compose-project
Docker Compose を用いて仮想ネットワークを構築し、
ファイアウォールの導入による通信制御の挙動を検証する。
さらに、監視ツール（ELKスタック＋Suricata）を実装し、
ネットワーク上の通信を可視化・分析するセキュリティ学習用プロジェクトです。
---

※※注意事項※※  
- この環境はインターネットとは接続されていません。
- 今の時点ではセキュリティ対策は実施しておりません  
- 実際の環境でこの構成をそのまま使用することは、重大なセキュリティリスクを伴います。  
- この環境は学習用です。※本プロジェクトでは現在、学習目的のため `docker Compose` による構築を行っています。  
  



### 構成の目的

ファイアウォールの重要性を理解するために、Docker Composeでネットワーク環境を構築し、さらにELKスタックやSuricataといった監視ツールを設置して、通信の可視化・分析ができるセキュアな学習環境を作ることを目的としました。
　
---
## Docker コンテナ一覧

```bash
version: '3.9'

services:
  api-server:
    build:
      context: ./api-server
      dockerfile: Dockerfile.dmz
    networks:
      - dmz-net
    ports:
      - "4000:4000"
    restart: unless-stopped

  web-servs:
    image: nginx:latest
    networks:
      - dmz-net

  dmz-client:
    image: alpine
    command: tail -f /dev/null
    networks:
      - dmz-net
  fake-api:
    build:
      context:  ./fake-api
      dockerfile: Dockerfile.dmz
    networks:
      - dmz-net
    ports:
      - "7000:7000"
    restart: unless-stopped

  fake-db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 　#任意のパスワード
      MYSQL_DATABASE: customer_data
    networks:
      - dmz-net
    ports:
      - "3307:3306"
    volumes:
      - ./fake-db/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

  fake-admin:
    image: httpd:2.4
    volumes:
      - ./fake-admin:/usr/local/apache2/htdocs/
    networks:
      - dmz-net
    ports:
      - "8081:80"
    restart: unless-stopped

  api-server-internal:
    build:
      context: ./api-server
      dockerfile: Dockerfile.internal
    networks:
      - internal-net
    ports:
      - "5000:5000"
    restart: unless-stopped
  internal-server:
    image: ubuntu
    command: tail -f /dev/null
    networks:
      - internal-net

  internal-db:
    image: mysql:8.0.33
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: seoiatum
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - internal-net

  internal-client1:
    image: alpine
    command: tail -f /dev/null
    networks:
      - internal-net

  internal-client2:
    image: alpine
    command: tail -f /dev/null
    networks:
      - internal-net

  suricata:
    image: jasonish/suricata:latest
    command: -i eth0
    networks:
      - elk-net
      - dmz-net
    volumes:
      - ./suricata/logs:/var/log/suricata
      - ./suricata/etc:/etc/suricata
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    networks:
      - elk-net
    ports:
      - "9200:9200"
    restart: unless-stopped

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    volumes:
      - /home/seisom/my-docker-network/logstash/pipeline:/usr/share/logstash/pipeline
      - /home/seisom/my-docker-network/suricata/logs:/var/log/suricata
    networks:
      - elk-net
    depends_on:
      - elasticsearch
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk-net
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    restart: unless-stopped

networks:
  dmz-net:
    driver: bridge

  internal-net:
    driver: bridge

  elk-net:
    driver: bridge
```

## 構成について
この構成は、外部に公開してはいけない情報をやり取りするネットワークと、一部を外部に公開するネットワークを分けた構成です

### コンテナの区分
| コンテナ名 | 用途 | 所属ネットワーク | 備考 |
|-------------|------|------------------|------|
| **api-server** | クライアントからのリクエストを受け取る DMZ 側 API サーバ | DMZ ネットワーク（`dmz-net`） | `Dockerfile.dmz` からビルド、ポート **4000** 番で公開 |
| **web-servs** | Web サーバ（Nginx） | DMZ ネットワーク（`dmz-net`） | 静的コンテンツ配信に使用 |
| **dmz-client** | DMZ 内での通信確認・検証用クライアント | DMZ ネットワーク（`dmz-net`） | `alpine` ベース |
| **fake-api** | 疑似 API サーバ（テスト用） | DMZ ネットワーク（`dmz-net`） | ポート **7000** 番で公開、`Dockerfile.dmz` からビルド |
| **fake-db** | 疑似データベース | DMZ ネットワーク（`dmz-net`） | MySQL 5.7、`init.sql` により初期化 |
| **fake-admin** | 疑似管理用 Web サーバ | DMZ ネットワーク（`dmz-net`） | Apache 2.4、ポート **8081** 番で公開 |
| **api-server-internal** | 内部ネットワーク側の API サーバ | 内部ネットワーク（`internal-net`） | `Dockerfile.internal` からビルド、ポート **5000** 番で公開 |
| **internal-server** | 内部サーバ（通信・制御テスト用） | 内部ネットワーク（`internal-net`） | `ubuntu` ベース |
| **internal-db** | 内部データベース | 内部ネットワーク（`internal-net`） | MySQL 8.0.33、root パスワード `seoiatum`、`init.sql` により初期化 |
| **internal-client1** | 内部クライアント1 | 内部ネットワーク（`internal-net`） | `alpine` ベース |
| **internal-client2** | 内部クライアント2 | 内部ネットワーク（`internal-net`） | `alpine` ベース |
| **suricata** | IDS（侵入検知システム） | ELK ネットワーク（`elk-net`）、DMZ ネットワーク（`dmz-net`） | `/var/log/suricata` にログ出力、`NET_ADMIN` 権限付与 |
| **elasticsearch** | ログ蓄積サーバ | ELK ネットワーク（`elk-net`） | ポート **9200** 番公開、単一ノード構成 |
| **logstash** | ログ収集・変換 | ELK ネットワーク（`elk-net`） | Suricata のログを解析し Elasticsearch へ転送 |
| **kibana** | ログ可視化ツール | ELK ネットワーク（`elk-net`） | ポート **5601** 番公開、Elasticsearch に接続 |





## ネットワーク一覧
```bash
$ docker network ls
NETWORK ID     NAME                             DRIVER    SCOPE
b93162f2fe3a   my-docker-network_dmz-net        bridge    local
4c76e2cb632f   my-docker-network_elk-net        bridge    local
8771d012dc53   my-docker-network_internal-net   bridge    local

```
## 各ネットワークについて

| ネットワーク名       | 用途・役割                                         |
|----------------------|----------------------------------------------------|
|my-docker-network_dmz-net  |DMZとして使用                | 
| my-docker-network_internal-net|内部ネットワーク| 
| my-docker-network_elk-net|監視ツール(ELKスタック+suricata)を実装し、ネットワーク通信を可視化・分析するために使用|

---

## 構築をする際の環境設定
- ubuntuをインストール



## 構築の流れ

Dockerファイルを作成。ファイル名は自身で決めるられる。

```bash
$ mkdir my-docker-network

```
ファイルが入ってるディレクトリに入る

```bash
$ cd my-docker-network

```
ファイルを開く。

```bash
$ nano dock-net.yml

```


ファイルの中に以下ようにしてコンテナを記述する。また、以下の内容は、説明のために省略した部分がある。全文は、「Docker　コンテナ一覧」にある。
```yaml
version: '3.9'  

services:
  web-servs:
    image: nginx:latest
    networks:
      - dmz-net
networks:
  dmz-net:
    driver: bridge
```
-インデントがずれると正常に機能しないため、インデントは正しい位置にする。
- version:　バージョンによって書式が変わるため、指定は絶対に行う。
- service: コンテナの作成を行う。(名前は、任意の名前でよい)
- serviceの中に入っているimage:は、コンテナの名前を設定し、networks:は、どのネットワークに置くかを決める。
- 一番上のservice:と同じインデントに書かれているnetworks:は、ネットワークの作成を行う。
- networks:のインデントから二段下げたところにネットワーク名を設定する。(任意の名前)
- driver:どこにネットワークを置くかを決定。
## 通信の確認

- ネットワークが正常に構成されているかどうかを確認するため、DMZ用ネットワークおよび内部ネットワークにおける通信テストを実施した。
- 準備
APIサーバを動かすためのDockerfileを作成する

```yaml
api-server:
    build:
      context: ./api-server
      dockerfile: Dockerfile.dmz
    networks:
      - dmz-net
    ports:
      - "4000:4000"
    restart: unless-stopped

```

その作成したDockerファイルの中に以下を記述する。
```Dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y iputils-ping
WORKDIR /app
CMD ["python3", "-m", "http.server", "4000"]


```
上記のDockerファイルの作成、ファイルの中に情報を入れるという動作は、それぞれのAPIサーバで行う。

---
-DMZ用ネットワークの通信の確認と結果 

pingを送った
pingコマンドの後には、一個空欄を開けて、通信確認をしたいコンテナのIPを入力。

```bash
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-docker-network-api-server-1
172.24.0.2
$ ping 172.24.0.2
PING 172.24.0.2 (172.24.0.2) 56(84) bytes of data.
64 bytes from 172.24.0.2: icmp_seq=1 ttl=64 time=0.573 ms
64 bytes from 172.24.0.2: icmp_seq=2 ttl=64 time=0.074 ms
64 bytes from 172.24.0.2: icmp_seq=3 ttl=64 time=0.082 ms
64 bytes from 172.24.0.2: icmp_seq=4 ttl=64 time=0.079 ms
64 bytes from 172.24.0.2: icmp_seq=5 ttl=64 time=0.076 ms
64 bytes from 172.24.0.2: icmp_seq=6 ttl=64 time=0.071 ms
64 bytes from 172.24.0.2: icmp_seq=7 ttl=64 time=0.084 ms
64 bytes from 172.24.0.2: icmp_seq=8 ttl=64 time=0.086 ms
64 bytes from 172.24.0.2: icmp_seq=9 ttl=64 time=0.081 ms
64 bytes from 172.24.0.2: icmp_seq=10 ttl=64 time=0.101 ms
^C
--- 172.24.0.2 ping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 9326ms
rtt min/avg/max/mdev = 0.071/0.130/0.573/0.147 ms
seisom@LAPTOP-P5CT19JG:~/my-docker-network$
```

```bash


```

以下は、内部ネットワークでの通信確認である

```bash
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-docker-network-api-server-internal-1
172.23.0.5
$ ping 172.23.0.5
PING 172.23.0.5 (172.23.0.5) 56(84) bytes of data.
64 bytes from 172.23.0.5: icmp_seq=1 ttl=64 time=0.682 ms
64 bytes from 172.23.0.5: icmp_seq=2 ttl=64 time=0.082 ms
64 bytes from 172.23.0.5: icmp_seq=3 ttl=64 time=0.107 ms
64 bytes from 172.23.0.5: icmp_seq=4 ttl=64 time=0.075 ms
64 bytes from 172.23.0.5: icmp_seq=5 ttl=64 time=0.072 ms
64 bytes from 172.23.0.5: icmp_seq=6 ttl=64 time=0.069 ms
64 bytes from 172.23.0.5: icmp_seq=7 ttl=64 time=0.066 ms
64 bytes from 172.23.0.5: icmp_seq=8 ttl=64 time=0.080 ms
64 bytes from 172.23.0.5: icmp_seq=9 ttl=64 time=0.074 ms
64 bytes from 172.23.0.5: icmp_seq=10 ttl=64 time=0.144 ms
64 bytes from 172.23.0.5: icmp_seq=11 ttl=64 time=0.115 ms
^C
--- 172.23.0.5 ping statistics ---
11 packets transmitted, 11 received, 0% packet loss, time 10370ms
rtt min/avg/max/mdev = 0.066/0.142/0.682/0.172 ms
seisom@LAPTOP-P5CT19JG:~/my-docker-network$


```

DMZ、内部ネットワークの両方で通信が確認できた。

## nmapでポートスキャンを行う
---

以下は、DMZと内部ネットワークの中のAPIサーバ、内部ネットワーク(データベース、サーバ)、DMZ(webサーバ)のコンテナについてのスキャンを行った結果である。

```bash
$ docker run -it --rm --network my-docker-network_internal-net instrumentisto/nmap nmap internal-server
Unable to find image 'instrumentisto/nmap:latest' locally
latest: Pulling from instrumentisto/nmap
9824c27679d3: Pull complete
5410630e007a: Pull complete
e5da4a54c84a: Pull complete
Digest: sha256:5d8a634e7d6cb95f26a1a18b33df7e07a2d6db9278854e0a2bf4a8396c1a9d07
Status: Downloaded newer image for instrumentisto/nmap:latest
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-21 12:27 UTC
Failed to resolve "nmap".
Nmap scan report for internal-server (172.23.0.5)
Host is up (0.000013s latency).
rDNS record for 172.23.0.5: my-docker-network-internal-server-1.my-docker-network_internal-net
All 1000 scanned ports on internal-server (172.23.0.5) are in ignored states.
Not shown: 1000 closed tcp ports (reset)
MAC Address: 02:51:FD:20:5E:5D (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.47 seconds

$ docker run -it --rm --network my-docker-network_internal-net instrumentisto/nmap nmap internal-db
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-21 12:28 UTC
Failed to resolve "nmap".
Nmap scan report for internal-db (172.23.0.3)
Host is up (0.000012s latency).
rDNS record for 172.23.0.3: my-docker-network-internal-db-1.my-docker-network_internal-net
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE
3306/tcp open  mysql
MAC Address: 56:A9:4D:EE:56:1D (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.40 seconds

$ docker run -it --rm --network my-docker-network_internal-net instrumentisto/nmap nmap api-server-internal
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-21 12:29 UTC
Failed to resolve "nmap".
Nmap scan report for api-server-internal (172.23.0.2)
Host is up (0.000013s latency).
rDNS record for 172.23.0.2: my-docker-network-api-server-internal-1.my-docker-network_internal-net
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE
5000/tcp open  upnp
MAC Address: 42:59:6C:92:54:CE (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.43 seconds
$ docker run -it --rm --network my-docker-network_dmz-net instrumentisto/nmap nmap api-server
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-21 12:55 UTC
Failed to resolve "nmap".
Nmap scan report for api-server (172.24.0.2)
Host is up (0.000013s latency).
rDNS record for 172.24.0.2: my-docker-network-api-server-1.my-docker-network_dmz-net
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE
4000/tcp open  remoteanything
MAC Address: 32:B7:CC:08:91:04 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.43 seconds

$ docker run -it --rm --network my-docker-network_dmz-net instrumentisto/nmap nmap  web-servs
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-21 12:56 UTC
Failed to resolve "nmap".
Nmap scan report for web-servs (172.24.0.4)
Host is up (0.000012s latency).
rDNS record for 172.24.0.4: my-docker-network-web-servs-1.my-docker-network_dmz-net
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 8A:90:3A:33:66:08 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.46 seconds
```

| コンテナ名          | ポート番号 | 状態  | サービス名  |
|---------------------|------------|-------|-------------|
| internal-server      | closed        | 全閉  | -           |
| internal-db          | 3306       | open  | mysql       |
| api-server-internal  | 5000       | open  | upnp        |
| api-server           | 4000       | open  | remoteanything |
| web-servs            | 80         | open  | http        |

- remoteanythingについて

  これは、nmapのデフォルトのポートであり、サービスの名前ではないです。


## ファイアウォールの設置(ネットワーク間)
---

- 目的
  
Docker Compose には、異なるネットワーク間での通信を細かく制御する機能はありません。そこで、明示的に通信を遮断するためにファイアウォール（iptables）を利用しました。
## 設置方法
- ネットワーク名の確認

```bash
$ docker network ls
NETWORK ID     NAME                             DRIVER    SCOPE
e1198e6d04aa   my-docker-network_dmz-net        bridge    local
fba4cc2edb93   my-docker-network_internal-net   bridge    local
```

NAMEに書かれているものがネットワーク名です。このネットワーク名を用いて通信の制御を行います。

- ファイアウォールの設置でつまずいていた部分

以下の記録に、docker network inspect my-docker-network_dmz-net | grep "com.docker.network.bridge.name"　というコマンドがあるが、これはネットワーク名からブリッジ名を表示させようとして失敗している。そこで、ネットワーク名を明記せずにブリッジを表示するコマンド( ip link show | grep br-)で対処した。

```bash
$ docker network inspect my-docker-network_dmz-net | grep "com.docker.network.bridge.name"
$ docker network inspect my-docker-network_internal-net | grep "com.docker.network.bridge.name"
$ ip link show | grep br-
3: br-e1198e6d04aa: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
4: br-f35992645f42: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
5: br-fba4cc2edb93: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
6: br-09f9b00bb76a: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
7: br-15f6afe0212b: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
9: br-a908670cf30b: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
10: br-e0e6aa3e4816: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
11: veth91260ee@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-fba4cc2edb93 state UP mode DEFAULT group default
12: veth979d698@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-e1198e6d04aa state UP mode DEFAULT group default


```

- 設置のためのコマンド

  ```bash
  $ sudo iptables -I FORWARD -i br-e1198e6d04aa -o br-fba4cc2edb93 -j DROP
  [sudo] password for seisom:
 
  ```

このコマンドは、-i以降にDMZのネットワークの名前を書き、-o以降には内部ネットワークの名前を書いた。これを実行すると、DMZ
から内部ネットワークへの通信が遮断される。また、内部ネットワークからDMZへの通信は可能である。

- FORWARDについて
  
  このコマンドの中に入ってるFORWARDというものは、Linuxのiptables(パケットフィルタリング機能)におけるチェインの一つです。

- ファイアウォールを永続的にさせるために

iptablesで設定したルールは、OSを再起動すると消えてしまいます。  
そのため、以下の手順で永続化を行う必要があります。

以下のコマンドの途中に設定の確認でyesかnoを選択する場面がありますが、すべてyesで大丈夫です。
 ```bash
 $ sudo apt install iptables-persistent
 ```

以下のコマンドは、現在のiptables(IPv4)のルール設定をファイルに保存するコマンドです。

```bash
$ sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

## 偽のサービスの設置とELKスタック+suricataの実装
---
- 目的
  
  外からの攻撃を想定し、ハニーボットのような役割を持つ偽のサービスの設置と、そこを通る通信の監視を行うツールの実装を行う。

- 偽のサービスの設置について
  ---
  偽のサービスを設置する際は、本物のサービスを設置する際と動作はほとんど同じです。
  だが、侵入されやすいサービス、ポートがあるため、それに基づいた設定をする必要があります。

### ELSスタック+suricataの実装
  ---
  
### ツール一覧

| ツール名          | 概要 |           
|---------------------|------------|
| Suricata      | 通信を監視し、アラートやフロー情報を eve.json に出力    | 
|Logstash       |  Suricataのログを収集・整形し、Elasticsearchへ送信 | 
| Elasticsearch | 受信したログデータを保存・インデックス化 | 
|Kibana|Elasticsearch上のデータを可視化|

### 手順
- suricata実装
　通常は、サーバに直接インストールするものです。しかし、仮想環境なので今回はDockerコンテナ版を使用します。
  docker-compose.ymlファイルの中に以下を書き足す
  ```bash
  suricata:
    image: jasonish/suricata:latest
    command: -i eth0
    networks:
      - elk-net
      - dmz-net
    volumes:
      - ./suricata/logs:/var/log/suricata
      - ./suricata/etc:/etc/suricata
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped
   ```
その後、以下を実行する。また、
  ```bash
  mkdir -p ./suricata/etc
   ```
-p オプションによって、親ディレクトリが存在していなくてもまとめて作成できる。
以下のコマンドで、作成したディレクトリに入る。
  ```bash
  mkdir ./suricata/etc
   ```
入ったディレクトリで以下を実行する。以下のコマンドでは、suricataルールを取得可能である。
  ```bash
  sudo wget https://rules.emergingthreats.net/open/suricata-6.0.0/emerging.rules.tar.gz

   ```
このルールをマウントすると、外部からの攻撃や不正通信を検知可能となる。

- suricataのルールが読めない場合
  suricataのルールが読めない原因として、emerging.rules.tar.gzが解凍されていない場合がある。その時は以下を実行する。
  そのうえで、ルールを読み込む。
  ```bash
  tar zxvf emerging.rules.tar.gz
   ```
  
### ELKスタックの実装 

 以下をdockerファイルの中に書き足す。
 ```bash
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    networks:
      - elk-net
    ports:
      - "9200:9200"
    restart: unless-stopped

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    volumes:
      - /home/seisom/my-docker-network/logstash/pipeline:/usr/share/logstash/pipeline
      - /home/seisom/my-docker-network/suricata/logs:/var/log/suricata
    networks:
      - elk-net
    depends_on:
      - elasticsearch
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk-net
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    restart: unless-stopped

   ```
 その後、以下のコマンドでLogstashのpipeline設定用のディレクトリを作成
 ```bash
 $mkdir -p logstash/pipeline
   ```
logstash/pipeline/suricata.confに以下を書き入れて保存
 ```bash
 input {
  file {
    path => "/var/log/suricata/eve.json"
    codec => json
  }
}

filter {
  date {
    match => [ "timestamp", "ISO8601" ]
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "suricata-%{+YYYY.MM.dd}"
  }
}
   ```
elkディレクトリを作成する。
 ```bash
 $mkdir elk
```
elkディレクトリに入る。

 ```bash
 $cd elk
   ```
以下のコマンドで、先ほど設定したELKスタックが立ち上がる。
 ```bash
 $docker compose up -d
   ```
ここまでの操作で、監視ツールの基本的な部分は設定できる。
