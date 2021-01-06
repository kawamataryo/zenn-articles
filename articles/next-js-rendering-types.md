---
title: "Next.jsのレンダリング方式をざっくり理解する CSR, SSR, SSG, ISR"
emoji: "💍"
type: "tech"
topics: ["nextjs", "react","typescript"]
published: false
---

最近、Next.js を勉強し始めました。最初に詰まった Next.js の複数のレンダリング方式についてまとめてみました。

以下で紹介するアプリは全てこちらのリポジトリにあります。

https://github.com/kawamataryo/next-js-rendering-type-sample

データ取得のタイミング

|type| timing |
|---|---|
|CSR|アクセス時（クライアント側）|
|SSG|アプリケーション デプロイ時|
|SSR|アクセス時（サーバー側）|
|ISR||

# `CSR` - Client Side Rendering -

- CSR とは？
- 実装方法

https://next-js-rendering-type-sample.vercel.app/csr

```tsx
const CSRPage: React.FC = () => {
  const [time, setTime] = useState("");
  useEffect(() => {
    (async () => {
      const res = await fetchCurrentTime();
      setTime(res);
    })();
  }, []);

  return (
    <PageLayout color="warning" type="csr">
      <h1 className="title">CSR</h1>
      <h2 className="subtitle">
        Client Side Rendering
        <TimeText time={time} />
      </h2>
      <PageLink />
    </PageLayout>
  );
};

export default CSRPage;
```


# `SSR` - Server Side Rendering -

https://next-js-rendering-type-sample.vercel.app/ssr

```tsx
const SSRPage: React.FC<PropsType> = (props) => {
  return (
    <PageLayout color="primary" type="ssr">
      <h1 className="title">SSR</h1>
      <h2 className="subtitle">
        Server Side Rendering
        <br />
        <TimeText time={props.time} />
      </h2>
      <PageLink />
    </PageLayout>
  );
};

export const getServerSideProps: GetServerSideProps = async () => {
  const time = await fetchCurrentTime();

  return {
    props: {
      time: time,
    },
  };
};

export default SSRPage;
```

# `SSG` - Static Site Generation -

https://next-js-rendering-type-sample.vercel.app/ssg

```tsx
const SSGPage: React.FC<PropsType> = (props) => {
  return (
    <PageLayout color="info" type="ssg">
      <h1 className="title">SSG</h1>
      <h2 className="subtitle">
        Static Site Generation
        <br />
        <TimeText time={props.time} />
      </h2>
      <PageLink />
    </PageLayout>
  );
};

export const getStaticProps: GetStaticProps = async () => {
  const time = await fetchCurrentTime();

  return {
    props: {
      time,
    },
  };
};

export default SSGPage;
```

# `ISR` - Incremental Static Regeneration -

https://next-js-rendering-type-sample.vercel.app/isr

```tsx
const ISRPage: React.FC<PropsType> = (props) => {
  return (
    <PageLayout color="danger" type="isr">
      <h1 className="title">ISR</h1>
      <h2 className="subtitle">
        Incremental Static Regeneration
        <br />
        <TimeText time={props.time} />
      </h2>
      <PageLink />
    </PageLayout>
  );
};

export const getStaticProps: GetStaticProps = async () => {
  const time = await fetchCurrentTime();

  return {
    props: {
      time,
    },
    revalidate: 5,
  };
};

export default ISRPage;
```

# 参考

- [Getting Started | Next.js](https://nextjs.org/docs)
- [Next.jsにおけるSSG（静的サイト生成）とISRについて（自分の）限界まで丁寧に説明する - Qiita](https://qiita.com/thesugar/items/47ec3d243d00ddd0b4ed)