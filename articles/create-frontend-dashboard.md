---
title: "zx + Datadog + GitHub Actions でフロントエンドのコードベースの健全性を可視化する"
emoji: "📊"
type: "tech"
topics: ["zx", "datadog", "typescript", "shell"]
published: true
---

フロントエンドのダッシュボードを作ってみたらいい感じだったので紹介です。

# 作ったもの

zx と Datadog、GitHub Actions を使って以下画像のように、フロントエンドのコードベースの各指標を可視化するダッシュボードを作りました。

![](https://i.gyazo.com/8f1b5b8719ae6f8b5a797c3e73f65fe0.png)
*値はデモ用に書き換えています*

現在、計測している指標はこちらです。

* Vue SFCファイルにしめるTypeScriptの割合
* Vue SFCファイルにしめるComposition APIの割合
* strict: trueにした場合のType Errorの数（tsc & vue-tsc）
* Jestの各種カバレッジ

各指標は毎朝9時に更新していて、時系列での推移も確認できます。

# なぜ作った？

技術的負債解消等コードベースのリファクタリングの活動は、機能追加に比べ進捗を把握しにくい、成果が伝わりにくいという問題があり、それを解消したいと考えたからです。

このダッシュボードには日時でデータが蓄積されていくので、どのようなペースで負債を解消していったのかが一目瞭然です。また、しきい値を定めれば、目標への到達具合も把握できてモチベーションにも繋がります。その他、コードベースの良くない傾向（例：カバレッジの継続的な低下）なども、このダッシュボードを定期確認することで検知できます。

# 実装方法

計測スクリプトの実装方法をまとめます。

基本的な構成はこちらです。

![](https://i.gyazo.com/124bacd9974cd1f3f124c501bd77b5a9.png)

zxを使って、node.jsで実行できる計測スクリプトを作り、それをGitHub Actionsで計測対象のリポジトリ（毎回GitHub Actions上でPull）に対して実行。その結果をDatadogにメトリクスとして送信しています。
Datadogのメトリクスを使う利点は、結果の永続化のために専用のRDBを新たに作る必要がないという点です。

計測スクリプトのフォルダー構成はこちらです。
`target_project` となっているところは、計測対象のプロジェクトを想定しています。このフォルダーは `.gitignore` に指定して、毎回GitHub Actionsで計測スクリプトを実行する際に、最新コードをpullする想定です。

```
app/
├── github/
│   └── workflows/
│       └── metrics.yml #定期実行のためのGitHub Actions定義
├── src/
│   ├── index.ts #計測スクリプト
│   ├── lib/
│   │   └── ...
│   └── ...
└── target_project/ #毎回GitHub Actions上で最新コードをpullする
    ├── tsconfig.json
    └── frontend/
        └── ...
```


## zxでの各指標の計測
zxはJSでシェルスクリプトを記述できるツールです。

https://github.com/google/zx

こちらを用いて、各種指標を計測するスクリプトを書いています。
以下実装コードです。

１つ目はTypeScript化の指標の集計です。

計測対象のプロジェクトのパスを受け取り、そのパスに対して`tsc`や`vue-tsc`コマンドを実行しています。
そしてzxで標準出力の値を文字列として受け取り、その結果に対して`match(/error TS/g || []).length`でエラー数を計測し、戻り値としています。

```ts:src/aggregate.ts
import { $, nothrow, ProcessOutput } from "zx";

// 型エラーの集計
export const aggregateTypeErrors = async (target: string) => {
  const result = {
    tsc_error_count: 0,
    vue_tsc_error_count: 0,
  };

  // vue-tscのエラー数
  const vueTscResult = await nothrow($`yarn vue-tsc --noEmit -p ${target}/tsconfig.json --strict --allowJs`);
  result.vue_tsc_error_count = (vueTscResult.stdout.match(/error TS/g) || []).length;

  // tscのエラー数
  const tscResult = await nothrow($`yarn tsc --noEmit --p ${target}/tsconfig.json --strict --allowJs`);
  result.tsc_error_count = (tscResult.stdout.match(/error TS/g) || []).length;

  // ts-expect-error or ts-ignoreでのコメントアウト数を追加
  const tsExpectErrorCount = parseInt((await nothrow($`grep -r --include='*.vue' --include='*.ts' -e '@ts-expect-error' -e '@ts-ignore' ${target}/frontend/ | wc -l`)).stdout.trim(), 10);
  result.tsc_error_count += tsExpectErrorCount;
  result.vue_tsc_error_count += tsExpectErrorCount;

  return result;
};
```

2つ目はVueファイルの集計です。
こちらでは、`find`と`grep`と`wc`を組み合わせてVueの総数、その内にしめるTypeScript化されたVueファイルの総数、Composition APIが使われているVueファイルの総数を計測しています。

```ts:src/aggregate.ts
export const aggregateVueFiles = async (target: string) => {
  const vueFileCount = await $`find ./${target}/frontend -name "*.vue" | wc -l`;
  const compositionApiFileCount =
    await nothrow($`grep -r -l --include='*.vue' "defineComponent({" ./${target}/frontend/ | wc -l`);
  const vueTsFileCount =
    await nothrow($`grep -r -l --include='*.vue' -E 'script lang=.ts.' ./${target}/frontend/ | wc -l`);

  return {
    vue_file_count: parseInt(vueFileCount.stdout.trim(), 10),
    vue_composition_api_file_count: parseInt(
      compositionApiFileCount.stdout.trim(),
      10
    ),
    vue_ts_file_count: parseInt(vueTsFileCount.stdout.trim(), 10),
  };
};
```

最後はカバレッジの計測です。
zxでjestのカバレッジをJSONに出力するコマンドを実行し、そのJSONに対して`jq`で各指標を取り出し、戻り値としています。

```ts:aggregate.ts
export const aggregateCoverages = async (target: string) => {
  await nothrow($`yarn --cwd ./${target} test:coverage --coverageReporters=json-summary`);
  const result =
    await nothrow($`cat ${target}/frontend_coverage/coverage-summary.json | jq -r '.total | keys[] as $k | {("coverage_" + $k):(.[$k].pct)}' | jq -s add`);
  return JSON.parse(result.stdout);
};
```

共通するポイントはzxの `nothorw` を使っているところです。通常の実行だと、exit codeが1以外のシェルスクリプトを実行すると例外が投げられてしまいます。実行するスクリプトによっては、適切に終了していて標準出力から値が取れるものの、exit codeが1以外となる場合があるので（`grep`でマッチしない値がある時など）、意図せぬ例外を防ぐためnothrowでシェルスクリプトのコマンドを囲っています。

## Datadogへメトリクスとして送信

次にDatadogのメトリクスへの送信部分です。

Datadogへのクライアントの実装は以下です。
単純に受け取った値を元に、node-fetchでDatadogのREST APIを叩いているのみです。

```ts:src/lib/ddClient.ts
import * as dotenv from "dotenv";
import fetch from "node-fetch";

dotenv.config();
const DD_API_KEY = process.env.DD_API_KEY as string;
export class DDClient {
  private apiUrl: string;

  constructor() {
    this.apiUrl = `https://api.datadoghq.com/api/v1/series?api_key=${DD_API_KEY}`;
  }

  async sendMetrics(metricsName: string, value: number) {
    const requestBody = JSON.stringify({
      series: [
        {
          metric: metricsName,
          points: [[Math.floor(Date.now() / 1000), value]],
          type: "gauge",
        },
      ],
    });
    return await this.post(requestBody);
  }

  private async post(requestBody: string) {
    return await fetch(this.apiUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: requestBody,
    });
  }
}
```

各種指標の集計から、Datadogへの送信までは以下ですね。
さきほど紹介した、各指標の集計メソッドを実行して、その結果をDatadogに送信しています。


```ts:src/index.ts
import {
  aggregateTypeErrors,
  aggregateVueFiles,
  aggregateCoverages,
} from "./lib/aggregate";
import { DDClient } from "./lib/ddClient";

const aggregateMetricsAndSendDataDog = async (client: any, target: string) => {
  const [typeErrors, vueFiles, coverages] = await Promise.all([
    aggregateTypeErrors(target),
    aggregateVueFiles(target),
    aggregateCoverages(target),
  ]);

  try {
    for (const [key, value] of Object.entries<string | number>({
      ...typeErrors,
      ...vueFiles,
      ...coverages,
    })) {
      await client.sendMetrics(`application.${target}.frontend.${key}`, value);
    }
  } catch (err) {
    console.error(err);
  }
};

const main = async () => {

  const ddClient = new DDClient();
  await aggregateMetricsAndSendDataDog(ddClient, "プロジェクトのパス");
};

main();
```

## GitHub Actionsでの定期実行

あとは、上記のスクリプトをGitHub Actionsで定期実行すれば完了です。
GitHub Actionsのスケジュールトリガーで毎日UTCの1時に実行されるように設定しています。
スクリプト実行前に、計測対象のリポジトリをpullしているので、実行時点での最新の指標を計測できます。

```yml:github/workflows/metrics.yml
name: Metrics

on:
  schedule:
    - cron:  '0 1 * * *'
  workflow_dispatch:

jobs:
  measure-metrics:
    runs-on: ubuntu-20.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '14'

      - name: Checkout hoge
        uses: actions/checkout@v2
        with:
          repository: lapras-inc/hoge
          token: ${{ secrets.GH_PAT }}
          path: hoge
          submodules: recursive


      - name: Install packages.
        run: yarn install && yarn --cwd ./hoge

      - name: Measure metrics.
        run: yarn ts-node src/index.ts
        env:
          DD_SITE: https://app.datadoghq.com
          DD_API_KEY: ${{ secrets.DATADOG_API_KEY }}
```

これで、日時でDatadog上のメトリクスにコードベースの指標の集計結果が蓄積されていくので、あとはその値を利用してダッシュボードを作れば完成です。

# おわりに
シェルコマンドを組み合わせて何かしたい時に、zx は本当に便利ですね。
今後もこの指標をもとに、地道な改善活動を頑張っていきたいです。