# クラウドDB（Supabase）セットアップ手順

システムにはクラウド同期機能（v6.2）が組み込み済み。Supabaseのプロジェクトを1つ作り、キーを `index.html` に記入して push すれば、全端末（ドライバーのスマホ・本部PC）でデータが共有される。

## 1. Supabaseプロジェクトの作成（約5分）

1. https://supabase.com/ → 「Start your project」→ GitHubアカウントでサインイン
2. 「New project」→ 名前: `nippo-system`、リージョン: `Northeast Asia (Tokyo)`、DBパスワードは任意（メモ不要）
3. 作成完了まで1〜2分待つ

## 2. テーブルの作成

ダッシュボード左メニュー「SQL Editor」→ 以下を貼り付けて「Run」:

```sql
create table if not exists app_rows (
  tbl text not null,
  id text not null,
  data jsonb not null,
  updated_at timestamptz not null default now(),
  primary key (tbl, id)
);

alter table app_rows enable row level security;

-- PoC用: anonキーでの読み書きを許可（本運用では認証導入を推奨。下記「セキュリティ」参照）
create policy "anon_all" on app_rows
  for all using (true) with check (true);
```

## 3. キーの記入

ダッシュボード「Project Settings」→「API」から以下2つをコピー:

- **Project URL**（例: `https://abcdefgh.supabase.co`）
- **anon public** キー（`eyJ...` で始まる長い文字列）

`index.html` の `CLOUD_CONFIG` に記入:

```js
const CLOUD_CONFIG = {
  url: 'https://abcdefgh.supabase.co',
  anonKey: 'eyJ...',
};
```

コミットして push すれば約1分で全端末に反映される。

## 4. 動作確認

1. https://koyanagisho-crypto.github.io/nippo-system/ を開く → ヘッダーに「☁ 同期済み」と表示されればOK
2. 最初に開いた端末のデータ（車両116台等）が自動でクラウドに移行される
3. 別の端末（スマホ）で開く → 同じデータが見えることを確認
4. スマホで日報を送信 → 本部PCの管理画面に表示される（60秒ごと自動更新／ヘッダーの☁ボタンで即時更新）

## 仕組み

- 保存先はSupabaseの `app_rows` テーブル1本（`tbl`=reports/drivers/vehicles/folders、`data`=JSON）
- 書き込みは差分のみアップロード（last-write-wins）。読み込みは起動時・管理画面表示時・60秒間隔・手動☁ボタン
- 通信断のときは端末に保存され「⚠ 同期エラー」表示、回復後の次の保存/更新で自動再送
- ドライバーの入力中・管理者の編集中は自動更新を止めて上書き事故を防止

## セキュリティ（重要）

- anonキーは公開前提のキーだが、現状のRLSポリシーは全許可のため、**URLとキーを知っていれば誰でも読み書きできる**（キーは公開リポジトリのソースに含まれる）
- PoC段階の割り切り。本運用前に Supabase Auth（ドライバー=匿名ログイン+乗務員コード、管理者=メール+パスワード）とRLS絞り込みを実装すること
- 領収書・OCR画像はbase64でDBに保存している。件数が増えたら Supabase Storage への移行を検討（無料枠: DB 500MB / Storage 1GB）
