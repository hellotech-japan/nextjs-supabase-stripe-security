# Payment セキュリティ診断報告書（サンプル）

> **この報告書について**
> 筆者自身の自作リポジトリ（Payment）を対象に実施した、ソースコード診断（静的診断）の記録を、サンプルとして一部匿名化したものです。対象は自分の資産のみです。攻撃の再現手順は含みません。

## 1. 概要

公開前に対応すべき指摘が2件あります。Next.jsの既知の脆弱性と、注文確定用のDB関数を外部から呼べる可能性の2点です。認証、金額の決定、Webhookの署名検証、秘密情報の管理といった土台の部分は適切に実装されていました。

| 重要度 | 件数 | 内容 |
| --- | --- | --- |
| 高 | 2 | Next.jsの既知の脆弱性、注文確定関数の実行権限 |
| 中 | 7 | 決済APIの回数制限、入力検証、Webhookのエラー処理と状態管理、HTTPヘッダー、関数のsearch\_path、二重契約 |
| 低 | 4 | 管理用クライアントの分離、送信元の確認、DB変更の管理、認証コールバック |

高の2件目は、DBの既定権限によって成立するかどうかが変わります。コードからは判定できないため、7章の確認用SQLの結果で確定させてください。

診断後に高と中の指摘を手元のコードで修正しました。結果は9章にまとめています。DBへの適用と依存関係のインストールは未実施です。

## 2. 診断の範囲

| 項目 | 内容 |
| --- | --- |
| 対象 | 自作リポジトリ（Payment）の手元のクローン、`main` ブランチ、対象コミット（匿名化） |
| 構成 | Next.js 16.3.1、React 19.2.8、@supabase/ssr 0.5.2、@supabase/supabase-js 2.112.3、stripe 16.12.0 |
| 確認日 | 2026年10月12日 |
| 方法 | Git管理下の全ソースとSQLの読解、Gitの全履歴とビルド済みのブラウザ向けファイルに対する秘密情報の検索、`npm audit` |
| 基準 | 「Next.js・Supabase・Stripe構成 セキュリティ診断チェックリスト」 |

**確認していない範囲**

- `.env.local` の中身（存在と、Gitの管理対象外であることだけを確認）
- 稼働中のDBの実際の状態（RLS、権限、関数）。SQLファイルの記述から判断しています
- Supabase、Stripe、GitHub、ホスティングの各管理画面の設定
- 実際にリクエストを送る形での動作確認

この結果は確認時点のコードに対するものであり、問題がないことを保証するものではありません。

## 3. 指摘事項（高）

### 高-1 Next.js 16.3.1 に既知の重大な脆弱性がある

- **該当箇所**：`package.json` 16行目（`"next": "16.3.1"`）
- **起こりうる被害**：公開されている勧告には、認証なしでの遠隔コード実行が3件含まれます。サーバーで任意の処理を実行されると、管理用キーとStripeのシークレットキーが読み取られ、他の対策がすべて無効になります。
- **確認した事実**：`npm audit --omit=dev` の結果、nextに重大3件、高1件、中4件、低1件の勧告が該当しました。主なものは次のとおりです。
  - [Windows上のサーバーでの認証なしの遠隔コード実行](https://github.com/advisories/GHSA-p293-qw3h-jr36)（16.3.3未満）
  - [画像最適化APIでの遠隔コード実行](https://github.com/advisories/GHSA-2xp9-vwfh-vxw4)（16.3.3未満）
  - [next/og での遠隔コード実行](https://github.com/advisories/GHSA-vcvr-r3jv-pc5j)（16.3.6未満）
  - [画像最適化でのサーバーサイドリクエストフォージェリ](https://github.com/advisories/GHSA-cjq9-62q9-8jv4)（16.3.8未満）
- **術、対応**：各勧告がこのアプリの使い方で実際に成立するかは個別に検証していません。間接的な依存であるsharp、source-map-js、qsにも高または中の勧告があります。
- **修正方法**：`npm audit` が示す修正版は16.4.0で、メジャーバージョンの変更はありません。

```bash
npm install next@16.4.0
npm audit fix
npm audit --omit=dev
```

- **再確認の結果**：9章を参照

### 高-2 注文確定用の関数を、支払いなしで外部から呼べる可能性がある

- **該当箇所**：`supabase/schema_cart_atomic.sql` 13〜53行目（`mark_order_paid`）、`supabase/schema_grants.sql` 24行目と51行目、`app/api/payment-intent/route.ts` 203〜206行目
- **起こりうる被害**：支払いをせずに注文を「支払い済み」にでき、在庫も減ります。商品を無料で入手される、在庫をすべてゼロにされる、の両方が成立します。
- **確認した事実**：
  - `mark_order_paid` は `security definer` で定義されており、呼び出した人の権限に関係なく、RLSを無視して `orders` と `products` を更新します。
  - `schema_grants.sql` はテーブルの権限を一度すべて剥奪してから付け直していますが、関数については `service_role` への付与だけで、剥奪がありません。
  - PostgreSQLの標準では、関数の実行権限は作成時に全員（PUBLIC）へ付与されます。同ファイルのコメントは「この環境では所有者にしか付かない」としていますが、これはDBの既定権限の設定に依存します。
  - `/api/payment-intent` はログインなしで呼べ、応答に注文ID（`orderId`）が含まれます。
  - 以上から、実行権限が残っている環境では「注文を作成してIDを受け取る → 公開用キーで関数を呼ぶ」だけで成立します。
- **判定**：コード上は高。成立するかどうかは7章の確認用SQLで確定します。結果が「実行不可」でも、テーブルと同じ方針で明示的に剥奪しておくことを推奨します。
- **修正方法**：`schema_grants.sql` の関数の付与の前に剥奪を追加します。

```sql
revoke execute on function public.mark_order_paid(uuid)
  from public, anon, authenticated;
grant execute on function public.mark_order_paid(uuid) to service_role;
```

- **再確認の結果**：9章を参照

## 4. 指摘事項（中）

### 中-1 決済準備APIに回数制限がなく、ログインなしで呼べる

- **該当箇所**：`app/api/payment-intent/route.ts` 24行目以降
- **起こりうる被害**：呼び出すたびに注文の行とStripeの決済が1件ずつ作られます。大量に呼ばれると、DBに未決済の注文が溜まり、発行された決済を使って盗難カードの有効性を試す用途に悪用されるおそれがあります。
- **確認した事実**：本人確認は任意（142行目で未ログインなら `user_id` を空にして続行）で、回数や頻度を制限する処理がありません。
- **修正方法**：IPアドレス単位の回数制限を入れる。未決済のまま一定時間が過ぎた注文を定期的に無効化する。Stripe側の不正対策（Radar）のルールも合わせて確認する。

### 中-2 決済準備APIの入力検証が不足している

- **該当箇所**：`app/api/payment-intent/route.ts` 36〜79行目
- **起こりうる被害**：想定外の入力でDBやStripeの呼び出しが失敗し、500エラーになります。配送先に極端に長い文字列を保存させることもできます。
- **確認した事実**：数量は1以上の整数であることだけを確認しており、上限がありません。商品の種類数にも上限がありません。配送先の各項目は「空でないこと」だけを確認しており、文字列であることと長さを確認していません。商品IDの形式も確認していません。
- **修正方法**：スキーマ検証（zodなど）を導入し、数量と件数の上限、各文字列の型と最大長、商品IDの形式を定義する。

### 中-3 WebhookがDB更新の失敗を確認せずに成功を返す

- **該当箇所**：`app/api/stripe/webhook/route.ts` 33〜41行目、52〜58行目、90〜93行目、103行目
- **起こりうる被害**：DBの更新が失敗しても、Stripeには受信成功（200）を返します。Stripeは再送しないため、支払ったのに有料プランにならない利用者や、解約したのに有料のままの利用者が発生します。
- **確認した事実**：`profiles` と `orders` の更新結果の `error` を確認している箇所は、注文確定の関数呼び出し（70〜82行目）だけです。
- **修正方法**：各更新の `error` を確認し、失敗した場合は500を返してStripeの再送に任せる。更新対象が0件だった場合も記録する。

### 中-4 Webhookで注文の状態が巻き戻る場合があり、返金と異議申し立てを処理していない

- **該当箇所**：`app/api/stripe/webhook/route.ts` 86〜96行目、98〜100行目
- **起こりうる被害**：1回目の支払いに失敗し、2回目で成功した注文について、失敗の通知が遅れて届くと、支払い済みの注文が「失敗」に上書きされます。また、返金や異議申し立てが起きても注文の状態は変わりません。
- **確認した事実**：`payment_intent.payment_failed` の処理は、現在の状態を条件にせずに `failed` へ更新しています。`charge.refunded` と `charge.dispute.created` は未対応のイベントとして無視されます。
- **修正方法**：失敗への更新に「現在の状態が `pending` の場合のみ」の条件を付ける。返金と異議申し立てのイベントを処理し、注文の状態を更新する。

```ts
await supabase
  .from("orders")
  .update({ status: "failed" })
  .eq("id", orderId)
  .eq("status", "pending");
```

### 中-5 セキュリティ関連のHTTPヘッダーを設定していない

- **該当箇所**：`next.config.js` 2〜4行目
- **起こりうる被害**：他サイトの枠内にマイページを表示し、利用者に気づかれないまま課金登録のボタンを押させる攻撃を防げません。スクリプトの読み込み元の制限もないため、他の問題が見つかった場合の被害が広がりやすくなります。
- **確認した事実**：`headers()` の設定がなく、Content-Security-Policy、`frame-ancestors`、`Strict-Transport-Security`、`X-Content-Type-Options`、`Referrer-Policy` のいずれも出力されません。
- **修正方法**：`next.config.js` に `headers()` を追加する。Content-Security-Policyでは、Stripeの入力部品が使う `js.stripe.com` などを許可する必要があります。

### 中-6 SECURITY DEFINERの関数でsearch\_pathを固定していない

- **該当箇所**：`supabase/schema.sql` 24〜31行目（`handle_new_user`）、`supabase/schema_cart_atomic.sql` 13〜53行目（`mark_order_paid`）
- **起こりうる被害**：所有者の権限で動く関数が、呼び出し側の設定次第で意図しないスキーマのテーブルや関数を参照する余地が残ります。単独での悪用は難しいものの、Supabaseのセキュリティアドバイザーでも警告される項目です。
- **確認した事実**：どちらの関数にも `set search_path` の指定がありません。`mark_order_paid` はテーブル名をスキーマなしで参照しています。
- **修正方法**：

```sql
alter function public.mark_order_paid(uuid) set search_path = public, pg_temp;
alter function public.handle_new_user() set search_path = public, pg_temp;
```

### 中-7 有料プランの二重契約を防いでいない

- **該当箇所**：`app/api/checkout/route.ts` 56〜62行目
- **起こりうる被害**：すでに有料プランの利用者がこのAPIを再度呼ぶと、2つ目の契約が作られ、二重に請求されます。返金対応と信用の問題になります。
- **確認した事実**：決済画面を作成する前に、現在の契約状態を確認していません。画面上はボタンが出し分けられていますが（`app/dashboard/page.tsx` 40行目）、APIは直接呼び出せます。
- **修正方法**：`profiles.subscription_status` が有効な場合は、決済画面を作らずに請求情報の管理画面へ案内する。

## 5. 指摘事項（低）

| 番号 | 件名 | 該当箇所 | 確認した事実 | 修正方法 |
| --- | --- | --- | --- | --- |
| 低-1 | 管理用クライアントがサーバー専用として分離されていない | `lib/supabase/server.ts` 39〜46行目 | 通常のクライアントと同じファイルにあり、`server-only` の指定がありません。現状の呼び出し元はAPIルート3つだけで、ブラウザ向けファイルへの混入はありませんでした | 別ファイル（`lib/supabase/admin.ts`）に分け、先頭に `import "server-only"` を置く |
| 低-2 | 更新系のAPIで送信元を確認していない | `app/api/checkout/route.ts` 6行目、`app/api/stripe/portal/route.ts` 6行目 | Cookieで認証するPOSTで、`Origin` ヘッダーの確認がありません。認証CookieのSameSite属性に防御を依存しています | `Origin` が自サイトと一致しないリクエストを拒否する |
| 低-3 | DBの変更がマイグレーションで管理されていない | `supabase/schema*.sql`（4ファイル） | SQLエディタで順番に手動実行する前提です。流し忘れや順序の誤りで、権限の剥奪が適用されないままになるおそれがあります。サンプル商品の投入もスキーマに含まれています（`schema_cart.sql` 83〜85行目） | `supabase/migrations/` に移し、サンプルデータは `seed.sql` に分ける |
| 低-4 | 認証コールバックが失敗を無視する | `app/api/auth/callback/route.ts` 9〜14行目 | コードの交換に失敗しても、結果を確認せずにマイページへ転送します。被害はありませんが、失敗の記録が残りません | `error` を確認し、失敗時は記録してログイン画面へ戻す |

## 6. 問題がなかった項目

次の項目はコード上で適切に実装されていることを確認しました。

| 領域 | 確認した内容 | 根拠 |
| --- | --- | --- |
| 認証 | サーバー側の本人確認はすべて `getUser()` を使用。クライアントはリクエストごとに生成 | `proxy.ts` 28行目、各APIルート、`app/dashboard/page.tsx` 8行目 |
| 認証 | ログイン後の転送先は固定で、入力から決めていない | `app/api/auth/callback/route.ts` 14行目 |
| 認可 | 4テーブルすべてでRLSを有効化。利用者には参照権限だけを付与し、書き込みはサーバー側に限定 | `supabase/schema_grants.sql` 24〜45行目 |
| 認可 | 利用者が自分でプランを有料に書き換えられる問題は修正済み | 対象コミット（匿名化）（2026年9月23日） |
| 決済 | 金額はDBの価格からサーバー側で再計算。ブラウザから送られた金額は使わない | `app/api/payment-intent/route.ts` 93〜136行目 |
| 決済 | 定期課金の価格はサーバー側の価格IDで指定 | `app/api/checkout/route.ts` 59行目 |
| 決済 | 顧客ポータルは本人の顧客IDでのみ作成 | `app/api/stripe/portal/route.ts` 19〜35行目 |
| Webhook | 加工前の本文で署名を検証し、失敗時は400を返す | `app/api/stripe/webhook/route.ts` 10〜23行目 |
| Webhook | 注文確定と在庫の減算は1つのトランザクションで行い、同じ通知が重複しても二重に減らない | `supabase/schema_cart_atomic.sql` 21〜49行目 |
| カード情報 | カード番号はStripeの入力部品で受け取り、自前のサーバーを通らない | `components/CheckoutForm.tsx` 56行目 |
| 秘密情報 | `.env.local` はGitの管理対象外。履歴にコミットされた環境変数ファイルは見本（`.env.local.example`）だけ | `.gitignore` 3〜4行目、Git履歴 |
| 秘密情報 | Gitの全履歴に、Stripeのキー、Webhookの署名シークレット、JWS、DB接続文字列の形式に一致する文字列は0件 | パターン検索 |
| 秘密情報 | ビルド済みのブラウザ向けファイルに含まれるJWTは公開用（anon）の1件だけ。管理用キーとStripeのシークレットキーは含まれない | `.next/static` の検索 |
| フロントエンド | `dangerouslySetInnerHTML` の使用なし。ブラウザ向けのソースマップの公開なし | ソース検索、`.next/static` |
| 依存関係 | ロックファイルをリポジトリに含めている | `package-lock.json` |

## 7. 未確認の項目と確認手順

コードからは判断できず、管理画面やDBでの確認が必要な項目です。

**最優先：高-2の確定**

SupabaseのSQLエディタで次を実行してください。`anon` または `authenticated` の行が `true` であれば、高-2は成立します。

```sql
select r.rolname,
       has_function_privilege(r.rolname, 'public.mark_order_paid(uuid)', 'execute') as can_execute
from pg_roles r
where r.rolname in ('anon', 'authenticated', 'service_role');
```

**DBの実際の状態**

全テーブルのRLSの有効状態と、残っているポリシーを確認します。修正前の「自分のプロフィールを更新できる」が残っていないことが要点です。

```sql
select tablename, rowsecurity from pg_tables where schemaname = 'public';
select tablename, policyname, cmd, roles from pg_policies where schemaname = 'public';
```

**管理画面で確認する項目**

- [ ] Supabase：セキュリティアドバイザーの指摘内容
- [ ] Supabase：本番のRedirect URLsが完全一致のURLだけになっていること。手元の `supabase/config.toml` は完全一致で設定されていますが、本番の設定は別管理です
- [ ] Supabase：メール送信の回数制限と、CAPTCHAの要否（ログインリンクの送信は誰でも実行できるため）
- [ ] Supabase：DBのバックアップの有効状態と、復元の試行
- [ ] Stripe：Webhookの登録先と署名シークレットが、テスト用と本番用で分かれていること
- [ ] Stripe：受信するイベントの種類が、処理しているものと一致していること
- [ ] ホスティング：環境変数が本番、プレビュー、開発で分かれていること。`NEXT_PUBLIC_` で始まる変数に秘密の値がないこ
- [ ] GitHub、ホスティング、Supabase、Stripe：管理アカウントの多要素認証
- [ ] 利用者Aと利用者Bのテストアカウントで、AからBの注文とプロフィールが読めないことの実機確認

## 8. 修正の優先順位

1. **高-1**：Next.jsを16.4.0へ更新し、`npm audit` で再確認する
2. **高-2 と 中-6**：確認用SQLを実行したうえで、関数の実行権限の剥奪とsearch\_pathの固定をまとめて適用する
3. **中-3 と 中-4**：Webhookのエラー処理と状態の条件付き更新。同じファイルなので一度に直す
4. **中-1 と 中-2**：決済準備APIの回数制限と入力検証
5. **中-7**：二重契約の防止
6. **中-5**：HTTPヘッダーの追加。Stripeの入力部品が動くことを画面で確認しながら調整する
7. **低-1〜低-4**：次回の改修時

修正後は、各指摘の「再確認の結果」を同じ手順で確認して更新します。

## 9. 修正の実施結果

高と中の8件を手元のフォルダで修正しました。7件は修正済み、2件は一部対応です。修正済みのうちDBに関わる2件は、ローカルDBで適用を確認しました。PR #1 のマージ後、本番にもマイグレーションで適用し、同じ状態であることを確認しました。変更は PR #1 としてマージ済みです。

| 番号 | 状態 | 実施した内容 | 再確認の結果 |
| --- | --- | --- | --- |
| 高-1 | 修正済み（ビルドで確認） | `package.json` と `package-lock.json` でNext.jsを16.4.0に更新。sharp、qs、source-map-jsも修正版に更新 | 手元で `npm install` を実行した後、`npm audit` は0件、`npm run build` は成功（2026年10月12日） |
| 高-2 | 修正済み（ローカルDBで確認） | `schema_grants.sql` に、`mark_order_paid` の実行権限を全員から剥奪して `service_role` にだけ付け直す処理を追加 | ローカルDBで、mark\_order\_paid を実行できるのが postgres と service\_role だけで、anon、authenticated、PUBLIC は実行できないことを確認（2026年10月12日）。本番DBでも同じ結果を確認（2026年10月12日） |
| 中-1 | 一部対応 | IPアドレスごとに10分あたり10回までの制限を追加（`lib/rate-limit.ts`） | 11回目が429になることをテストで確認。数えているのはサーバープロセス内だけです |
| 中-2 | 修正済み | 件数、数量、合計金額の上限と、商品IDの形式、配送先の型と長さの検証を追加 | 不正な入力16種類がすべて400になり、注文とStripeの決済が作られないことをテストで確認 |
| 中-3 | 修正済み | DB更新の失敗時に500を返してStripeの再送に任せるよう変更。対象の行がない場合は記録 | Stripeのテストイベントで、本番環境での動作を確認（2026年10月13日） |
| 中-4 | 修正済み | 失敗への更新を `pending` の注文に限定。全額返金と異議申し立てのイベントを処理し、状態 `refunded` と `disputed` を追加 | Mark_order_paid は pending と failed の注文だけを確定するよう変更 | Stripeのテストイベントで、本番環境での動作を確認（2026年10月13日） |
| 中-5 | 一部対応 | `frame-ancestors`、`X-Frame-Options`、`X-Content-Type-Options`、`Referrer-Policy`、`Strict-Transport-Security`、`Permissions-Policy` を追加 | 設定の読み込みを確認。スクリプトの読み込み元の制限は未対応です |
| 中-6 | 修正済み（ローカルDBで確認） | 2つの関数の定義と `schema_grants.sql` に `search_path` の固定を追加 | ローカルDBで、2つの関数の search\_path が public, pg\_temp に固定されていることを確認（2026年10月12日）。本番DBでも同じ結果を確認（2026年10月12日） |
| 中-7 | 修正済み | 契約が残っている利用者は、新しい契約を作らずに請求情報の管理画面へ転送 | 型チェックのみ。画面での確認は未実施 |

低-3は、スキーマの4ファイルを supabase/migrations/ にマイグレーションとして配置しました。元のファイルは削除し、二重管理は解消しました。サンプル商品の投入はマイグレーションに残っており、本番にも2件登録されています（意図とおり。PR #1 に記載）。db reset でマイグレーションだけからDBを作り直し、4件が順に適用され、権限とRLSが手動適用時と同じ状態になることを確認しました（2026年10月12日）。低-1、低-2、低-4は未対応です。

**全体の確認**

- TypeScriptの型チェック（`tsc --noEmit`）は修正の前後とも成功
- `next build` と画面での動作確認は未実施。手元の `node_modules` がWindows用で、こちらの環境では実行できなかったためです

**反映に必要な作業**

- [x] `npm install` を実行し、`npm audit` と `npm run build` を確認する
- [x] マイグレーション（`supabase/migrations/`）をローカル（`db reset`）と本番（`db push`）に適用する
- [x] 7章の確認用SQLをローカルと本番で実行し、`anon` と `authenticated` が `false` になることを確認する
- [x] 本番の `orders` で `service_role` に UPDATE 権限があることを確認する（`true`、2026年10月12日）
- [x] StripeのWebhookの受信イベントに `charge.refunded` と `charge.dispute.created` を追加する
- [x] Stripeのテストイベントで、中-3と中-4の動作を本番で確認する（2026年10月13日）
- [x] 本番の Redirect URLs を完全一致のURLだけに設定する

**残っている課題**

- 中-1：サーバーレスや複数台の構成では、インスタンスごとに別々に数えるため制限が緩くなります。その構成で公開する場合は、共有ストア（Redisなど）での実装が必要です。未決済のまま残った注文の定期的な無効化も未対応です
- 中-5：スクリプトの読み込み元を制限するContent-Security-Policyは、Next.jsのインラインスクリプトへのnonce付与と、Stripeの入力部品の動作確認が必要です
- 中-4：返金時に在庫は自動では戻りません、返品の確認後に手動で戻す運用を前提にしています
