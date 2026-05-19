# 横浜ファニーズ 公式サイト 技術解説

草野球チーム「横浜ファニーズ」の公式サイト技術解説リポジトリ。

サイト本体は private リポ（https://github.com/yasumorishima/yokohama-funnies）。

公開 cron: [yokohama-funnies-public-cron](https://github.com/yasumorishima/yokohama-funnies-public-cron) （ubuntu-latest 無料枠で `/weather` ISR cache warm + 週次 `/schedule` SSR fire で Supabase auto-pause 回避、 anon key 不要、 RPi5 outage 影響圏外）。

## サイトについて

- **本番**: https://yokohama-funnies.vercel.app/ （公開済）
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
- 球場の天気予報 (Open-Meteo API、 WBGT 当日 hourly ピーク判定、 横浜スタジアム 先頭 + デフォルト展開、 降水量 mm 表示 「累計 N.Nmm」 (日別) + 毎時 mm (時間帯別)、 雨アイコン 4 段階 小雨/中雨/大雨/暴風雨 を mm で判定)
- ホーム CONTENTS ナビバンド (5 色 card-style + 横スライド snap で 5 セクションへ即ジャンプ、 hint label 2 段表示)

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

## 視覚デザイン

- **フォント**: Zen Maru Gothic (Japanese 丸ゴシック) + Fredoka (英字)
- **配色**: navy `#1e3a8a` (チームカラー) + orange `#ea580c` (アクセント) + warm cream `#fff7ed` (背景) + pastel yellow (セクション交互)
- **レイアウト**: Photo Magazine スタイル長スクロール。 全幅 hero (`h-[85vh]`) + bottom-aligned 巨大タイトル、 editorial `№ 01-05` セクションラベル、 大型 Card (`rounded-3xl` + ソフト影)
- **トップ構成**: Hero → CONTENTS ナビバンド (5 色 card-style + 横スライド snap) → 横浜スタジアム 3 日天気 (M/D + 曜日 + 降水% + 累計 N.Nmm + 雨強度別アイコン、 クリックで /weather に遷移し横浜スタジアムが先頭で自動展開) → チームの一言 → ABOUT US → MOMENTS (写真) → NEWS → UP NEXT (Google Calendar 風 月間ビュー) → RECORD

## セキュリティ

- **権限**: Supabase RLS で全 table 行レベル制御、 5 段階 role
- **secret 検知**: gitleaks workflow が main push / PR / 手動実行で全履歴スキャン、 binary は SHA256 検証で取得（supply-chain 防御）
- **env 規約**: `.env.production` は `NEXT_PUBLIC_*` のみ（client 公開前提値）、 service_role / DB password / admin token 系は GitHub Actions secrets で管理

## ステータス

着手: 2026-05-10、 公開: 2026-05-11 (Vercel Hobby plan)。 minami-baseball-ob（横浜市立南高校 野球部 OB 会サイト）を template として、 OB 専用機能を全削除し草野球チーム向けに再設計中。 試合スケジュール・選手名簿・出欠・成績集計を順次実装中。
