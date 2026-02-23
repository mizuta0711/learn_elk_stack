# ELKスタック 予知保全PoC

工場設備（ポンプ・モーター）のセンサーデータを ELK スタックに取り込み、**異常検知 → 可視化 → アラート通知**の一連の流れを体験する学習・検証環境。

## 構成図

```
センサーデータ(CSV)
        │
        ▼
   ┌──────────┐     ┌───────────────┐     ┌──────────┐
   │ Logstash │────▶│ Elasticsearch │◀────│ Python   │
   │ (取込み) │     │ (蓄積・検索)  │     │ (ML推論) │
   └──────────┘     └───────┬───────┘     └──────────┘
                            │
                    ┌───────┴───────┐
                    ▼               ▼
              ┌──────────┐   ┌─────────────┐
              │  Kibana  │   │ ElastAlert2 │
              │ (可視化) │   │  (アラート) │
              └──────────┘   └─────────────┘
```

## 技術スタック

| コンポーネント | バージョン | 役割 |
|---|---|---|
| Elasticsearch | 8.17 | データ蓄積・検索・ML 異常検知 |
| Kibana | 8.17 | 可視化・ダッシュボード・ML 管理 |
| Logstash | 8.17 | CSV データの取り込み・変換 |
| ElastAlert2 | latest | 閾値ベースのアラート通知 |
| Python 3.11 | - | scikit-learn + LightGBM による ML 学習・推論 |

## 前提条件

- Docker Desktop がインストール済み
- Docker にメモリ 8GB 以上を割り当て済み（推奨 16GB）
- ディスク空き容量 20GB 以上

> Docker Desktop がない場合は、[GitHub Codespaces でブラウザだけで学習](docs/03_GitHub_Codespacesでの学習.md)することもできる。

## クイックスタート

```bash
# 1. リポジトリをクローン
git clone https://github.com/mizuta0711/learn_elk_stack.git
cd learn_elk_stack

# 2. 環境変数ファイルを準備
copy .env.sample .env

# 3. コンテナを起動（初回はイメージダウンロードに数分かかる）
docker compose up -d

# 4. Elasticsearch の稼働確認
curl -u elastic:changeme http://localhost:9200/_cluster/health?pretty

# 5. kibana_system ユーザーのパスワードを設定（初回のみ）
docker exec -it elasticsearch bin/elasticsearch-reset-password -u kibana_system -i
# プロンプトが表示されたら "changeme" を入力

# 6. Kibana を再起動
docker compose restart kibana
```

起動後、ブラウザで http://localhost:5601 を開く（ユーザー: `elastic` / パスワード: `changeme`）。

## サンプルデータ

3台の設備 x 3ヶ月分（約38,880行）のセンサーデータを同梱。Logstash が起動時に自動で Elasticsearch に投入する。

| 設備ID | 設備名 | 種別 |
|---|---|---|
| pump-01 | ポンプA-01 | ポンプ |
| pump-02 | ポンプA-02 | ポンプ |
| motor-01 | モーターB-01 | モーター |

データには **故障の7日前**から徐々にセンサー値が悪化するパターンが組み込まれており、異常検知の学習に最適。

## ディレクトリ構成

```
├── docker-compose.yml          # Docker Compose 定義
├── .env.sample                 # 環境変数のサンプル（.env にコピーして使用）
├── elasticsearch/config/       # Elasticsearch 設定
├── kibana/config/              # Kibana 設定（日本語化含む）
├── logstash/
│   ├── config/                 # Logstash 設定
│   └── pipeline/               # CSV 取り込みパイプライン
├── elastalert/
│   ├── config.yaml             # ElastAlert2 基本設定
│   └── rules/                  # アラートルール
├── python/
│   ├── Dockerfile              # Python ML コンテナ
│   ├── generate_sample_data.py # サンプルデータ生成
│   ├── train_model.py          # ML 学習
│   └── batch_inference.py      # バッチ推論
├── .devcontainer/              # GitHub Codespaces 設定
├── data/                       # センサーデータ（CSV）
└── docs/                       # ドキュメント
```

## 学習手順

詳細な手順は [docs/01_学習手順.md](docs/01_学習手順.md) を参照。

| Step | 内容 | 概要 |
|---|---|---|
| 1 | 環境起動 | Docker Compose でコンテナを起動し、Kibana にログイン |
| 2 | データ投入確認 | Logstash による CSV 取り込みを確認、Discover でデータ閲覧 |
| 3 | 異常検知（GUI） | Kibana ML で Anomaly Detection ジョブを作成・実行 |
| 4 | 異常検知（Python） | LightGBM で故障予兆スコアを算出（オプション） |
| 5 | ダッシュボード作成 | センサー推移・異常スコア・アラート一覧を可視化 |
| 6 | アラート通知 | ElastAlert2 による異常検知アラートを確認 |

## よく使うコマンド

```bash
# 起動 / 停止
docker compose up -d
docker compose down

# ログ確認
docker compose logs -f elasticsearch

# データ件数確認
curl -u elastic:changeme "http://localhost:9200/sensor-data-*/_count?pretty"

# ML 学習・推論（Python）
docker exec python-ml python train_model.py
docker exec python-ml python batch_inference.py

# 完全クリーンアップ（データも削除）
docker compose down -v
```

## ドキュメント

- [概要と目的](docs/00_概要と目的.md) — プロジェクトの目的・ゴールイメージ・Azure 構成との対応
- [学習手順](docs/01_学習手順.md) — ステップバイステップの操作手順
- [無料オンライン環境の調査](docs/02_無料オンライン環境の調査.md) — GitHub Codespaces 等の調査結果
- [GitHub Codespaces での学習](docs/03_GitHub_Codespacesでの学習.md) — ブラウザだけで学習する方法

## 注意事項

- `.env` にパスワードが記載されているため、`.gitignore` で除外済み。`.env.sample` をコピーして使用すること
- Elasticsearch ML（X-Pack）は **Trial ライセンス（30日間）**。期限切れ後は `docker compose down -v` で再セットアップが必要
- Docker に最低 **8GB のメモリ**を割り当てること（推奨 16GB）
