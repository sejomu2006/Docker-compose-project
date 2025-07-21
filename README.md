# Docker-compose-project
Docker Compose で仮想ネットワークを構築し、ファイアウォールの導入と通信制御の挙動を検証するセキュリティ学習用プロジェクトです。

--

※※注意事項※※  
- この環境はインターネットとは接続されていません。
- セキュリティ対策は実施しておりません  
- 実際の環境でこの構成をそのまま使用することは、重大なセキュリティリスクを伴います。  
- この環境は学習用です。※本プロジェクトでは現在、学習目的のため `docker Compose` による構築を行っています。  
  



### 構成の目的
- ファイアウォールの重要性を認識するためのに、Docker Composeで構築を行いました。
　

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
      MYSQL_ROOT_PASSWORD: 
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

networks:
  dmz-net:
    driver: bridge

  internal-net:
    driver: bridge

```

## 構成について
この構成は、外部に公開してはいけない情報をやり取りするネットワークと、一部を外部に公開するネットワークを分けた構成です

### コンテナの区分
| コンテナ名         | 用途                     | 所属ネットワーク | 備考                              |
|--------------------|--------------------------|------------------|-----------------------------------|
|api-server|クライアントからのリクエストを受け取る|DMZ用ネットワーク(dnz-net)||
|api-server-internal|クライアントからのリクエストを受け取る|内部ネットワーク(internal-net)||
| my-docker-network-internal-server-1 | 内部サーバ|内部ネットワーク(internal-net)| ubuntuベース         |
| my-docker-network-internal-db-1 | 内部データベース |内部ネットワーク(internal-net)|mysql rootパスワード設定済み    |
| my-docker-network-internal-client1-1|クライアント1|内部ネットワーク(internal-net)|alpine使用|
|my-docker-network-internal-client2-1|クライアント2|内部ネットワーク(internal-net) | alpine使用  |
|  my-docker-network-dmz-client-1| DMZ内での通信確認用、クライアントとしても機能|DMZ用（dmz-net)|alpine使用 |
|my-docker-network-web-servs-1|webサーバとして使用|DMZ用ネットワーク(dmz-net)||




## ネットワーク一覧
```bash
$ docker network ls
NETWORK ID     NAME                             DRIVER    SCOPE
e1198e6d04aa   my-docker-network_dmz-net        bridge    local
fba4cc2edb93   my-docker-network_internal-net   bridge    local

```
## 各ネットワークについて

| ネットワーク名       | 用途・役割                                         |
|----------------------|----------------------------------------------------|
|my-docker-network_dmz-net  |DMZとして使用                | 
| my-docker-network_internal-net|内部ネットワーク| 

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




