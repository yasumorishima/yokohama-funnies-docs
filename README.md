# Yokohama Funnies - 横浜ファニーズ 公式サイト

草野球チーム「横浜ファニーズ」の公式Webアプリケーション。試合結果・予定管理、選手名簿、試合別の打撃・投手成績、出欠管理、オーダー表（打順表）、写真ギャラリーなどを、5段階の権限制御のもとで運用しています。minami-baseball-ob（横浜市立南高校 野球部OB会サイト）を template として 2026-05-10 から構築。

**https://yokohama-funnies.vercel.app/** | ソースコード: private | 公開 cron workflow: [yokohama-funnies-public-cron](https://github.com/yasumorishima/yokohama-funnies-public-cron)

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | **Next.js 15** (App Router / React 19 / Server Components) |
| Language | **TypeScript 5.8** (strict mode) |
| Styling | **Tailwind CSS 4** (PostCSS-first, `@theme` CSS variables) |
| Database | **Supabase** (PostgreSQL + RLS + DB Triggers, Tokyo region / Free plan) |
| Auth | **Supabase Auth** (Google OAuth / SSR cookie pattern)。ログイン失敗時は生の技術エラー（PKCE 等）を表示せずやさしい日本語に変換し、PKCE/verifier 系は1回だけ自動リトライ（ループ防止）で多くは無音でログイン完了 |
| Storage | **Supabase Storage** (photos + scorebook images, client-side resize) |
| Hosting | **Vercel** (Hobby plan, git push auto-deploy) |
| CI/CD | **GitHub Actions** (会員申請 / role sync / monitoring は public repo [yokohama-funnies-public-cron](https://github.com/yasumorishima/yokohama-funnies-public-cron) で実行 — private repo の GHA quota 枯渇対策、GitHub App installation token で private repo に push/PR back) |
| Analytics | **Google Analytics 4** (`G-PJQFLXL3P6`)。オプトアウト方式（既定で計測 ON・同意ポップアップなし、プライバシー / 設定の「計測を止める」トグルで停止可）。**公開ダッシュボード `/insights`**（GA4 Data API・日別 PV / 人気ページ〔試合詳細は実名解決〕/ 参照元 / 端末を inline SVG 表示、ISR 1h、cookieless） |
| Weather | **Open-Meteo API** (free, no API key, 30-min ISR cache + external cron warm) |
| Testing | **Playwright** (e2e: skeleton / navigation / weather / screenshot) |
| GitHub integration | **GitHub App `yokohama-funnies-bot`** (Installation token via `@octokit/auth-app`, PAT-less) |
| External | **Google Apps Script** (Member form dispatch + feedback Gmail notification + hourly heartbeat) |

---

## Architecture

```
                         +-----------------+
                         |   Google Form   |
                         |   (Member)      |
                         +--------+--------+
                                  |
                         Google Apps Script
                                  | HMAC (WEBHOOK_SECRET)
                                  v
                         +-----------------+
                         |  Vercel API     |
                         |  /api/gas-proxy |  (GitHub App yokohama-funnies-bot)
                         +--------+--------+
                                  | repository_dispatch (target: yokohama-funnies-public-cron)
                                  v
     +----------------------------+----------------------------+
     |     GitHub Actions (yokohama-funnies-public-cron, public)|
     |  member-request  sync-roles  health-check (hourly)      |
     +---+---------------------+-------------------------------+
         | App token push      | App token + Supabase REST
         | + PR back to        | + GAS approval-notify
         | private repo        |
     +---v---------------------v---+     +-------------------+
     |         GitHub Repo         |     |   Vercel (CDN)    |
     |  config/members.yml (RBAC)  +---->+   Auto Deploy     |
     |                             |push |                   |
     +-----------------------------+     +--------+----------+
                                                  |
                                         +--------v----------+
                                         |  Next.js 15 App   |
                                         +--------+----------+
                                                  |
                               +------------------+------------------+
                               |                  |                  |
                      +--------v---+    +---------v----+   +---------v---+
                      | Supabase   |    | Supabase     |   | Supabase      |
                      | PostgreSQL |    | Auth (OAuth)  |   | Storage       |
                      | + RLS/Trig |    | Google SSO   |   | photos/       |
                      |            |    | 5-tier RBAC  |   | scorebooks/   |
                      +------------+    +--------------+   +---------------+
```

---

## Project Scale

<!-- stats-start (auto-updated by GitHub Actions) -->
| Metric | Count |
|--------|-------|
| TypeScript/TSX files | <!--stat:ts_files-->146<!--/stat--> |
| Lines of code | <!--stat:loc-->~16900<!--/stat--> |
| Page routes | <!--stat:pages-->43<!--/stat--> |
| API routes | <!--stat:apis-->8<!--/stat--> |
| Reusable components | <!--stat:components-->53<!--/stat--> |
| DB tables (+ history) | <!--stat:tables_main-->21<!--/stat--> + <!--stat:tables_hist-->5<!--/stat--> |
| DB migrations | <!--stat:migrations-->53<!--/stat--> |
| GitHub Actions workflows | <!--stat:workflows-->4<!--/stat--> |
| e2e tests | 18 |
| Hosting cost | ¥0 / month |
<!-- stats-end -->

---

## Key Features

### 5-Tier Role-Based Access Control

Middleware + RLS の2層で認可を実施。全操作を権限に応じて制御。

| Level | Role | Access |
|-------|------|--------|
| 1 | Guest | Public pages |
| 2 | `viewer` | Logged in, awaiting approval |
| 3 | `member` | + Member-only pages (選手成績 / 出欠 / オーダー表 / 会員専用投稿 / スコアブック閲覧) |
| 4 | `editor` | + Content CRUD (8 edit pages + inline editing) |
| 5 | `admin` | + User management, audit logs, trash |

- **Next.js Middleware**: Route-level access control, redirect unauthorized users
- **Supabase RLS**: Row-level policies using `get_user_role()` DB function
- **Component-level**: `useAuth()` hook for conditional UI rendering
- New OAuth signups auto-receive `viewer` via an `auth.users` trigger

### Automated Member Management Pipeline (PR-based)

Google Forms から PR 作成、マージで権限反映まで、**個人情報をGitに一切残さず**に完結する会員管理フロー。minami-baseball-ob と同一 topology。

```
Google Form submit (氏名 + 背番号)
  -> Google Apps Script gas-member-form (HMAC-signed call to Vercel proxy)
    -> Vercel /api/gas-proxy/dispatch (GitHub App yokohama-funnies-bot via Installation token)
      -> GitHub repository_dispatch (target: yokohama-funnies-public-cron)
        -> GitHub Actions member-request.yml: auto-create PR editing config/members.yml
        -> GAS notifyAdmin: Gmail notification to admin
          -> Admin merges PR
            -> GitHub Actions sync-roles.yml (*/5 polling): config/members.yml + config/members/*.yml -> Supabase user_roles
            -> GAS approval-notify: approval email to the applicant
```

- 会員承認 = 管理者が PR をマージするだけ（viewer→member 昇格は冪等同期）
- 編集権限が必要な人は admin が `config/members.yml` の `editor:` に移してマージ（編集権限の要不要は申請者に聞かない方針）
- funnies private repo は GHA quota 枯渇のため、Actions は public repo `yokohama-funnies-public-cron` で実行

### In-App Feedback Form

メンバーが GitHub アカウント不要でフィードバックを送信できる仕組み。サイト内 `/feedback` ページで完結。

```
/feedback page (Google login required)
  -> Next.js API route /api/feedback
    -> GitHub App yokohama-funnies-bot: create GitHub Issue on private repo
    -> GAS Web App doPost (FEEDBACK_GAS_URL): Gmail notification to admin
```

- Google login required (Supabase Auth)
- Issue auto-created on the private repo so the team can triage on GitHub

### Per-Game Stats & Scorebook Images

紙のスコアブックを起点に、試合別の打撃・投手成績を DB 化して集計表示する仕組み。

- **Per-game batting / pitching tables**: `game_player_batting`（1 試合 1 選手 = 1 row、PA/AB/H/2B/3B/HR/RBI/BB+HB/犠/盗/三振）と `game_player_pitching`（IP/R/H/K/BB/W/L）が `/members-only/stats` の primary source。試合別に分解できない年度は `season_aggregates_batting` / `season_aggregates_pitching`（`season_year` + `player_id` 単位のシーズン通算）で補完（2024 シーズン = 打撃 19 名 + 投手 7 名）
- **Aggregate views**: `player_batting_stats`（打率 / 出塁率 / 長打率）+ `player_pitching_stats`（ERA / WHIP / K9）、season filter は client 側
- **Editor 手入力 UI** `/edit/game-stats` (hub) + `/edit/game-stats/[result_id]`: 上半分にスコアブック画像、下半分に打撃 / 投手タブ + 選手 picker + inline input + bulk UPSERT。mobile (≤640px) は 1 選手 = 1 card。data 整合性 guard（`2B+3B+HR > H` や `AB > PA` を throw）
- **Scorebook viewer**: private bucket `scorebooks/<game_date>/*` convention、admin client で list + 1h signed URL、member+ は試合詳細 `/results/[id]` で画像を inline 閲覧。editor は同 page で 90°/180° step 回転（sharp 経由 API `/api/scorebooks/rotate`）+ client upload UI（multi file / 10MB 上限）
- **Auto-rotate on upload**: editor がアップした画像は Storage 直アップ後に `/api/scorebooks/normalize`（sharp）が EXIF 正規化＋縦向きなら 90° 回転で横向き（既存の 3300×2550 に統一）へ自動整正。既に横向きなら skip。撮影向きに依存せず常に横向きで保存される（手動回転は override として存置）
- **Scorebook ingestion**: companion repo `baseball-scorebook-ocr`（private R&D）+ 既存スプレッドシートからの移行
- **Stats page** `/members-only/stats`: 打撃 / 投手タブ + 年度フィルタ（全年 / 2025 / 2024）+ sortable headers。打撃は全選手表示（0 打席含む、休部は grayed-out）、投手は登板選手のみ。12 月の試合は翌年 season 扱い。**規定打席（チーム試合数×1）到達者の打率・OPS（投手は防御率・WHIP）をアクセント色で強調**。試合別データの無い年度（2024）は `season_aggregates_*` の通算値を per-game の年度とシームレスに合算

### Roster & Public ROSTER Section

- **Player roster** `/members-only/players`: 背番号 / ポジション / 打投左右 / status / team_role / 写真 + コメント
- **Guest players** (`players.is_guest`): 名簿 form の「助っ人」 checkbox、ON で `jersey_number=NULL` 強制、`/stats` 累計から除外、名簿末尾 group
- **Public ROSTER (№ 06)**: home top の公開 section。`players_public` view（anon-readable、sensitive 列除外 + active + 非削除 + 助っ人除外）経由で写真 + 背番号 + 役職 + 名前 + ポジション + コメントを grid card 表示
- **対外名 (public_name)**: 選手ごとに公開ページ専用の表示名を設定可能。`players_public` view が `COALESCE(NULLIF(public_name,''), name)` で解決し、設定があれば公開ロスターはその名前、空欄なら本名を表示。会員向けページ（選手名簿 / 成績 / box score）は常に本名

### Attendance Management

- 出欠回答ページ + 試合詳細ページ内の出欠 toggle（○/△/×、member 以上、集計と名簿表示）
- **本格運用前（仮表示）**: 出欠の各入口（出欠一覧 / 予定 / 試合詳細）に「本格運用前（仮表示）」を明示。正式公開は **2026年7月** 予定、当面の実出欠はサークルスクエア
- `attendances` table は `schedule_id + user_id` UNIQUE、集計 view `attendances_with_user`

### Lineup / オーダー表 (打順表)

- 試合詳細ページ内で editor 以上が**打順（守備位置つき）+ 控え + 助っ人**のオーダー表を作成・共有（試合ごと 1 枚、`lineups` の `schedule_id` UNIQUE、`starters` / `bench` は JSONB）
- 打順は可変枠（既定 9、追加 / 削除 / 並べ替え）+ 守備位置（投捕一二三遊左中右指）。控えは別枠。**助っ人は名簿外でも自由入力**（名前 + 背番号を保持）。選手選択は当該試合の**出欠 ○ 回答者を先頭表示**
- **下書き / 公開トグル**（`is_published`）で試合直前まで準備 → 会員に共有。下書きは RLS で会員から不可視
- **共有ボタン**で打順をプレーンテキスト化し `navigator.share`（モバイルは LINE 等の共有シート）で共有

### Content Management (Custom CMS)

外部CMSを使わず、**Supabase + Next.js で構築した独自CMS**。ソフトデリート、変更履歴、監査ログを標準装備。

- **8 editor pages** (3 groups: 試合関連 / チーム情報 / その他): Results, Schedule, Announcements, Media, Game Stats, Manager Comment, Team Message, Account
- **Inline editing**: Edit content directly on detail pages (no page transition)
- **Update tracking**: All tables have `updated_by` (auto-set by DB trigger via `auth.uid()`)
- **Soft delete + 7-day trash**: Auto-purge via scheduled GitHub Actions
- **Change history**: DB triggers auto-save previous versions on UPDATE/DELETE
- **Audit logs**: All privilege changes and deletions are recorded
- **Bidirectional linking**: Schedule <-> Results linked by `schedule_id`, photos shared across both (`photos.schedule_id` ties photos to result-less events such as 練習 / 飲み会)
- **Manager comment**: `/results/[id]` 試合詳細に member 限定で監督コメント本文を inline 表示、editor のみ投稿 / 編集、1 試合 1 件、履歴 trigger 付き
- **Members-only posts** (`members_posts`): 2 categories — 会計記録 / 連絡事項
- **Team message** `/edit/team-message`: editor 以上、home top に DB-backed 表示、履歴保持

### Game Type Taxonomy (草野球向け)

- **Results**: 公式戦 ワイワイリーグ / 公式戦 区民大会 / 練習試合 / その他
- **Schedule**: 上記 + 練習 / 飲み会・懇親会
- **Result filter** (`/results`): すべて / ワイワイ / 区民大会 / 練習試合 / その他
- 試合結果ページに「🏆 区民大会公式 試合結果ページ（横浜南区野球協会）↗」 外部リンク

### イニング別スコアボード (Line Score)

イニングデータを入力した試合は、試合結果一覧 (`/results`) と試合詳細 (`/results/[id]`) で野球中継風のラインスコア (各回得点 + R 合計) を表示。

- 先攻チームを上段 / 後攻チームを下段に配置 (`results.inning_scores` JSONB: `{ is_home, us[], them[] }`)
- 後攻のサヨナラ勝ち・最終回を攻撃しなかった場合 (×) は `x` 表記を自動判定
- 編集フォームから先攻/後攻 + 各回得点をカンマ区切りで入力。未入力の試合は従来どおり合計点 + 勝敗バッジのみ
- チーム名と R (合計) 列は横スクロール時も sticky 固定、モバイル幅でも判読可能

### 会場地図 (Venue Map)

試合詳細 (`/schedule/[id]`) に会場の地図を表示。会場名から Google Maps Embed で地図を描画し、Google Maps / Apple Maps の経路リンクを併設。

- 会場名で地図をセンタリング、ワンタップで経路案内を起動 (モバイルは各アプリで開く)
- 地図キー未設定の環境では地図を省略し、経路リンクのみを表示（機能は劣化せず動作）

### Weather Forecast Integration

Open-Meteo API（無料）を使った球場別天気予報。予定との連動で試合当日の天気をすぐ確認できる。

- **`/weather` page**: 横浜スタジアム（ベイスターズ本拠地）を先頭に並べ替えてデフォルト展開、expandable cards with 3-day forecast + hourly breakdown
- **Schedule integration**: 試合当日の天気を試合詳細・home top カードに表示
- **降水量 (mm)**: 日別カードに「累計 N.Nmm」、時間帯別に毎時 mm 値で常時表示
- **雨アイコン 4 段階**: mm 値で 小雨 / 中雨 / 大雨 / 暴風雨 に分割（daily 5/30/80 mm + hourly 1/5/15 mm 閾値）、確率同値でも強度差が視覚的に分かる
- **WBGT (熱中症指数) warning**: `0.725*T + 0.0368*RH + 0.00364*T*RH - 4.25` で算出、日本スポーツ協会基準で 注意/警戒/厳重警戒/危険 を判定。**判定は current 値ではなく当日 hourly の WBGT ピーク**を使い、朝の時点で午後ピークが来る日も確実に警戒として可視化
- **Open-Meteo API** (free, no key, `wind_speed_unit=ms`, `relative_humidity_2m`), 30-min ISR cache

### Silent-Fail Monitoring

会員申請 / フィードバックの通知経路が静かに止まる事故を防ぐための毎時監視。minami の 1 ヶ月 silent fail 事案の再演防止。

- `yokohama-funnies-public-cron` の `health-check.yml` が毎時 probe を実行: Vercel proxy dispatch / health-check-ack freshness / GAS gas-heartbeat freshness / feedback Web App secret-match
- `check-pipeline-health.py` が member-request / sync-roles の **ワークフロー実行失敗** と sync-roles の **liveness（cron 停止＝6h 以上未実行、GitHub の scheduled cron 間引き ~4h を考慮）** を検出し、異常時は private repo に GitHub issue 自動作成 + GAS Gmail 通知、復旧で自動 close（2026-05-26 追加、minami と監視パリティ）
- GAS `hourlyHealthCheck` time trigger が毎時 heartbeat 送信
- 異常時に admin へ Gmail 警告

---

## Database Design

20 main/utility tables + 5 history tables + views, all protected by Row-Level Security.

```
user_roles ----< results         (author)
    |      ----< schedule        (author)
    |      ----< announcements   (author)
    |      ----< members_posts   (author, member+ read, editor+ write)
    |      ----< bookmarks       (owner, RLS: self-only)
    |
    +-- audit_logs               (auto-recorded by DB triggers)
    +-- *_history (x5)           (auto-saved on UPDATE/DELETE)

players  (背番号 UNIQUE / bats/throws/status / is_guest / photo_path / comment / team_role)
    |  players_public (view: anon-readable roster, sensitive cols filtered)
    +--< game_player_batting     (1 game x 1 player = 1 row, 14 cols)
    +--< game_player_pitching    (1 game x 1 pitcher = 1 row)
    |        player_batting_stats / player_pitching_stats (aggregate views)
    +--< attendances             (○/△/×, schedule_id + user_id UNIQUE)
    at_bats / pitching_logs      (per-PA / per-pitch microdata, reserved)

schedule <---> results           (bidirectional via schedule_id)
photos ----< results | schedule | announcements

Storage buckets:
  photos/        (public, game + player photos, editor+ upload)
  scorebooks/    (private, <YYYY-MM-DD>/*.jpg|png|pdf, editor+ client write, 1h signed URL viewer)
```

### Tables

| Table | Description |
|-------|-------------|
| `user_roles` | Permissions (admin/editor/member/viewer) + display name |
| `players` | Player roster (背番号 UNIQUE, bats/throws/status, is_guest, photo_path, comment, team_role) |
| `game_player_batting` | Per-game batting (1 game x 1 player = 1 row, PA/AB/H/2B/3B/HR/RBI/BB+HB/SF/SB/K) |
| `game_player_pitching` | Per-game pitching (1 game x 1 pitcher = 1 row, IP/R/H/K/BB/W/L) |
| `at_bats` / `pitching_logs` | Per-PA / per-pitch microdata (reserved, currently unused) |
| `attendances` | Attendance (○/△/×, schedule_id + user_id UNIQUE) |
| `lineups` | Batting order sheet (打順表, schedule_id UNIQUE, starters/bench JSONB, draft/publish) |
| `results` | Game results (ワイワイリーグ / 区民大会 / 練習試合 / その他). Soft delete |
| `schedule` | Events (games, practice, social). Soft delete |
| `announcements` | News posts. Soft delete |
| `members_posts` | Members-only posts (2 categories: 会計記録 / 連絡事項). Soft delete |
| `photos` | Photo metadata (Storage integration, FK linkage incl. schedule_id) |
| `videos` | Videos (YouTube embed URL) |
| `audit_logs` | Audit trail (privilege changes, soft deletes via DB trigger) |
| `bookmarks` | User bookmarks (RLS: self-only) |
| `*_history` (x5) | Change history (auto-saved on UPDATE/DELETE via DB trigger) |

### Views

| View | Purpose |
|------|---------|
| `player_batting_stats` | Aggregate batting (打率 / 出塁率 / 長打率) |
| `player_pitching_stats` | Aggregate pitching (ERA / WHIP / K9) |
| `season_aggregates_batting` / `season_aggregates_pitching` | per-game データの無い年度のシーズン通算 (2024) |
| `players_public` | Anon-readable roster (sensitive cols filtered, active + non-deleted + non-guest) |
| `attendances_with_user` | Attendance joined with member display name |
| `*_with_author` | Content joined with author display name + `updated_by` |

### DB Functions & Triggers

| Function | Type | Purpose |
|----------|------|---------|
| `get_user_role()` | RLS | Get current user's role |
| `is_admin()` | RLS | Check if admin |
| `is_editor_or_above()` | RLS | Check if editor+ |
| `is_member_or_above()` | RLS | Check if member+ |
| `handle_new_user()` | Trigger | Auto-insert `user_roles(role='viewer')` on new OAuth signup |
| `log_user_roles_change()` | Trigger | Record role changes to `audit_logs` (with `auth.uid() IS NOT NULL` guard) |
| `log_soft_delete()` | Trigger | Record soft deletes to `audit_logs` |

### Key Design Decisions

- **Soft delete** on all content tables (`deleted_at` column) with 7-day auto-purge
- **DB triggers** for `updated_at`, `updated_by` (server-side via `auth.uid()`), audit logging, and history snapshots
- **`auth.users` trigger** auto-inserts `viewer` on new OAuth signup, with `auth.uid() IS NOT NULL` guard so anon-path signup INSERTs are not blocked by audit-log RLS
- **Editor delete policy**: `game_player_batting` / `game_player_pitching` DELETE は `is_editor_or_above()`（手入力ページで row 削除でき、累計の二重計上を防ぐ）
- **DB functions** for RLS: `get_user_role()`, `is_admin()`, `is_editor_or_above()`, `is_member_or_above()`
- **Versioned migrations** in `supabase/migrations/`

---

## CI/CD & Automation

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| **Member Request PR** (public-cron) | Google Form (GAS → Vercel proxy → `repository_dispatch` to yokohama-funnies-public-cron) | GitHub App installation token を mint → private repo の `config/members.yml` を編集する PR を自動作成、admin へ Gmail 通知 |
| **Sync Member Roles** (public-cron) | `*/5 * * * *` polling | App token で `config/members.yml`（既存ロスター）+ `config/members/<uid>.yml`（申請ごと 1 ファイル）を fetch → union → Supabase `user_roles` に diff 同期（idempotent、昇格検出時のみ GAS Gmail で承認通知）。mass-demote safety guard で誤設定時の会員一斉降格を防止 |
| **Health Check** (public-cron, monitoring) | Hourly | 4 probe（Vercel proxy dispatch / health-check-ack freshness / GAS gas-heartbeat freshness / feedback Web App secret-match。 Google 側 transient 誤発報回避に **3 回リトライ + 単発失敗は warning のみ・secret 不一致のみ hard fail**）、異常時 admin に Gmail |
| **Weather Warm / Schedule Fire** (public-cron) | `*/30` + 週次 | `/weather` の ISR cache を warm + 週次で `/schedule` SSR fire（Supabase auto-pause 回避、anon key 不要） |
| **Update README Stats** (public-cron) | Monthly (1st, JST 09:00) / manual | ファイル数・LOC・ページ数・table 数等を算出し `<!--stat:KEY-->` マーカーを private + docs README に同期 |

All workflows use **minimal `permissions`** (principle of least privilege).

### Google Apps Script Integrations

| Script | Purpose |
|--------|---------|
| `gas-member-form/` | 会員申請 onFormSubmit → Vercel proxy `/api/gas-proxy/dispatch` → PR auto-creation（HMAC `WEBHOOK_SECRET`、no PAT in GAS）。フィードバック通知 `doPost` + `hourlyHealthCheck` heartbeat も兼務 |

---

## Security

| Measure | Implementation |
|---------|---------------|
| **Row-Level Security** | All tables. `members_posts`: member+ read. Private storage bucket with signed URLs |
| **Server-only admin client** | `import "server-only"` prevents client-side import of service_role key |
| **Auth callback validation** | Open redirect prevention in `/auth/callback` |
| **Workflow permissions** | Every GitHub Actions workflow declares minimal permissions |
| **Privacy-first membership** | Personal names never appear in Git history (`config/members.yml` stores UID + 背番号 only) |
| **アクセス解析** | オプトアウト方式（既定 ON・外部送信内容を privacy に開示・利用者はいつでも計測停止可能）。個人を直接特定する情報は送信しない |
| **GitHub App authentication** | `yokohama-funnies-bot` GitHub App の Installation token 方式（classic PAT を保持しない）。token は 1h 自動更新、permission は narrow scope |
| **Secret centralization** | GAS は Vercel proxy に HMAC `FEEDBACK_GAS_SECRET` で authenticate するのみで GitHub 認証情報を保持しない。サーバ側クレデンシャルは Vercel env (Sensitive) に集約 |
| **Audit log tamper resistance** | `audit_logs_insert` RLS policy uses `WITH CHECK (user_id = auth.uid())`。`log_user_roles_change` trigger は `auth.uid() IS NOT NULL` guard を持ち、anon-path の OAuth signup INSERT を audit-log RLS で block しない（migration 047 で「Database error saving new user」を完全 fix） |
| **Source secret leak prevention** | `gitleaks` GHA が main push / PR / 手動実行で全履歴スキャン。スキャンは基本 GitHub Actions 無料枠（`ubuntu-latest`, x86_64）で走り、 無料枠枯渇等で hosted job が起動できない時は RPi5 self-hosted runner（`rpi5-funnies`, arm64）に自動 fallback してスキャンが途切れない（primary は `continue-on-error` + クリーン通過時のみ立つ output で gate するため、 実 secret 検出時も fallback が走り検知を維持）。binary は x86_64/arm64 とも SHA256 検証で取得（supply-chain 防御）。`.gitleaks.toml` は `NEXT_PUBLIC_*` placeholder を allowlist。GitHub native Secret Scanning / Push Protection は free plan の private repo では使えないため gitleaks が唯一の検知層 |
| **Env file 規約** | `.env.production` は `NEXT_PUBLIC_*` のみ（client 公開前提値）。service_role / DB password / admin token 系は GH Actions secrets / Vercel env のみ |
| **Silent-fail monitoring** | 会員申請 + フィードバックの通知経路全段を `health-check.yml` で hourly 検査 + GAS `hourlyHealthCheck` から内側 probe (Vercel heartbeat、 毎時実行をまたいだ連続失敗カウンタ `evaluateProbe_` で debounce = 単発の transient は次の毎時実行で自己回復するため無音、 連続 2 回 ≒2 時間継続で初めて通知し誤発報を抑制)、異常時は admin へ Gmail |
| **Dependency vulnerability management** | 2026-06-10 全数監査 (姉妹サイトと同時): 本番依存パッケージを OSV.dev (Google 脆弱性DB) で照会し、next 15.5.14 の既知脆弱性 14 件 (HIGH: middleware/proxy バイパス GHSA-267c-6grr-h53f / GHSA-26hh-7cqf-hhc6 / GHSA-492v-c6pp-mqqv、DoS、SSRF ほか) を検出 → 即日 next 15.5.18 / ws 8.21.0 へ bump。本サイトは認可を middleware で行うためバイパス系は直撃構成だった (データ自体は RLS の defense-in-depth で保護)。2026-07-05、残存の moderate 3 件 (postcss XSS・js-yaml DoS・brace-expansion DoS。いずれもビルド/開発時のみで実行時露出なし) を `package.json` の `overrides` で patched 版に固定して解消 (`npm audit` = 0 / Dependabot open = 0) |
| **Live-tested access control** | 2026-06-10 実環境検証: SET ROLE anon で user_roles / players は 0 行 (公開テーブルのみ可視)、書き込みポリシーは全て役割ゲート (匿名開放ゼロ)、匿名アクセスで /players・/admin・/scorebooks は 307 → /login、管理系 API は 401。storage は members-docs / scorebooks が private、photos / videos のみ public (設計通り) |
| **Membership pipeline privacy** | 2026-06-11: 会員申請パイプラインから個人名を全面排除。PR タイトル・commit message・公開 Actions ログを背番号表記 (背番号未記入は UID 8桁) に修正し、過去の public run ログ 24 件を削除、既存 PR 9 件の title/body も背番号表記に改名。申請者氏名は Supabase の会員管理 (display_name) のみに保存され Git/公開面に残らない |
| **Feedback image privacy** | 2026-06-11: feedback 添付画像の保存先を public repo から **private Supabase Storage bucket `feedback-images`** + 10 年署名付き URL へ移行。ファイル名はタイムスタンプ+乱数+ASCII 化で投稿者名を含まない。E2E 検証済 (署名 URL 200 / token 無し 400 / public repo への新規追加ゼロ) |

---

## UX & Accessibility

- Mobile-first responsive design (base font 18px, line-height 1.7)
- **PC viewport typography scaling**: `sm:` / `md:` / `lg:` breakpoints で text size を段階的に拡大（mobile/sm の見え方は keep）。公開全 page を PC ブラウザで可読性のあるサイズに調整
- **Font size toggle (文字サイズ切替)**: ヘッダーに「大きく / 標準」トグルを常時表示。`html[data-font-size="large"] { font-size: 112.5% }` で rem ベースの文字・余白・アイコンをサイト全体で一括ズーム。localStorage 永続化 + `<head>` inline script で初回ペイント前に適用（フラッシュ防止）、body は px→rem 化で既定サイズは従来値（mobile 18px / desktop 16px）と厳密一致
- All touch targets >= 44px
- Dark mode with full color sweep (league badge sky-200, tournament badge rose-200, attendance pill emerald/amber/rose-200, calendar 土曜 + 天気 min temp blue-300 で navy bg 上の視認性確保)
- `aria-label` on result badges (Win/Loss/Draw)
- **CONTENTS ナビバンド**: 5 色 card-style + 横スライド snap で 5 セクション（ABOUT / MOMENTS / NEWS / UP NEXT / RECORD）へ即ジャンプ、hint label 2 段表示
- **Skeleton loading**: Suspense-based skeleton UI（layout changes auto-propagate to loading state, verified by Playwright e2e）
- **Unsaved warning**: Click capture intercepts navigation on edit pages
- **Share button**: Web Share API (mobile native share sheet) with LINE fallback (desktop)
- **Google Calendar button**: One-tap calendar registration from schedule detail (URL scheme, no API key)
- **LINE browser support**: Auto-detect LINE in-app browser on login — redirects to external browser for Google OAuth compatibility
- **Safe delete UX**: Delete buttons hidden from list views, accessible only after entering edit mode, always followed by a confirmation modal
- **Scroll-to-top fix**: `window.scrollTo({ behavior: 'auto' })` 明示で PageTransition remount と scroll-behavior:smooth の干渉を解消
- アクセス解析はオプトアウト方式（同意ポップアップ廃止・GA4 は既定で計測・プライバシー / 設定ページから停止可能）
- 公開アクセス解析ページ `/insights`（GA4 Data API・日別 PV / 人気ページ / 参照元 / 端末）
- Error boundaries (`error.tsx` / `global-error.tsx`)

---

## Page Structure

```
Public
  /                        Top page (hero + CONTENTS nav + 横浜スタジアム天気 + roster + news + calendar)
  /about                   About the team
  /results                 Game results (filter: ワイワイ/区民大会/練習試合/その他)
  /results/[id]            Game detail (photos, inline stats, scorebook viewer, manager comment)
  /schedule                Upcoming events
  /schedule/[id]           Event detail (weather forecast + calendar registration)
  /announcements           News
  /announcements/[id]      News detail
  /gallery                 Photo gallery (folder view)
  /league                  League info
  /players/[id]            Player detail
  /search                  Cross-table full-text search
  /weather                 Venue weather forecast (3-day + hourly + WBGT)
  /feedback                Feedback form (Google login required)

Auth / Member
  /login                   Google OAuth login
  /account                 Profile (背番号 input), analytics opt-out
  /insights                Public access-analytics dashboard (GA4 Data API)
  /bookmarks               Saved articles
  /attendances             Attendance response
  /members-only            Hub
  /members-only/players    Player roster
  /members-only/stats      Per-player stats (batting/pitching tabs + season filter + sortable)
  /members-only/posts      Members-only posts (会計記録 / 連絡事項)
  /scorebooks/[result_id]  Scorebook viewer + editor upload

Editor (8 pages)
  /edit/results            Game results CRUD
  /edit/schedule           Events CRUD
  /edit/announce           Announcements CRUD
  /edit/media              Photo management
  /edit/game-stats         Per-game stats input hub + [result_id]
  /edit/manager-comment    Manager comment CRUD
  /edit/team-message       Home team message
  /edit/account            (editor account utilities)

Admin
  /admin                   Dashboard
  /admin/roles             User role management
  /admin/trash             Soft-deleted items (restore/permanent delete)
  /admin/audit             Audit log viewer
  /admin/settings          Site settings (GA4 measurement ID etc.)
```

---

## Design

| Element | Value |
|---------|-------|
| Primary color | Navy `#1e3a8a` (team color) |
| Accent | Orange `#ea580c` |
| Background (light) | White `#ffffff` + section-alt sky-blue `#dbeafe` (blue-100) + border slate `#cbd5e1` |
| Mobile font | 18px / line-height 1.7 |
| Layout | Photo Magazine style — full-width hero (`h-[85vh]`) + bottom-aligned title + editorial `№ 01-05` section labels + 大型 Card (`rounded-3xl border-2 shadow-md`) |
| Fonts | Zen Maru Gothic (Japanese 丸ゴシック) + Fredoka (Latin) |
| GameType badge | ワイワイリーグ = navy (`bg-team/15`) / 区民大会 = rose / その他公式戦 = orange / 練習試合・その他 = gray |
| Loading icon | `public/cap.png` (background 透過済、帽子シルエットのみ回転)、OG image にも同じ cap を中央配置 |
| CSS Architecture | Tailwind CSS 4 `@theme` with CSS variables for light/dark |
| Component library | Custom (no external UI framework) |

---

## Top Page Composition

Hero → トップ通知バー (最新の新着情報1件、押下で #news へジャンプ) → CONTENTS ナビバンド (5 色 card-style + 横スライド snap) → 横浜スタジアム 3 日天気 (M/D + 曜日 + 降水% + 累計 N.Nmm + 雨強度別アイコン、クリックで /weather に遷移) → チームの一言 → № 01 ABOUT US → № 02 MOMENTS (写真) → № 03 NEWS → № 04 UP NEXT (Google Calendar 風 月間ビュー) → № 05 RECORD → № 06 ROSTER (公開選手名簿)

なお試合予定 (一覧・詳細) には試合当日の天気を「試合日」ラベル付きで表示し、試合の 16 日前から出る (Open-Meteo 無料枠上限)。会場座標は球場マスタ (`lib/venues.ts`、本拠地の清水ヶ丘公園を含む) で解決する。
