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
├── articles/        # Zenn 記事（スラッグ名.md）
├── books/           # Zenn 本（未使用）
├── package.json     # zenn-cli 依存
└── .gitignore
```

## 運用
- 記事ソースは `articles/` に格納（Zenn 形式 Markdown）
- ローカルプレビュー: `npx zenn preview` → http://localhost:8000
- dev.to への投稿は手動（Markdown をコピペ or dev.to API）
- **英語のみ。** 日本語記事は `odakin/zenn-articles` に格納

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
