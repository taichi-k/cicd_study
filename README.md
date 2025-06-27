# GitHubのCI/CD勉強メモ

## CI / Integration
- コードをコードベースに適用し、検証すること  
  - 検証作業を自動化すること

## CD / Delivery
- リリースしてユーザに届けること  
  - リリース作業を自動化すること
- CIはCDに包含される

---

## GitHub Actions

### ワークフロー定義
```yaml
# ワークフローは .github/workflows/*.yml に配置
on:
  push:
    branches: [ main ]
  workflow_dispatch:    # 手動実行
  schedule:             # 定期実行
    - cron: '0 0 * * *'
```

### ランナー
- `runs-on: ubuntu-latest` など最新OS指定がおすすめ
- 事前に各種ソフトがインストール済み  
  → 特定バージョンが必要ならステップ内でインストール

### エフェメラル環境
- 実行中のみ存在する一時環境

### Context と環境変数
```yaml
env:
  ACTOR: ${{ github.actor }}
steps:
  - run: echo "${ACTOR}"
```
- `${{ … }}` は必ずダブルクォートで囲む
- Settings > Secrets and variables で事前登録
  - `secrets.*` はログにマスクされ、登録後は閲覧不可

### オブジェクトフィルター
- 一部の値を選択可能  
  例）`${{ github.contexts.*.html_url }}`

### 条件付き実行
```yaml
jobs:
  build:
    if: github.event_name == 'push'
```

### ステップ間の値受け渡し
- **GITHUB_OUTPUT**（おすすめ／ジョブ内限定）
  ```bash
  echo "RESULT=hello" >> "$GITHUB_OUTPUT"
  ```
- **GITHUB_ENV**（グローバル環境変数）
  ```bash
  echo "RESULT=hello" >> "$GITHUB_ENV"
  ```

### トークン & パーミッション
- `secrets.GITHUB_TOKEN`：ワークフロー中のみ有効
- `permissions` セクションでスコープ指定可  
  ```yaml
  permissions:
    contents: read
    issues: write
  ```
- 他リポジトリへのアクセスは不可

---

## CIでよく使うジョブ
- 自動テスト
- 静的解析
- リポジトリ混入クレデンシャル検出
- コンテナ依存関係脆弱性検査
- IaC設定ミス／ポリシー違反検知

---

## ソースコードのチェックアウト
- 明示的に `actions/checkout` が必要

---

## `on:` イベント
- `issue`：`types` 指定しなければ全てのアクティビティ
- `pull_request`：既定で特定イベントのみ

---

## タイムアウト／シェル／自動キャンセル
```yaml
jobs:
  test:
    timeout-minutes: 30    # デフォルト6h→手動設定推奨
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash --login -euo pipefail
```
- デフォルトシェルは `bash`  
- 明示指定してオプション統一
- **concurrency**  
  ```yaml
  concurrency:
    group: ${{ github.ref }}
    cancel-in-progress: true
  ```

---

## CI運用の3原則
1. **クリーンを保つ**  
   - エラーを無視せず徹底対応
2. **高速に実行する**  
   - 理想5分以内、遅くとも10分以内
3. **ノイズを減らす**  
   - 必要なログだけを出力

---

## テスト戦略
- **Unit テスト**：単体振る舞い検証・高速・独立  
- **Integration テスト**  
- **E2E テスト**  
- テストピラミッド比率：Unit > Integration > E2E  
- フレーキーテスト（稀に失敗） → 1%以下に抑制  
- フラジャイル／スローテストの監視

---

## 警告の抑止
- 静的解析警告を抑止する場合、理由をコミットログに明記

---

## プルリクエスト実行
- PR発行時に自動実行が一般的

---

## デバッグ／ロググループ
- Bashの `-x`（トレーシング）活用  
- `::group::` / `::endgroup::` でログ区切り

---

## 並列実行とマトリックス
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [14, 16, 18]
```
- `needs` で逐次化
- キャッシュキーに `os` や `lockfile` のハッシュを含める

---

## 自作 Action
- `action.yml` に丁寧な description を記述
- `shell`, `inputs`, `outputs` を活用  
- README にパーミッション要件を明記
- ログは `::group::` で整理

---

## コードオーナーと技術的負債
- 個人オーナーは複数人に設定、最小人数維持

---

## クレデンシャル対策
- 検知次第ローテーション・無効化  
- 年1回の棚卸し推奨  
- `secretlint`, `gitleaks`, Gitフックで自動検知

---

## Dependabot
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```
- `ignore` 除外設定時は理由を明記  
- `GITHUB_TOKEN` のみアクセス可能

---

## リリースノート自動化
- ラベルでリリースノート選別  
- `.github/release.yml` で設定

---

## GitHub CLI
- 使用時は `GITHUB_TOKEN` を忘れずに

---

## OpenID Connect (OIDC)
- 静的クレデンシャル不要  
- AWS 例：
  1. IAM ID プロバイダ設定  
  2. IAM ロール／ポリシー作成  
  3. Secrets にアカウント名／ロール名登録

---

## アクションのテスト（4フェーズ）
1. setup  
2. exercise  
3. verify  
4. teardown  
> teardown はエラー可とし、手動対応を容易に

---

## 再利用可能ワークフロー
- `workflow_call` で呼び出し  
- 呼び出し元のパーミッションを継承（ただし明記推奨）  
- ネストは控えめに

---

## その他Tips
- `fromJSON()` で文字列→数値キャスト  
- `continue-on-error: true` はリカバリー不要時に  
- `fail-fast: false` で他ジョブを維持

---

## セキュリティ強化
- 攻撃面最小化・多層防御・最小権限  
- サードパーティActionは信頼性確認  
- OpenSSF Scorecards でスコア確認  
- スクリプトインジェクション対策に中間環境変数を習慣化  
- `shellcheck` でシェルの脆弱箇所検出  
- workflow-level permissions (`permissions: {}`) とジョブ分割

---

## 組織パフォーマンス指標（DORA）
- 変更リードタイム  
- デプロイ頻度  
- 変更失敗率  
- 回復期間
