# GCPにWebARコンテンツをアップロードしてHTTPS公開（iOS対応）する手順

このドキュメントは、手元で `npm run dev` で起動できる WebAR コンテンツを **GCP上でHTTPS配信** し、**iOS Safari でもアクセス（カメラ権限など）** できるようにするための手順と注意点をまとめたものです。

> 重要: iOS Safari でカメラやセンサー等の機能を使う WebAR は、基本的に **HTTPS（有効な証明書）** が必須です。自己署名証明書やHTTP配信では動作しない／権限が出ないケースが多いです。

---

## 0. 先に確認すべきこと（最短で失敗を減らす）

GCPへ上げる前に、プロジェクトが **「静的サイト」** か **「Nodeサーバ（SSR/API含む）」** かで手順が変わります。

### A. `package.json` の scripts を確認

- **静的ビルドがある**: `build` がある（例: `vite build`, `next build`, `react-scripts build`）
- **本番起動がある**: `start` がある（例: `next start`, `node server.js`）
- `dev` は開発用であり、**GCP上の本番運用にそのまま使うべきではありません**

### B. ビルド成果物の出力先を確認

代表例:

- Vite: `dist/`
- Create React App: `build/`
- Next.js:
  - 静的エクスポートなら `out/`（`next export`）
  - SSRなら `.next/` を `next start` で起動

### C. WebAR固有の依存確認

- **カメラ権限**: `getUserMedia` を使うなら HTTPS 必須
- **外部リソース**（CDN/モデル/画像/動画）: 全て HTTPS か（混在コンテンツ禁止）
- **iOSの制約**: 同時再生・自動再生・WebXR対応状況などで挙動が変わる

---

## 1. 推奨方針（迷ったらこれ）

構成が不明な場合、最も安全なのは **Cloud Run** で「コンテナ化してHTTPS公開」する方法です。

- **メリット**:
  - Node/SSR/静的配信/簡易APIなど、形が何でも対応しやすい
  - HTTPSはGCP側で標準対応（カスタムドメイン＋証明書も管理しやすい）
- **デメリット**:
  - Dockerfile作成が必要
  - 純静的サイトだけなら、Firebase Hostingの方が簡単なこともある

以降は以下の2パターンで説明します:

- **パターンA（推奨）**: Cloud Run（Node/静的どちらも可）
- **パターンB（静的のみ）**: Firebase Hosting（最短でHTTPS配信）

---

## 2. パターンA: Cloud Run でHTTPS公開（推奨）

### 2.1 事前準備（GCP）

- GCPプロジェクト作成（または既存プロジェクトを使用）
- 課金（Billing）有効化（Cloud Run/Cloud Buildで必要）
- Cloud Run / Cloud Build を有効化

### 2.2 本番起動方法を決める（重要）

Cloud Run は **HTTPで待ち受けるプロセス** が必要です。

#### ケース1: Nodeサーバ（Next.js SSRなど）

- `npm run build` → `npm run start` がある構成
- Cloud Run では `PORT` 環境変数でポートが渡されるため、**アプリが `process.env.PORT` を使って listen する必要があります**
  - Next.js の `next start` は Cloud Run と相性が良い部類です

#### ケース2: 静的サイト（Vite/CRAなど）

- `npm run build` で `dist/` や `build/` が生成される構成
- Cloud Run では、静的ファイルを配信する軽量サーバ（例: `serve` / `nginx` / `express`）を使うのが一般的です

### 2.3 Dockerfile（例）

#### 例1: 静的サイトをNodeで配信（`serve` を利用）

- 前提:
  - `npm run build` で `dist/` ができる（Vite想定）
  - `serve` を devDependency ではなく dependency に入れるか、npxで実行できるようにする

Dockerfile例（要調整）:

```dockerfile
FROM node:20-slim
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

ENV PORT=8080
EXPOSE 8080

CMD ["npx", "serve", "-s", "dist", "-l", "8080"]
```

#### 例2: Next.js SSR（`next start`）

Dockerfile例（要調整）:

```dockerfile
FROM node:20-slim
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

ENV PORT=8080
EXPOSE 8080

CMD ["npm", "run", "start"]
```

> 注意: `npm run start` が `next start -p $PORT` のように `PORT` を参照していない場合、Cloud Run で起動しても疎通できません。startスクリプトを調整してください。

### 2.4 デプロイ（Cloud Build でビルド→Cloud Runへ）

ローカルで gcloud を使う場合の概念手順:

1. コンテナをビルド（Cloud Build またはローカルDocker）
2. Artifact Registry にpush
3. Cloud Run にデプロイ（`--allow-unauthenticated` を基本）

Cloud Run設定の要点:

- **Ingress**: 外部からのアクセスを許可
- **認証**: 公開コンテンツなら「未認証呼び出しを許可」
- **環境変数**: `NODE_ENV=production`、必要ならAPIキー等
- **リージョン**: 日本向けなら `asia-northeast1` など

### 2.5 HTTPS（iOSで動かすための要点）

Cloud Run のデフォルトURLはHTTPSですが、iOS配布を意識するなら **独自ドメイン** 推奨です。

- Cloud Run の「カスタムドメイン」機能を使用
- Cloud DNS でレコード設定（`CNAME`/`A` は案内に従う）
- **Google管理SSL証明書** を有効化して、証明書発行を待つ（数分〜最大数時間）

---

## 3. パターンB: Firebase Hosting（静的サイトのみ、最短）

以下に該当するなら Firebase Hosting が最短です:

- `npm run build` で静的ファイルができる
- SSRやNode APIが不要

概略手順:

1. `npm run build` で成果物作成（例: `dist/`）
2. Firebase CLI を初期化（Hosting）
3. 公開ディレクトリに `dist/` を指定
4. `firebase deploy` でデプロイ

特徴:

- HTTPSが自動
- カスタムドメインもGUIで設定しやすい
- ただし **SSRやサーバ処理** が必要なら別途 Cloud Functions/Cloud Run が必要

---

## 4. iOS Safari / WebAR の注意点（HTTPS以外）

### 4.1 Mixed Content（混在コンテンツ）を避ける

- HTML/JS/CSSはHTTPSでも、モデル/画像/動画/音声/外部JSがHTTPだとブロックされます
- 参照先はすべて `https://` に統一してください

### 4.2 カメラ権限の挙動

- 権限ダイアログは **ユーザー操作（クリック等）** がトリガーにならないと出ないことがあります
- iOSでは、初回許可後も「設定」で無効化されるケースがあるため、UIでリカバリ導線を作ると安全です

### 4.3 キャッシュと更新（特にiOS）

- iOS Safari はキャッシュが強く残ることがあります
- 推奨:
  - ファイル名にハッシュ（ビルドツールで多くは自動）
  - `Cache-Control` を適切に設定（HTMLは短め、アセットは長めなど）

### 4.4 MIMEタイプ

- `glb/gltf`, `wasm`, `mp4` などが正しい `Content-Type` で配信されないと失敗します
- Cloud Run（Node配信）ならサーバ側設定、Firebase Hostingなら通常は問題になりにくいです

### 4.5 CORS

- 外部ドメインからモデルやAPIを取得する場合はCORSが必要
- iOSはCORS違反がエラーになっても原因が追いづらいので、まずは **同一オリジン配信** を優先してください

---

## 5. 本番運用のチェックリスト

- **HTTPS**: 有効な証明書（自己署名不可）
- **リダイレクト**: `http -> https`（可能なら）
- **外部リソース**: 全てHTTPS／CORS確認
- **起動確認**: Cloud Run のヘルスチェック（コンテナ起動後に応答するか）
- **iOS実機確認**:
  - iPhoneのSafariでアクセス
  - カメラ権限が出る/ARが動く
  - LTE/5Gなど回線でも読み込みが破綻しない

---

## 6. 進め方（最短で確定させるために）

このリポジトリで次を確認できれば、手順を「あなたの構成に完全一致」する形に絞り込めます。

- `package.json` の `scripts`（`build` / `start` の有無）
- フレームワーク（Vite/Next/React/Three.jsなど）
- ビルド出力先（`dist/`/`build/`/`out/`）

上記が分かったら、Cloud Run用の Dockerfile とデプロイコマンド（またはFirebase Hosting設定）まで含めて、手順を確定版に更新してください。

