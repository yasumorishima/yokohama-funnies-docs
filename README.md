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
- 個人成績ページ `/members-only/stats` (打撃 / 投手 タブ切替 + 年度フィルタ button group + 全 header クリックでソート、 打撃は全選手表示 [0 打席含む]、 投手は登板選手のみ、 列順は見たい指標 [打率/OPS/防御率/WHIP/K-9] を左、 12 月の試合は翌年 season 扱い特別対応)
- 出欠管理（試合・練習）
- 写真ギャラリー
- お知らせ・会員専用投稿 (2 カテゴリ: 会計記録 / 連絡事項)
- 会員申請フォーム → admin が `/admin/roles` で承認 (新規 OAuth signup で trigger が viewer 自動付与、 admin が member/editor 昇格)
- 球場の天気予報 (Open-Meteo API、 WBGT 当日 hourly ピーク判定、 横浜スタジアム 先頭 + デフォルト展開、 降水量 mm 表示 「累計 N.Nmm」 (日別) + 毎時 mm (時間帯別)、 雨アイコン 4 段階 小雨/中雨/大雨/暴風雨 を mm で判定)
- ホーム CONTENTS ナビバンド (5 色 card-style + 横スライド snap で 5 セクションへ即ジャンプ、 hint label 2 段表示)
- スコアブック viewer (会員以上、 試合詳細 `/results/[id]` page で画像 inline 閲覧 + 監督コメント本文 inline、 別 page `/scorebooks/[result_id]` も残置、 editor は同 page で 90°/180° step 回転 + 画像クリックで原寸別タブ拡大、 sharp 経由 download + rotate + upsert)
- 監督コメント (`/results/[id]` 試合詳細に会員限定で本文表示、 editor のみ投稿/編集 link、 1 試合 1 件、 履歴 trigger 付き)

## DB スキーマ

PostgreSQL (Supabase)、 主要 table:

- **players**（選手名簿） — 背番号、 ポジション、 打/投左右、 入団年、 active/OB 区分、 team_role 自由テキスト
- **game_player_batting**（試合別打撃集計） — **1 試合 1 選手 = 1 row**、 PA/AB/H/2B/3B/HR/RBI/BB+HB/犠/盗/三振、 stats page の primary source、 xlsx ground truth
- **game_player_pitching**（試合別投手集計） — **1 試合 1 投手 = 1 row**、 IP/R/H/K/BB/W/L
- **at_bats**（打席ごと microdata、 未使用予備）、 **pitching_logs**（投球ごと microdata、 未使用予備）
- **attendances**（出欠） — ○/△/×/未回答
- **members_posts**（会員投稿） — 2 カテゴリ: 会計記録 / 連絡事項
- **trigger on auth.users** — 新規 OAuth signup で `user_roles(role='viewer')` を auto-INSERT

## 認証

Google OAuth、 5 段階権限制御（admin / editor / member / viewer / 未ログイン）。

## データ取り込み

- スコアブック OCR (`baseball-scorebook-ocr`、 RPi5 ローカル): 紙のスコアブック → at_bats / pitching_logs に同期
- スプレッドシート（既存）: 移行期間のみ並行運用

## 視覚デザイン

- **フォント**: Zen Maru Gothic (Japanese 丸ゴシック) + Fredoka (英字)
- **配色**: navy `#1e3a8a` (チームカラー) + orange `#ea580c` (アクセント) + 白 `#ffffff` (背景、 favicon の navy cap + orange F + 白 と整合) + sky-blue `#dbeafe` (section-alt) + slate `#cbd5e1` (border)
- **レイアウト**: Photo Magazine スタイル長スクロール。 全幅 hero (`h-[85vh]`) + bottom-aligned 巨大タイトル、 editorial `№ 01-05` セクションラベル、 大型 Card (`rounded-3xl` + ソフト影)
- **トップ構成**: Hero → CONTENTS ナビバンド (5 色 card-style + 横スライド snap) → 横浜スタジアム 3 日天気 (M/D + 曜日 + 降水% + 累計 N.Nmm + 雨強度別アイコン、 クリックで /weather に遷移し横浜スタジアムが先頭で自動展開) → チームの一言 → ABOUT US → MOMENTS (写真) → NEWS → UP NEXT (Google Calendar 風 月間ビュー) → RECORD

## セキュリティ

- **権限**: Supabase RLS で全 table 行レベル制御、 5 段階 role
- **secret 検知**: gitleaks workflow が main push / PR / 手動実行で全履歴スキャン、 binary は SHA256 検証で取得（supply-chain 防御）
- **env 規約**: `.env.production` は `NEXT_PUBLIC_*` のみ（client 公開前提値）、 service_role / DB password / admin token 系は GitHub Actions secrets で管理

## ステータス

着手: 2026-05-10、 公開: 2026-05-11 (Vercel Hobby plan)。 minami-baseball-ob（横浜市立南高校 野球部 OB 会サイト）を template として、 OB 専用機能を全削除し草野球チーム向けに再設計。 **2026-05-23 時点**で試合スケジュール / 試合結果 / 選手名簿 (23 名) / 試合別打撃 + 投手 DB / /stats page (打撃投手タブ + 年度フィルタ + sortable) / 会員専用投稿 / スコアブック inline 表示 + 90°/180° 回転 / 監督コメント / Open-Meteo 天気予報 + WBGT 全実装済。 Google OAuth は 2026 年 6 月開通予定。
