# セットアップガイド

コトコトバの開発環境構築からデプロイまでの詳細な手順です。

## 目次

- [前提条件](#前提条件)
- [リポジトリのクローン](#リポジトリのクローン)
- [バックエンドのセットアップ](#バックエンドのセットアップ)
- [フロントエンドのセットアップ](#フロントエンドのセットアップ)
- [Docker によるバックエンド起動](#docker-によるバックエンド起動)
- [デプロイ](#デプロイ)

---

## 前提条件

| ツール     | バージョン | 用途               |
| ---------- | ---------- | ------------------ |
| Git        | 2.x+       | サブモジュール管理 |
| Node.js    | 20+        | フロントエンド     |
| pnpm       | 9+         | パッケージ管理     |
| Python     | 3.11+      | バックエンド       |
| [uv](https://docs.astral.sh/uv/) | 最新 | Python パッケージ管理 |
| Docker     | 20.10+     | コンテナ実行（任意）|
| メモリ     | 4 GB+      | word2vec モデル用  |

---

## リポジトリのクローン

このリポジトリは git submodule を使用してフロントエンドとバックエンドを管理しています。

```bash
# サブモジュールを含めてクローン
git clone --recursive https://github.com/nu-chotech/coto2-ba.git
cd coto2-ba
```

既にクローン済みの場合、サブモジュールを初期化・更新します:

```bash
git submodule update --init --recursive
```

### サブモジュール構成

| サブモジュール | リポジトリ                                            | パス       |
| -------------- | ----------------------------------------------------- | ---------- |
| Frontend       | https://github.com/nu-chotech/coto2-ba-frontend.git   | `frontend/` |
| Backend        | https://github.com/nu-chotech/coto2-ba-backend.git    | `backend/`  |

---

## バックエンドのセットアップ

### 1. 依存パッケージのインストール

```bash
cd backend
uv sync
```

### 2. word2vec モデルのダウンロード

[WikiEntVec](https://github.com/singletongue/WikiEntVec) の学習済みモデル（200 次元）を使用します。

```bash
cd app/models

# ダウンロード（圧縮時約 588 MB）
wget https://github.com/singletongue/WikiEntVec/releases/download/20190520/jawiki.word_vectors.200d.txt.bz2

# 解凍（解凍後約 1.6 GB）
bzip2 -d jawiki.word_vectors.200d.txt.bz2

cd ../..
```

> ⚠️ モデルファイルは `.gitignore` に含まれており、リポジトリにはコミットされません。

### 3. 開発サーバーの起動

```bash
uv run task dev
```

- API: http://localhost:8000
- Swagger UI: http://localhost:8000/docs

### タスク一覧

| コマンド            | 説明                                     |
| ------------------- | ---------------------------------------- |
| `uv run task dev`   | 開発サーバーを起動（ホットリロード有効） |
| `uv run task start` | 本番用サーバーを起動                     |
| `uv run task lint`  | Ruff によるリントチェック                |

---

## フロントエンドのセットアップ

### 1. 依存パッケージのインストール

```bash
cd frontend
pnpm install
```

### 2. 開発サーバーの起動

```bash
pnpm dev
```

http://localhost:3000 でアクセスできます。

> 開発環境では PWA チェック（iOS Safari 専用制限）は自動バイパスされます。

### コマンド一覧

| コマンド        | 説明                   |
| --------------- | ---------------------- |
| `pnpm dev`      | 開発サーバーを起動     |
| `pnpm build`    | プロダクションビルド   |
| `pnpm start`    | ビルド済みアプリを起動 |
| `pnpm lint`     | Biome によるリント     |
| `pnpm format`   | Biome によるフォーマット |

---

## Docker によるバックエンド起動

Docker Compose を使えば、モデルのダウンロードを含めて自動でセットアップできます。

### 起動

```bash
cd backend
docker compose up -d --build
```

初回起動時に WikiEntVec モデル（約 1.6 GB）を自動ダウンロードします。  
ダウンロード済みモデルは名前付きボリュームに永続化されるため、2 回目以降はスキップされます。

### 動作確認

```bash
curl http://localhost:8000/
# => {"status":"ok","message":"Coto2-Ba API is running 🚀"}
```

### 運用コマンド

```bash
# 停止
docker compose down

# 停止 & モデルデータも削除
docker compose down -v

# 再ビルド（コード変更後）
docker compose up -d --build

# ログを確認
docker compose logs -f
```

### リソース要件

- メモリ: 最低 2 GB（推奨 4 GB）
- ディスク: 約 2 GB（モデル + Docker イメージ）

---

## デプロイ

### フロントエンド → Vercel

フロントエンドは [Vercel](https://vercel.com/) にデプロイされています。

- リポジトリ: `nu-chotech/coto2-ba-frontend`
- フレームワーク: Next.js（自動検出）
- URL: `https://coto2-ba.ut42tech.com`

### バックエンド → Render

バックエンドは [Render](https://render.com/) にデプロイされています。

- 設定ファイル: [`backend/render.yaml`](../backend/render.yaml)
- ビルドコマンド: `pip install uv && uv sync --frozen && uv cache prune --ci`
- 起動コマンド: `uv run uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- ヘルスチェック: `GET /`
- Python バージョン: 3.11

---

## トラブルシューティング

### モデルのダウンロードに失敗する

wget が使えない環境では curl を使用してください:

```bash
curl -LO https://github.com/singletongue/WikiEntVec/releases/download/20190520/jawiki.word_vectors.200d.txt.bz2
```

### メモリ不足エラー

word2vec モデルの読み込みには約 2 GB のメモリが必要です。Docker 環境の場合、Docker Desktop のメモリ割り当てを 4 GB 以上に設定してください。

### サブモジュールが空になっている

```bash
git submodule update --init --recursive
```
