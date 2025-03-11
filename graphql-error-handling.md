---
marp: true
theme: default
paginate: true
footer: "© 2025 Roppongi.rb"
style: |
  section {
    font-family: 'Arial', 'Helvetica', sans-serif;
    font-size: 24px;
  }
  code {
    background-color: #f0f0f0;
    padding: 0.2em 0.4em;
    font-size: 0.8em;
  }

  section.lead {
    text-align: center;
    justify-content: center;
  }
  section.lead h1 {
    font-size: 2.5em;
  }
  section.lead h2 {
    font-size: 1.8em;
  }
---

<!-- _class: lead -->

## GraphQL のエラーレスポンス設計における意思決定

2025 年 3 月 13 日（木）Roppongi.rb #28

<!--
_footer: ''
_header: ''
-->

---

# 自己紹介

<div style="display: flex; align-items: center; justify-content: space-between;">

<div style="flex: 1;">

## 栃川 晃佑

- X: [@Web_TochiTech](https://twitter.com/Web_TochiTech)
- Ruby 歴: 4 年
  - 人材紹介会社で Rails エンジニアとして働いています！

<br>

「コミュニティ活動」への参加をこれから積極的にしようと思い、飛び込みでLT応募しました🙌
もしかしたら今後ちょくちょく顔を出すかもしれないです🙏

</div>
</div>

---

# はじめに

ごめんなさい 🙇‍♂️

このスライドは、初めて「Cursor + Marp」で作ったので、
時折違和感のあるスライドやコード記述があります。

何卒ご容赦いただけますと幸いです。。。。

---

# 今日お話しすること

- GraphQL Spec で推奨されないエラーレスポンスの方法で実装した意思決定の理由を紹介します
- **ぜひフィードバックいただきたいです！**

---

# アジェンダ

- エラーレスポンス設計の選択肢
- 自分がした意思決定とその理由
- 実装例
- まとめ

---

# エラーレスポンス設計の選択肢

---

## そもそもエラーレスポンスの設計ってどんなのがあるの？

---

# エラーレスポンス設計の選択肢

前提として、GraphQL のエラー設計には、大きく以下の 3 つのアプローチがある

1. **標準仕様**
2. **Errors as data パターン**
3. **Union / Result Types パターン**

それぞれの方法にはメリットとデメリットがあり、組織の状況やプロジェクトの要件によって適切な選択が求められる。

※ このあと説明します

---

# 1. 標準仕様に従う

GraphQL の標準仕様では、エラーは`data`フィールドと同階層の `errors` フィールドに格納される。

```json
{
  "errors": [
    {
      "message": "指定のユーザが見つかりませんでした",
      .
      .

      "path": ["users"]
    }
  ],
  "data": null //　データはNULL
}
```

---

# 1. 標準仕様に従う

## メリット

- 実装が最も簡単で手軽
- クライアント側で特別なクエリを書く必要がない

## デメリット

- エラーの型や構造がスキーマで定義されないため予測が困難
- すべてのエラーが`errors`フィールドに混在するため、クライアント側での判別が難しい
  - GraphQL Spec の作者も、エラーは**例外的な状況を表すべき**であり、ユーザーデータをエラーとして表現すべきではないと主張している。

---

# 2. Errors as data

エラーも「**データ**」としてクライアント側に返してあげる方法。
エラー内容をデータと一緒に返してあげる。

```graphql
type SignUpPayload {
  userErrors: [UserError!]! # エラーがあればサーバーサイドでメッセージを格納
  account: Account # 取得しようとするデータ
}

type UserError {
  message: String!
  field: [String!]
  code: UserErrorCode
}
```

---

# 2. Errors as data

▽ レスポンスの返却例

```json
{
  "data": {
    "signUp": {
      "userErrors": [
        {
          "message": "パスワードは8文字以上である必要があります",
          "field": ["password"],
          "code": "INVALID_PASSWORD"
        },
        {
          "message": "このメールアドレスは既に使用されています",
          "field": ["email"],
          "code": "EMAIL_TAKEN"
        }
      ],
      "account": null
    }
  }
}
```

---

# 2. Errors as data

## メリット

- スキーマでエラーを管理できるため予測しやすい
- 独自のエラーレスポンスのデータ構造を定義可能

## デメリット

- クライアントが `userErrors` フィールドを明示的にクエリする必要がある
- 開発者が常に `userErrors` を意識する必要があり認知負荷が高まる

---

# 3. Union / Result Types

Union 型を利用して成功時とエラー時の異なる結果を表現する方法

```graphql
type Mutation {
  signUp(email: String!, password: String!): SignUpPayload
}

# Union型としてそれぞれレスポンスを表現
union SignUpPayload = SignUpSuccess | UserNameTaken | PasswordTooWeak

type SignUpSuccess {
  account: Account
}

type UserNameTaken {
  message: String!
  suggestedUsername: String
}

# 省略
```

---

# 3. Union / Result Types（続き）

## メリット

- 成功パターンとエラーパターンを型として明確に分離、各エラーシナリオごとにカスタムフィールドを追加可能
- 強い型付けによる安全性

## デメリット

- スキーマが肥大化しやすい
- クエリが煩雑化しやすい
- クライアントがすべての可能な型に対してフラグメントを定義する必要がある

---

# 3. Union / Result Types

返ってくる型に合わせてクライアントも用意をしておく必要がある。

▽ クライアント側のクエリ例

```graphql
mutation {
  signUp(email: "user@example.com", password: "P@ssword") {
    ... on SignUpSuccess {
      account {
        id
      }
    }
    ... on UserNameTaken {
      message
      suggestedUsername
    }
    ... on PasswordTooWeak {
      message
      passwordRules
    }
  }
}
```

---

# 各設計手法の比較

| **手法**                 | **メリット**                                       | **デメリット**                         |
| ------------------------ | -------------------------------------------------- | -------------------------------------- |
| **標準仕様に従う**       | ・手軽に実装できる                                 | ・予測しにくい<br>・スキーマ外で管理   |
| **Errors as Data**       | ・独自のエラーデータ構造<br>・スキーマで管理       | ・クライアント側での認知負荷が高い     |
| **Union / Result Types** | ・成功とエラーを型として明確に分離<br>・強い型付け | ・スキーマの肥大化<br>・クエリの煩雑化 |

---

## で、結局何がいいん？

---

## 結論、正解はない

- システムエラー以外のエラーはデータとして返すべきというのが GraphQL Spec の作者の主張。

- 標準仕様の errors フィールドに全てのエラーメッセージを格納するのはあまり推奨されてない

---

# 自分がした意思決定とその理由

---

## 結論、私はあらゆるエラーを標準仕様の errors フィールドに格納するという意思決定をした

- **標準仕様に則ることが一番恩恵を受けられるから**

---

# 意思決定するにあたってのポイント

- 意思決定するにあたり 3 つのポイント
  - アーキテクチャとプロダクト特性の関係
  - 組織体制とメンバー
  - 直近の技術動向

---

# 意思決定ポイント 1: アーキテクチャとプロダクト特性の関係

- アーキテクチャ

  - マイクロフロントエンド（Next.js） + 単一 API の構成で、**複数サービス**を運営している

- プロダクトの特性
  - **各サービスごとに求められる基準値が大きく異なる**
  - 例
    - プロダクト A では、ユーザの CV 数が事業の要になるので、エラーの内容を詳細に返す必要がある
    - プロダクト B では、社内向けである程度リテラシーがあるので、エラーの内容が詳細を返す必要はない

```
💡 エラーメッセージそのものをAPI側で管理するより、各サービスのフロントでした方が柔軟性がある
```

---

# 意思決定ポイント 2: 組織体制とメンバー

- 事業の急拡大によって、やりたいこと・やるべきことが山積みで、まだまだ**開発エンジニアが足りてない**。
  - 新たにメンバーの加入があれば、 **すぐに立ち上がれるようにしておきたい**
- 所属メンバーは全員フルスタックエンジニアで、**フロントとサーバーサイドの両方の知識を有している**

<br>

```
💡 新規メンバーができるだけ実装や学習コストを下げることが、事業の売り上げに直結する

   サーバーサイドのコードを理解できるので、 標準仕様のデメリットであるスキーマ外のエラー管理を許容できる
```

---

# 意思決定ポイント 3: 直近の技術動向

- RSC の登場で、**今後も GraphQL を採用しつづけるとは限らない**
  - データフェッチは ServerComponents で行うことがベストプラクティス
    - 参考: [Data Fetching and Caching](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching#fetching-data-on-the-server)
  - RSC に GraphQL を組み合わせることはメリットよりデメリットの方が多くなる可能性がある
    - 参考: [Next.js の考え方](https://zenn.dev/akfm/books/nextjs-basic-principle/viewer/part_1_server_components)

<br>

```
💡 今GraphQLスキーマを作り込むことにメリットがあるかわからない。（頑張ったのに、技術スタックの変更がありうる）
```

---

# 意思決定ポイントまとめ

- まめるとこんな感じ

| 要因                               | 状況                                         | ポイント                                                 |
| ---------------------------------- | -------------------------------------------- | -------------------------------------------------------- |
| アーキテクチャと<br>プロダクト特性 | ・プロダクトごとに要件が異なる               | ・**エラーメッセージはフロントで管理して柔軟性担保**     |
| 組織体制と<br>メンバー             | ・リソース不足<br>・メンバーの守備範囲の広さ | ・**実装・学習コストを抑える**<br>・**デメリットの許容** |
| 技術動向                           | ・RSC の台頭<br>・技術スタックの変更可能性   | ・**作り込むメリットが見えなかった**                     |

---

## うん、標準仕様に乗っかるので良さそうだ！

![GraphQLエラーハンドリングの意思決定まとめ](data:image/jpeg;base64,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)

---

# 実装の紹介

---

# GraphQL Ruby での実装例（サーバーサイド）

- ユーザ作成のための mutation で考えてみる

```ruby
module Mutations
  class CreateUser < BaseMutation
    argument :name, String, required: true

    field :user, Types::UserType, null: true

    def resolve(name:)
      user = User.new(name:)

      if user.save
        { user: }
      else
        raise_errors(user) # 後述
      end
    end
  end
end
```

---

# GraphQL Ruby での実装例（サーバーサイド）

- `raise_errors`メソッドで、errors フィールドにエラーコードを格納する処理

```ruby
  private

    def raise_errors(user)
      # userオブジェクトからエラーオブジェクトを取り出す
      user.errors.each do |error|
        # エラーの種類に応じて、エラーコードを選択する
        extension_code = case error.type
                         when :taken
                          'NAME_DUPLICATED'
                        else
                          'USER_INPUT_ERROR'
                        end

        # errorsフィールドを追加
        context.add_error(GraphQL::ExecutionError.new(error.message, extensions: { code: extension_code }))
      end
    end
```

- サーバーサイドではこれだけ。

---

# GraphQL Ruby での実装例（フロント）

- エラーコードを元に、メッセージを出し分けるだけで良い

```js
createUser({ variables: { input: { name: data.name } } })
  .then(() => {
    console.log("作成成功！");
  })
  .catch((error) => {
    const errors = error.graphQLErrors;

    errors.forEach((error) => {
      const extensionCode = error.extensions.code;

      if (extensionCode === "NAME_DUPLICATED") {
        console.log("すでに登録済みの名前です");
      }

      if (extensionCode === "USER_INPUT_ERROR") {
        console.log("名前が不正です");
      }
    });
  });
```

---

# 実際に採用してみて

---

# 実際に採用してみて

- よかった点

  - 楽かつシンプルな実装できる点はやっぱりよかった
  - メンバーのキャッチアップは楽にできる
  - （個人的には）フロント側でエラーを意識せずにクエリを書かなくていいので認知負荷は低い

- 少し残念な点
  - extensions の自由度が高いのでチーム内でルールをちゃんと決める必要がありそう
    - エラーコードはなんでもいいので、「USER_INVALID」でも「INVALID_USER」でも同じ意味なのに複数のエラーコードができてしまう可能性がある
  - 属性ごとの細かいエラーを返す場合は、コードがすぐに複雑になる
    - 例: name のエラー、address のエラーそれぞれを分けて表示したいなど

---

# まとめ

---

# まとめ

- GraphQL のエラーレスポンスには複数の設計アプローチがあってそれぞれにメリデリがある
  - エラーレスポンス設計をする際は、技術的な面だけでなく**プロダクトの性質やチーム状況に合わせて設計方針を固める**と良さそう

- 個人的には標準実装はおすすめです
  - 小規模チームでかつ、フルスタックエンジニアのチームであれば、エラーをスキーマ外で管理するデメリットがそんなに大きいとは感じてない  
  - 手軽に実装できるのが本当に楽
  - ただし、extensions属性は自由度が高いので、**チーム内でルールは決めた方が良さそう**です

---

<!-- _class: lead -->

# ご清聴ありがとうございました

## ぜひ意思決定の過程や、エラーレスポンス設計周りについてのフィードバックもらえると嬉しいです！

---

# 参考文献

- [GraphQL Spec](https://spec.graphql.org/October2021/)
- [Production Ready GraphQL](https://book.productionreadygraphql.com/)
- [GraphQL Server とエラー処理](https://kakudou3.github.io/%E8%A8%AD%E8%A8%88/2024/09/27/design-error-handling-on-graphql-server.html)
- [Shopify GraphQL Design Tutorial](https://github.com/Shopify/graphql-design-tutorial/blob/master/lang/TUTORIAL_JAPANESE.md)
