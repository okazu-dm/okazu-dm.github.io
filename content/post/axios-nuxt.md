+++
date = "2019-01-07T22:21:33+09:00"
title = "Nuxt.jsのaxios-moduleで頑張ったが敗北した話"

+++

# やりたかったこと
一部のホストとかにだけ別の設定をしたaxios-module(`Secure and Easy Axios integration with Nuxt.js` らしい)のインスタンスを用意したいという話
そして(少なくともスマートなやり方では)できなかったという話

## 例
例えば、以下のようなNuxt.jsのプラグインがあったとする ( https://axios.nuxtjs.org/extend より引用 )

```javascript
export default function ({ $axios, redirect }) {
  $axios.onRequest(config => {
    console.log('Making request to ' + config.url)
  })
​
  $axios.onError(error => {
    const code = parseInt(error.response && error.response.status)
    if (code === 400) {
      redirect('/400')
    }
  })
}
```

これは、エラーコードが400ならエラーページに飛ばす、という処理である(と思われる)
で、これが例えば `/api/` 以下に対してはやってほしいが、 `/health` とかに対してはやってほしくない、みたいなことがある(かもしれない)
そんな時に、プラグインの中で処理を分けるのではなく、別のインスタンスとして分けてしまえば管理が楽であろうというのが最初のモチベーション。
ついでに、ホスト名のprefixとかも個別に設定できたりして便利

```javascript
async fetchResource ({ $apiClient }) {
  return await $apiClient.get('/api/foo')
  // これが400 errorのときはredirectしてほしい
}

async healthCheck ({ $healthCheckClient }) {
  return await $healthCheckClient.get('/')
  // これがerrorでもredirectまではしてほしくない
  // そもそもこれで /health にリクエスト飛ぶような設定がしたい
}
```

# なぜできなかったのか
結論としては、axios-moduleのインスタンスに生えている関数の `this` はdeepCloneしてもオリジナルのインスタンスを指すようになっていたから

## 現象
まず、pluginで $axios をdeepCopyしてみた(lodashのcloneDeepを利用)
```javascript
import cloneDeep from 'lodash.clonedeep'

export default function ({ $axios }, inject) {
const newClient = cloneDeep($axios)
newClient.onRequest..... //みたいな感じで処理を挟んで

inject('newClient', newClient) // これでcontext.app.$newClient が生える
}
```

ということをやっていたのだが、 `newClient.onRequest` の処理結果が想定と違っていた。
この処理はaxios-module内から始まっている。
https://github.com/nuxt-community/axios-module/blob/dev/lib/plugin.template.js#L20
このinterceptorsというのがaxios側で定義されている。
https://github.com/axios/axios/blob/master/lib/core/InterceptorManager.js#L17-L23
`this.handlers` にonRequestで定義したフック処理が連なっていくのだが、この `this` が `newClient` ではなくもともとの `$axios` を指しており変更できなさそうだということがわかった
(実際console.logやらデバッガやらを使ってnewClientと$axiosのinterceptors.requestの中身を見ればわかる)

## 理由
> `this.handlers` にonRequestで定義したフック処理が連なっていくのだが、この `this` が `newClient` ではなくもともとの `$axios` を指しており変更できなさそうだということがわかった

と書いているが、仕組みとしては `onRequest` 内の `this` が外から変更できないということが原因になる。
https://github.com/nuxt-community/axios-module/blob/dev/lib/plugin.template.js#L20
この行の `this` は以下の箇所の処理によってaxios-moduleのプラグインによって生成されたaxiosに束縛されている
https://github.com/nuxt-community/axios-module/blob/dev/lib/plugin.template.js#L42-L46
```javascript
const extendAxiosInstance = axios => {
  for (let key in axiosExtra) {
    axios[key] = axiosExtra[key].bind(axios)
  }
}
```

この処理によって、axiosの外部であるaxios-moduleのプラグインで定義したメソッドをaxiosに生やしているのだが、今回やりたいことからすると困ったことになる。
というのも(たまたま知らなかっただけだが)bindでthisを束縛しなおしたり、bindを解除してthisをundefinedに戻したりはできないらしい(rebindもunbindもない)
https://stackoverflow.com/questions/31656593/javascript-function-bind-override-how-to-bind-it-to-another-object
上のstack overflowの記事が端的に今回の困りポイントを表していて分かりやすい。(ES2015のspecは下)
http://www.ecma-international.org/ecma-262/6.0/index.html#sec-function.prototype.bind

## (一応考えられうるワークアラウンド)Function.prototype.bind自体をunbind可能なものに置き換える
https://gist.github.com/cowboy/5373000
一応これで(おそらく)できるという話はあるが、かなりダーティハック感がある。

# 結論: このやり方はだめだがどうすればいいのかは検討中
* 生axiosを使うか
* axios-moduleを使ったまま onRequest内で条件分岐をするか

