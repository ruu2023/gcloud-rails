# README

This README would normally document whatever steps are necessary to get the
application up and running.

Things you may want to cover:

* Ruby version

* System dependencies

* Configuration

* Database creation

* Database initialization

* How to run the test suite

* Services (job queues, cache servers, search engines, etc.)

* Deployment instructions



```bash
chmod +x bin/docker-entrypoint
git update-index --chmod=+x bin/docker-entrypoint

# deploy
gcloud run deploy gcloud-rails \
  --source . \
  --platform managed \
  --region asia-northeast1 \
  --allow-unauthenticated \
  --set-secrets="RAILS_MASTER_KEY=rails-master-key:latest"
```

* ...

## Secretの初回セットアップ

`RAILS_MASTER_KEY` はSecret Managerに登録し、Cloud Runには `--set-secrets` で注入する(平文を `--set-env-vars` やシェル履歴に残さないため)。

```bash
# 1. master.keyをシークレットとして登録
gcloud secrets create rails-master-key \
  --data-file=config/master.key

# 2. Cloud Runのランタイムサービスアカウントに閲覧権限を付与
gcloud secrets add-iam-policy-binding rails-master-key \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## master.keyのローテーション手順

漏洩時やチームメンバー退職時など、鍵を作り直す必要がある場合の手順。**ローテーションすると既存のCookie/セッションは全て無効になる**(`secret_key_base` も同時に入れ替わるため)。

```bash
# 1. 既存の鍵と暗号化ファイルを削除して作り直す(中身は残った状態で再暗号化される)
rm config/master.key config/credentials.yml.enc
EDITOR="vi" bin/rails credentials:edit

# 2. 新しいmaster.keyをSecret Managerの新バージョンとして登録
gcloud secrets versions add rails-master-key --data-file=config/master.key

# 3. 新しい credentials.yml.enc をコミット(master.key自体はgitignore対象なのでコミットしない)
git add config/credentials.yml.enc
git commit -m "Rotate Rails master key"

# 4. 再デプロイ
gcloud run deploy gcloud-rails \
  --source . \
  --platform managed \
  --region asia-northeast1 \
  --allow-unauthenticated \
  --set-secrets="RAILS_MASTER_KEY=rails-master-key:latest"
```

**注意**:
* `master.key` と `credentials.yml.enc` は鍵と暗号化データのペア。片方だけ更新すると復号エラーで起動しなくなる。
* すでに `RAILS_MASTER_KEY` を通常の環境変数(`--set-env-vars`)として設定したことがある場合、`--set-secrets` に切り替える際は同じデプロイコマンドに `--remove-env-vars="RAILS_MASTER_KEY"` を追加しないと型の競合でデプロイが失敗する。

## Cloud Run向けの変更点

### Thrusterを外してPumaを直接起動

Rails標準Dockerfileの `CMD` は `./bin/thrust ./bin/rails server` でしたが、Cloud Runでは常に502エラーになった。

**原因**: Thrusterはポートを即座に開いてPumaを子プロセスとして起動するプロキシ構成。Cloud Runは「ポートが開いた=起動完了」とみなしてCPUをスロットリングするため、まだ起動中のPumaがCPUを奪われて `:3000` を開く前にタイムアウトし、Thrusterへの接続が `connection refused` → 502になっていた。

**対応**: `CMD` を `./bin/rails server` に変更し、Pumaを直接起動するように修正。`config/puma.rb` が `ENV.fetch("PORT", 3000)` でリッスンポートを決めているため、Cloud Runが注入する `PORT` 環境変数(デフォルト8080)をそのまま拾える。これによりポートが開くタイミング=アプリが実際にリクエストを処理できるタイミングと一致し、CPUスロットリングとの競合が解消された。

これに伴い `gcloud run deploy` コマンドから `--port=80` オプションを削除(Cloud Runのデフォルトポート8080をそのまま使用)。
