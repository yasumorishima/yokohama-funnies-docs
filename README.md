# 横浜ファニーズ 公式サイト 技術解説

草野球チーム「横浜ファニーズ」の公式サイト技術解説リポジトリ。

サイト本体は private リポ（https://github.com/yasumorishima/yokohama-funnies）。

## サイトについて

- **本番**: https://yokohama-funnies.vercel.app/ （準備中）
- **対象**: 横浜ファニーズのメンバー向け
- **目的**: 試合・練習・成績・出欠の一元管理

## 技術スタック

- **フロント**: Next.js 15 (App Router) + TypeScript + Tailwind CSS 4
- **バックエンド**: Supabase（Postgres + Auth + Storage）
- **ホスティング**: Vercel
- **CI**: GitHub Actions

## 主要機能（実装中）

- 試合スケジュール / 試合結果管理
- 個人成績（打率 / 出塁率 / 防御率 / WHIP）
- 出欠管理
- 写真ギャラリー
- お知らせ・会員専用投稿
- 球場の天気予報

## 認証

Google OAuth、 5 段階権限制御（admin / editor / member / viewer / 未ログイン）。

## ステータス

着手: 2026-05-10。 minami-baseball-ob（横浜市立南高校 野球部 OB 会サイト）を template として作成中。
