# zenn-articles-en

## 概要
英語記事のソース管理リポ。投稿先は dev.to（Zenn は日本語専用）。

## リポジトリ情報
- パス: `~/Claude/zenn-articles-en/`
- ブランチ: `main`
- リモート: `odakin/zenn-articles-en` (public, GitHub)

## 構造
```
zenn-articles-en/
├── CLAUDE.md        # このファイル
├── README.md        # 記事一覧・使い方
├── .github/workflows/
│   └── publish-devto.yml  # push → dev.to 自動投稿
├── articles/        # 記事ソース
│   ├── *-en.md      # Zenn 形式（原本）
│   └── *-devto.md   # dev.to 形式（投稿用、GitHub Actions で自動投稿）
├── books/           # 未使用
├── package.json     # zenn-cli 依存（プレビュー用）
└── .gitignore
```

## 記事管理ワークフロー
1. Zenn 形式で原稿を書く（`articles/<slug>-en.md`）
2. dev.to 形式に変換（`articles/<slug>-devto.md`）
3. dev.to に投稿（手動コピペ or API）

### Zenn → dev.to 変換ルール
| Zenn | dev.to | 対応 |
|---|---|---|
| `topics: ["A", "B"]` | `tags: a, b` | 配列→カンマ区切り、最大4つ、小文字 |
| `emoji: "🧠"` | 削除（`cover_image` で代替可） | |
| `type: "tech"` | 削除 | |
| — | `description: "..."` | 追加（SEO 用） |
| — | `canonical_url: URL` | 追加（クロスポスト時） |
| `:::message` ... `:::` | `> **Note:** ...` | ブロック引用に変換 |
| `:::message alert` | `> **Warning:** ...` | ブロック引用に変換 |
| `# 見出し` | `## 見出し` | dev.to は `#` をタイトルに使うため、本文は `##` 始まり |
| `https://github.com/user/repo` | `{% github user/repo %}` | GitHub embed |

### dev.to 投稿（自動）
- `articles/*-devto.md` を変更して push → GitHub Actions が自動で dev.to に投稿
- `published: false` → ドラフト、`published: true` → 公開
- アクション: `sinedied/publish-devto@v2`
- API キー: GitHub secret `DEVTO_API_KEY` に登録済み
- 初回投稿後、アクションが frontmatter に `id` フィールドを自動追加（記事追跡用）

## 運用
- ローカルプレビュー: `npx zenn preview` → http://localhost:8000（Zenn 形式のみ）
- **英語のみ。** 日本語記事は `odakin/zenn-articles` に格納

## How to Resume
1. SESSION.md は不要（記事リポのため）
2. README.md の記事一覧を参照
3. 変更後は commit + push

## 英語記事のルール
- **dev.to 記事（`*-devto.md`）に日本語・非 ASCII テキストを入れない。** リンクテキストも英語のみ（例: `[Japanese](URL)` ○、`[日本語版](URL)` ✗、`[Japanese (日本語)](URL)` ✗）。コードブロック内の記号（`←`, `→`, `―` 等）は許容。

## 安全規則（公開リポ）
**このリポは public。** 以下を絶対にコミットしない:
- 実名（GitHub ユーザー名 `odakin` は可）
- メールアドレス
- 非公開リポ名
- 金融データ・口座情報
- 所属機関名
- 他ユーザーのユーザー名
- dev.to API キー

## 規約
- `~/Claude/CONVENTIONS.md` に従う
