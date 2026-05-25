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
- 会員申請フォーム → admin が `/admin/roles` で承認 (新規 OAuth signup で trigger が viewer 自動付与、 admin が member/editor 昇格)。 **申請が届くと自動で GitHub issue 起票 + 管理者にメール通知** (Google Form → Apps Script → Vercel proxy → public repo の GitHub Actions → GitHub App、 private repo の Actions 枠を使わず public repo で実行)
- フィードバックフォーム `/feedback` (Google ログイン要、 GitHub App `yokohama-funnies-bot` 経由で private repo に issue 自動起票、 **送信時に管理者へメール通知も送信**、 運営が GitHub 上で管理)
- silent-fail 監視 (会員申請 / フィードバックの通知経路を毎時自動チェックし、 認証鍵の不一致や経路断などで通知が静かに止まったら管理者にメール警告。 「気付かないうちに申請が届かない」 事故を防止)
- 球場の天気予報 (Open-Meteo API、 WBGT 当日 hourly ピーク判定、 横浜スタジアム 先頭 + デフォルト展開、 降水量 mm 表示 「累計 N.Nmm」 (日別) + 毎時 mm (時間帯別)、 雨アイコン 4 段階 小雨/中雨/大雨/暴風雨 を mm で判定)
- ホーム CONTENTS ナビバンド (5 色 card-style + 横スライド snap で 5 セクションへ即ジャンプ、 hint label 2 段表示)
- スコアブック viewer (会員以上、 試合詳細 `/results/[id]` page で画像 inline 閲覧 + 監督コメント本文 inline、 別 page `/scorebooks/[result_id]` も残置、 editor は同 page で 90°/180° step 回転 + 画像クリックで原寸別タブ拡大、 sharp 経由 download + rotate + upsert)
- 監督コメント (`/results/[id]` 試合詳細に会員限定で本文表示、 editor のみ投稿/編集 link、 1 試合 1 件、 履歴 trigger 付き)
- 試合別成績手動入力 (`/edit/game-stats` hub + `/edit/game-stats/[result_id]` 入力ページ、 editor+ 限定、 上半分 scorebook 画像表示 + 下半分 打撃/投手 タブ + 選手 picker + inline input + bulk UPSERT、 mobile (≤640px) は 1 選手 1 card、 入力済試合は /results/[id] の inline 成績で member+ に表示、 助っ人選手は名簿に先行追加してから picker で選択)
- 助っ人選手 (`players.is_guest BOOLEAN`、 名簿 form の checkbox、 ON で 背番号 input disable + jersey_number=NULL 強制、 /stats 累計から除外、 選手一覧 末尾 group、 試合別 inline 成績の 背番号 欄に「助っ人」 表示)
- 選手ごと 写真 + コメント (`players.photo_path` 列追加 [2026-05-24]、 既存 `notes` 列をコメント用に再利用、 /members-only/players edit form に写真 upload + 削除 button + コメント textarea、 list page で 円形 thumbnail + コメント 2 行 truncate 表示)
- ホーム 公開 ROSTER section (`players_public` view 経由で anon でも閲覧可能、 sensitive 列 [user_id] 除外 + active + 非削除 + 助っ人除外、 写真 + 背番号 + 役職 + 名前 + ポジション + コメント を grid card 表示、 mobile 2 列 / sm 3 列 / md 4 列)
- スコアブック画像 client upload UI (editor+ が `/scorebooks/[result_id]` の画面から直接アップロード、 multi file 選択 + 10MB 上限、 filename は `<timestamp>_<safe_name>` で sort 順保証)

## DB スキーマ

PostgreSQL (Supabase)、 主要 table:

- **players**（選手名簿） — 背番号、 ポジション、 打/投左右、 入団年、 active/OB 区分、 team_role 自由テキスト、 **is_guest** (助っ人フラグ)、 **photo_path** (Storage 写真 path、 2026-05-24 追加)、 **notes** (コメント、 自由記述)
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

着手: 2026-05-10、 公開: 2026-05-11 (Vercel Hobby plan)。 minami-baseball-ob（横浜市立南高校 野球部 OB 会サイト）を template として、 OB 専用機能を全削除し草野球チーム向けに再設計。 **2026-05-24 時点**で試合スケジュール / 試合結果 / 選手名簿 (23 名) / 試合別打撃 + 投手 DB / 試合別成績手動入力ページ (editor+、 /edit/game-stats) / 助っ人選手機能 / /stats page (打撃投手タブ + 年度フィルタ + sortable) / 会員専用投稿 / スコアブック inline 表示 + 90°/180° 回転 / 監督コメント / Open-Meteo 天気予報 + WBGT 全実装済。 編集メニューは 3 グループ (試合関連 / チーム情報 / その他) に整理。 Google OAuth は 2026-05-24 に Production publish 完了 (任意の Google アカウントで login 可、 fwyasu11 以外も sign up 成功)、 会員申請 Form メアドは Verified mode で自動取得 (本人検証済 + 編集不可)。 フィードバック窓口は GitHub App 経由 private repo issue 化で 2026-05-23 から稼働。 Google Analytics は minami と独立な専用 GA4 property (G-PJQFLXL3P6) を 2026-05-24 開設。 2024 シーズン試合結果 13 件 (ダブルヘッダー + 中止 + 紅白戦含む) を取り込み済 (通算 27 試合)。 名簿に写真 + コメント機能、 ホーム 06 ROSTER 公開 section、 スコアブック画像の editor 直 upload UI を 2026-05-24 追加。 **2026-05-25 に会員申請通知 + フィードバック通知 + silent-fail 監視を minami と同一運用で構築** (会員申請・フィードバックとも送信時に GitHub issue + 管理者メール、 通知経路は public repo の GitHub Actions で毎時自動監視)。
