---
title: "deno-puppeteerとGitHub Actionsで外形監視をお手軽に"
emoji: "📹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "githubactions", "puppeteer", "slack"]
published: false
---

はるか昔に受託で作ったWPサイトが最近たびたび落ちるので、deno-puppeteerとGitHub ActionsとSlackでお手軽に外型監視ツールを作ってみました。

# 作ったもの 📹

以下のように毎日指定時刻に、指定したURLにアクセスして、HTTPレスポンスの確認とスクリーンショットの撮影をして、Slackに通知してくれるツールです。

![](/images/edade2d3f991e9/2023-05-07-17-22-14.png)
_スクショ画像はdummyです_

もし、200以外のステータスコードが返ってきた場合は、異常とみなしメンション付きで通知してくれます。

![](/images/edade2d3f991e9/2023-05-07-17-22-37.png)

これで、サイトが落ちてたら依頼者からのお叱りの連絡が来る前に、自分で対応できるようになります。よかった！

# 仕組み・実装

仕組みはとても単純で、GitHub Actionsで定期実行するdenoのスクリプトで指定したURLにアクセスして、HTTPレスポンスの確認とスクリーンショットの撮影を行い結果をSlackに送信しているだけです。

![](/images/edade2d3f991e9/2023-05-07-17-34-09.png)

:::message alert
PublicリポジトリのGitHub Actionsの定期実行の場合は注意が必要です。リポジトリに変化がないと、60日で停止します。Privateリポジトリであれば、この制限はないようです。
https://docs.github.com/en/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
:::

main関数はこちらです。
`findInvalidSites` 関数で、指定したURLにアクセスして、200以外のステータスコードが返ってくるものを抽出しています。

```ts:main.ts
const main = async () => {
  const slackClient = new SlackClient(
    Deno.env.get("SLACK_TOKEN")!,
    Deno.env.get("NOTIFICATION_SLACK_CHANNEL")!,
  );

  // 外型監視
  const invalidSites = await findInvalidSites(TARGET_SITES);

  // スクリーンショットの撮影
  const screenshots = await takeScreenshots(TARGET_SITES);
  const uploadImageLinks = await slackClient.uploadImage(screenshots);

  // Slackへの通知
  if (invalidSites.length === 0) {
    await slackClient.notify({
      text: "🎉 全てのサイトが正常に稼働しています。",
    });
  } else {
    await slackClient.notify(createInvalidSiteMessage(invalidSites));
  }
  await slackClient.notify({
    text: uploadImageLinks.map((link) => `<${link}| >`).join(" "),
  });

  console.log("Done.🎉");
};

if (import.meta.main) {
  main();
}
```




# 参考

- [Slack Web APIを使ってslackに任意の画像を複数枚投稿する](https://zenn.dev/yui/articles/2d965fedca620c)
  - Slackへの画像送信の部分で参考にさせていただきました🙏
