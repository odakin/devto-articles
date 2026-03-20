# zenn-articles-en

## 概要
Zenn.dev 英語記事の一元管理リポ。zenn-cli + GitHub 連携で自動デプロイ。

## リポジトリ情報
- パス: `~/Claude/zenn-articles-en/`
- ブランチ: `main`
- リモート: `odakin/zenn-articles-en` (public, GitHub)

## 構造
```
zenn-articles-en/
├── CLAUDE.md        # このファイル
├── README.md        # 記事一覧・使い方
├── articles/        # Zenn 記事（スラッグ名.md）
├── books/           # Zenn 本（未使用）
├── package.json     # zenn-cli 依存
└── .gitignore
```

## 運用
- 記事の追加: `npx zenn new:article --slug my-slug` → 編集 → push
- プレビュー: `npx zenn preview` → http://localhost:8000
- 公開: frontmatter の `published: false` → `true` に変えて push
- デプロイ: GitHub 連携により `articles/` を含む push で自動デプロイ
- **記事は英語のみ。** 日本語記事は `odakin/zenn-articles` に格納

## How to Resume
1. SESSION.md は不要（記事リポのため）
2. README.md の記事一覧を参照
3. 変更後は commit + push

## 安全規則（公開リポ）
**このリポは public。** 以下を絶対にコミットしない:
- 実名（GitHub ユーザー名 `odakin` は可）
- メールアドレス
- 非公開リポ名
- 金融データ・口座情報
- 所属機関名
- 他ユーザーのユーザー名

## 規約
- `~/Claude/CONVENTIONS.md` に従う
