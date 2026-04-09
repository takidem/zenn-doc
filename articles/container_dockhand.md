---
title: "DockhandでDockerコンテナを可視化してみた"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["infra", "Docker"]
published: false
---
# DockhandでDockerコンテナを可視化してみた

Dcokerコンテナの可視化ツールとして、「Dockhand」というものが出てきたようなので少し触ってみました。

source code: 
https://github.com/takidem/monitor-cn

## yamlファイルの作成

Dockhand公式に従って作成します。
https://dockhand.pro/

```yml
version: '3.9'
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dockhand_data:/app/data
volumes:
  dockhand_data: {}
```

## Dockhandコンテナの作成&動作確認

upします

```bash
$ docker compose up -d
```

### 初期画面

アップデート情報等が一覧表示されるようです。
![](https://storage.googleapis.com/zenn-user-upload/7a8f644692b0-20260125.png)

### Dashboard

Environmentというのを作成していれば、ここにその一覧と関連情報が表示されるようです。

![](https://storage.googleapis.com/zenn-user-upload/c249fee8ba77-20260125.png)

### Container

コンテナの一覧を確認できます。
右上の＋ボタンを押下すれば、コンテナイメージを指定してコンテナを作成することも可能です。
docker run コマンドのオプションで指定可能なオプションもGUIで指定ができるため、とても見やすいです。

![](https://storage.googleapis.com/zenn-user-upload/31e1fe72a9fa-20260125.png)
![](https://storage.googleapis.com/zenn-user-upload/8ff4a54c3371-20260125.png)
![](https://storage.googleapis.com/zenn-user-upload/f93d95317735-20260125.png)

## ネットワークの状態

dockhandのコンテナをdocker composeで作成するとネットワークは以下のようになっています。専用のBridgeネットワーク(dockerhand_default)が作成され、dockhandコンテナ自身がそこに所属しているのが見て取れます。

![](https://storage.googleapis.com/zenn-user-upload/9b559f5bfddb-20260125.png)

- ↓ip addr showの結果一部抜粋
![](https://storage.googleapis.com/zenn-user-upload/e2703d5d1c2d-20260125.png)

## サービスコンテナを立ててみる

では、続いてこの状態でサービスコンテナを同じネットワーク内に構築した場合にどんな挙動になるかを見たいと思います。

### Prometheus/node-exporter/grafanaのサービスを定義する

なんとなく最近よく使っている３つを採用
docker-compose.ymlにprometheus/node-exporter/grafanaのコンテナの定義を追加してみるとどうなるかを見てみます。

```yml
version: '3.9'
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dockhand_data:/app/data

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    expose:
      - 9090

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - 5000:3000
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  dockhand_data: {}
  prometheus_data: {}
  grafana_data: {}
```
簡単ですがprometheus.ymlも作っておきます
```yml
global:
  scrape_interval: 1m

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 1m
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

ではここの内容でupします。
```bash
docker compose up -d
WARN[0000] /home/nikomi/work/dockerhand/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] up 5/5
 ✔ Network dockerhand_default Created                                                                                                                   0.1s
 ✔ Container prometheus       Created                                                                                                                   0.2s
 ✔ Container grafana          Created                                                                                                                   0.2s
 ✔ Container node-exporter    Created                                                                                                                   0.2s
 ✔ Container dockhand         Created       
 ```

 dockhandのWbeUIをリロードしてみます

 Dashboardには何も表示されず・・・

![](https://storage.googleapis.com/zenn-user-upload/79d87a1a8ed7-20260125.png)
 
 Environmentを作る必要があるのかな？

### Environmentを作成してみる

GUI上でEnvironmentを作成してみます。
通信方式はUNIX Socketでいい気がしますので、あとは適当な名づけとデフォルトで・・・
一番下のTest Connectionを押下すると右下に5つのコンテナへの通信が成功しているようなログが出力されました。

![](https://storage.googleapis.com/zenn-user-upload/7ed40f07053b-20260125.png)

Environmentが作成され、ステータスがConnectedになりました。

![](https://storage.googleapis.com/zenn-user-upload/881b43960b0d-20260125.png)

Containersに移動すると、docker composeで立ち上げたコンテナがすべて表示されているのを確認できました。
Environmentを作成しないとダメなんですね。

![](https://storage.googleapis.com/zenn-user-upload/ffbc33344f2f-20260125.png)

Logsを見ると、それぞれのコンテナのログを一覧でパッと見ることができるようになっています。これは便利！

![](https://storage.googleapis.com/zenn-user-upload/4de84c92a2bf-20260125.png)

また、Shellでは各コンテナに接続し、bashのコマンドを打てるようになっています。トラブルシューティングにも役に立ちそうです。

![](https://storage.googleapis.com/zenn-user-upload/ca3d55b3bb13-20260125.png)

Stackの画面では、そのCompose Stackの各種情報やざっくりなメトリクスを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/3e8069339260-20260125.png)

dockhandというDockerコンテナの可視化ツールが新しく出てきたということでデモ的な感じでいろんな画面とかサービスコンテナを作成した時の動き等を見てきました。
今回見た機能はほんの一部に過ぎないので、Git連携やCI/CD、認証認可等他にも触っていきたいなと思います。
