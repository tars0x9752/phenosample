# Phenyl ハンズオン

Phenyl は複数のライブラリからなるライブラリ**群**です。

ライブラリの数は軽く 10 個を超え、どこから手を付けて良いかわからないかもしれません。

このハンズオンではその中からコアなライブラリ数個を厳選して触っていき、Phenyl の概要をなんとなく理解することを目指します。

## はじめの一歩

まずは DB クライアントから見ていきましょう。

Phenyl には 2 種類の DB クライアントがあります。

一つは MongoDB のクライアントである `@phenyl/mongodb`。もう一つはインメモリ DB のライブラリである `@phenyl/memory-db` です。

MongoDB 環境を用意しなくても良いように、今回はより簡単な `@phenyl/memory-db` を使うことにします。

> `@phenyl/memory-db` はインメモリ DB のため、プログラムを終了するとデータは揮発します。

### `entityClient`

`src/server.ts` に次のようにコードを書き、`@phenyl/memory-db` の `entityClient` を用意しましょう。

```ts
import { createEntityClient } from '@phenyl/memory-db'

const entityClient = createEntityClient()
```

`Phenyl` の世界で言う `Entity` とは、`id` を持つオブジェクトのことです。

このハンズオンでは例としてタスク管理システムを作っていくことにします。

タスク管理システムの `Entity` として `TaskCollection` と `PersonCollection` があるとします。（この辺の設定はテキトーで深い意味はありません。）

`src/type-map.ts` を開き、以下のように `Task` と `Person` の型を定義しましょう。

```ts
export type PersonId = `PID-${string}`
export type Person = {
  id: PersonId
  name: string
}
export type PersonCollection = {
  id: string
  personList: Person[]
}

export type TaskStatus = 'DONE' | 'WIP' | 'TODO'
export type TaskId = `TID-${string}`
export type Task = {
  id: TaskId
  name: string
  status: TaskStatus
  assignee?: PersonId
}
export type TaskCollection = {
  id: string
  taskList: Task[]
}
```

次に `entityClient` で管理したい `Entity` 全てを列挙した `EntityMap` を定義します。ここでは、`taskCollection` と `personCollection` の二つの `Entity` があるので、これらを `EntityMap` に書きます。

```ts
export type EntityMap = {
  taskCollection: TaskCollection
  personCollection: PersonCollection
}
```

`src/server.ts` に戻り、この `EntityMap` をインポートして `createEntityClient` の型引数に渡しましょう。そうすることで `entityClient` が扱う `Entity` を明示できます。

```ts
const entityClient = createEntityClient<EntityMap>()
```

これで `entityClient` を使う準備が整いました。早速 DB に `Entity` を追加するコードを書いてみましょう。

まずは `personCollection` を 追加してみます。`insertOne` で 1 つの `Entity` を追加できます。

```ts
entityClient.insertOne({
  entityName: 'personCollection',
  value: {
    id: 'person-collection-1',
    personList: [
      {
        id: 'PID-1',
        name: 'a',
      },
      {
        id: 'PID-2',
        name: 'b',
      },
    ],
  },
})
```

> このハンズオンの執筆時点ではまだ `id` の型推論は `template literal types` に対応しておらず強制的に `string` になってしまうようです。(無念)

`entityClient` の諸々のメソッドは非同期なので、ここで一連のコードを `async` 関数にまとめることにしましょう。ゆくゆくはサーバーにしたいので、`serve` と名付けることにします。

```ts
const serve = async () => {
  const entityClient = createEntityClient<EntityMap>()

  await entityClient.insertOne({
    entityName: 'personCollection',
    value: {
      id: 'person-collection-1',
      personList: [
        {
          id: 'PID-1',
          name: 'a',
        },
        {
          id: 'PID-2',
          name: 'b',
        },
      ],
    },
  })
}

serve()
```

つぎに `taskCollection` を追加してみましょう。

```ts
await entityClient.insertOne({
  entityName: 'taskCollection',
  value: {
    id: 'task-collection-1',
    taskList: [
      {
        id: 'TID-1',
        name: 'Do hands-on',
        status: 'TODO',
      },
    ],
  },
})
```

実際に `Entity` を追加できているのかどうか確認してみましょう。

`find` で `DB` が保持している `Entity` を検索できます。

```ts
const persons = await entityClient.find({
  entityName: 'person',
  where: {},
})

const tasks = await entityClient.find({
  entityName: 'task',
  where: {},
})

console.log(JSON.stringify({ persons, tasks }, null, 2))
```

`yarn serve` を実行します。以下のようなログが表示されるはずです。

```json
{
  "personCollection": {
    "entities": [
      {
        "id": "person-collection-1",
        "personList": [
          {
            "id": "PID-1",
            "name": "a"
          },
          {
            "id": "PID-2",
            "name": "b"
          }
        ]
      }
    ],
    "versionsById": {
      "person-collection-1": "0kqfddogaXRhFshQhBya4HaJ"
    }
  },
  "taskCollection": {
    "entities": [
      {
        "id": "task-collection-1",
        "taskList": [
          {
            "id": "TID-1",
            "name": "Do hands-on",
            "status": "TODO"
          }
        ]
      }
    ],
    "versionsById": {
      "task-collection-1": "0kqfddogbbiRLzmrRKqSLvd7"
    }
  }
}
```

> `versionId` はランダムです。

`Entity` を追加できてそうですね！第一部完！🎉

> **Note:** 興味のある人は試しに `entityClient` のメソッドをいろいろ試してみてください。

> **Tips:** 他の Phenyl ファミリーに存在する `entityClient` も基本的に同じような API を持っているため、同じ操作で使う事ができます。

## `@phenyl/rest-api`

ここまでで `entityClient` を通じた DB 操作を見てきました。次に DB から REST API を作成したいと思います。

Phenyl マナーで REST API 化できる `@phenyl/rest-api` というライブラリを使います。

> これは Phenyl の中でも最もコアとなるライブラリです。

早速、`@phenyl/rest-api` から `PhenylRestApi` をインポートし、先ほどの `serve` 関数の最後に `RestApiHandler` を用意しましょう。

```ts
const restApiHandler = new PhenylRestApi()
```

`PhenylRestApi` のコンストラクタの引数の型を見てみましょう。何やら、`FunctionalGroup` というものと `params` が必要なことがわかります。

`params` に `entityClient` と `sessionClient` を渡します。`FunctionalGroup` は次で見ていきましょう。

```ts
const functionalGroup = {}

const restApiHandler = new PhenylRestApi(functionalGroup, {
  entityClient,
  sessionClient: entityClient.createSessionClient(),
})
```

## `FunctionalGroup` と `TypeMap`

`FunctionalGroup` とは一体なんでしょうか？ これは `PhenylRestApi` に伝えたいドメインの実装定義...みたいなやつです。よくわからないと思いますが、とりあえず `@phenyl/interfaces` に `FunctionalGroup` の型があるので見てみましょう。

どうやら、`TypeMap` という型引数が必須のようです。この `TypeMap` から見ていきましょう。

### `TypeMap`

`TypeMap` は全ての `Entity` についてのリクエストやレスポンスの型、および Auth に関する情報をまとめた型定義です。

早速 `TypeMap` を定義していきましょう。

`src/type-map.ts` を開き、まずは全ての `Entity` についてのリクエストやレスポンスをまとめた `EntityRestInfoMap` という型を書きます。

```ts
export type EntityRestInfoMap = {
  taskCollection: {
    request: TaskCollection
    response: TaskCollection
  }
  personCollection: {
    request: PersonCollection
    response: PersonCollection
  }
}
```

`task` も `person` もリクエストとレスポンスの型は同じとしたいと思います。

そのような場合は `request` と `response` の二つを定義せずとも、以下のように書けます。

```ts
export type EntityRestInfoMap = {
  taskCollection: {
    type: TaskCollection
  }
  personCollection: {
    type: PersonCollection
  }
}
```

次にこの `EntityRestInfoMap` を使って `TypeMap` を定義しましょう。名前は適当に `MyTypeMap` とします。

> auth やカスタム XXX については今回は考えないこととします。

```ts
export interface MyTypeMap extends GeneralTypeMap {
  entities: EntityRestInfoMap
  customQueries: {}
  customCommands: {}
  auths: {}
}
```

これで `TypeMap` の定義はできたので改めて `FunctionalGroup` を作っていきます。

### `FunctionalGroup`

`FunctionalGroup` には 4 つのプロパティがあり、それぞれ

- `nonUsers`: `authenticate` 機能を持たない `Entity`
- `users`: `authenticate` 機能を持つ `Entity`
- `customQueries`: カスタムクエリ―
- `customCommands`: カスタムコマンド

です。

今回のタスク管理システムでは、`authenticate` 機能を持つ `Entity` は要らないので、`nonUsers` だけ使います。

> FunctionalGroup の細かい機能についてはここではこれ以上深追いしません。

```ts
const functionalGroup: FunctionalGroup<MyTypeMap> = {
  users: {},
  nonUsers: {
    personCollection: {},
    taskCollection: {},
  },
  customCommands: {},
  customQueries: {},
}
```

`FunctionalGroup` と `TypeMap` を用意し、`PhenylRestApi` の準備までできました！🎉

ただ、まだこれだけではサーバーとして機能はしません。

## `@phenyl/http-server`

次に、`PhenylRestApi` をホストするサーバーを作ります。

シンプルに `Nodejs` の `http` でサーバーを実装する方法の他に、サーバーは `express` で実装し、`Express` の `middleware` として `PhenylRestApi` を動かす方法などもありますが、今回は前者のシンプルな方法で実装することにします。

そのためには `@phenyl/http-server` というライブラリを使います。

`src/server.ts` にて、`http` の `createServer` と `@phenyl/http-server` をインポートします。

```ts
import { createServer } from 'http'
import PhenylHttpServer from '@phenyl/http-server'
```

あとは `serve` 関数の末尾に以下を追記するだけです。🎉

```ts
// PhenylRestApiをホストするサーバー
const server = new PhenylHttpServer(createServer(), { restApiHandler })

server.listen(8080)

console.log('server started')
```

サーバー編、完！🎉

## `@phenyl/http-client`

さて、ようやくクライアント編に突入です。

`@phenyl/http-client` を使ってクライアントを実装していきます。

`src/index.ts` にクライアントのコードを書いていきましょう。まずは `PhenylHttpClient` を用意し、`main` 関数作ります。

```ts
import PhenylHttpClient from '@phenyl/http-client'

const main = async () => {
  const client = new PhenylHttpClient<MyTypeMap>({
    url: 'http://localhost:8080',
  })
}

main()
```

これだけで `PhenylHttpClient` の準備は完了です！試しにサーバーから `personCollection` を取得してくるコードを書いてみましょう。

```ts
const res = await client.find({
  entityName: 'personCollection',
  where: {},
})

console.log(JSON.stringify(res, null, 2))
```

実際に動かしてみます。`yarn serve` でサーバーを立ち上げてから、`yarn start` してクライアントのログを確認してみましょう。

> ここから先はサーバーは立ち上げたままで良いです。

```json
{
  "entities": [
    {
      "id": "person-collection-1",
      "personList": [
        {
          "id": "PID-1",
          "name": "a"
        },
        {
          "id": "PID-2",
          "name": "b"
        }
      ]
    }
  ],
  "versionsById": {
    "person-collection-1": "0kqfdkf7h14kg7ItARb939Ic"
  }
}
```

サーバーからデータを取得できました！🎉

では試しに、`task-collection-1` の `taskList` に新たにタスクを追加してみたいと思います。`main` を次のように書き換えます。

```ts
const main = async () => {
  const client = new PhenylHttpClient<MyTypeMap>({
    url: 'http://localhost:8080',
  })

  await client.updateById({
    entityName: 'taskCollection',
    id: 'task-collection-1',
    operation: {
      $addToSet: {
        taskList: {
          id: 'TID-2',
          name: 'Implement client',
          status: 'WIP',
          assignee: 'PID-1',
        },
      },
    },
  })

  const res = await client.find({
    entityName: 'taskCollection',
    where: {},
  })

  console.log(JSON.stringify(res, null, 2))
}
```

`operation` はこのように MongoDB 風に書けます。

もう一度 `yarn start` してみましょう。

```json
{
  "entities": [
    {
      "id": "task-collection-1",
      "taskList": [
        {
          "id": "TID-1",
          "name": "Do hands-on",
          "status": "TODO"
        },
        {
          "id": "TID-2",
          "name": "Implement client",
          "status": "WIP",
          "assignee": "PID-1"
        }
      ]
    }
  ],
  "versionsById": {
    "task-collection-1": "0kqfdq3blbYvXFn3FEDqmGHw"
  }
}
```

`update` もできました！🎉

次に既存のタスクのステータスの更新をしてみたいと思います。

`"Do hands-on"` のステータスを `WIP` に、`assignee` を `PID-1` にしてみましょう。

```ts
const main = async () => {
  const client = new PhenylHttpClient<MyTypeMap>({
    url: 'http://localhost:8080',
  })

  const tasks = await client.find({
    entityName: 'taskCollection',
    where: {
      id: 'task-collection-1',
    },
  })

  await client.updateById({
    entityName: 'taskCollection',
    id: 'task-collection-1',
    operation: {
      $set: {
        taskList: tasks.entities[0].taskList.map((e) => {
          if (e.id === 'TID-1') {
            return {
              ...e,
              status: 'WIP',
              assignee: 'PID-1',
            }
          }

          return e
        }),
      },
    },
  })

  const res = await client.find({
    entityName: 'taskCollection',
    where: {},
  })

  console.log(JSON.stringify(res, null, 2))
}
```

実行してみます。

```json
{
  "entities": [
    {
      "id": "task-collection-1",
      "taskList": [
        {
          "id": "TID-1",
          "name": "Do hands-on",
          "status": "WIP",
          "assignee": "PID-1"
        },
        {
          "id": "TID-2",
          "name": "Implement client",
          "status": "WIP",
          "assignee": "PID-1"
        }
      ]
    }
  ],
  "versionsById": {
    "task-collection-1": "0kqfelnibEJMVd4dEy1y4FTM"
  }
}
```

いい感じです。あとは削除もやってみましょう。

`task-collection-1` を消してみます。

```ts
const main = async () => {
  const client = new PhenylHttpClient<MyTypeMap>({
    url: 'http://localhost:8080',
  })

  await client.delete({
    entityName: 'taskCollection',
    id: 'task-collection-1',
  })

  const res = await client.find({
    entityName: 'taskCollection',
    where: {},
  })

  console.log(JSON.stringify(res, null, 2))
}
```

```json
{
  "entities": [],
  "versionsById": {}
}
```

消えていますね。削除もできました！👍🏿

## State Synchronization via `@phenyl/redux`

さて、このハンズオンもとうとう最後の章まできました。

最後は `@phenyl/redux` を見ていきます。

`@phenyl/redux` はサーバーの DB とクライアントの store 間で `Git-like` な `synchronization` を実現します。

### 掃除

これまで書いてきたコードを一旦掃除します。サーバー起動時に何も Entity 追加しないようにしましょう。 `src/server.ts` の `serve` から `entityClient` を操作していた部分を消しておきます。

```ts
const serve = async () => {
  const entityClient = createEntityClient<EntityMap>()

  const functionalGroup: FunctionalGroup<MyTypeMap> = {
    users: {},
    nonUsers: {
      personCollection: {},
      taskCollection: {},
    },
    customCommands: {},
    customQueries: {},
  }

  // PhenylRestApi
  const restApiHandler = new PhenylRestApi(functionalGroup, {
    entityClient,
    sessionClient: entityClient.createSessionClient(),
  })

  // PhenylRestApiをホストするサーバー
  const server = new PhenylHttpServer(createServer(), { restApiHandler })

  server.listen(8080)

  console.log('server started')
}
```

また、`src/index.ts` についても `client` のメソッドを呼んでいた箇所をまるっと消してしまいましょう。

```ts
const main = async () => {
  const client = new PhenylHttpClient<MyTypeMap>({
    url: 'http://localhost:8080',
  })
}
```

### `store` を作る

`store` を作っていきます。

`src/index.ts` に以下をインポートします。

```ts
import { createRedux } from '@phenyl/redux'
import { createStore, applyMiddleware, combineReducers } from 'redux'
import { ActionWithTypeMap, EntityNameOf, LocalState, UserEntityNameOf } from '@phenyl/interfaces'
```

`main` に以下を追記します。

```ts
const { reducer, middleware, actions } = createRedux({
  client,
  storeKey: 'phenyl',
})

const storeEnhancer = applyMiddleware(middleware)

const store = createStore<
  {
    phenyl: LocalState<EntityRestInfoMap, {}>
  },
  ActionWithTypeMap<MyTypeMap, EntityNameOf<MyTypeMap>, UserEntityNameOf<MyTypeMap>>,
  {},
  {}
>(combineReducers({ phenyl: reducer }), storeEnhancer)
```

これで store の準備ができました。
