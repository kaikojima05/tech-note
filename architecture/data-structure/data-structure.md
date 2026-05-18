# データ構造から考える Webアプリケーション設計の教科書

## はじめに

「一流のエンジニアはデータ構造から考える」とは、単に配列・木・グラフに詳しいという意味ではありません。Webアプリケーションにおいては、**業務上の意味を持つ情報が、どこで生まれ、どの形で保持され、誰に更新され、どの境界を越え、いつ消えるのかを先に設計する**という意味です。

画面、API、DB、フロントエンドstate、ジョブ、ログ、イベントはすべて「データの置き場所」または「データが通る通路」です。データ構造から考えるとは、これらを別々に場当たり的に作るのではなく、**正本・派生・一時状態・履歴・制約・更新者・ライフサイクル**を一貫して扱うことです。

この文書は、Webアプリケーション開発で設計を始めるときに使える「思考の型」として書かれています。DB設計だけ、TypeScriptの型だけ、DDDの用語だけに閉じず、バックエンド、フロントエンド、API、運用まで接続して考えます。

## 目次

1. [そもそも「データ構造」とは何か](#1-そもそもデータ構造とは何か)
2. [「データ構造から考える」とは具体的に何をすることか](#2-データ構造から考えるとは具体的に何をすることか)
3. [良いデータ構造とは何か](#3-良いデータ構造とは何か)
4. [悪いデータ構造とは何か](#4-悪いデータ構造とは何か)
5. [データ構造が悪いことで起きる被害](#5-データ構造が悪いことで起きる被害)
6. [データ構造を検討するとき、何から考え始めるべきか](#6-データ構造を検討するとき何から考え始めるべきか)
7. [ゴールをどう設定すべきか](#7-ゴールをどう設定すべきか)
8. [どのような点に注意して考えるべきか](#8-どのような点に注意して考えるべきか)
9. [考えるのに不要なこと・後回しでよいこと](#9-考えるのに不要なこと後回しでよいこと)
10. [よくある勘違い](#10-よくある勘違い)
11. [Webアプリケーションでの具体例](#11-webアプリケーションでの具体例)
12. [TypeScriptでの考え方](#12-typescriptでの考え方)
13. [RDB設計での考え方](#13-rdb設計での考え方)
14. [フロントエンドstateでの考え方](#14-フロントエンドstateでの考え方)
15. [API設計での考え方](#15-api設計での考え方)
16. [状態遷移とデータ構造](#16-状態遷移とデータ構造)
17. [データ構造のレビュー観点](#17-データ構造のレビュー観点)
18. [思考の型を作る方法](#18-思考の型を作る方法)
19. [実務での進め方](#19-実務での進め方)
20. [まとめ](#20-まとめ)

---

# 1. そもそも「データ構造」とは何か

## 1.1 アルゴリズムの教科書におけるデータ構造

アルゴリズムの文脈でのデータ構造は、**データを効率よく保存・探索・更新・削除するための形**です。

| データ構造 | 得意なこと | 苦手なこと | Web実務でのイメージ |
|---|---|---|---|
| 配列 | 順序付きの走査、インデックスアクセス | 中間挿入・削除 | 一覧、フォーム項目、明細行 |
| 連想配列 / Map / ハッシュテーブル | キーによる高速取得 | 順序や範囲検索 | IDからEntityを引く、キャッシュ |
| Set | 重複排除、存在確認 | 順序を持つ処理 | 選択済みID、権限集合 |
| Stack | 後入れ先出し | 任意位置アクセス | Undo、パンくず、処理履歴 |
| Queue | 先入れ先出し | 優先度制御 | ジョブキュー、通知送信待ち |
| Tree | 階層表現 | 横断的検索 | カテゴリ、組織、コメントスレッド |
| Graph | 多対多の関係 | 単純な集計 | ユーザー関係、依存関係、権限 |
| Heap / Priority Queue | 優先度順処理 | 全件走査 | 期限順ジョブ、リトライ制御 |

ここで重要なのは、「配列が良い」「Mapが良い」ではなく、**何を頻繁に行うかによって形が変わる**ということです。

- 追加が多いのか
- 検索が多いのか
- 並び順が重要なのか
- 重複してはいけないのか
- 親子関係があるのか
- 多対多の関係があるのか

Webアプリケーション設計でも同じです。保存する情報の意味、更新頻度、参照頻度、関係、制約によって、DB、API、stateの形が変わります。

## 1.2 Webアプリケーション設計におけるデータ構造

Webアプリケーションでは、データ構造という言葉はより広く使われます。

| 場所 | データ構造の例 | 主な関心 |
|---|---|---|
| DB | テーブル、カラム、制約、インデックス | 正本、永続化、整合性、検索性 |
| ドメイン | Entity、Value Object、Aggregate | 業務概念、ルール、不変条件 |
| API | Request DTO、Response DTO、Error DTO | 境界、互換性、公開範囲 |
| フロントエンド | server state、UI state、form state | 画面操作、一時状態、キャッシュ |
| URL | query params、path params | 共有可能な状態、検索条件、ページ位置 |
| キャッシュ | key-value、normalized cache | 再利用、鮮度、無効化 |
| ログ | event log、audit log | 調査、監査、再現性 |
| イベント | domain event、integration event | 非同期連携、履歴、疎結合 |
| ジョブキュー | job payload、retry state | 非同期処理、失敗時復旧 |
| フォーム | 入力値、dirty、touched、validation errors | 入力途中、送信可否、エラー表示 |

たとえば「注文」という概念でも、場所によって形は変わります。

```ts
// DBから取得した永続化モデルに近い形
export type OrderRecord = {
  id: string;
  customer_id: string;
  status: 'draft' | 'confirmed' | 'cancelled';
  total_amount: number;
  created_at: string;
  updated_at: string;
};

// ドメインで扱う業務上の注文
export type Order = {
  id: OrderId;
  customerId: CustomerId;
  status: OrderStatus;
  items: OrderItem[];
  billingAddress: Address;
};

// APIで返す画面向けDTO
export type OrderResponse = {
  id: string;
  statusLabel: string;
  totalAmount: number;
  customer: {
    id: string;
    name: string;
  };
  items: Array<{
    productName: string;
    quantity: number;
    unitPrice: number;
  }>;
};

// フォーム入力中の状態
export type OrderFormState = {
  selectedProductIds: string[];
  quantitiesByProductId: Record<string, number>;
  couponCodeInput: string;
  validationErrors: Record<string, string>;
  isSubmitting: boolean;
};
```

これらを全部同じ型にしてしまうと、責務が混ざります。逆に、何でも別型にしすぎると変換地獄になります。大事なのは、**境界が違えば形が違ってよい。ただし意味と対応関係は明確にする**ことです。

## 1.3 「データ構造」と「データモデル」「スキーマ」「型」「状態」「ドメインモデル」の違い

| 用語 | 意味 | 例 | 注意点 |
|---|---|---|---|
| データ構造 | データの持ち方・関係・操作のされ方を含む広い概念 | 配列、DBテーブル、APIレスポンス、state | 文脈により指す範囲が違う |
| データモデル | ある対象領域をデータとしてどう表すか | 注文モデル、ユーザーモデル | 業務理解が必要 |
| スキーマ | 保存・通信・検証のための構造定義 | SQL DDL、JSON Schema、Zod schema | 制約を機械的に表す |
| 型 | プログラム上で値の形を表すもの | TypeScript type/interface | 型だけでは業務制約は完結しない |
| 状態 | 時間と操作により変化する値 | loading、status、form input | 状態遷移を考える必要がある |
| ドメインモデル | 業務概念とルールを表すモデル | 契約、請求、在庫引当 | DBや画面都合と分ける |

「データ構造から考える」ときは、これらを混同しないことが重要です。

- DBテーブルは永続化の都合を表す
- API DTOは境界を越える公開データを表す
- TypeScript型は実装上の形を表す
- ドメインモデルは業務概念と不変条件を表す
- フロントエンドstateはユーザー操作中の一時状態を表す

## 1.4 実務ポイント

- 「データ構造」はDBだけではなく、API、型、state、イベント、ログまで含む広い言葉として扱う。
- 同じ業務概念でも、保存用、業務処理用、公開用、表示用、入力中の状態では形が違ってよい。
- 形が違う場合は、どの値が同じ意味で、どの値が変換・派生された値なのかを明確にする。
- 最初に「このデータはどこに置かれるのか」ではなく、「このデータは何を意味するのか」を考える。

## 1.5 よくある失敗

- データ構造をDBテーブル設計だけだと思う。
- TypeScriptのinterfaceを作っただけで設計した気になる。
- APIレスポンスの形をそのままDBに保存しようとする。
- 画面に表示しやすい形をそのまま正本にしてしまう。
- DTO、Domain Model、DB Model、View Model、Form Stateを区別せず、1つの巨大な型で済ませる。

---

# 2. 「データ構造から考える」とは具体的に何をすることか

## 2.1 画面や処理からではなく、なぜデータから考えるのか

画面から考えると、最初の実装は速くなります。しかし画面は変わりやすいです。キャンペーン、権限、レスポンシブ対応、管理画面、CSV出力、分析、通知、外部連携などが増えると、画面都合で作ったデータ構造はすぐに苦しくなります。

処理から考える場合も同様です。「このボタンを押したらこのAPIを呼ぶ」から始めると、一見自然ですが、データの正本、状態遷移、履歴、不変条件が後回しになります。その結果、処理ごとに似たような更新ロジックが散らばります。

データから考えるとは、次のような問いを先に立てることです。

- この業務で本当に存在する概念は何か
- その概念はどんな属性を持つのか
- どの属性は必須で、どの属性は後から決まるのか
- どの値が正本なのか
- どの値は計算できるのか
- 誰が、いつ、どの条件で変更できるのか
- 過去の値を残す必要があるのか
- 不可能な状態を作らないためには、どんな制約が必要か

画面は「データの見せ方」です。処理は「データの変え方」です。APIは「データの渡し方」です。DBは「データの保存の仕方」です。根本にあるのは、業務上のデータそのものです。

## 2.2 データのライフサイクルを見る

データのライフサイクルとは、**そのデータが生まれてから消えるまでの流れ**です。

| 観点 | 問い | 例 |
|---|---|---|
| 発生 | どこで生まれるか | ユーザー入力、外部API、バッチ、管理者操作 |
| 検証 | どこで妥当性を確認するか | フロント、API、DB制約、ドメインサービス |
| 永続化 | 保存するか | 注文、請求、監査ログは保存する |
| 更新 | 誰がいつ変えるか | 顧客、管理者、決済Webhook、在庫バッチ |
| 参照 | 誰が見るか | 顧客、管理画面、分析基盤、通知処理 |
| 派生 | 計算できるか | 合計金額、表示ラベル、ステータス文言 |
| 履歴 | 過去を残すか | 契約変更履歴、価格履歴、監査ログ |
| 削除 | いつ消えるか | 退会、保存期間満了、論理削除 |
| 復旧 | 壊れた時に戻せるか | event log、audit log、再計算 |

## 2.3 正本、派生値、一時状態、永続化、再計算、履歴

データ構造設計で最も重要な分類は、次の6つです。

| 分類 | 意味 | 例 | 設計上の注意 |
|---|---|---|---|
| 正本 | その値の最終的な真実 | 注文明細、契約開始日、決済ID | 1つに決める。重複させない |
| 派生値 | 正本から計算できる値 | 合計金額、税込金額、表示ラベル | 保存するなら同期ルールが必要 |
| 一時状態 | 操作中だけ必要な値 | 入力中テキスト、モーダル開閉、選択行 | DBに保存しないことが多い |
| 永続化データ | 長期保存が必要な値 | 注文、請求、ユーザー設定 | 制約と移行を考える |
| 再計算可能データ | 失っても作り直せる値 | 集計キャッシュ、検索インデックス | 無効化と再構築手段が必要 |
| 履歴データ | 過去の事実として残す値 | 契約履歴、価格改定履歴 | 現在値と混ぜない |

悪い例です。

```ts
type Order = {
  items: Array<{ price: number; quantity: number }>;
  totalAmount: number; // itemsから計算できるが、いつ更新されるか不明
};
```

良い例です。

```ts
type Order = {
  items: Array<{ unitPrice: number; quantity: number }>;
};

const calculateTotalAmount = (order: Order): number => {
  return order.items.reduce((sum, item) => sum + item.unitPrice * item.quantity, 0);
};
```

ただし、常に派生値を保存してはいけないわけではありません。大量データの集計、請求確定時の金額、法的な記録などは保存すべきです。その場合は「派生値」ではなく「確定時点の事実」として扱います。

```ts
type Invoice = {
  id: string;
  orderId: string;
  // 注文明細から計算できるように見えても、請求確定時の金額として保存する
  issuedTotalAmount: number;
  issuedAt: string;
};
```

## 2.4 「どこで生まれ、どこで更新され、どこで参照され、どこで消えるか」を書く

設計時には、最低限この表を書きます。

| データ | 発生源 | 正本 | 更新者 | 参照者 | 消える条件 | 履歴 |
|---|---|---|---|---|---|---|
| ユーザー名 | 登録フォーム | users.name | ユーザー本人、管理者 | プロフィール、管理画面 | 退会後匿名化 | 監査ログのみ |
| 注文ステータス | 注文処理 | orders.status | 決済処理、キャンセル処理 | 一覧、通知、分析 | 注文削除不可 | イベントログあり |
| 合計金額 | 明細から計算 | order_items | 注文明細変更 | 詳細、請求 | 注文確定後は請求に保存 | 請求金額は履歴 |
| 検索キーワード | URL query | URL | ユーザー操作 | 一覧画面 | URL変更 | 不要 |
| フォーム入力値 | 入力欄 | local form state | ユーザー操作 | 入力画面 | 送信完了/離脱 | 不要 |

## 2.5 実務ポイント

- 設計開始時に「正本・派生・一時状態・履歴」を分類する。
- 派生値を保存する場合は、保存する理由と同期ルールを明記する。
- 画面単位ではなく、データの発生・更新・参照・削除の流れで考える。
- 「これは誰が更新できるのか」「いつ変更されるのか」を必ず確認する。

## 2.6 よくある失敗

- 合計金額、件数、表示名などの派生値を雑に保存し、同期漏れを起こす。
- 一時的な画面状態をDBに保存してしまう。
- 現在値と履歴を同じカラムで表現しようとする。
- 複数の場所に同じ意味の値を置き、どちらが正しいか分からなくなる。
- 「後で考える」と言って、更新者・更新タイミングを決めない。

---

# 3. 良いデータ構造とは何か

## 3.1 良いデータ構造の条件

良いデータ構造とは、見た目がきれいな構造ではありません。**業務の意味を自然に表し、変更・検証・運用・調査に耐える構造**です。

| 条件 | 良い例 | 悪い例 | なぜ重要か |
|---|---|---|---|
| 変更に強い | 状態をenum/unionで表す | booleanを追加し続ける | 状態追加時の影響範囲が明確になる |
| 意味が明確 | `billingAddress` | `address1` | 何の住所か分かる |
| 責務が分かれている | DomainとDTOを分ける | DBモデルを画面に直返し | 境界ごとの変更に強い |
| 正本と派生が区別される | 明細を正本、合計は計算 | 合計も明細も更新 | 不整合を防ぐ |
| 不整合が起きにくい | DB制約と型で表現 | コメントだけでルールを書く | 実行時にも守れる |
| 業務ルールを自然に表現 | `ContractPeriod` | `startDate`, `endDate`が散在 | ルールを集約できる |
| 必要な制約を表現できる | `NOT NULL`, `CHECK`, union | すべてnullable string | 不可能な値を防ぐ |
| 読みやすい | 用語が業務と一致 | 略語だらけ | チームで理解できる |
| テストしやすい | 純粋関数で計算 | state更新に埋め込む | 仕様を検証しやすい |
| 拡張しやすい | 明細テーブル | `item1`, `item2` | 個数増加に耐える |
| クエリしやすい | 検索対象をカラム化 | JSONに全詰め | 検索・集計・indexが容易 |
| パフォーマンス問題を起こしにくい | 参照頻度に応じてindex | 毎回全件scan | 運用時に破綻しにくい |
| 変換が少ない | 境界ごとのDTOを意識 | DB/UI/APIで意味がズレる | バグの温床を減らす |

## 3.2 変更に強い構造

悪い例です。

```ts
type Order = {
  isPaid: boolean;
  isCancelled: boolean;
  isShipped: boolean;
  isRefunded: boolean;
};
```

この構造では、次のような不可能な状態が表現できます。

- キャンセル済みなのに発送済み
- 返金済みなのに未決済
- 発送済みなのに未決済

良い例です。

```ts
type OrderStatus =
  | 'created'
  | 'payment_pending'
  | 'paid'
  | 'shipping_preparing'
  | 'shipped'
  | 'cancelled'
  | 'refunded';

type Order = {
  id: string;
  status: OrderStatus;
};
```

状態が増えたとき、遷移ルールを考えやすくなります。

## 3.3 意味が明確な構造

悪い例です。

```ts
type User = {
  name: string;
  address: string;
};
```

何の住所か分かりません。自宅住所、配送先住所、請求先住所、勤務先住所のどれでしょうか。

良い例です。

```ts
type Customer = {
  displayName: string;
  defaultShippingAddress?: Address;
  billingAddress?: Address;
};
```

「意味の違い」を名前で表すことは、データ構造設計そのものです。

## 3.4 責務が分かれている構造

悪い例です。

```ts
type User = {
  id: string;
  email: string;
  passwordHash: string;
  role: string;
  lastLoginAt: string | null;
  displayName: string;
};

// このままAPIで返してしまう
```

良い例です。

```ts
type UserRecord = {
  id: string;
  email: string;
  password_hash: string;
  role: 'admin' | 'member';
  last_login_at: string | null;
  display_name: string;
};

type UserProfileResponse = {
  id: string;
  displayName: string;
  role: 'admin' | 'member';
};
```

DBに必要な情報と、APIで公開してよい情報は異なります。

## 3.5 クエリしやすい構造

悪い例です。

```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  payload jsonb NOT NULL
);
```

何でもJSONに入れると、最初は楽ですが、検索、集計、制約、移行が難しくなります。

良い例です。

```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  customer_id uuid NOT NULL REFERENCES customers(id),
  status text NOT NULL CHECK (status IN ('created', 'paid', 'cancelled')),
  ordered_at timestamptz NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_orders_customer_ordered_at
  ON orders (customer_id, ordered_at DESC);
```

検索・集計・制約を前提にするなら、主要な値はカラムとして表現します。

## 3.6 実務ポイント

- 良いデータ構造は、業務ルール、制約、更新頻度、参照頻度を自然に表せる。
- 名前は単なるラベルではなく、意味の境界を表す設計要素である。
- 変更に強い構造は、状態や種類の追加時に修正箇所が予測しやすい。
- クエリ・集計・運用で使う値は、JSONに隠さず構造化する。

## 3.7 よくある失敗

- 「実装が短い」ことを「良い設計」と勘違いする。
- booleanで一時的に済ませ、状態が増えたときに破綻する。
- ドメイン、DB、API、Viewの責務を1つの型に詰め込む。
- 派生値を保存する理由を考えずにカラム化する。
- JSONを「将来拡張しやすい魔法の箱」として使う。

---
# 4. 悪いデータ構造とは何か

## 4.1 悪いデータ構造の典型例

悪いデータ構造は、単に「見た目が汚い」構造ではありません。**意味が曖昧で、制約を表現できず、更新ルールが不明で、将来の変更や調査で人間に負荷をかける構造**です。

| 悪い匂い | 例 | なぜ悪いか | 将来起きる問題 |
|---|---|---|---|
| 名前だけでは意味が分からない | `data`, `value`, `type`, `flag` | 業務上の意味が読み取れない | 修正時に仕様確認が必要になる |
| nullableが多すぎる | `string | null`だらけ | いつnullなのか分からない | null分岐が増え、漏れる |
| 1つのフィールドが複数の意味を持つ | `address`が配送先にも請求先にもなる | 文脈依存になる | 仕様追加で壊れる |
| booleanフラグが増殖する | `isPaid`, `isCancelled`, `isRefunded` | 組み合わせ爆発が起きる | 不可能な状態が生まれる |
| statusが雑 | `status: string` | 値域と遷移が不明 | typo、未対応状態、条件漏れ |
| 正本とコピーが混ざる | `userName`を注文にも保存 | どちらが真実か不明 | 更新同期漏れ |
| 派生値を不用意に保存 | `totalAmount`を明細と別更新 | 再計算値とズレる | 請求・分析の不整合 |
| 配列・JSONに詰め込みすぎる | `settings jsonb`, `items jsonb` | 制約・検索・移行が難しい | indexが効かない、分析不能 |
| DB都合だけでドメインが歪む | 中間テーブルを業務概念として扱う | 業務ルールが散る | 変更に弱い |
| 画面都合だけでAPI/DBが歪む | 一覧画面の列順に合わせたDB | UI変更がDB変更になる | 画面追加で破綻 |
| 同じ意味のデータが複数箇所に存在 | `email`がusersとprofilesにある | 正本が不明 | 差分が発生する |
| 更新タイミングが曖昧 | `lastStatus`がいつ更新か不明 | 処理順に依存する | レースコンディション |
| 履歴・現在値・予定値が混ざる | `price`に現在価格も予約価格も入れる | 時点の意味が消える | 請求や監査で困る |
| とりあえずoptional | `name?: string` | 未設定なのか不要なのか不明 | 分岐とバグが増える |
| とりあえずstring | `amount: string`, `status: string` | 値域・単位・制約がない | 計算や比較で事故る |
| とりあえずJSON | `metadata: any` | 何が入るか誰も知らない | 移行と保守が困難 |

## 4.2 nullableが多すぎる例

悪い例です。

```ts
type Contract = {
  id: string;
  startedAt: string | null;
  endedAt: string | null;
  cancelledAt: string | null;
  trialEndedAt: string | null;
};
```

この構造では、どの組み合わせが正しいのか分かりません。

- `startedAt`がnullで`endedAt`があるのはあり得るか
- `cancelledAt`と`endedAt`は同時に入るのか
- trial中と本契約中をどう判定するのか

良い例です。

```ts
type Contract =
  | {
      status: 'trial';
      trialStartedAt: string;
      trialEndsAt: string;
    }
  | {
      status: 'active';
      startedAt: string;
      currentPeriodEndsAt: string;
    }
  | {
      status: 'cancelled';
      startedAt: string;
      cancelledAt: string;
      endedAt: string;
    };
```

状態ごとに必要なデータが違うなら、状態ごとに型を分けます。

## 4.3 booleanフラグが増殖する例

悪い例です。

```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  is_paid boolean NOT NULL DEFAULT false,
  is_cancelled boolean NOT NULL DEFAULT false,
  is_shipped boolean NOT NULL DEFAULT false,
  is_refunded boolean NOT NULL DEFAULT false
);
```

このテーブルでは、DB上で以下のような状態を防げません。

- `is_cancelled = true` かつ `is_shipped = true`
- `is_refunded = true` かつ `is_paid = false`

良い例です。

```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  status text NOT NULL CHECK (
    status IN ('created', 'payment_pending', 'paid', 'shipping_preparing', 'shipped', 'cancelled', 'refunded')
  )
);
```

さらに状態遷移はアプリケーション層で明示します。

```ts
const allowedTransitions: Record<OrderStatus, OrderStatus[]> = {
  created: ['payment_pending', 'cancelled'],
  payment_pending: ['paid', 'cancelled'],
  paid: ['shipping_preparing', 'refunded'],
  shipping_preparing: ['shipped', 'cancelled'],
  shipped: [],
  cancelled: [],
  refunded: [],
};
```

## 4.4 「とりあえずJSON」の危険

JSONカラムは便利ですが、次の用途に向いていません。

- 頻繁に検索・集計する
- 必須項目や一意制約を守りたい
- 値の変更履歴を追いたい
- 外部キーで参照したい
- BIや障害調査で頻繁に読む

悪い例です。

```sql
CREATE TABLE products (
  id uuid PRIMARY KEY,
  attributes jsonb NOT NULL
);
```

この中に、価格、在庫、色、サイズ、公開状態、カテゴリまで入れると、ほぼDBの強みを捨てることになります。

JSONが向く例です。

- 外部サービスから返る原文payloadの保存
- 表示用の柔軟な追加メタデータ
- 検索・集計しない小さな設定
- 将来スキーマ化する前の一時的な実験。ただし期限を決める

```sql
CREATE TABLE webhook_events (
  id uuid PRIMARY KEY,
  provider text NOT NULL,
  event_type text NOT NULL,
  received_payload jsonb NOT NULL,
  received_at timestamptz NOT NULL DEFAULT now(),
  processed_at timestamptz
);
```

## 4.5 実務ポイント

- 悪いデータ構造は、実装直後ではなく、仕様変更・調査・運用で本性を現す。
- null、boolean、string、JSONは便利だが、意味と制約を消しやすい。
- 現在値、履歴、予定値、入力中の値を同じ場所に混ぜない。
- 「正本はどこか」「いつ更新されるか」が答えられない構造は危険である。

## 4.6 よくある失敗

- `status: string`にして、値域をドキュメントだけで管理する。
- 画面で使うからという理由で、DBにも同じ派生値を保存する。
- DBマイグレーションが面倒だからJSONに入れる。
- null許容にしておけば柔軟だと思う。
- booleanを追加して短期的な修正を繰り返し、状態空間を把握できなくなる。

---

# 5. データ構造が悪いことで起きる被害

## 5.1 実務で起きる具体的な被害

データ構造の悪さは、単独のバグとしてではなく、開発速度、品質、運用、チーム理解にじわじわ影響します。

| 被害 | 実例 | 根本原因 |
|---|---|---|
| バグが増える | キャンセル済み注文に発送通知が送られる | boolean組み合わせで不可能状態を防げない |
| 条件分岐が増える | `if (x && !y || z)`だらけになる | 状態が明示されていない |
| 仕様変更が怖くなる | 新しい契約状態を追加できない | 状態遷移とデータが散在 |
| テストが書きづらい | どのnullパターンをテストすべきか不明 | nullableが無秩序 |
| APIが複雑になる | 画面ごとに似たDTOが乱立する | 正本・View Modelの境界がない |
| state管理が破綻する | 一覧と詳細で同じ値がズレる | server stateを複数箇所にコピー |
| DBマイグレーション困難 | JSON内の古い形式を変換できない | スキーマが明示されていない |
| パフォーマンス問題 | JSONから条件抽出して全件scan | クエリ対象がカラム化されていない |
| データ不整合 | 請求金額と注文明細合計が違う | 派生値の同期ルールがない |
| 集計・分析が困難 | 契約状態が文字列自由入力 | 値域が統一されていない |
| 障害調査が難しい | いつ誰が値を変えたか分からない | 監査ログ・履歴がない |
| 新人が理解できない | `type=3`の意味が不明 | 命名とドキュメント不足 |
| 値の意味が不明になる | `deleted_at`が退会・非表示・停止を兼ねる | 1フィールド複数意味 |

## 5.2 フロントエンドstateが破綻する例

悪い例です。

```ts
const [users, setUsers] = useState<User[]>([]);
const [selectedUser, setSelectedUser] = useState<User | null>(null);

// 一覧のusersを更新しても、selectedUserは古いままになる可能性がある
```

`users`と`selectedUser`が同じユーザー情報を別々に保持しているため、片方だけ更新されるとズレます。

良い例です。

```ts
type UserState = {
  usersById: Record<string, User>;
  userIds: string[];
  selectedUserId: string | null;
};

const selectedUser = state.selectedUserId
  ? state.usersById[state.selectedUserId]
  : null;
```

正本は`usersById`に置き、選択状態はIDだけで表現します。

## 5.3 APIが複雑になる例

悪い例です。

```json
{
  "id": "ord_1",
  "user_name": "田中太郎",
  "user_email": "taro@example.com",
  "product_names": "Book,Pen",
  "paid": true,
  "cancelled": false
}
```

このレスポンスは、DBの一部、画面表示、一時的な文字列加工、状態管理が混ざっています。

良い例です。

```json
{
  "id": "ord_1",
  "status": "paid",
  "customer": {
    "id": "usr_1",
    "displayName": "田中太郎"
  },
  "items": [
    { "productId": "prd_1", "productName": "Book", "quantity": 1 },
    { "productId": "prd_2", "productName": "Pen", "quantity": 2 }
  ]
}
```

構造化すると、クライアント側で扱いやすく、状態や明細の意味も明確になります。

## 5.4 障害調査が難しくなる例

悪い構造では、障害時に以下の質問に答えられません。

- いつ値が変わったのか
- 誰が変えたのか
- API経由か、管理画面か、バッチか、Webhookか
- 変更前の値は何だったのか
- その時点の関連データは何だったのか

監査が必要なデータには、現在値だけでなく変更イベントを残します。

```sql
CREATE TABLE order_status_events (
  id uuid PRIMARY KEY,
  order_id uuid NOT NULL REFERENCES orders(id),
  from_status text,
  to_status text NOT NULL,
  changed_by_type text NOT NULL CHECK (changed_by_type IN ('user', 'admin', 'system', 'webhook')),
  changed_by_id uuid,
  reason text,
  changed_at timestamptz NOT NULL DEFAULT now()
);
```

## 5.5 実務ポイント

- データ構造の問題は、バグ修正、仕様変更、調査、分析、運用のコストとして現れる。
- stateの重複はフロントエンドの不整合を生むため、ID参照や正規化を使う。
- 監査・調査が必要な業務データは、現在値だけでなくイベントや履歴を残す。
- 集計や検索で使うデータは、早めに構造化・制約化する。

## 5.6 よくある失敗

- 「今の画面では困らない」だけで設計を終える。
- 障害調査や分析の観点を後回しにする。
- フロントエンドで同じデータを複数stateにコピーする。
- APIレスポンスに表示文字列と業務値を混ぜる。
- DB制約を入れず、アプリ側のバリデーションだけに頼る。

---

# 6. データ構造を検討するとき、何から考え始めるべきか

## 6.1 実務で使える15ステップ

設計は才能ではなく、問いの順番です。次の流れで考えると、抜け漏れが減ります。

| Step | やること | 問い |
|---:|---|---|
| 1 | ユースケースを洗い出す | 誰が、何のために、どんな操作をするか |
| 2 | 登場する概念・名詞を抽出する | ユーザー、注文、請求、契約、商品などは何か |
| 3 | 概念の責務を考える | その概念は何を知り、何を判断するか |
| 4 | データの正本を決める | 真実はDB、外部サービス、イベント、URLのどこか |
| 5 | ライフサイクルを考える | どこで生まれ、更新され、参照され、消えるか |
| 6 | 状態遷移を考える | どんな状態があり、どの順序で変わるか |
| 7 | 不変条件を考える | 常に守るべきルールは何か |
| 8 | 制約を考える | NOT NULL、unique、check、型で何を防ぐか |
| 9 | 更新頻度・参照頻度を考える | 何が頻繁に読まれ、何が頻繁に変わるか |
| 10 | 履歴が必要か考える | 過去の値、変更理由、変更者を残すか |
| 11 | API境界を考える | 外に出してよい情報、隠すべき情報は何か |
| 12 | DBにどう保存するか考える | 正規化、外部キー、index、履歴テーブルは必要か |
| 13 | フロントエンドでどう持つか考える | server state、form state、URL state、UI stateを分けるか |
| 14 | 型でどう表現するか考える | union、branded type、DTO、View Modelを使うか |
| 15 | 変更に耐えられるか検証する | 状態追加、項目追加、権限追加、外部連携追加に耐えるか |

## 6.2 Step 1: ユースケースを洗い出す

まず画面一覧ではなく、操作と目的を書きます。

| ユースケース | 主体 | 成功条件 | 失敗条件 |
|---|---|---|---|
| 注文を作成する | 顧客 | 注文が作成され、支払い待ちになる | 在庫不足、入力不備 |
| 決済を完了する | 決済Webhook | 注文が支払い済みになる | 金額不一致、重複通知 |
| 注文をキャンセルする | 顧客/管理者 | キャンセル状態になる | 発送済み、返金不可 |
| 注文一覧を見る | 顧客 | 自分の注文だけ見える | 権限なし |

## 6.3 Step 2: 概念・名詞を抽出する

ユースケースから名詞を拾います。

- 顧客
- 注文
- 注文明細
- 商品
- 在庫
- 決済
- 請求
- キャンセル
- 発送
- 通知

ここで重要なのは、すべてをテーブルにすることではありません。**業務上、独立したライフサイクルと責務を持つか**を考えます。

## 6.4 Step 3: 責務を考える

| 概念 | 責務 | 持たない責務 |
|---|---|---|
| 注文 | 顧客が購入する商品のまとまり、注文状態 | 決済処理の詳細 |
| 決済 | 外部決済との取引状態 | 商品明細の意味づけ |
| 請求 | 確定した請求金額と発行履歴 | 入力途中のカート |
| 在庫 | 商品の利用可能数と引当 | 顧客プロフィール |

## 6.5 Step 4: 正本を決める

| データ | 正本 | 理由 |
|---|---|---|
| 商品名 | products.name | 自社が管理する商品情報 |
| 決済状態 | payment provider + payments table | 外部決済が最終事実を持つが、自社でも同期状態を保存 |
| 請求金額 | invoices.issued_total_amount | 発行時点の法的・業務的事実 |
| 検索条件 | URL query | 共有・復元したい画面状態 |
| フォーム入力中の値 | form state | 未確定の一時状態 |

## 6.6 Step 5: ライフサイクルを書く

```text
注文作成
  -> 支払い待ち
  -> 支払い済み
  -> 発送準備中
  -> 発送済み

注文作成
  -> 支払い待ち
  -> キャンセル

支払い済み
  -> 返金済み
```

ライフサイクルを書かずにDBカラムを作ると、状態追加時に破綻しやすくなります。

## 6.7 Step 6〜15: 具体的な問い

| 観点 | 問い |
|---|---|
| 状態遷移 | どの状態からどの状態へ進めるか。戻れるか |
| 不変条件 | 合計金額は0以上か。終了日は開始日以後か |
| 制約 | DBで守るか、アプリで守るか、両方か |
| 更新頻度 | 書き込みが多い値と読み込みが多い値を分けるか |
| 履歴 | 変更前後、変更者、変更理由を残すか |
| API | 内部情報を漏らしていないか。互換性はあるか |
| DB | 外部キー、unique、check、indexは必要か |
| FE state | URL、server cache、local state、form stateのどれか |
| 型 | 不可能な状態を型で表現できないか |
| 変更耐性 | 新状態、新権限、新画面、新外部連携に耐えるか |

## 6.8 実務ポイント

- 設計は「画面→API→DB」ではなく、「ユースケース→概念→正本→ライフサイクル→境界→保存」の順で考える。
- すべてのデータについて、発生源、正本、更新者、参照者、削除条件を表にする。
- 状態があるものは、必ず状態遷移を書いてから型やDBに落とす。
- 初期設計では、完璧な抽象化よりも、意味・責務・制約・正本の明確化を優先する。

## 6.9 よくある失敗

- 画面項目一覧をそのままDBテーブルにする。
- 「必要そうなカラム」を先に並べ、業務概念を後から当てはめる。
- 正本を決めずに、API、DB、stateに同じ値を置く。
- 状態遷移を考えずに`status`だけ作る。
- 履歴の必要性をリリース直前まで考えない。

---
# 7. ゴールをどう設定すべきか

## 7.1 データ構造設計のゴール

データ構造設計のゴールは、完璧に美しいモデルを作ることではありません。実務上のゴールは、**業務上の意味を正しく表し、変更・整合性・運用・理解に耐える形を作ること**です。

| ゴール候補 | それだけをゴールにするとどうなるか | 現実的な位置づけ |
|---|---|---|
| 完璧な正規化 | クエリが複雑になり、画面表示が重くなることがある | 正本の整合性を保つ基本として使う |
| 綺麗なクラス設計 | 実装都合の抽象化に寄りすぎる | 業務ルールを閉じ込める手段として使う |
| DB設計 | API、画面、イベントの都合が抜ける | 永続化と整合性の基盤として扱う |
| 将来変更に耐える | 過剰抽象化になりやすい | 既知の変更可能性に備える |
| 不整合を防ぐ | 柔軟性や移行性が落ちる場合がある | 重要データでは強く優先する |
| 業務概念を正しく表現 | 実装が複雑になる場合がある | 長期運用では最重要 |
| チームが迷わず扱える | 完全性より分かりやすさを優先する場合がある | 実務では非常に重要 |

## 7.2 目指すべき現実的な状態

実務で目指すべき状態は、次のようなものです。

1. **名前から意味が分かる**
2. **正本が明確である**
3. **派生値と一時状態が区別されている**
4. **不可能な状態が作られにくい**
5. **更新者と更新タイミングが明確である**
6. **履歴が必要なものは履歴として残る**
7. **API、DB、UI、ドメインの境界が説明できる**
8. **将来の主要な変更に対して、壊れる箇所が予測できる**
9. **新人が読み、レビューで議論できる**
10. **障害調査で原因を追える**

## 7.3 完璧ではなく、説明可能性を目指す

良い設計は、必ずしも一目で「美しい」と感じるものではありません。むしろ、次の質問に説明できることが重要です。

- なぜこの値はDBに保存しているのか
- なぜこの値は保存せず計算しているのか
- なぜこのテーブルは分かれているのか
- なぜこのAPIはDBと違う形なのか
- なぜこのstateはURLに置いているのか
- なぜこのstatusにはこの値域しかないのか
- なぜこの履歴は残し、この履歴は残さないのか

## 7.4 実務ポイント

- ゴールは「正規化」「DDDっぽさ」「きれいな型」ではなく、業務意味・整合性・変更耐性・チーム理解である。
- 設計の正しさは、説明可能性で測る。
- 完璧な未来予測ではなく、現在分かっている業務ルールと高確率の変更に備える。
- データ構造は、実装者だけでなく、レビュー者、運用者、分析者が読める必要がある。

## 7.5 よくある失敗

- 正規化を目的化し、読みにくく遅い構造にする。
- 「将来使うかも」で汎用化し、現在の仕様が分かりにくくなる。
- DDDの用語を使うことがゴールになり、業務理解が浅いままになる。
- DBだけを整えて、APIやフロントエンドstateが破綻する。
- チームが理解できない高度な抽象化を入れる。

---

# 8. どのような点に注意して考えるべきか

## 8.1 注意点の全体像

データ構造をレビューするときは、次の観点を横断的に見ます。

| 観点 | 見るべきポイント |
|---|---|
| 意味 | 名前から業務上の意味が分かるか。似た概念と区別されているか |
| 責務 | そのデータは何を表すか。何を表さないか |
| 所有者 | 誰・どのシステムがその値を管理するか |
| 正本 | 真実の値はどこにあるか |
| 派生 | 他の値から計算できるか。保存するなら同期ルールはあるか |
| 更新 | 誰が、いつ、どの条件で更新するか |
| 参照 | 誰が、どの頻度で、どの条件で読むか |
| 履歴 | 過去値、変更者、変更理由が必要か |
| 状態遷移 | 有効な状態と遷移が明確か |
| 不変条件 | 常に満たすべきルールは何か |
| nullability | nullは「未設定」「不要」「不明」のどれか |
| 命名 | 業務用語と一致しているか。略語が過剰でないか |
| 型 | 値域、単位、IDの種類を表せているか |
| 制約 | DB制約、型、バリデーションの役割分担は適切か |
| 粒度 | 大きすぎず小さすぎないか |
| 集約 | 一緒に変更されるデータをまとめているか |
| 分割 | ライフサイクルが違うデータを分けているか |
| 境界 | DB/API/UI/Domainの都合が混ざっていないか |
| 永続化 | 保存すべき事実か。一時状態ではないか |
| 一時状態 | UI操作中だけ必要な値を永続化していないか |
| UI都合 | 表示ラベルや開閉状態が正本に混ざっていないか |
| API都合 | 公開用DTOが内部モデルを漏らしていないか |
| DB都合 | join都合で業務概念が歪んでいないか |
| パフォーマンス | 主要クエリにindexがあるか。N+1や全件scanが起きないか |
| 運用 | 手動修正、リカバリ、再実行が可能か |
| 障害調査 | いつ誰が何を変えたか追えるか |
| 監査 | 法務・契約・請求上必要な記録が残るか |
| 移行 | スキーマ変更やデータ移行が現実的か |
| テスト | 正常系、異常系、境界値、状態遷移をテストできるか |

## 8.2 nullabilityの見方

`null`には少なくとも3つの意味が混ざりがちです。

| nullの意味 | 例 | より良い表現 |
|---|---|---|
| まだ未設定 | プロフィール画像未設定 | `profileImageUrl: string | null`でもよいが意味を明記 |
| この状態では不要 | 法人でないため法人番号なし | 状態ごとに型を分ける |
| 不明 | 外部連携で取得不能 | `unknown`相当の状態や取得失敗理由を持つ |

悪い例です。

```ts
type Customer = {
  companyName: string | null;
  personalName: string | null;
};
```

良い例です。

```ts
type Customer =
  | { kind: 'individual'; personalName: string }
  | { kind: 'corporation'; companyName: string; corporateNumber?: string };
```

## 8.3 粒度と集約の見方

粒度が大きすぎると変更に弱くなり、小さすぎると関連が追いにくくなります。

| 悪い粒度 | 問題 | 改善 |
|---|---|---|
| `User`にプロフィール、権限、通知設定、請求先、契約を全部持たせる | 更新頻度・責務が違う | `UserProfile`, `UserRole`, `NotificationPreference`, `BillingAccount`に分ける |
| 住所を`postalCode`, `prefecture`, `city`で各所に散らす | ルールが重複 | `Address` Value Objectにまとめる |
| 注文明細を注文文字列に結合 | 個数、金額、商品IDを扱えない | `order_items`として分ける |

## 8.4 境界の見方

境界ごとに関心が違います。

| 境界 | 関心 | 例 |
|---|---|---|
| DB | 永続化、整合性、検索 | 外部キー、unique、index |
| Domain | 業務ルール、不変条件 | 契約期間、状態遷移 |
| API | 公開範囲、互換性 | DTO、error structure |
| Frontend | 操作性、一時状態、表示 | form state、URL query |
| Event | 時点の事実、非同期連携 | `OrderPaid`, `ContractCancelled` |

## 8.5 実務ポイント

- 注意点は暗記ではなく、レビュー時の観点表として使う。
- null、状態、履歴、正本、更新者は特にバグに直結しやすい。
- DB、API、UI、Domainはそれぞれ関心が違うため、必要に応じて型を分ける。
- 粒度は「一緒に変わるか」「ライフサイクルが同じか」で判断する。

## 8.6 よくある失敗

- `null`を単なる「値がない」とだけ捉える。
- ライフサイクルが違うデータを1つのテーブルや型にまとめる。
- UI都合の表示ラベルをDBの正本にする。
- API DTOとDB Modelを同じにして、内部情報を漏らす。
- 履歴や監査を後から足せると思い込み、現在値しか保存しない。

---

# 9. 考えるのに不要なこと・後回しでよいこと

## 9.1 初期設計で考えすぎると悪くなること

データ構造から考えることは、最初からすべてを厳密に決めることではありません。初期に考えすぎると、かえって複雑になります。

| 後回しでよいこと | なぜ危険か | 初期に考えるべきこと |
|---|---|---|
| 最初から過剰に抽象化 | 実際の差分が見えず、抽象が外れる | 具体的なユースケースと概念 |
| あり得る全未来に備える | 現在の仕様が読みにくくなる | 高確率の変更だけ想定 |
| 使うか分からない汎用化 | 汎用名だらけになる | 業務名で具体的に表す |
| DBパフォーマンスだけを最優先 | 意味が壊れる | 正本・制約・主要クエリを先に見る |
| UI都合だけで決める | 画面変更でデータ構造が壊れる | 業務概念を先に整理 |
| フレームワーク都合を優先 | 技術変更に弱くなる | ドメインと境界を明確にする |
| きれいな名前探しだけに時間を使う | 実体の責務が決まらない | 候補名で進め、責務を明文化 |
| DDD用語に引っ張られすぎる | 用語が目的化する | 業務ルールと不変条件を見る |

## 9.2 初期に必ず考えるべきこと

初期に考えるべきなのは、次のような後から変更しづらいことです。

- 正本はどこか
- 状態の種類と遷移は何か
- 履歴・監査が必要か
- 外部サービスが持つ事実は何か
- 一意制約は何か
- 削除・退会・キャンセル時にどうなるか
- 権限によって見えるデータは変わるか
- 集計や検索で必ず使う軸は何か
- データ移行が必要になりそうか

## 9.3 後からでも比較的直しやすいこと

- 表示順
- 表示ラベル
- UIコンポーネント内の局所state
- APIレスポンスに追加する後方互換フィールド
- index追加。ただしデータ量やロックには注意
- View Modelの小さな変換
- 命名の軽微な改善。ただし公開APIやDBカラム名は重い

## 9.4 実務ポイント

- 初期設計では「正本・状態・履歴・制約・境界」に集中する。
- 汎用化は、複数の具体例から共通性が見えた後に行う。
- パフォーマンスは無視しないが、意味を壊してまで最初から最適化しない。
- DDD用語よりも、業務担当者と同じ言葉で説明できることを優先する。

## 9.5 よくある失敗

- `BaseEntity`, `GenericStatus`, `metadata`など、汎用的すぎる名前で意味を消す。
- 未来の全機能を想像して、現在の実装を複雑にする。
- UIライブラリの都合でデータ構造を決める。
- 速度だけを理由に正本と派生値を混ぜる。
- 「DDDではこう呼ぶらしい」に引っ張られて、実際の業務を見ない。

---

# 10. よくある勘違い

## 10.1 初心者〜中級者がしがちな勘違い

| 勘違い | なぜ誤解か | 正しい考え方 |
|---|---|---|
| データ構造 = DBテーブル設計 | API、state、DTO、イベントにも構造がある | 境界ごとのデータ構造を考える |
| データ構造 = TypeScriptの型 | 型は形を表すが、正本や履歴は表しきれない | 型、DB制約、業務ルールを組み合わせる |
| interfaceを作れば設計完了 | 値の意味、更新者、ライフサイクルが未定 | 型の前に概念と責務を決める |
| 画面に必要な形 = 保存すべき形 | 画面は変わりやすい | 保存形と表示形を分ける |
| APIレスポンスの形 = DBの形 | APIは公開契約、DBは永続化 | DTOを設計する |
| null許容にすれば柔軟 | nullの意味が増え、分岐が増える | 状態ごとの型やNOT NULLを使う |
| JSONにすれば将来拡張しやすい | 制約・検索・移行が難しい | 重要データは構造化する |
| booleanを追加すれば簡単 | 組み合わせ爆発が起こる | 状態として表す |
| statusをstringで持てば十分 | 値域・遷移・typoを防げない | union、CHECK制約、遷移ルールを使う |
| 正規化すれば常に良い | 読み込み性能や複雑性が悪化することがある | 正本は正規化、必要なら読み取り用に非正規化 |
| 非正規化は常に悪い | キャッシュや集計では有効 | 同期ルールと再計算手段を持つ |
| 型があれば不整合は防げる | DB更新、外部連携、並行処理では崩れる | DB制約、トランザクション、監査も必要 |
| バリデーションがあれば雑でよい | バリデーションは入口だけ | 構造自体で不正を作りにくくする |

## 10.2 「柔軟」と「曖昧」は違う

初心者がよく使う柔軟な設計は、実際には曖昧な設計であることが多いです。

悪い例です。

```ts
type FlexibleData = {
  type: string;
  value?: string | number | boolean | object | null;
  metadata?: Record<string, unknown>;
};
```

この構造は何でも入りますが、何が正しいか分かりません。

良い例です。

```ts
type NotificationPreference =
  | { channel: 'email'; email: string; enabled: boolean }
  | { channel: 'slack'; webhookUrl: string; enabled: boolean }
  | { channel: 'sms'; phoneNumber: string; enabled: boolean };
```

これは拡張可能ですが、各状態の意味と必要なデータが明確です。

## 10.3 型とバリデーションの役割分担

| 仕組み | 得意 | 苦手 |
|---|---|---|
| TypeScript型 | 実装時の構造、値域、分岐網羅 | 実行時入力、DB内データ、外部API |
| バリデーション | 外部入力の検査 | 長期的な整合性維持 |
| DB制約 | 永続データの最後の防衛線 | 複雑な業務判断 |
| ドメインロジック | 業務ルール、不変条件 | すべての入口を通らない更新 |
| 監査ログ | 変更追跡、調査 | 不正値の事前防止 |

## 10.4 実務ポイント

- 「柔軟」と言う前に、値域・更新者・制約・検索性を確認する。
- TypeScript型は強力だが、DB制約や実行時バリデーションの代わりではない。
- 正規化・非正規化は善悪ではなく、正本と読み取り最適化の使い分けで考える。
- API、DB、画面の形が同じである必要はない。

## 10.5 よくある失敗

- 「後で何でも入れられるから」とJSONや`Record<string, unknown>`を使う。
- `string`で状態を持ち、typoや未知状態を許す。
- 型定義を作っただけで、状態遷移や不変条件を考えない。
- DB制約を入れず、アプリ側だけで整合性を保とうとする。
- 画面の表示都合を永続化モデルに混ぜる。

---
# 11. Webアプリケーションでの具体例

## 11.1 ユーザー

ユーザーは一見単純ですが、実務では複数の責務が混ざりやすい概念です。

| 概念 | 例 | 分ける理由 |
|---|---|---|
| 認証ユーザー | email、password hash、MFA設定 | セキュリティ責務 |
| プロフィール | 表示名、アイコン、自己紹介 | 表示・編集責務 |
| 権限 | role、permission、所属組織 | 認可責務 |
| 通知設定 | email通知、push通知 | 設定責務 |
| 請求先 | billing account、支払い方法 | 決済責務 |

悪い例です。

```ts
type User = {
  id: string;
  email: string;
  passwordHash: string;
  name: string;
  iconUrl: string | null;
  role: string;
  billingAddress: string | null;
  receiveEmail: boolean;
  receivePush: boolean;
};
```

良い例です。

```ts
type AuthUser = {
  id: UserId;
  email: EmailAddress;
  passwordHash: string;
};

type UserProfile = {
  userId: UserId;
  displayName: string;
  iconUrl?: string;
};

type NotificationPreference = {
  userId: UserId;
  emailEnabled: boolean;
  pushEnabled: boolean;
};
```

## 11.2 契約

契約では、現在状態、期間、更新予定、解約、履歴が重要です。

悪い例です。

```sql
CREATE TABLE contracts (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL,
  status text NOT NULL,
  start_date date,
  end_date date,
  cancel_date date,
  next_plan text
);
```

良い例です。

```sql
CREATE TABLE contracts (
  id uuid PRIMARY KEY,
  customer_id uuid NOT NULL REFERENCES customers(id),
  status text NOT NULL CHECK (status IN ('trial', 'active', 'scheduled_to_cancel', 'ended')),
  started_at timestamptz NOT NULL,
  current_period_start timestamptz NOT NULL,
  current_period_end timestamptz NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE contract_plan_changes (
  id uuid PRIMARY KEY,
  contract_id uuid NOT NULL REFERENCES contracts(id),
  from_plan_id uuid NOT NULL,
  to_plan_id uuid NOT NULL,
  effective_at timestamptz NOT NULL,
  requested_at timestamptz NOT NULL DEFAULT now()
);
```

現在契約と予定変更を分けることで、「現在値」と「未来の予定」が混ざらなくなります。

## 11.3 注文・決済・請求

注文、決済、請求は似ていますが責務が違います。

| 概念 | 役割 | 正本 |
|---|---|---|
| 注文 | 顧客が何を購入したいか | 自社DB |
| 決済 | お金の移動・外部決済状態 | 決済プロバイダ + 自社payments |
| 請求 | 確定した請求事実 | 自社invoices |

悪い例です。

```ts
type Order = {
  id: string;
  items: string;
  amount: number;
  isPaid: boolean;
  invoiceNo?: string;
  paymentProviderResponse?: unknown;
};
```

良い例です。

```ts
type Order = {
  id: OrderId;
  customerId: CustomerId;
  status: 'created' | 'payment_pending' | 'paid' | 'cancelled';
  items: OrderItem[];
};

type Payment = {
  id: PaymentId;
  orderId: OrderId;
  provider: 'stripe' | 'paypal' | 'bank_transfer';
  providerPaymentId: string;
  status: 'requires_action' | 'succeeded' | 'failed' | 'refunded';
  amount: Money;
};

type Invoice = {
  id: InvoiceId;
  orderId: OrderId;
  invoiceNumber: string;
  issuedTotalAmount: Money;
  issuedAt: string;
};
```

## 11.4 商品・在庫

商品と在庫も分けます。商品は「何を売るか」、在庫は「どれだけ利用可能か」です。

```sql
CREATE TABLE products (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  status text NOT NULL CHECK (status IN ('draft', 'published', 'archived'))
);

CREATE TABLE inventory_items (
  product_id uuid PRIMARY KEY REFERENCES products(id),
  quantity_on_hand integer NOT NULL CHECK (quantity_on_hand >= 0),
  quantity_reserved integer NOT NULL CHECK (quantity_reserved >= 0)
);
```

在庫では、単なる数量だけでなく「手元在庫」と「引当済み」を区別します。

```ts
type Inventory = {
  productId: ProductId;
  quantityOnHand: number;
  quantityReserved: number;
};

const availableQuantity = (inventory: Inventory) =>
  inventory.quantityOnHand - inventory.quantityReserved;
```

## 11.5 予約

予約は状態遷移と時間範囲が重要です。

```ts
type Reservation =
  | {
      status: 'requested';
      requestedAt: string;
      slot: TimeRange;
    }
  | {
      status: 'confirmed';
      confirmedAt: string;
      slot: TimeRange;
    }
  | {
      status: 'cancelled';
      cancelledAt: string;
      cancelReason: string;
      slot: TimeRange;
    };
```

DBでは重複予約を防ぐ制約やロックが重要になります。

```sql
CREATE TABLE reservations (
  id uuid PRIMARY KEY,
  resource_id uuid NOT NULL,
  starts_at timestamptz NOT NULL,
  ends_at timestamptz NOT NULL,
  status text NOT NULL CHECK (status IN ('requested', 'confirmed', 'cancelled')),
  CHECK (starts_at < ends_at)
);

CREATE INDEX idx_reservations_resource_time
  ON reservations (resource_id, starts_at, ends_at);
```

## 11.6 通知

通知は「通知すべき事実」と「配信結果」を分けます。

| 概念 | 例 | 注意 |
|---|---|---|
| 通知イベント | 注文完了、契約更新、パスワード変更 | 何が起きたか |
| 通知メッセージ | 件名、本文、宛先 | 何を送るか |
| 配信試行 | email送信、push送信、retry | 送れたか |

```sql
CREATE TABLE notification_messages (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL,
  type text NOT NULL,
  title text NOT NULL,
  body text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE notification_deliveries (
  id uuid PRIMARY KEY,
  message_id uuid NOT NULL REFERENCES notification_messages(id),
  channel text NOT NULL CHECK (channel IN ('email', 'push', 'slack')),
  status text NOT NULL CHECK (status IN ('pending', 'sent', 'failed')),
  attempted_at timestamptz,
  error_message text
);
```

## 11.7 権限

権限はroleだけで雑に済ませると壊れやすくなります。

悪い例です。

```ts
type User = {
  id: string;
  role: 'admin' | 'user';
};
```

良い例です。

```ts
type Permission =
  | 'order:read'
  | 'order:write'
  | 'invoice:read'
  | 'user:invite'
  | 'admin:access';

type Role = {
  id: string;
  name: string;
  permissions: Permission[];
};
```

権限が組織、プロジェクト、テナントに紐づく場合は、スコープもデータ構造に入れます。

```ts
type Membership = {
  userId: string;
  organizationId: string;
  roleId: string;
};
```

## 11.8 ステータス管理

ステータスは単なる表示値ではなく、**許可される操作を決めるデータ**です。

```ts
type InvoiceStatus = 'draft' | 'issued' | 'paid' | 'void';

const canEditInvoice = (status: InvoiceStatus): boolean => {
  return status === 'draft';
};
```

`statusLabel`を正本にせず、正本は機械的な値、表示は派生にします。

```ts
const invoiceStatusLabels: Record<InvoiceStatus, string> = {
  draft: '下書き',
  issued: '発行済み',
  paid: '支払い済み',
  void: '無効',
};
```

## 11.9 フォーム入力

フォームは未確定の一時状態です。DBの型と同じにしない方がよいことが多いです。

```ts
type ProductFormState = {
  nameInput: string;
  priceInput: string; // 入力中は空文字や全角数字があり得る
  categoryIdInput: string | null;
  errors: Record<string, string>;
};

type ProductCreateRequest = {
  name: string;
  price: number;
  categoryId: string;
};
```

フォームでは「入力途中」を表すため、Domain Modelより緩い形になります。送信時にRequest DTOへ変換します。

## 11.10 検索条件・一覧画面

検索条件はURLに置くと、共有、戻る、再読み込みに強くなります。

```ts
type OrderSearchParams = {
  q?: string;
  status?: OrderStatus;
  page: number;
  sort: 'ordered_at_desc' | 'ordered_at_asc';
};
```

APIではページネーションも構造化します。

```json
{
  "items": [
    { "id": "ord_1", "status": "paid" }
  ],
  "pageInfo": {
    "page": 1,
    "pageSize": 20,
    "totalCount": 128
  }
}
```

## 11.11 詳細画面・編集画面・履歴表示・管理画面

| 画面 | データ構造の注意 |
|---|---|
| 詳細画面 | 正本に近い情報と表示用派生値を分ける |
| 編集画面 | 保存済みデータと入力中フォームを分ける |
| 履歴表示 | 現在値ではなく、時点ごとの事実を表示する |
| 管理画面 | 権限に応じて見える項目・操作可能項目を分ける |
| 一覧画面 | 軽量なsummary DTOを返す。詳細DTOをそのまま並べない |

```ts
type OrderListItemResponse = {
  id: string;
  status: OrderStatus;
  customerName: string;
  orderedAt: string;
  totalAmount: number;
};

type OrderDetailResponse = OrderListItemResponse & {
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
  }>;
  statusHistory: Array<{
    from: OrderStatus | null;
    to: OrderStatus;
    changedAt: string;
  }>;
};
```

## 11.12 実務ポイント

- ユーザー、注文、決済、請求、在庫、通知、権限は、似た名前でも責務が違う。
- 一覧DTO、詳細DTO、編集フォーム、履歴表示は同じ構造にしなくてよい。
- 検索条件はURL、入力途中はform state、確定値はDB、表示派生値はView Modelというように置き場所を分ける。
- 履歴表示には、現在値ではなくイベントや履歴テーブルが必要になる。

## 11.13 よくある失敗

- 注文、決済、請求を1つの巨大テーブルにまとめる。
- ユーザーにプロフィール、認証、権限、請求先、通知設定を全部持たせる。
- 一覧画面のためだけに詳細情報を大量に返す。
- フォーム状態をDomain Modelと同じ型にして、入力途中の値を表現できなくなる。
- 管理画面だけの表示項目をDBの正本にしてしまう。

---
# 12. TypeScriptでの考え方

## 12.1 TypeScriptの型は「仕様の一部」

TypeScriptの型は、単なる補完のための道具ではありません。値域、責務、境界、状態遷移を表す設計要素です。ただし、TypeScript型だけでDBや外部入力の整合性は守れません。実行時バリデーション、DB制約、ドメインロジックと組み合わせます。

## 12.2 typeとinterfaceの使い分け

| 使う場面 | type | interface |
|---|---|---|
| union型 | 得意 | 不可 |
| primitiveの別名 | 得意 | 不可 |
| mapped type / conditional type | 得意 | 不可 |
| オブジェクト形状 | どちらでも可 | 拡張される公開契約に向く |
| class implements | 可 | 慣用的に向く |

実務では、DTOやドメイン型は`type`で統一しても問題ありません。拡張可能なライブラリAPIやclass実装では`interface`が向くことがあります。

```ts
type OrderStatus = 'created' | 'paid' | 'cancelled';

type Order = {
  id: OrderId;
  status: OrderStatus;
};

interface Logger {
  info(message: string): void;
  error(message: string): void;
}
```

## 12.3 union型とdiscriminated union

状態ごとに持つデータが違う場合は、discriminated unionが強力です。

悪い例です。

```ts
type Payment = {
  status: 'pending' | 'succeeded' | 'failed';
  paidAt?: string;
  failureReason?: string;
};
```

良い例です。

```ts
type Payment =
  | { status: 'pending'; createdAt: string }
  | { status: 'succeeded'; paidAt: string; providerTransactionId: string }
  | { status: 'failed'; failedAt: string; failureReason: string };

function renderPayment(payment: Payment): string {
  switch (payment.status) {
    case 'pending':
      return '支払い待ち';
    case 'succeeded':
      return `支払い完了: ${payment.paidAt}`;
    case 'failed':
      return `失敗: ${payment.failureReason}`;
  }
}
```

## 12.4 enumを使うべきか

TypeScriptでは、`enum`よりも文字列unionを使うことが多いです。

```ts
type OrderStatus = 'created' | 'paid' | 'cancelled';
```

文字列unionは、実行時の余計なオブジェクトを作らず、APIの値とも対応しやすいです。ただし、既存コードや外部仕様で`enum`が有効な場面もあります。

```ts
enum LegacyOrderStatus {
  Created = 'created',
  Paid = 'paid',
  Cancelled = 'cancelled',
}
```

## 12.5 nullableとoptionalの扱い

| 表現 | 意味 | 注意 |
|---|---|---|
| `field?: T` | フィールド自体が存在しない可能性 | PATCHやフォーム初期値で使うことがある |
| `field: T | null` | フィールドは存在するが値がない | APIレスポンスでは意味を明確にする |
| `field: T | undefined` | JS上の未定義 | 外部公開DTOでは避けることが多い |

悪い例です。

```ts
type User = {
  name?: string | null;
};
```

`undefined`と`null`の両方を許すと、意味が曖昧になります。

良い例です。

```ts
type UserProfile = {
  displayName: string;
  avatarUrl: string | null; // 未設定ならnull。キーは必ず返す
};
```

PATCHでは、未指定とnull指定を分けます。

```ts
type UpdateProfileRequest = {
  displayName?: string;       // 指定があれば変更
  avatarUrl?: string | null;  // nullなら削除
};
```

## 12.6 branded type

IDや金額の取り違えを防ぐには、branded typeが有効です。

```ts
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
type Yen = Brand<number, 'Yen'>;

function getOrder(orderId: OrderId) {
  // ...
}

const userId = 'usr_1' as UserId;
// getOrder(userId); // 型エラーにできる
```

## 12.7 Value Object的な型

Value Objectは、値とルールをまとめる考え方です。

```ts
type Money = {
  amount: number;
  currency: 'JPY' | 'USD';
};

function addMoney(a: Money, b: Money): Money {
  if (a.currency !== b.currency) {
    throw new Error('currency mismatch');
  }
  return { amount: a.amount + b.amount, currency: a.currency };
}
```

金額を単なる`number`にすると、通貨や単位が消えます。

## 12.8 DTO、Domain Model、Form State、View Model、DB Model

| 型 | 役割 | 例 |
|---|---|---|
| DB Model | DB行に近い形 | snake_case、nullable、timestamp |
| Domain Model | 業務ルールを表す | Value Object、状態遷移 |
| API DTO | 境界を越える形 | JSON、後方互換、公開範囲 |
| Form State | 入力途中の形 | 文字列入力、dirty、error |
| View Model | 表示しやすい形 | label、formatted value |

```ts
type ProductRecord = {
  id: string;
  price_amount: number;
  price_currency: string;
};

type Product = {
  id: ProductId;
  price: Money;
};

type ProductResponse = {
  id: string;
  price: { amount: number; currency: string };
  priceLabel: string;
};

type ProductFormState = {
  priceInput: string;
  currencyInput: 'JPY' | 'USD';
};
```

## 12.9 型を共通化しすぎる危険性

悪い例です。

```ts
// DB、API、画面、フォームで全部使う
export type User = {
  id: string;
  email: string;
  passwordHash?: string;
  displayName?: string;
  createdAt?: string;
};
```

良い例です。

```ts
type UserRecord = {
  id: string;
  email: string;
  password_hash: string;
  display_name: string;
  created_at: string;
};

type UserResponse = {
  id: string;
  email: string;
  displayName: string;
};

type UserEditFormState = {
  displayNameInput: string;
  error?: string;
};
```

共通化すべきなのは「同じ意味の値」であって、「たまたま形が似ている値」ではありません。

## 12.10 実務ポイント

- TypeScript型は、状態・値域・境界を表す設計道具として使う。
- 状態ごとに必要な値が違うなら、optionalの羅列ではなくdiscriminated unionを検討する。
- ID、金額、日付範囲など取り違えやすい値はValue Objectやbranded typeで守る。
- DB Model、Domain Model、DTO、Form State、View Modelは、必要に応じて分ける。

## 12.11 よくある失敗

- `any`や`Record<string, unknown>`で設計を先送りする。
- `type`と`interface`の宗教論に時間を使い、値の意味を考えない。
- nullableとoptionalを混ぜて、未指定・削除・未設定を区別できなくする。
- API DTOとDomain Modelを共通化しすぎる。
- 型があるからDB制約や実行時バリデーションは不要だと思う。

---

# 13. RDB設計での考え方

## 13.1 RDBは「正本」と「制約」の場所

RDBは、単にデータを置く箱ではありません。複数の処理、複数のアプリケーション、将来の運用から見たときの**最後の整合性防衛線**です。

## 13.2 テーブル設計の基本

| 要素 | 考えること |
|---|---|
| テーブル | 独立したライフサイクルを持つ概念か |
| 主キー | 安定して一意か。外部公開IDと内部IDを分けるか |
| 外部キー | 参照整合性をDBで守るか |
| unique制約 | 業務上一意なものは何か |
| check制約 | 値域、正数、状態、日付関係を守るか |
| NOT NULL | 必須値をDBでも守るか |
| index | 主要クエリ、JOIN、ソート、絞り込みに必要か |
| 正規化 | 正本の重複を避けるか |
| 非正規化 | 読み取り性能や集計のために持つか |

## 13.3 良いスキーマ例

```sql
CREATE TABLE customers (
  id uuid PRIMARY KEY,
  email text NOT NULL UNIQUE,
  display_name text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id uuid PRIMARY KEY,
  customer_id uuid NOT NULL REFERENCES customers(id),
  status text NOT NULL CHECK (status IN ('created', 'payment_pending', 'paid', 'cancelled')),
  ordered_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  id uuid PRIMARY KEY,
  order_id uuid NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id uuid NOT NULL REFERENCES products(id),
  quantity integer NOT NULL CHECK (quantity > 0),
  unit_price_amount integer NOT NULL CHECK (unit_price_amount >= 0),
  unit_price_currency text NOT NULL CHECK (unit_price_currency IN ('JPY', 'USD'))
);

CREATE INDEX idx_orders_customer_created_at ON orders (customer_id, created_at DESC);
CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

## 13.4 履歴テーブル

履歴が必要な場合、現在値のテーブルに過去値を詰め込まないようにします。

```sql
CREATE TABLE contract_status_history (
  id uuid PRIMARY KEY,
  contract_id uuid NOT NULL REFERENCES contracts(id),
  from_status text,
  to_status text NOT NULL,
  changed_by uuid,
  reason text,
  changed_at timestamptz NOT NULL DEFAULT now()
);
```

履歴テーブルは、監査・障害調査・時系列分析に役立ちます。

## 13.5 中間テーブル

多対多は中間テーブルで表します。

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY,
  email text NOT NULL UNIQUE
);

CREATE TABLE organizations (
  id uuid PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE organization_memberships (
  user_id uuid NOT NULL REFERENCES users(id),
  organization_id uuid NOT NULL REFERENCES organizations(id),
  role text NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
  joined_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, organization_id)
);
```

## 13.6 statusカラム

statusカラムは、値域だけでなく遷移ルールも必要です。

```sql
CREATE TABLE invoices (
  id uuid PRIMARY KEY,
  status text NOT NULL CHECK (status IN ('draft', 'issued', 'paid', 'void')),
  issued_at timestamptz,
  paid_at timestamptz,
  CHECK (
    (status = 'draft' AND issued_at IS NULL AND paid_at IS NULL)
    OR (status = 'issued' AND issued_at IS NOT NULL AND paid_at IS NULL)
    OR (status = 'paid' AND issued_at IS NOT NULL AND paid_at IS NOT NULL)
    OR (status = 'void')
  )
);
```

複雑すぎるCHECKは保守が難しくなるため、アプリケーション側のドメインロジックと役割分担します。

## 13.7 JSONカラム

JSONカラムを使う場合は、用途を限定します。

| 向く | 向かない |
|---|---|
| 外部payloadの原文保存 | 検索軸 |
| 低頻度の任意メタデータ | 一意制約が必要な値 |
| 一時的な実験項目 | 外部キーが必要な値 |
| 表示補助 | 集計対象 |

## 13.8 論理削除

論理削除は便利ですが、意味を分ける必要があります。

| 状態 | 意味 |
|---|---|
| deleted | データとして削除扱い |
| archived | 表示しないが履歴として残す |
| suspended | 一時停止 |
| cancelled | 契約・注文がキャンセル |

これらを全部`deleted_at`で表すと、意味が壊れます。

## 13.9 監査ログと集計用テーブル

監査ログは「誰が何を変えたか」を残します。

```sql
CREATE TABLE audit_logs (
  id uuid PRIMARY KEY,
  actor_type text NOT NULL CHECK (actor_type IN ('user', 'admin', 'system')),
  actor_id uuid,
  action text NOT NULL,
  target_type text NOT NULL,
  target_id uuid NOT NULL,
  before_data jsonb,
  after_data jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

集計用テーブルは非正規化ですが、再計算手段を持つことが重要です。

```sql
CREATE TABLE daily_sales_summaries (
  sales_date date PRIMARY KEY,
  total_amount integer NOT NULL,
  order_count integer NOT NULL,
  recalculated_at timestamptz NOT NULL
);
```

## 13.10 マイグレーションしやすさ

良いデータ構造は、マイグレーションしやすいです。

- nullable追加 → backfill → NOT NULL化の順に進める
- 新status追加時に既存コードが壊れないようにする
- JSON内の値をカラム化する場合、移行スクリプトを用意する
- 公開APIの削除は段階的に行う
- 大量テーブルのindex追加はロックと時間を考える

## 13.11 実務ポイント

- RDBでは、業務上一意なもの、必須なもの、値域があるものを制約として表す。
- statusは値域だけでなく、遷移と関連timestampの整合性まで考える。
- JSONカラム、論理削除、非正規化は悪ではないが、意味と運用ルールが必要である。
- マイグレーション手順まで含めて、スキーマ設計を考える。

## 13.12 よくある失敗

- 外部キーやunique制約を入れず、アプリ側だけで守る。
- すべての削除・停止・終了を`deleted_at`で表す。
- statusを文字列自由入力にする。
- 履歴が必要なデータを現在値だけで保存する。
- 検索・集計対象をJSONに入れてしまう。

---
# 14. フロントエンドstateでの考え方

## 14.1 フロントエンドstateの分類

フロントエンドの状態は、すべてをグローバルstateに入れるものではありません。

| 種類 | 例 | 置き場所 |
|---|---|---|
| サーバー由来の状態 | ユーザー一覧、注文詳細 | React Query、SWR、Apolloなどのcache |
| UIだけの状態 | モーダル開閉、タブ選択 | local state |
| フォーム状態 | 入力値、dirty、touched、errors | form library / component state |
| URLに持つ状態 | 検索条件、ページ番号、sort | query params |
| グローバルstate | ログイン中ユーザー、テーマ、横断的通知 | Zustand、Reduxなど |
| キャッシュ | API結果、normalized data | server state cache |
| optimistic update | 送信前に仮反映する値 | cache + rollback情報 |
| derived state | filtered list、合計、表示ラベル | 原則として計算する |

## 14.2 サーバーstateとUI stateを分ける

悪い例です。

```ts
const useStore = create<{
  orders: Order[];
  selectedOrder: Order | null;
  isModalOpen: boolean;
  searchKeyword: string;
  setOrders: (orders: Order[]) => void;
}>(() => ({
  orders: [],
  selectedOrder: null,
  isModalOpen: false,
  searchKeyword: '',
  setOrders: () => {},
}));
```

サーバー由来の注文、選択状態、UIの開閉、検索条件が混ざっています。

良い例です。

```ts
// サーバーstateはキャッシュに任せる
const { data: orders } = useQuery({
  queryKey: ['orders', searchParams],
  queryFn: () => fetchOrders(searchParams),
});

// UIだけの状態はlocal state
const [isFilterOpen, setFilterOpen] = useState(false);

// 選択はIDで持つ
const [selectedOrderId, setSelectedOrderId] = useState<string | null>(null);
```

## 14.3 URLに持つべき状態

URLに持つとよい状態です。

- 検索キーワード
- filter条件
- sort
- page
- タブ。ただし共有・復元したい場合

```ts
type OrderListQuery = {
  q?: string;
  status?: OrderStatus;
  page: number;
  sort: 'created_at_desc' | 'created_at_asc';
};
```

URLに持つべきでないことが多い状態です。

- モーダルの一時的な開閉
- 入力中のパスワード
- 入力途中の大量フォーム
- 一時的なhover状態

## 14.4 derived stateを重複させない

悪い例です。

```ts
const [items, setItems] = useState<CartItem[]>([]);
const [totalAmount, setTotalAmount] = useState(0);
```

`totalAmount`は`items`から計算できます。別stateにするとズレます。

良い例です。

```ts
const totalAmount = useMemo(() => {
  return items.reduce((sum, item) => sum + item.unitPrice * item.quantity, 0);
}, [items]);
```

## 14.5 form stateはDomain Modelと同じにしない

入力中には、空文字、不正な数値、未選択、検証エラーが存在します。

```ts
type ProfileFormState = {
  displayName: string;
  birthDateInput: string; // 入力途中は不正な日付もあり得る
  errors: {
    displayName?: string;
    birthDateInput?: string;
  };
  isDirty: boolean;
  isSubmitting: boolean;
};

type ProfileUpdateRequest = {
  displayName: string;
  birthDate: string | null;
};
```

## 14.6 optimistic update

optimistic updateでは、仮反映と失敗時rollbackを考えます。

```ts
async function toggleTodo(todoId: string) {
  const previous = queryClient.getQueryData<Todo[]>(['todos']);

  queryClient.setQueryData<Todo[]>(['todos'], todos =>
    todos?.map(todo =>
      todo.id === todoId ? { ...todo, completed: !todo.completed } : todo
    )
  );

  try {
    await api.toggleTodo(todoId);
  } catch (error) {
    queryClient.setQueryData(['todos'], previous);
  }
}
```

ここで重要なのは、サーバー正本とクライアント仮状態を混同しないことです。

## 14.7 Zustandで設計するとき

Zustandなどのグローバルstateには、アプリ横断で必要なUI状態やセッション情報を置きます。サーバー由来データを大量にコピーし続ける用途には注意します。

```ts
type AppUiState = {
  sidebarOpen: boolean;
  toastMessages: Array<{ id: string; message: string }>;
  setSidebarOpen: (open: boolean) => void;
  pushToast: (message: string) => void;
};
```

サーバーの注文一覧やユーザー一覧は、React Queryなどのserver state cacheに任せる方が自然なことが多いです。

## 14.8 実務ポイント

- フロントエンドstateは、server state、UI state、form state、URL stateに分類する。
- 同じデータを複数stateにコピーせず、正本とID参照を使う。
- 検索条件やページ番号など共有・復元したい状態はURLに置く。
- derived stateは保存せず、原則として計算する。
- optimistic updateでは、仮状態とサーバー正本を区別し、rollbackを用意する。

## 14.9 よくある失敗

- すべての状態をグローバルstoreに入れる。
- APIレスポンスをlocal stateにコピーし、その後キャッシュとズレる。
- フォーム入力中の値をDomain Modelで表そうとする。
- 検索条件をcomponent stateに閉じ込め、URL共有できない。
- derived stateを別stateとして持ち、同期漏れを起こす。

---

# 15. API設計での考え方

## 15.1 APIのデータ構造は「公開契約」

APIのRequest/Responseは、内部実装ではなく外部との契約です。一度公開した形は、クライアント、外部システム、モバイルアプリ、社内ツールに依存されます。DBの形をそのまま返すと、内部変更がAPI互換性を壊します。

## 15.2 DBの形をそのまま返してよいのか

原則として、DB行をそのまま返すのは避けます。

悪い例です。

```json
{
  "id": "usr_1",
  "email": "taro@example.com",
  "password_hash": "...",
  "deleted_at": null,
  "created_at": "2026-01-01T00:00:00Z"
}
```

良い例です。

```json
{
  "id": "usr_1",
  "email": "taro@example.com",
  "displayName": "Taro",
  "createdAt": "2026-01-01T00:00:00Z"
}
```

APIでは、公開してよい情報だけをDTOとして返します。

## 15.3 画面専用APIは悪いのか

画面専用APIは常に悪ではありません。

| APIの種類 | 向く場面 | 注意 |
|---|---|---|
| 汎用リソースAPI | 複数画面・複数クライアントで再利用 | 画面側の組み立て負荷が増える |
| 画面専用API | 複雑な一覧、管理画面、ダッシュボード | 画面変更にAPIが引きずられる |
| BFF | Web/モバイルごとに最適化 | ドメインロジックを入れすぎない |

大事なのは、画面専用APIでも**正本を歪めないこと**です。

## 15.4 Request/Responseの設計

```ts
type CreateOrderRequest = {
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  couponCode?: string;
};

type CreateOrderResponse = {
  id: string;
  status: 'payment_pending';
  paymentUrl: string;
};
```

Requestではクライアントが指定してよい値だけを受け取ります。価格、合計金額、権限、作成者IDなどはサーバー側で決めるべきことが多いです。

## 15.5 PUTとPATCH、partial update

| メソッド | 意味 | データ構造 |
|---|---|---|
| PUT | リソース全体の置換 | 全項目が必要 |
| PATCH | 部分更新 | 指定された項目だけ変更 |

PATCHでは、`undefined`相当の「未指定」と`null`の「明示的削除」を区別します。

```json
{
  "displayName": "新しい名前",
  "avatarUrl": null
}
```

この例では、`displayName`は変更、`avatarUrl`は削除です。送られていないフィールドは変更しません。

## 15.6 エラー構造

エラーもデータ構造です。

悪い例です。

```json
{
  "message": "エラーです"
}
```

良い例です。

```json
{
  "code": "VALIDATION_ERROR",
  "message": "入力内容に誤りがあります",
  "fields": [
    { "path": "items[0].quantity", "code": "MIN_VALUE", "message": "数量は1以上にしてください" }
  ]
}
```

クライアントが分岐できるように、機械的な`code`を持たせます。

## 15.7 ページネーション、検索条件、ソート条件

```json
{
  "items": [
    { "id": "ord_1", "status": "paid" }
  ],
  "pagination": {
    "type": "cursor",
    "nextCursor": "eyJpZCI6...",
    "hasNext": true
  }
}
```

検索条件とソート条件は、値域を決めます。

```ts
type ListOrdersRequest = {
  status?: OrderStatus;
  q?: string;
  sort?: 'created_at_desc' | 'created_at_asc';
  cursor?: string;
  limit?: number;
};
```

## 15.8 権限によって返すデータが変わる場合

権限で返すデータが変わる場合は、レスポンス構造を明確にします。

```ts
type UserResponse = {
  id: string;
  displayName: string;
  email?: string; // 権限がある場合のみ返す、などのルールを文書化
};
```

より明示するなら、閲覧者の権限に応じたDTOを分けます。

```ts
type PublicUserResponse = {
  id: string;
  displayName: string;
};

type AdminUserResponse = PublicUserResponse & {
  email: string;
  lastLoginAt: string | null;
};
```

## 15.9 バージョニングと後方互換性

API変更では、次を意識します。

- フィールド追加は比較的安全
- フィールド削除・名前変更は破壊的
- enum/statusの値追加もクライアントにとっては破壊的になり得る
- nullable変更は破壊的になり得る
- 意味の変更は最も危険

```json
{
  "id": "ord_1",
  "status": "paid",
  "statusLabel": "支払い済み"
}
```

`statusLabel`の追加は安全寄りですが、`status`の意味変更は危険です。

## 15.10 実務ポイント

- API DTOは公開契約であり、DB Modelとは分ける。
- Requestでは、クライアントに決めさせてよい値だけを受け取る。
- PATCHでは未指定、null、空文字の意味を明確にする。
- エラー、ページネーション、検索条件、ソート条件も設計対象である。
- enum/statusの値追加はクライアント影響を確認する。

## 15.11 よくある失敗

- DB行をそのままAPIで返し、内部情報を漏らす。
- APIレスポンスに画面表示ラベルだけを返し、機械的な値を返さない。
- PATCHでnullと未指定を区別しない。
- エラーを文字列だけで返し、クライアントが分岐できない。
- statusの値を追加して、既存クライアントのswitch文を壊す。

---
# 16. 状態遷移とデータ構造

## 16.1 statusをどう設計するか

statusは表示のための文字列ではありません。**許可される操作、必要なデータ、次に進める状態を決める値**です。

状態を設計するときは、次を明確にします。

- 状態の一覧
- 初期状態
- 終了状態
- 遷移可能な組み合わせ
- 各状態で必要なデータ
- 各状態で許可される操作
- 状態変更時に残す履歴

## 16.2 booleanフラグを避けるべき理由

booleanは2値しか表現できません。状態が3つ以上ある業務では、booleanの組み合わせが爆発します。

```ts
type BadReservation = {
  isRequested: boolean;
  isConfirmed: boolean;
  isCancelled: boolean;
};
```

この形では、すべてtrueのような不可能状態が作れます。

良い例です。

```ts
type ReservationStatus = 'requested' | 'confirmed' | 'cancelled';

type Reservation = {
  id: string;
  status: ReservationStatus;
};
```

## 16.3 状態遷移図

```text
requested
  -> confirmed
  -> cancelled

confirmed
  -> cancelled
  -> completed

cancelled
  -> 終了状態

completed
  -> 終了状態
```

この図がないまま実装すると、各所で独自の条件分岐が生まれます。

## 16.4 不可能な状態を表現できないようにする

状態ごとに必要なデータが違う場合、discriminated unionを使います。

```ts
type Reservation =
  | {
      status: 'requested';
      requestedAt: string;
      requestedBy: string;
    }
  | {
      status: 'confirmed';
      requestedAt: string;
      confirmedAt: string;
      confirmedBy: string;
    }
  | {
      status: 'cancelled';
      requestedAt: string;
      cancelledAt: string;
      cancelReason: string;
    }
  | {
      status: 'completed';
      requestedAt: string;
      completedAt: string;
    };
```

`cancelledAt`があるのに`status`が`confirmed`のような状態を型で防ぎやすくなります。

## 16.5 DBではどう表現するか

DBでは、status値域、関連timestamp、履歴を組み合わせます。

```sql
CREATE TABLE reservations (
  id uuid PRIMARY KEY,
  status text NOT NULL CHECK (status IN ('requested', 'confirmed', 'cancelled', 'completed')),
  requested_at timestamptz NOT NULL,
  confirmed_at timestamptz,
  cancelled_at timestamptz,
  completed_at timestamptz,
  cancel_reason text,
  CHECK (
    (status = 'requested' AND confirmed_at IS NULL AND cancelled_at IS NULL AND completed_at IS NULL)
    OR (status = 'confirmed' AND confirmed_at IS NOT NULL AND cancelled_at IS NULL AND completed_at IS NULL)
    OR (status = 'cancelled' AND cancelled_at IS NOT NULL)
    OR (status = 'completed' AND completed_at IS NOT NULL)
  )
);

CREATE TABLE reservation_status_events (
  id uuid PRIMARY KEY,
  reservation_id uuid NOT NULL REFERENCES reservations(id),
  from_status text,
  to_status text NOT NULL,
  changed_at timestamptz NOT NULL DEFAULT now(),
  reason text
);
```

DBのCHECK制約で完全な業務ロジックを表す必要はありませんが、明らかに不正な状態は防ぎます。

## 16.6 APIではどう表現するか

APIでは、statusと状態ごとのデータを明確に返します。

```json
{
  "id": "res_1",
  "status": "cancelled",
  "requestedAt": "2026-01-01T10:00:00Z",
  "cancelledAt": "2026-01-02T10:00:00Z",
  "cancelReason": "user_request"
}
```

クライアントが操作可能かどうかも返すと、画面の条件分岐が安定します。

```json
{
  "id": "res_1",
  "status": "confirmed",
  "availableActions": ["cancel", "complete"]
}
```

ただし、`availableActions`は派生値です。サーバー側の権限・状態ルールを正本として計算します。

## 16.7 実務ポイント

- statusは表示用ではなく、操作可否と業務ルールの中心である。
- booleanフラグが2つ以上になったら、状態として表せないか検討する。
- 状態ごとに必要なデータが違うなら、TypeScriptではdiscriminated unionを使う。
- DBでは値域、timestamp整合性、状態履歴を組み合わせる。
- APIでは、機械的なstatusと必要なら派生の`availableActions`を返す。

## 16.8 よくある失敗

- `isActive`, `isDeleted`, `isCancelled`, `isExpired`を同時に持つ。
- status値はあるが、遷移ルールがどこにも書かれていない。
- 状態ごとの必須データをoptionalで表し、null分岐だらけにする。
- DBではどんなstatusでも保存できるようにしてしまう。
- APIで表示ラベルだけ返し、機械的なstatusを返さない。

---

# 17. データ構造のレビュー観点

## 17.1 設計レビューで使えるチェックリスト

| チェック項目 | なぜ確認すべきか |
|---|---|
| このデータの正本はどこか？ | 重複や同期漏れを防ぐため |
| この値は保存すべきか、計算できるか？ | 派生値の不整合を防ぐため |
| nullは本当に必要か？ | null分岐と曖昧さを減らすため |
| このbooleanの組み合わせで不可能な状態は生まれないか？ | 状態爆発を防ぐため |
| statusの遷移は明確か？ | 無効な操作を防ぐため |
| 同じ意味の値が複数箇所にないか？ | 正本不明を防ぐため |
| 名前から意味が分かるか？ | チーム理解と保守性のため |
| 更新者は誰か？ | 権限、監査、競合制御のため |
| 更新タイミングはいつか？ | レースコンディションや同期漏れを防ぐため |
| 履歴は必要か？ | 監査、調査、請求、契約のため |
| 削除されたらどうなるか？ | 参照整合性、復旧、個人情報対応のため |
| 将来増える状態に耐えられるか？ | 仕様変更時の影響を予測するため |
| API/DB/UIの都合が混ざっていないか？ | 境界の変更に強くするため |
| テストしやすいか？ | 仕様を検証し続けるため |
| 障害調査しやすいか？ | 原因追跡と復旧を早くするため |

## 17.2 レビュー時の聞き方

設計レビューでは、否定から入るよりも、問いで確認します。

- この値は誰が最後に更新しますか？
- この値がズレた場合、どちらを信じますか？
- このstatusに新しい値が増えたら、どこを直しますか？
- このnullはどんな業務状態を意味しますか？
- このJSONの中身は検索・集計されますか？
- このAPIレスポンスは将来フィールド削除できますか？
- このデータの変更履歴は障害調査で必要ですか？

## 17.3 レビューの観点別テンプレート

```markdown
## データ構造レビュー

### 対象
- 機能:
- 主なユースケース:
- 主要データ:

### 正本
- 正本データ:
- 派生データ:
- 一時状態:

### ライフサイクル
- 発生:
- 更新:
- 参照:
- 削除:
- 履歴:

### 状態遷移
- 初期状態:
- 終了状態:
- 遷移:
- 不可能な状態:

### 境界
- DB:
- API:
- Domain:
- Frontend state:
- Event/Log:

### 制約
- NOT NULL:
- unique:
- check:
- foreign key:
- validation:

### 懸念
- 不整合リスク:
- 移行リスク:
- パフォーマンスリスク:
- 調査・監査リスク:
```

## 17.4 実務ポイント

- レビューではコードの形だけでなく、正本、派生、更新者、履歴、状態遷移を見る。
- boolean、nullable、JSON、string statusは重点的に確認する。
- 「この値がズレたらどちらを信じるか」は強力な質問である。
- レビュー観点をテンプレート化し、毎回同じ問いを使う。

## 17.5 よくある失敗

- 命名やコードスタイルだけをレビューし、データの意味を見ない。
- DB制約、API DTO、フロントstateを別々にレビューする。
- 状態遷移を口頭だけで済ませる。
- 監査や障害調査の観点をレビューに入れない。
- 「今回は小さい機能だから」と正本や履歴を確認しない。

---

# 18. 思考の型を作る方法

## 18.1 毎回使える問いのテンプレート

データ構造を考えるときは、次の問いを毎回使います。

```markdown
## データ構造設計メモ

### 1. この機能で扱う概念
- 

### 2. 各概念の責務
- 

### 3. 正本
- この値の正本はどこか:
- コピーや派生値はあるか:

### 4. ライフサイクル
- どこで生まれるか:
- 誰が更新するか:
- どこで参照されるか:
- いつ消えるか:

### 5. 状態
- 状態一覧:
- 初期状態:
- 終了状態:
- 遷移:
- 不可能な状態:

### 6. 制約
- 必須:
- 一意:
- 値域:
- 外部キー:
- 日付・数量・金額の制約:

### 7. 境界
- DB:
- API Request:
- API Response:
- Domain Model:
- Frontend state:
- Form state:
- URL params:

### 8. 履歴・監査
- 残す履歴:
- 変更者:
- 変更理由:

### 9. パフォーマンス・運用
- 主要クエリ:
- index:
- 再計算できるデータ:
- 障害時の調査方法:

### 10. 変更耐性
- 増えそうな状態:
- 増えそうな属性:
- 増えそうな権限:
- 外部連携追加時の影響:
```

## 18.2 既存機能を分析する練習

既存機能を読むときは、次を逆引きします。

1. 主要テーブルを1つ選ぶ
2. 各カラムの意味を書く
3. nullの意味を分類する
4. statusの値域と遷移を書く
5. 派生値を探す
6. 同じ意味のデータが他にないか探す
7. APIレスポンスとの対応を書く
8. フロントエンドstateでコピーされている箇所を探す
9. 障害調査時に必要なログがあるか確認する
10. 改善案を書く

## 18.3 悪い設計を良い設計に直す練習

練習用の悪い型です。

```ts
type Subscription = {
  id: string;
  userId: string;
  plan: string;
  status: string;
  isTrial: boolean;
  isCancelled: boolean;
  startDate?: string;
  endDate?: string;
  nextPlan?: string;
  metadata?: any;
};
```

改善観点です。

- statusの値域を決める
- trial、active、scheduled_to_cancel、endedを状態として表す
- 現在プランと予定プラン変更を分ける
- `metadata`の中身を確認し、重要項目は構造化する
- 開始日、終了日、次回更新日の意味を分ける

改善例です。

```ts
type Subscription =
  | {
      status: 'trial';
      userId: UserId;
      planId: PlanId;
      trialStartedAt: string;
      trialEndsAt: string;
    }
  | {
      status: 'active';
      userId: UserId;
      planId: PlanId;
      currentPeriodStart: string;
      currentPeriodEnd: string;
    }
  | {
      status: 'scheduled_to_cancel';
      userId: UserId;
      planId: PlanId;
      currentPeriodEnd: string;
      cancellationScheduledAt: string;
    }
  | {
      status: 'ended';
      userId: UserId;
      planId: PlanId;
      endedAt: string;
    };
```

## 18.4 状態遷移図を書く練習

練習対象です。

- 注文
- 決済
- 請求
- 契約
- 予約
- 問い合わせチケット
- 承認ワークフロー

書くべき内容です。

```text
状態A
  -> 状態B: 条件、実行者、残す履歴
  -> 状態C: 条件、実行者、残す履歴
```

## 18.5 ER図・型・DB・API・stateに落とす練習

同じ題材を、複数の表現に落とします。

| 表現 | 練習内容 |
|---|---|
| ER図 | 概念、関係、多重度、ライフサイクルを書く |
| TypeScript | union、Value Object、DTOを書く |
| DBスキーマ | テーブル、外部キー、制約を書く |
| API | Request/Response/Errorを書く |
| Frontend state | server state、form state、URL stateを分ける |

## 18.6 練習問題

### 問題1: 予約システム

美容室の予約機能を設計してください。

- 顧客が予約を作成する
- 店舗スタッフが確定する
- 顧客がキャンセルする
- 無断キャンセルを記録する
- 同じ時間枠に重複予約できない

考えること:

- 予約の状態
- 時間枠の正本
- 顧客とスタッフの関係
- キャンセル履歴
- DB制約
- API DTO
- フロントエンドstate

### 問題2: 請求書管理

請求書の作成、発行、支払い、無効化を設計してください。

考えること:

- draftとissuedで必要なデータの違い
- 発行後に金額を変えられるか
- 支払い履歴
- 請求番号の一意性
- APIのエラー構造

### 問題3: 権限管理

組織、ユーザー、ロール、権限を設計してください。

考えること:

- roleとpermissionの違い
- organizationごとの権限
- 管理者だけが招待できる制約
- 権限変更履歴

## 18.7 実務ポイント

- 思考力は、毎回同じ問いを使うことで再現性が上がる。
- 既存機能を読む練習は、設計力を上げる最短ルートである。
- 悪い設計を直す練習では、null、boolean、string status、JSONを重点的に見る。
- 1つの題材を、ER図、TypeScript、DB、API、stateに変換する練習が有効である。

## 18.8 よくある失敗

- 経験だけに頼り、毎回問いが変わる。
- ER図だけ、型だけ、DBだけの練習に偏る。
- 悪い設計を見ても、なぜ悪いかを言語化しない。
- 状態遷移図を書かずにstatusだけ作る。
- 練習問題で運用・障害調査・履歴まで考えない。

---
# 19. 実務での進め方

## 19.1 フェーズごとに見るべきこと

| フェーズ | 何を見るか | 成果物 |
|---|---|---|
| 要件確認時 | ユースケース、登場概念、正本、履歴、権限 | 概念メモ、用語表、状態候補 |
| 実装前 | DB、API、Domain、FE stateの境界 | 設計メモ、ER図、状態遷移図、DTO案 |
| PR前 | 実装が設計と一致しているか | セルフレビュー、テストケース |
| PRレビュー時 | 正本、派生、null、status、制約、移行 | レビューコメント、設計修正 |
| リリース後 | 実データ、ログ、性能、問い合わせ | 監視、メトリクス、改善メモ |
| 障害発生時 | 変更履歴、イベント、正本、復旧手順 | 事後分析、再発防止策 |
| 仕様変更時 | 既存データへの影響、互換性、移行 | 変更設計、migration plan |

## 19.2 要件確認時

要件確認では、画面項目より先に概念とライフサイクルを確認します。

聞くべき質問です。

- この値は誰が入力しますか？
- 入力後に変更できますか？
- 変更履歴は必要ですか？
- この状態になった後、戻れますか？
- 外部システムが正本を持つ値はありますか？
- 退会、解約、キャンセル、削除時にどうなりますか？
- 管理者と一般ユーザーで見えるデータは違いますか？

## 19.3 実装前

実装前には、軽量でよいので設計メモを作ります。

```markdown
# 設計メモ: 注文キャンセル機能

## ユースケース
- 顧客が支払い前の注文をキャンセルする
- 管理者が支払い後の注文をキャンセルし、必要なら返金する

## 正本
- 注文状態: orders.status
- 決済状態: payments.status + 決済プロバイダ
- キャンセル履歴: order_status_events

## 状態遷移
- created -> cancelled
- payment_pending -> cancelled
- paid -> refunded
- shipped -> キャンセル不可

## DB変更
- order_status_events追加
- orders.statusの値域確認

## API
- POST /orders/{id}/cancel
- Request: reason
- Response: updated order status

## FE state
- 注文詳細はserver state
- キャンセル理由はform state
- モーダル開閉はlocal state
```

## 19.4 PR前

PR前のセルフレビューでは、次を確認します。

- 追加したnullableは本当に必要か
- statusの値追加に対してswitchが網羅されているか
- DB制約が不足していないか
- APIで内部情報を返していないか
- フロントエンドstateに重複がないか
- テストは状態遷移と不正状態を含んでいるか
- migrationは既存データで動くか

## 19.5 PRレビュー時

レビュアーは、実装の細部だけでなく「データの意味」を確認します。

良いレビューコメント例です。

- `isCancelled`を追加するより、既存の`status`に`cancelled`を追加する方が状態の整合性を保ちやすそうです。
- `totalAmount`は明細から計算できそうですが、保存する理由は請求確定時の金額として残すためですか？
- `metadata`に入る項目のうち、検索・集計で使うものはカラム化した方がよさそうです。
- `avatarUrl?: string | null`は、未指定と削除を区別する目的でしょうか？API仕様に明記しましょう。

## 19.6 リリース後

リリース後は、設計が実データに耐えているかを見ます。

- 不正なstatusの組み合わせがないか
- nullが想定以上に多くないか
- migration後のデータ欠損がないか
- 主要クエリが遅くないか
- ログで更新者・変更理由を追えるか
- APIクライアントが未知のstatusで落ちていないか

## 19.7 障害発生時

障害時は、データ構造の弱点が見えます。

- 正本が曖昧で復旧判断に迷ったか
- 履歴がなく原因を追えなかったか
- 派生値がズレたか
- 状態遷移が不明で不正操作が起きたか
- DB制約があれば防げたか
- ログに必要なIDや前後値がなかったか

障害後は、バグ修正だけでなく、データ構造の改善も検討します。

## 19.8 仕様変更時

仕様変更では、既存データと互換性が重要です。

- 新しいstatusを追加するだけでよいか
- 既存statusの意味が変わるか
- 既存APIクライアントが未知値を扱えるか
- 既存データをどの状態に移行するか
- 履歴テーブルも移行が必要か
- 古い画面と新しい画面が同時に存在する期間はあるか

## 19.9 ドキュメント化の粒度

すべてを巨大な設計書にする必要はありません。重要なのは、後から判断できる情報を残すことです。

| 機能の規模 | 残すもの |
|---|---|
| 小さい修正 | PR本文に正本・変更理由を書く |
| 中規模機能 | 設計メモ、DTO、状態遷移、DB変更 |
| 大規模機能 | 用語集、ER図、状態遷移図、migration plan、運用設計 |
| 監査・請求・契約系 | 履歴、監査ログ、復旧手順まで明文化 |

## 19.10 実務ポイント

- 要件確認時に、正本、履歴、削除、権限を確認する。
- 実装前に、軽量な設計メモと状態遷移図を作る。
- PRでは、コードの書き方だけでなくデータの意味と制約をレビューする。
- 障害後は、ロジックだけでなくデータ構造を改善する。
- ドキュメントは巨大でなくてよいが、判断理由を残す。

## 19.11 よくある失敗

- 要件確認で画面項目だけを確認し、ライフサイクルを確認しない。
- 実装前に状態遷移を書かない。
- PRで命名やスタイルだけを見て、正本・履歴・制約を見ない。
- リリース後に実データの状態を確認しない。
- 障害後に応急処置だけして、データ構造の原因を直さない。

---

# 20. まとめ

## 20.1 「データ構造から考える」とは一言でいうと何か

**業務上の情報について、正本・意味・責務・状態・制約・ライフサイクルを先に決め、そのうえでDB、API、型、フロントエンドstate、ログ、イベントに落とすこと**です。

## 20.2 初心者がまず意識すべき3つのこと

1. **名前から意味が分かるようにする**
   - `value`, `data`, `flag`, `type`ではなく、業務用語で表す。
2. **正本と派生値を分ける**
   - どこが真実か、何が計算値かを決める。
3. **nullとbooleanを雑に増やさない**
   - nullの意味、booleanの組み合わせを確認する。

## 20.3 中級者が意識すべき5つのこと

1. **ライフサイクルを見る**
   - 発生、更新、参照、削除、履歴を考える。
2. **状態遷移を明示する**
   - statusの値域と遷移を図にする。
3. **境界ごとに型を分ける**
   - DB Model、Domain Model、DTO、Form State、View Modelを必要に応じて分ける。
4. **制約を適切な場所に置く**
   - 型、バリデーション、DB制約、ドメインロジックを使い分ける。
5. **運用・調査まで考える**
   - 監査ログ、履歴、再計算、復旧を考える。

## 20.4 実務で毎回使えるチェックリスト

- このデータの正本はどこか？
- これは保存すべき値か、計算できる値か？
- この値は一時状態ではないか？
- nullは本当に必要か？意味は何か？
- optionalは未指定なのか、不要なのか？
- booleanの組み合わせで不可能な状態は生まれないか？
- statusの値域と遷移は明確か？
- 同じ意味の値が複数箇所にないか？
- 名前から業務上の意味が分かるか？
- 更新者と更新タイミングは明確か？
- 履歴、監査、変更理由は必要か？
- 削除、解約、キャンセル時にどうなるか？
- API、DB、UI、Domainの都合が混ざっていないか？
- 検索・集計で使う値がJSONに隠れていないか？
- テストしやすいか？
- 障害調査しやすいか？
- 将来増える状態・属性・権限に耐えられるか？

## 20.5 設計時のテンプレート

````markdown
# データ構造設計テンプレート

## 1. ユースケース
- 誰が:
- 何をする:
- 成功条件:
- 失敗条件:

## 2. 概念
| 概念 | 意味 | 責務 | 持たない責務 |
|---|---|---|---|
| | | | |

## 3. 正本・派生・一時状態
| データ | 分類 | 正本 | 派生元 | 保存理由 |
|---|---|---|---|---|
| | | | | |

## 4. ライフサイクル
| データ | 発生 | 更新 | 参照 | 削除 | 履歴 |
|---|---|---|---|---|---|
| | | | | | |

## 5. 状態遷移
```text
初期状態
  -> 次の状態
```

## 6. 不変条件・制約
- 必須:
- 一意:
- 値域:
- 参照整合性:
- 日付・数量・金額:

## 7. DB
- テーブル:
- 主キー:
- 外部キー:
- index:
- migration:

## 8. API
- Request:
- Response:
- Error:
- バージョニング:

## 9. Frontend state
- server state:
- form state:
- URL state:
- local UI state:
- cache:

## 10. 運用
- 監査ログ:
- 障害調査:
- 再計算:
- 復旧:
````

上のテンプレートは、Markdown内にコードフェンスを含むため、外側を4連バッククォートにしています。

## 20.6 悪い匂いリスト

| 匂い | 見直すこと |
|---|---|
| `data`, `value`, `flag`, `type` | 名前で意味を表せているか |
| `string | null`だらけ | 状態ごとに分けられないか |
| booleanが3つ以上 | statusや状態遷移にできないか |
| `status: string` | 値域と遷移を定義できないか |
| `metadata: any` | 重要項目を構造化できないか |
| APIとDBが同じ型 | 公開範囲と永続化責務が混ざっていないか |
| formとDomainが同じ型 | 入力途中を表現できているか |
| 合計値を保存 | 派生値か、確定事実か |
| 履歴がない | 調査・監査で困らないか |
| deleted_atに何でも入る | 削除、停止、解約、アーカイブを区別できているか |

## 20.7 迷ったときの判断基準

| 迷い | 判断基準 |
|---|---|
| 保存するか計算するか | 計算可能で性能・監査上問題なければ計算。確定事実や高頻度集計なら保存 |
| nullにするか | nullの意味を1つに説明できるなら可。複数意味なら型や状態を分ける |
| booleanかstatusか | 3状態以上、または排他的状態ならstatus |
| JSONかカラムか | 検索・集計・制約・外部キーが必要ならカラム |
| 型を共通化するか | 同じ意味なら共通化。形が似ているだけなら分ける |
| 画面専用APIか | 複雑な画面では可。ただし正本やドメインを歪めない |
| 非正規化するか | 読み取り性能や集計目的なら可。同期・再計算手段を持つ |
| 履歴を残すか | 契約、請求、権限、状態変更、監査対象なら残す |

## 20.8 今日から使える行動指針

- 新しい機能を作る前に、5分だけ「正本・派生・一時状態」を書く。
- statusが出たら、必ず状態遷移図を書く。
- nullableを追加するときは、nullの意味をコメントではなく構造で表せないか考える。
- booleanを追加するときは、既存booleanとの組み合わせ表を書く。
- API DTO、DB Model、Form Stateを同じ型にしようとしたら、一度立ち止まる。
- JSONカラムや`metadata`を使うときは、検索・集計・移行の予定を確認する。
- PRレビューで「この値の正本はどこですか？」と聞く。
- 障害対応後に「データ構造で防げたか？」を振り返る。

## 20.9 実務ポイント

- データ構造から考える力は、特別な才能ではなく、毎回同じ問いを使う習慣で育つ。
- 重要なのは、DB、API、TypeScript、フロントエンドstateを別々に見るのではなく、同じ業務データの異なる表現として接続して考えること。
- 良いデータ構造は、不可能な状態を作りにくくし、変更時の影響を予測しやすくし、障害調査を助ける。
- 「正本はどこか」「これは保存すべきか」「この状態は本当に可能か」は、毎回使える最重要質問である。

## 20.10 よくある失敗

- 最後まで画面中心で考え、データの正本を決めない。
- 型・DB・API・stateを別々に設計し、意味がズレる。
- 状態遷移、履歴、監査、削除を後回しにする。
- 「柔軟」という言葉で曖昧な構造を正当化する。
- 設計の判断理由を残さず、数か月後に誰も意味を説明できなくなる。

---

# 付録: 最小実践フロー

最後に、実務で最小限これだけやればよい流れをまとめます。

1. ユースケースを3〜5個書く。
2. 名詞を抽出し、概念として残すものを決める。
3. 各データを「正本・派生・一時状態・履歴」に分類する。
4. 状態があるものは、状態遷移図を書く。
5. DB制約、API DTO、Frontend stateをそれぞれ設計する。
6. 「null、boolean、string status、JSON」を重点的に見直す。
7. PR前にチェックリストでセルフレビューする。

これを毎回繰り返すだけで、データ構造を見る目は確実に鍛えられます。
