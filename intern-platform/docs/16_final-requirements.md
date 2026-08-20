# 16. 最終要件定義・設計書（実装版）— ツナカレインターン

作成日: 2026-08-20 / ステータス: **実装前の最終確認版**
方針: **スピードより精度**。稼働中システム（piecetimes-crm）とのコンフリクトゼロを設計原則とする。

---

## 0. 決定ログ（01〜15の全決定の集約）

| 領域 | 決定 |
|---|---|
| ブランド | サブブランド「ツナカレインターン」 |
| ターゲット | 優秀な低学年一般学生（4類型: 挑戦系/専門系/発信系/未覚醒優等生）。体育会はおまけ |
| モデル | 審査制・招待制タレントプール × 面談レス（非同期AI審査・自動推薦文・剥奪可能な会員資格） |
| USP | 審査1回→書類選考なし・面談確約 / 落ちてもフィードバック / 実績書の自動蓄積 / 新卒まで一気通貫 |
| 募集形態 | 公開ショーウィンドウ（求人閲覧可・応募ボタンなし）× 審査制メンバーシップ |
| 価格 | 採用15万円（30日以内離脱50%返金）+ 新卒転換フィー逓減（6ヶ月未満35万/6〜12ヶ月25万/12ヶ月超15万/卒業時消滅）。単一プラン・事前明示・迂回防止条項 |
| 審査 | 5軸（学歴=表示のみ・実績・課題・行動・志向）。合否はF2+F3+F4。落ちた人全員にフィードバック+再挑戦 |
| 集客 | オファー階段（30秒診断/ガクチカ言語化→審査→活性）× 3層チャネル + 広告調整弁（CPA上限2,000円/申込） |
| 法務 | 職業紹介許可 2026-11-02発行予定。発行前は媒体型運用 |
| ゲート | Gate 3（6ヶ月）: 累計10決定・プール300人・CAC3,000円未満 → 未達でpivot |
| 共通ID | ツナカレインターンの学生基盤が PieceTimes 共通基盤の初代（グリーンフィールド確認済み） |

## 1. システム全体像とコンフリクト回避原則

### 現状の稼働資産（確認済み・変更しない）

| 資産 | 実体 | 本プロジェクトとの関係 |
|---|---|---|
| **piecetimes-crm** | Next.js 15 + Prisma + SQLite。ローカル運用（Vercel設定あり）。営業パイプライン・請求・財務・SNS・メディア管理・Student/University/Organization マスタ（Student には bigFive フィールドあり） | **一切変更しない**。連携は §4 の疎結合インターフェースのみ |
| rugbymania / tsunakare-clubs | 静的メディア（GitHub Pages） | 変更しない。学生向け導線を張るのみ |
| tsunakare-backend / frontend / app / management-frontend | 空リポジトリ | 実装先として利用可能 |

### コンフリクト回避原則
1. **別リポジトリ・別DB・別デプロイ**: ツナカレインターンは新規リポジトリ+Supabase（マネージドPostgres）+Vercelで構築。CRMのコード・DB・運用に触れない
2. **共通IDの実体はリポジトリではなくSupabaseプロジェクト**（Auth+DB）。将来CRMや他アプリが繋ぎたくなったら同じSupabaseプロジェクトを参照する
3. CRMとのデータ連携は一方向・非同期・失敗しても互いに壊れない形（§4）

### リポジトリ配置（提案・要承認）

- **`tsunakare-frontend`** をプロダクトリポジトリとする: Next.js（App Router）+ `supabase/`（migrations・seed・RLSポリシー）+ テストを同居させるモノレポ。1人開発+AI補助で精度を保つには単一リポが最適
- `tsunakare-backend` は将来専用APIが必要になった時のために温存。`tsunakare-management-frontend` の運営管理機能は当面プロダクト内の `/admin`（権限制御）で実装

## 2. アーキテクチャ

```
[公開レイヤー]                 [メンバーレイヤー]              [運営レイヤー]
Next.js SSG/ISR               Next.js (認証必須)             /admin (RBAC)
・LP/求人一覧(応募ボタンなし)   ・マッチ提示/面談確約           ・審査キュー(AI一次+人力監査)
・30秒診断/ガクチカ言語化      ・実績書/フィードバック          ・求人/企業審査
・審査エントリー               ・適性検査(フル版)              ・請求/転換フィー管理
        │                          │                          │
        └────────── Supabase (Auth: LINE/Email, Postgres+RLS, Storage) ──────────┘
                     │                                   │
              LINE Messaging API                  Claude API（審査一次評価・
              (通知・リマインド・診断Bot)           推薦文生成・ガクチカ言語化）
```

- ホスティング: Vercel（プレビュー環境=PRごと、本番=main）。DB: Supabase（無料枠→Pro）
- 決済/請求: 請求書発行SaaS連携（Phase 1。手動発行から開始）
- 監視: Vercel Analytics + Sentry（エラー）+ Supabase ログ

## 3. データモデル（確定版）

15 §2 のスキーマを CRM 実態に合わせて確定:

```
students             id, auth_user_id(UNIQUE), email, line_user_id, crm_student_code(NULL可・CRM Student.code対応),
                     created_at   ← 共通IDの正
student_profiles     student_id, name, kana, university, faculty, grade, admission_year, graduation_year,
                     affiliation_type(club/circle/org/none), affiliation_detail, bio
assessments          id, type('aptitude_full'|'aptitude_short'|'screening_task_*'), version, definition(jsonb)
assessment_results   id, student_id, assessment_id, scores(jsonb), raw_answers(jsonb), taken_at
memberships          student_id, status(applied/passed/rejected/suspended/withdrawn), review_scores(jsonb: F2/F3/F4/F5),
                     reviewed_at, feedback(text), invite_quota, trust_score, suspended_reason
invitations          code, inviter_student_id, used_by_student_id, used_at
achievements         id, student_id, kind(task/internship/review), title, body(jsonb), verified, created_at
companies            id, name, url, industry, size, crm_company_code(NULL可), status(pending/approved/rejected),
                     contract_plan(jsonb: 転換フィー条件・同意日時)
jobs                 id, company_id, title, 構造化フォーマット各項目(目的/業務具体/1日/条件/合う人・合わない人),
                     min_days_per_week, remote, hourly_wage, status(draft/review/published/closed)
matches              id, student_id, job_id, status(proposed/accepted_by_student/interview_set/interviewed/
                     offered/started/declined/expired), ai_recommendation(text), proposed_at, responded_at
placements           id, match_id, started_at, day30_confirmed_at, left_at, refund_status,
                     conversion(内定承諾日, フィー区分, 請求状態)  ← 課金イベントの正
club_units           CRM Organization のミラー(code, name, university, type)。個人ID紐付け用
audit_logs           全ステータス遷移の監査ログ（請求根拠・精度重視の要）
```

- RLS: 学生は自分のデータのみ / 企業は自社求人と提示されたマッチのみ / admin はサービスロール
- **金額・課金条件はコードに埋めず `settings` テーブル+契約スナップショット（companies.contract_plan）に保存**（後の価格変更で過去契約が壊れない）

## 4. 稼働中CRMとの連携インターフェース（疎結合・CRM無変更）

| 連携 | 方向 | 方式 | 時期 |
|---|---|---|---|
| 学生マスタ初期投入 | CRM → platform | CRMのStudent/OrganizationをCSVエクスポート→インポートスクリプト（crm_student_code で紐付け。**照合キーは code、なければ作らない**＝name照合はしない） | Phase 1 |
| 掲載申込→営業管理 | platform → CRM | 新規企業の掲載申込をCSV/JSONで日次エクスポート（CRM側は既存のインポート運用で取り込み。CRM改修なし） | Phase 1 |
| 適性検査結果 | platform 内 | assessment_results に集約。CRM Student.bigFive とのマッピングは適性検査ドキュメント確認後に定義 | Phase 2 |
| 新卒送客 | platform → ツナカレ運用 | membershipsから対象学生リストを運営が抽出（手動→将来自動） | Phase 2 |

## 5. 機能要件（フェーズ別・実装粒度）

### Phase 0（公開サイト+審査+手動マッチング）— 精度優先で6〜8週
- P0-1 公開サイト: LP / 求人一覧・詳細（JobPosting構造化データ・応募ボタンなし・CTA=審査エントリー）/ 参加企業 / 審査の説明 / 記事
- P0-2 30秒志向診断（適性検査ショート版※11ファイル確認後に設問確定）→ 結果画面 → LINE登録導線
- P0-3 ガクチカ言語化ツール（Claude API。出力が審査エントリーの実績申告の下書きになる）
- P0-4 審査エントリー: プロフィール+実績申告+職種別15分課題+非同期ヒアリング。AI一次評価→adminキューで人力確定→合否通知（不合格はフィードバック付き）
- P0-5 企業: 掲載申込フォーム+契約同意（転換フィー条項の明示・同意ログ）→ admin審査 → 求人公開
- P0-6 マッチング（11/2以降）: adminがマッチ提示作成→学生がLINE/メールで受諾→企業へAI推薦文付き送客→面談確約フローのステータス管理
- P0-7 admin: 審査キュー/求人審査/マッチ管理/監査ログ閲覧

### Phase 1（セルフサーブ化+請求）
- 企業ダッシュボード（応募者管理・ステータス更新）/ 稼働開始の両側確認 / 請求生成 / 転換フィー請求（内定承諾報告→逓減計算） / CRM連携（§4）/ 招待制の自動化

### Phase 2（適性検査フル版・スカウト・共通基盤拡張）
- 適性検査フル版（別紙A〜G準拠）/ スコアのマッチング活用 / 企業検索・スカウト（プール500人条件）/ 口コミ（クローズド）

## 6. テスト戦略（「精度重視」の実装）

| レイヤー | 対象 | 手段 |
|---|---|---|
| ユニット | 転換フィー逓減計算・返金計算・審査スコア合成・招待コード・ステータス遷移の全分岐 | Vitest。**課金に関わるロジックはテストファースト** |
| 統合 | RLSポリシー（他人のデータが見えないこと）・API境界 | Supabase local + テストDB |
| E2E | 診断→エントリー→合否通知 / 掲載申込→公開 / マッチ提示→面談確約 の3クリティカルパス | Playwright |
| 契約テスト | 適性検査モジュールのスコアJSONスキーマ | zodスキーマで実行時検証+CI |
| CI | GitHub Actions: lint / typecheck / unit / e2e(smoke) を PRごと。mainへ直pushしない | |
| データ保全 | Supabase自動バックアップ+audit_logs | |

## 7. 実装マイルストーン

| 週 | 内容 | 完了条件 |
|---|---|---|
| W1 | リポジトリ初期化・CI・Supabaseスキーマ+RLS・認証 | migrations+RLSテスト green |
| W2-3 | 公開サイト（LP/求人/構造化データ）+ 診断ショート版 + ガクチカツール | E2E: 診断→LINE導線 |
| W4-5 | 審査エントリー+AI一次評価+adminキュー+合否/フィードバック通知 | E2E: エントリー→合否 |
| W6 | 企業申込+契約同意ログ+求人審査・公開 | E2E: 申込→公開 |
| W7 | マッチ提示・面談確約フロー+監査ログ | E2E: 提示→確約 / 11/2運用リハーサル |
| W8 | 総合テスト・負荷/セキュリティ確認・本番環境・ドメイン設定 | 本番リリース判定 |

## 8. ブロッカーと依頼（実装開始前に必要）

1. **`docs/適性検査/`（11ファイル）が GitHub に未プッシュ**（piecetimes-crm は main のみで該当フォルダなし＝ローカルにのみ存在）。ローカルの crm フォルダで以下を実行してください:
   ```
   git add docs/適性検査 && git commit -m "適性検査ドキュメント追加" && git push
   ```
   → 受領後に P0-2（診断ショート版）と Phase 2（フル版）の仕様を確定します。CRM Student.bigFive との整合もここで判断
2. リポジトリ配置の承認: **tsunakare-frontend をプロダクトリポジトリにする**（§1）。実装は同リポジトリの新ブランチに行い、レビュー用にPRを作成します
3. Supabase / Vercel アカウント（無ければ新規作成の可否。どちらも無料枠で開始）
