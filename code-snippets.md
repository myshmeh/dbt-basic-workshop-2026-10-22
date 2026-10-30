# dbt Basic Workshop - コードスニペット集

ワークショップのStepごとに、コピペで使うコードをまとめています。
スライドのStep番号と対応しています。

## 目次

- [Step 6: ソーステーブルのデータ確認](#step-6)
- [Step 8: 最初のモデル（customers）の実装](#step-8)
- [Step 9: マテリアライゼーション方式の変更](#step-9)
- [Step 10: ステージングモデルの作成と ref() 関数](#step-10)
- [Step 12: ソースの定義と source() 関数](#step-12)
- [Step 13: 3種類のデータテストの追加と実行](#step-13)
- [Step 14: ドキュメント追加とカタログプレビュー](#step-14)
- [Step 18: dbt Developer Agentの体験](#step-18)

---

## <a id="step-6"></a>Step 6: ソーステーブルのデータ確認

### 顧客テーブル

```sql
select * from raw.jaffle_shop.customers
```

### 注文テーブル

```sql
select * from raw.jaffle_shop.orders
```

### 決済テーブル（任意）

```sql
select * from raw.stripe.payment
```

---

## <a id="step-8"></a>Step 8: 最初のモデル（customers）の実装

`models/customers.sql` を新規作成し、以下を貼り付け：

```sql
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from raw.jaffle_shop.customers

),

orders as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from raw.jaffle_shop.orders

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),

final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
```

### 実行コマンド

```text
dbt run
```

---

## <a id="step-9"></a>Step 9: マテリアライゼーション方式の変更

### 方法A: `dbt_project.yml` で一括設定（任意）
試行後に、+materializedの行のコメントアウトかviewに書き換える
`dbt_project.yml` の末尾に追記：

```yaml
models:
  jaffle_shop:
    +static_analysis: strict
    +materialized: table
```

### 方法B: SQLファイル内で個別指定

`models/customers.sql` の冒頭に追記：

```sql
{{
  config(
    materialized='table'
  )
}}
```

ショートカット入力：`_con`

---

## <a id="step-10"></a>Step 10: ステージングモデルの作成と ref() 関数

### `models/stg_customers.sql`

```sql
select
    id as customer_id,
    first_name,
    last_name

from raw.jaffle_shop.customers
```

### `models/stg_orders.sql`

```sql
select
    id as order_id,
    user_id as customer_id,
    order_date,
    status

from raw.jaffle_shop.orders
```

### `models/customers.sql` を ref() で書き換え

```sql
with customers as (

    select * from {{ ref('stg_customers') }}

),

orders as (

    select * from {{ ref('stg_orders') }}

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),

final as (

    select
        customers.*,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
```

ショートカット入力：`_ref`

---

## <a id="step-12"></a>Step 12: ソースの定義と source() 関数

### `models/sources.yml` を新規作成

```yaml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: customers
        columns:
          - name: id
            data_tests:
              - unique
              - not_null
      - name: orders
        columns:
          - name: id
            data_tests:
              - unique
              - not_null
```

### `models/stg_customers.sql` を source() で書き換え

```sql
select
    id as customer_id,
    first_name,
    last_name

from {{ source('jaffle_shop', 'customers') }}
```

### `models/stg_orders.sql` を source() で書き換え

```sql
select
    id as order_id,
    user_id as customer_id,
    order_date,
    status

from {{ source('jaffle_shop', 'orders') }}
```

---

## <a id="step-13"></a>Step 13: 3種類のデータテストの追加と実行

### Genericテスト：`models/schema.yml` を新規作成

```yaml
version: 2

models:
  - name: customers
    columns:
      - name: customer_id
        data_tests:
          - unique
          - not_null

  - name: stg_customers
    columns:
      - name: customer_id
        data_tests:
          - unique
          - not_null

  - name: stg_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: status
        data_tests:
          - accepted_values:
              arguments: # available in v1.10.5 and higher. Older versions can set the <argument_name> as the top-level property.
                values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']
      - name: customer_id
        data_tests:
          - not_null
          - relationships:
              arguments:
                to: ref('stg_customers')
                field: customer_id
```

### Singularテスト：`tests/assert_first_order_date_before_most_recent.sql` を新規作成

```sql
with customer_order_dates as (
    select
        customer_id,
        first_order_date,
        most_recent_order_date
    from {{ ref('customers') }}
)

select
    customer_id,
    first_order_date,
    most_recent_order_date
from customer_order_dates
where first_order_date > most_recent_order_date
```

### テスト実行コマンド

```text
dbt test
```

特定のテストのみ実行：

```text
dbt test --select customers
```

特定のSingularテストのみ実行：

```text
dbt test --select assert_first_order_date_before_most_recent
```

### Unitテスト：`models/schema.yml` に追記

注文がある顧客のテストと注文がない顧客のテスト：

```yaml
unit_tests:
  - name: test_customers_aggregates_orders_for_existing_customer
    model: customers
    given:
      - input: ref('stg_customers')
        rows:
          - {customer_id: 1, first_name: "John", last_name: "Doe"}
      - input: ref('stg_orders')
        rows:
          - {order_id: 1, customer_id: 1, order_date: "2023-01-01"}
          - {order_id: 2, customer_id: 1, order_date: "2023-06-15"}
    expect:
      rows:
        - {
            customer_id: , -- TODO
            first_name: , -- TODO
            last_name: , -- TODO
            first_order_date: "", -- TODO
            most_recent_order_date: "", -- TODO
            number_of_orders: -- TODO
          }
          
  - name: test_customers_defaults_to_zero_orders_when_no_orders_exist
    model: customers
    given:
      - input: ref('stg_customers')
        rows:
          - {customer_id: 1, first_name: "Jane", last_name: "Smith"}
      - input: ref('stg_orders')
        rows: []
    expect:
      rows:
        - {
            customer_id: , -- TODO
            first_name: "", -- TODO
            last_name: "", -- TODO
            first_order_date: , -- TODO
            most_recent_order_date: , -- TODO
            number_of_orders: -- TODO
          }
```

Unitテストのみ実行：

```text
dbt test --select "test_type:unit"
```

---

## <a id="step-14"></a>Step 14: ドキュメント追加とカタログプレビュー

### `models/schema.yml` に日本語の説明を追記

```yaml
version: 2

models:
  - name: customers
    description: 1レコードが1件の顧客を示す
    columns:
      - name: customer_id
        description: 顧客IDを示す主キー
        data_tests:
          - unique
          - not_null
      - name: first_name
        description: 顧客の名
        data_tests:
          - not_null
      - name: last_name
        description: 顧客の姓
        data_tests:
          - not_null
      - name: first_order_date
        description: 初回の注文日。NULLは顧客がまだ注文したことがないことを示す
      - name: most_recent_order_date
        description: 最新の注文日。NULLは顧客がまだ注文したことがないことを示す
      - name: number_of_orders
        description: 顧客の累計注文件数。注文がない場合は0
```

### ドキュメント生成コマンド

```text
dbt compile --write-catalog
```

または

```text
dbt run --write-catalog
```

---

## <a id="step-18"></a>Step 18: dbt Developer Agentの体験

### プロンプト例

```text
rawデータベースのjaffle_shopスキーマにあるcustomersとordersをソースとして定義して、staging でカラム名をきれいにして、customersっていうマートモデルで顧客ごとに初回注文日・最新注文日・注文数を集計する

注文がない人は注文数0にしたい。schema.ymlでモデルとカラムの説明を書いて、主キーや外部キー関係、statusの取りうる値のテストも入れて。最後に、初回注文日が最新注文日より後になるようなレコードがないことを確認するsingularテストも作成する

customersモデルに対するunitテストも実行する

標準的な dbt のフォルダ構成で、customersモデルはテーブルでマテリアライズする設定にしておいて
```

