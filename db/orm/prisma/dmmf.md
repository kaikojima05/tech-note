## DMMF について

Primsa には TypeScript で記述されたクエリを Rust に渡すための中間データを生成する仕組みがあり、これを DMMF（Data Model Meta Format）という。
<br />DMMF にはデータモデルの構造、利用可能な演算子、フィールドの型、リレーションなどが定義された json 形式のデータが含まれていて、TypeScript と Rust はそれぞれで DMMF を確認し、どんな型を生成すればいいのか、クエリがスキーマ通りかを確認している。

## お題

DMMF を利用して、モデルレベル[^1]、フィールドレベル[^2]で定義されたユニーク制約を取得する。

[^1]: モデル定義の最後にまとめて書かれるもの。複合ユニーク制約を定義するために使用されることが多い。
[^2]: フィールド定義の中に書かれるもの。単一フィールドのユニーク制約を定義するために使用されることが多い。

## 使用方法

DMMF は `@prisma/client` からインポートした `Prisma` オブジェクトを介してアクセスできる。

```TypeScript
import { Prisma } from '@prisma/client';

const dmmf = Prisma.dmmf.datamodel;
```

dmmf は datamodel プロパティを持ち、datamodel プロパティが `モデルの定義（models）` 、 `モデルの型定義（types）` 、 `enum（enum）` 、 `全モデルを横断したインデックス（indexes）` の定義などへのアクセスを可能にしている。
<br />ちなみに、モデルの定義（models）とモデルの型定義（types）のインターフェースは同じだが、**前者はモデルに定義された内容を取得、後者はモデルの型定義を取得するため**に使用される。

また、models プロパティは名前の通り、モデル定義を配列に入れて提供しているため、**単一もしくは条件に一致するモデルのみを取得するには配列のメソッドを利用して取り出す必要がある**。
<br />Prisma の型定義を見ると、それぞれのプロパティへどのメソッドを使用してアクセスすればいいかが分かる。

```TypeScript
// packages/client-common/src/dmmf.ts
import * as DMMF from '@prisma/dmmf'

export type BaseDMMF = {
  readonly datamodel: Omit<DMMF.Datamodel, 'indexes'>
}

// prisma/packages/dmmf/src/dmmf.ts
export type Datamodel = ReadonlyDeep<{
  models: Model[]
  enums: DatamodelEnum[]
  types: Model[]
  indexes: Index[]
}>

export type Model = ReadonlyDeep<{
  name: string
  dbName: string | null
  schema: string | null
  fields: Field[]
  uniqueFields: string[][]
  uniqueIndexes: uniqueIndex[]
  documentation?: string
  primaryKey: PrimaryKey | null
  isGenerated?: boolean
}>

export type DatamodelEnum = ReadonlyDeep<{
  name: string
  values: EnumValue[]
  dbName?: string | null
  documentation?: string
}>

export type Index = ReadonlyDeep<{
  model: string
  type: IndexType
  isDefinedOnField: boolean
  name?: string
  dbName?: string
  algorithm?: string
  clustered?: boolean
  fields: IndexField[]
}>

export type Field = ReadonlyDeep<{
  kind: FieldKind
  name: string
  isRequired: boolean
  isList: boolean
  isUnique: boolean
  isId: boolean
  isReadOnly: boolean
  isGenerated?: boolean // does not exist on 'type' but does on 'model'
  isUpdatedAt?: boolean // does not exist on 'type' but does on 'model'
  type: string
  nativeType?: [string, string[]] | null
  dbName?: string | null
  hasDefaultValue: boolean
  default?: FieldDefault | FieldDefaultScalar | FieldDefaultScalar[]
  relationFromFields?: string[]
  relationToFields?: string[]
  relationOnDelete?: string
  relationOnUpdate?: string
  relationName?: string
  documentation?: string
}>
```

インターフェースに従って、User モデルからリレーションフィールドを取得してみる。
<br />参照先フィールドは各フィールド自身が配列で保持しているため、`map()` や `filter()` を利用してリレーションフィールドを抽出しようとすると、戻り値が2次元配列になってしまう。
<br />そのため、今回は `foreignKeys` 変数の参照側が一工夫しなくてもいいように `flatMap()` で1次元配列にする。

```TypeScript
const userModel = dmmf.models.filter((model) => model.name === 'User');
const foreignKeys = userModel.fields.flatMap((field) => field.kind === 'object' ? field.relationFromFields)
```
 
