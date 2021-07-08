---
title: "「Vue.jsのコレはReactでどう書くの？」をまとめてみた"
emoji: "🙋‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "vue", "typescript"]
published: false
---

最近 React 勉強中なので、Vue Composition API の書き方と React Hooks の書き方の対比のメモです。

この記事は以下バージョン時点の情報です。

`React v17.0.2`
`Vue v3.1.1`

# コンポーネントの定義

### Vue

```vue
<template>
  <div>Hello</div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'Hello',
})
</script>
```

### React

```tsx
import * as React from 'react';

const Hello = () => <div>Hello</div>

export default Hello
```

# リアクティブなデータ

### Vue

```vue
<template>
    <button @click="countUp">Count is: {{count}}</button>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'Counter',
  setup: () => {
    const count = ref(0)
    const countUp = () => { count.value++ }

    return {
      count,
      countUp
    }
  }
})
</script>
```

### React

```tsx
import * as React from 'react';

const Counter = () => {
  const [count, setCount] = React.useState(0)
  const countUp = () => {
    setCount((prevCount) => prevCount + 1)
  }

  return <button onClick={countUp}>Count is: {count}</button>
}

export default Counter
```

# キャッシュされるプロパティ

### vue

```vue
<template>
  <button @click="countUp1">Count1 is: {{ count1 }}</button>
  <button @click="countUp2">Count2 is: {{ count2 }}</button>
  <p>Double Count2 is: {{ doubleCount }}</p>
</template>

<script lang="ts">
import { computed, defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'DoubleCounter',
  setup: () => {
    const count1 = ref(0)
    const count2 = ref(0)
    const countUp1 = () => {
      count1.value++
    }
    const countUp2 = () => {
      count2.value++
    }
    const doubleCount = computed(() => {
      for (let i = 0; i < 2000000000; i++) {}
      return count2.value * count2.value
    })

    // この書き方だとcountUp1の実行でもめちゃ重い
    // const doubleCount = () => {
    //   let i = 0
    //   while (i < 2000000000) { i++ }
    //   return count2.value * count2.value
    // }

    return {
      count1,
      count2,
      countUp1,
      countUp2,
      doubleCount
    }
  }
})
</script>
```

### React

```tsx
import * as React from 'react';

const DoubleCounter = () => {
  const [count1, setCount1] = React.useState(0)
  const [count2, setCount2] = React.useState(0)
  const countUp1 = () => { setCount1((prevCount) => prevCount + 1) }
  const countUp2 = () => { setCount2((prevCount) => prevCount + 1) }

  const doubleCount =  React.useMemo(() => {
    for (let i = 0; i < 2000000000; i++) {}
    return count2 * count2
  }, [count2])

  // この書き方だとcountUp1の実行でもめちゃ重い
  // const doubleCount = () => {
  //   for (let i = 0; i < 2000000000; i++) {}
  //   return count2 * count2
  // }

  return (
    <>
    <button onClick={countUp1}>Count1 is: {count1}</button>
    <button onClick={countUp2}>Count2 is: {count2}</button>
    <p>Twice count 2 is: {doubleCount}</p>
    </>
  )
}

export default DoubleCounter
```
# ライフサイクルフック
### Vue

```vue
<template>
    <button @click="countUp">Count is: {{count}}</button>
</template>

<script lang="ts">
import { defineComponent, onMounted, onUnmounted, onUpdated, ref } from 'vue'

export default defineComponent({
  name: 'Lifecycle',
  setup: () => {
    const count = ref(0)
    const countUp = () => { count.value++ }

    // Mount時に実行
    onMounted(() => {
      console.log('Mountされました')
    })

    // Update時に実行
    onUpdated(() => {
      console.log('Updateされました')
    })

    // Unmount時に実行
    onUnmounted(() => {
        console.log('Unmountされました')
    })

    return {
      count,
      countUp
    }
  }
})
</script>
```


### React

```tsx
import * as React from 'react';
import { useEffect, useRef } from "react";


const Lifecycle = () => {
  const [count, setCount] = React.useState(0)
  const countUp = () => {
    setCount((prevCount) => prevCount + 1)
  }

  // Mount時に実行
  useEffect(() => {
    console.log('Mountされました')
  }, [])

  // Update時に実行
  const isInitialMount = useRef(true);

  useEffect(() => {
    if (isInitialMount.current) {
      isInitialMount.current = false;
    } else {
      console.log('Updateされました')
    }
  });

  // Unmount時に実行
  useEffect(() => {
    return () => {
      console.log('Unmountされました')
    }
  }, [])

  return <button onClick={countUp}>Count is: {count}</button>
}

export default Lifecycle
```

# Styleの設定

# 子コンポーネントへのデータの受け渡し

# 子コンポーネントのイベント購読

# リストレンダリング

# 条件付きレンダリング

## Vue

```vue
<template>
  <div>
    <p v-if="toggle">Hello</p>
    <button @click="toggleIsShow" >toggle</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'

export default defineComponent({
  name: 'App',
  setup() {
    const isShow = ref(true)
    const toggle = () => {
      isShow.value = !isShow.value
    }

    return {
      isShow,
      toggle
    }
  }
})
</script>
```

## React

```tsx
import React, { useState } from 'react'

function App() {
  const [isShow, setIsShow] = useState(true)
  const toggle = () => {
    setIsShow((previous) => !previous)
  }

  return (
    <div>
      {isShow && <p>Hello</p>}
      <button onClick={toggle}>toggle</button>
    </div>
  )
}

export default App
```

# フォーム入力のバインディング


# 深くネストされたコンポーネント間でのデータの受け渡し

# 参考
https://react-typescript-cheatsheet.netlify.app
https://dev.to/voluntadpear/comparing-react-hooks-with-vue-composition-api-4b32