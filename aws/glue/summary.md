## Glue について

S3 に配置されたデータの metadata を持てるサービス
<br />S3 に保存されたオブジェクトをいずれかのサービス（Athena, Lambda, Glue のクローラー）で、外部から参照出来るように登録する

## 登録されたデータとは 

いずれかのサービスが特定のオブジェクトをバケット内から探そうとした時に、Glue が登録したデータを参照し値を確認する
<br />上記から、実値を確認するのはいずれかのサービスの責務になるため、Glue では登録するデータに実値を持たないようになっている

```json
// イメージ
{
  tableName: shop,
  dataSource: "csv/sample/salary-cap.csv",
  fileType: "csv",
  columns: {
    name: string,
    age: number,
    phoneNumber: number,
    address: string,
    salary: number
  }
}
```
