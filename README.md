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

## DB スキーマ

PostgreSQL (Supabase)、 主要 table:

- **players**（選手名簿） — 背番号、 ポジション、 打/投左右、 入団年、 active/OB 区分
- **at_bats**（打席記録） — 16 値 outcome (1B/2B/3B/HR/BB/HBP/K/SF/SH/FC/E/GO/FO/LO/DP/その他)、 OCR 対応
- **pitching_logs**（投球記録） — 投球回 / H / R / ER / BB / K / win/loss/save/hold
- **attendances**（出欠） — ○/△/×/未回答
- **members_posts**（会員投稿） — 試合振り返り / 練習メモ / 飲み会記録 / 連絡事項

集計 view: `player_batting_stats` (打率/出塁率/長打率) + `player_pitching_stats` (ERA/WHIP/K9)

## 認証

Google OAuth、 5 段階権限制御（admin / editor / member / viewer / 未ログイン）。

## データ取り込み

- スコアブック OCR (`baseball-scorebook-ocr`、 RPi5 ローカル): 紙のスコアブック → at_bats / pitching_logs に同期
- スプレッドシート（既存）: 移行期間のみ並行運用

## セキュリティ

- **権限**: Supabase RLS で全 table 行レベル制御、 5 段階 role
- **secret 検知**: gitleaks workflow が main push / PR / 手動実行で全履歴スキャン、 binary は SHA256 検証で取得（supply-chain 防御）
- **env 規約**: `.env.production` は `NEXT_PUBLIC_*` のみ（client 公開前提値）、 service_role / DB password / admin token 系は GitHub Actions secrets で管理

## ステータス

着手: 2026-05-10。 minami-baseball-ob（横浜市立南高校 野球部 OB 会サイト）を template として、 OB 専用機能を全削除し草野球チーム向けに再設計中。
