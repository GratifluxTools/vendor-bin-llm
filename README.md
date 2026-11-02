# Local LLM on GitHub Actions

プロンプトをWeb UIから送ると、GitHub Actions（無料のCPUランナー 4 vCPU / 16 GB）で
llama.cpp がローカルLLMを動かし、結果をリポジトリに書き戻してUIに返す仕組みです。
APIキー・外部LLMサービス不要。**モデルは actions/cache に保存され、2回目以降はダウンロードゼロ**です。

```
[ブラウザ] index.php ──POST workflow_dispatch──▶ [GitHub Actions]
    │                                              │
    │  results/<id>.json を4秒間隔でポーリング        ├─ llama.cpp (CPU) + GGUF
    └──────────────────────────────────────────────┘
```

## 1. ファイル構成

```
.
├── index.php                      # Web UI（添付デザインを踏襲）＋ APIプロキシ
├── .github/workflows/llm.yml      # 推論ワークフロー
├── results/                       # 結果JSONの置き場（自動生成）
│   └── .gitkeep
└── README.md
```

## 2. セットアップ

1. **リポジトリ作成**（Public 推奨 = Actions 無制限・無料）
2. このフォルダの中身をそのまま push（`results/.gitkeep` も忘れずに）
3. **Personal Access Token を作成**（Settings → Developer settings）
   - Fine-grained PAT: `Actions: Read and write` + `Contents: Read and write`（対象リポジトリを選択）
   - Classic PAT: `repo` + `workflow` スコープ
4. `index.php` の先頭を書き換え
   ```php
   const DEFAULT_REPO     = 'your-name/your-repo';
   const DEFAULT_WORKFLOW = 'llm.yml';
   ```
   （画面の Connection 欄からも変更でき、localStorage に保存されます）
5. `index.php` を PHP ホスティングにアップロードして開く
   - PHP が無い場合はファイル名を `index.html` にして GitHub Pages に置くだけでも動きます
     （GitHub API は CORS 対応なので、JS が自動で「direct mode」に切り替わります）
6. 画面でトークン → リポジトリ名を入れて、プロンプトを送信

## 3. モデル

| 既定 | サイズ | 速度の目安(4vCPU) | 切替方法 |
|---|---|---|---|
| **Qwen3-4B-Instruct-2507 Q4_K_M** | 約2.5 GB | 8〜15 tok/s | 既定のまま |
| Qwen3-8B-Instruct-2507 Q4_K_M | 約5.0 GB | 3〜6 tok/s | リポジトリ変数 `LLM_MODEL_URL` を変更 |

- Qwen3-2507 系は「thinking」なしの指示特化モデルなので、応答が速くCPU向きです。
- 変更する場合: Settings → Secrets and variables → Actions → **Variables** に登録
  - `LLM_MODEL_URL` … GGUF の直リンク（Hugging Face の `resolve/main/...gguf`）
  - `LLM_MODEL_NAME` … UI 表示名（例: `Qwen3-8B-Instruct-2507 (Q4_K_M)`）
  - `LLM_CTX` … コンテキスト長（既定 8192）
- キャッシュキーは URL のハッシュなので、**モデルを差し替えると自動で別キャッシュ**になります（10 GB 上限に注意）。

## 4. ダウンロード時間を削る仕組み

1. `actions/cache` で `~/models` を丸ごと保存 → 2回目以降は数十秒で復元（実質ゼロ）
2. `steps.prep` で URL ハッシュからキャッシュキーを生成（モデル変更時も衝突しない）
3. llama.cpp は公式リリースのプリビルド（Ubuntu x64）を使用。取得できなかった場合のみソースビルドにフォールバック
4. キャッシュヒット時はダウンロードステップをスキップ

## 5. 仕組みの要点

- `workflow_dispatch` の inputs で `prompt` / `request_id` を渡す（ブランチは `main`）
- ワークフローは `llama-server` を起動し、OpenAI 互換の `/v1/chat/completions` に POST
  → チャットテンプレートはモデル内蔵のものが自動適用されるので、プロンプト整形が不要
- 結果は `results/<request_id>.json` としてコミット
- UI は `results/<id>.json` を4秒ごとに取得（404 = まだ実行中）
- Artifacts と違い、**結果JSONの取得に認証トークン付きAPIで足りる**のでUIから素直に読めます

## 6. セキュリティ上の注意（重要）

- **このUIはトークンをブラウザの localStorage に保存します。** 共用PCでは使わないでください。
- **Public リポジトリにすると `results/*.json` は誰でも閲覧できます。** プロンプトと回答に機密情報を入れないこと。機密なら Private リポジトリにして、トークンは最小権限にしてください。
- **できないこと**: 静的サイトだけでトークンを隠すこと。GitHub の仕様上、ブラウザから dispatch するにはユーザーのトークンが必要です。他人に配布する場合は、トークンをサーバー側（自分のPHP）に隠す構成に変えるか、GitHub OAuth App を挟んでください。
- プロンプトは `env:` 経由でシェルに渡しているため、`${{ inputs.prompt }}` を `run:` に直接埋め込むような改変（シェルインジェクション）は避けてください。
- 実行はリポジトリの Actions 権限を持つ人のみ可能。必要なら Environment の承認ルールや、`github.actor` の許可リストを追加してください。

## 7. トラブルシューティング

| 症状 | 原因 / 対処 |
|---|---|
| `403` / `404` で dispatch 失敗 | ワークフロー名（`llm.yml`）、ブランチ名（`main`）、PAT権限（Actions: write）を確認 |
| いつまでも QUEUED | Actions が無効 / 同時実行制限。リポジトリの Actions タブで状態確認 |
| `results/` にファイルが増えない | workflow に `permissions: contents: write` があるか、branch protection で push がブロックされていないか |
| 初回が遅い | モデル2.5GBのダウンロード＋ランナー準備で 2〜4分。2回目以降は 40秒〜1分台 |
| 途中で失敗する | Actions のログで `/tmp/server.log` の tail を確認。コンテキスト長を超える長文は `LLM_CTX` を上げる |
| 日本語が崩れる | モデルは Qwen3 系（日本語可）。小さいモデルは精度が落ちるので 8B へ |

## 8. 拡張アイデア

- ストリーミング表示（`llama-server` は SSE 対応。Actions のログを poll して逐次表示）
- 会話履歴を `messages` 配列で渡してマルチターン化
- `results/` を `gh-pages` ブランチに分離してリポジトリの履歴を汚さない
- 画像入力（Qwen2.5-VL などの VLM + mmproj）でマルチモーダル化
