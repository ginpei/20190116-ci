# 自動で徳を積むための試験とCIハンズオン 2019-03-16

- Event: [自動で徳を積むための試験とCIハンズオン | Frog](https://frogagent.com/event/testtools-ci-workshop/)
- Original: https://github.com/ginpei/hannya-roll

---

## 目次

- サンプルプロジェクトの用意
- ESLintで綺麗にする
- Jestで試験する
- TravisCIで自動試験する
- CodeClimateで自動レビューする
- 試験Coverageを自動集計する
- GitHub Pagesへ自動デプロイする

## サンプルプロジェクトの用意

- [ginpei/20190116-ci: 自動で徳を積むぞ  
https://github.com/ginpei/20190116-ci](https://github.com/ginpei/20190116-ci)

```bash
$ git clone git@github.com:ginpei/20190116-ci.git
$ cd 20190116-ci
$ npm install
$ npm start
```

フォークしたりZIPでダウンロードしたりでもよろしい。

## Jestで試験する

- [Jest · 🃏 Delightful JavaScript Testing  
https://jestjs.io/](https://jestjs.io/)
- 試験の実行とライブラリーを統合したフレームワーク
- by Facebook

### 導入

- [Getting Started · Jest](https://jestjs.io/docs/en/getting-started.html)

```console
$ npm install jest
```

本来はまずコードを書く。

```js
// src/sum.test.js
function sum(a, b) {
  return a + b;
}
module.exports = sum;
```

続いて試験コードを書く。先の対象コードを `import` する。

```js
// src/sum.test.js
const sum = require('./sum');

test('adds 1 + 2 to equal 3', () => {
  expect(sum(1, 2)).toBe(3);
});
```

実行。

```console
$ npx jest
```

開発中はwatchしたい場合、`--watch` オプション。`^C` で終了。

```console
$ npx jest --watch
```

### npm scripts

に書いておくと吉。 `"test": "jest",` を追加。

```js
{
  "private": true,
  "scripts": {
    "build": "echo NODE_ENV=$NODE_ENV && webpack --config config/webpack.config.js",
    "publish": "npm run build && push-dir --dir=dist --branch=gh-pages --cleanup",
    "start": "webpack-dev-server --config config/webpack.config.js --open",
    "test": "jest",
    "type-check": "tsc"
  },
…
```

```console
$ npm run test
```

```console
$ npm run test -- --watch
```

## Jestの試験の書き方

ちょっとだけ。

### `it()`, `expect()`

試験項目。`it()` が実行され、`expect()` で結果を確認する。

```js
it('sums up numbers', () => {
  const result = sum(3, 2);
  const expected = 5;
  expect(result).toBe(expected);
});
```

`expect()` に実際の結果を、 `.toBe()` の方に想定する結果を与える。 "Expect xxx to be yyy" という英文。

### `toBe()` 他

`.toBe()` の部分を Matcher と呼ぶ。100億万種類くらいある。

- `toBe()` … 同じ値か。[`Object.is()`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/is) を利用
- `toEqual()` … オブジェクトの中身が同じか。別インスタンスでも可

その他たくさんあるので公式文書を確認。

- [Expect · Jest  
https://jestjs.io/docs/en/expect](https://jestjs.io/docs/en/expect)

### `not`

matcher の判定を逆にする。

```js
expect(10).toBe(10); // pass
expect(10).not.toBe(99); // pass
```

### 非同期の試験

普通に `async`, `await` を書けば動く。

```js
test('the data is peanut butter', async () => {
  expect.assertions(1);
  const data = await fetchData();
  expect(data).toBe('peanut butter');
});
```

`resolves`, `rejects` というのもある。

### 例外

`try`, `catch` でやれる。

`toThrow()` というのもある。この場合、`expect()` には関数を与える。

```js
expect(() => {
  throw new Error('It is not OK');
}).toThrow();
```

### 関数が呼ばれることを確認

`jest.fn()` でスパイする。

```js
it('calls back event listener', () => {
  const spy = jest.fn();
  const el = document.createElement('div');

  el.addEventListener('click', spy);
  el.dispatchEvent(new Event('click'));

  expect(spy).toBeCalled();
});
```

### 時間操作

`jest.useFakeTimers()` する。

### `describe()` でグループ化

複数の試験をまとめる。入れ子にもできる。

```js

describe('FooApp()' => {
  it('works nicely', () => ...);
  it('works finely', () => ...);

  describe('BarModule', () => {
    it('works straightforwardly', () => ...);
  });
});
```

結果も入れ子になって表示される。

### `beforeEach()`, `afterEach()`

毎試験前後に呼ばれる。

```js
describe('Foo', () => {
  let foo;

  beforeEach(() => {
    foo = new Foo();
  });

  afterEach(() => {
    //
  });

  it('is OK', () => {
    expect(foo).toBeInstanceOf(Foo);
  });
});
```

### モック

オブジェクトを試験用に置き換えるやつ。

例えば axios で通信する機能を内部に持つ場合、axio をモック化することで通信を偽造できる。

## 試験の考え方

public な interface の「入力」と「出力」の確認をする。内側へ秘匿されている機能は気にしない。（ブラックボックス試験。）　これで実装の変更があっても試験が通っていれば、結果は変わらないと判断できる。

試験自体が誤っていないことを確認するため、一度わざと失敗すると良い。失敗するはずなの成功する場合、前提が誤っていることがある。そもそも呼ばれてないとか。

外部リソース（ネットワーク経由の情報とか）は使わない。外部要因により成功したり失敗したりするのを防ぐため。（結合試験で確認する。）　ファイルは試験用のものを用意する。

## TravisCIで自動試験する

- [Travis CI - Test and Deploy Your Code with Confidence  
https://travis-ci.org/](https://travis-ci.org/)
- CI: continuous integration
- リポジトリーと連携して自動であれこれやってくれる

### GitHubで公開

`.git` を削除して、自分のプロジェクトとして登録。

```bash
$ rm -rf .git
$ git init
$ git add .
$ git commit --message init
```

`git remote add ...` とか何とかしてから `git push` する。

### リポジトリーへ接続

アカウント作って、GitHubと連携して、ダッシュボードの "+" ボタンを押してリポジトリー一覧を開いて、目的のものを選択。

### 設定ファイルを作成

プロジェクトルートに `.travis.yml` を作成。

```yaml
language: node_js
node_js:
  - "10"
script:
  - npm run test
```

設定ファイルをGitリポジトリーへ追加してサーバーへ上げる。Travisが変更を監視しているので、自動で試験が実行される。（しばし待て。）

失敗するとメールが飛んでくる。

`script` の `-` はYAMLの配列なので、複数書ける。lintとか。`script` 以外にもいろいろある。

- [Job Lifecycle - Travis CI  
https://docs.travis-ci.com/user/job-lifecycle](https://docs.travis-ci.com/user/job-lifecycle)

### vs local git-hooks

- [Git - Git Hooks  
https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)

`git commit` のたびにローカルで試験実行、なんかもできるけど、時間かかって面倒なのでCIあるならそれだけで良いのでは。

### バッジを付ける

よく見かけるアレをREADMEに追加しよう。

TravisCI内各プロジェクトの画面の "build: passing" みたいなバッジの画像をクリックすると、コピペ用コードを得られる。

```md
[![Build Status](https://travis-ci.org/ginpei/excell.svg?branch=master)](https://travis-ci.org/ginpei/excell)
```

## 他のCIサービス、ツール

### CircleCI

- https://circleci.com/
- こっちの方が有名かも

### GitLab CI, CD

- https://about.gitlab.com/product/continuous-integration/
- GitLabのサービスだが[GitHubとも連携できる](https://about.gitlab.com/solutions/github/)
- CD: Continuous Delivery


> ## What is CI/CD?
>
> **Continuous Integration** is the practice of integrating code into a shared repository and building/testing each change automatically, as early as possible - usually several times a day.
>
> **Continuous Delivery** adds that the software can be released to production at any time, often by automatically pushing changes to a staging system.
>
> **Continuous Deployment** goes further and pushes changes to production automatically.

### Heroku CI

- https://devcenter.heroku.com/articles/heroku-ci

### Jenkins

- https://jenkins.io/
- 超有名。CIツールの先駆け
- サービスではないので自分でサーバーを用意してインストールする

### その他

- [CIマニアから見た各種CIツールの使い所 - くりにっき](https://sue445.hatenablog.com/entry/2018/12/07/114638)
  - TravisCI vs CircleCIとか
  - TravisCI推し
- [History in 5 years of CircleCI and CyberAgent - Speaker Deck](https://speakerdeck.com/stormcat24/history-in-5-years-of-circleci-and-cyberagent)
  - 中身は日本語

## ESLintで綺麗にする

- [ESLint - Pluggable JavaScript linter  
https://eslint.org/](https://eslint.org/)
- linter、つまりコードの書き方を制限して綺麗な状態に保つツール

### 導入

NPMパッケージをインストールする。 `eslint` がアプリ。`eslint-config-airbnb-base` が下敷きにするルール集、`eslint-plugin-import` はその依存。

```console
$ npm install eslint eslint-config-airbnb-base eslint-plugin-import
```

プロジェクトルートに `.eslintrc.js` を作成。

```js
// npm install --save-dev eslint eslint-config-airbnb-base eslint-plugin-import

module.exports = {
  "env": {
    "browser": true,
    "es6": true,
    "jest": true,
  },
  "extends": "eslint-config-airbnb-base",
  "rules": {
    "arrow-parens": [
      "error",
      "always",
    ],
    "no-underscore-dangle": [
      "error",
      {
        "allowAfterThis": true,
      },
    ],
    "class-methods-use-this": "off",
    "import/extensions": [
      "error",
      "ignorePackages",
    ],
    "space-before-function-paren": [
      "error",
      "always",
    ],
    "no-unused-vars": [
      "off",
    ],
  },
};

```

実行。

```console
$ npx eslint src
```

エラーがあれば何か出る。

問題なければ何も出ない。

### 自動修正

セミコロンの抜けのような軽微なエラーは自動修正できる。`--fix` を付ける。

```console
$ npx eslint src --fix
```

ファイルは上書きされるが `git` があるので恐るるに足らぬ。

### npm scripts

に書いておくと吉。 `package.json` の `"scripts"` に `"lint"` を追加。

```js
{
  "scripts": {
    "lint": "eslint src"
…
```

```console
$ npm run lint
```

```console
$ npm run lint -- --fix
```

## Travis CIで自動Lintする

`.travis.yml` に `npm run lint` を追加。

```yaml
language: node_js
node_js:
  - "10"
script:
  - npm run lint
  - npm run test

```

わざと失敗してみたい場合、`.travis.yml` 中の `"no-underscore-dangle"` の `"allowAfterThis": true` を `"allowAfterThis": false` にしてみよう。

## Code Climateで自動レビューする

- [Quality | Code Climate  
https://codeclimate.com/quality/](https://codeclimate.com/quality/)
- コードを静的解析して、改善提案してくれたり点数を付けて変化を記録してくれたりする
- 同社製品の Velocity (青、$799+/mo) ではなく Quality (緑、$0+/mo) の方ね
- **Vue.js未対応**。ReactやTypeScriptには対応

### 導入

なんかぽちぽちすれば終わる。

連携してしばらく待つと出てくる。

## 試験Coverageを自動集計する

試験がコードをどれだけカバーしているか。（試験項目の有無ではなく試験時に行を通ったかどうかの判定のみ。）

CodeClimateが対応しているので、記録してもらおう。

### 試す

jestが計測機能を持っているので、それを使う。 `jest --coverage` でやれる。

```console
$ npm run test -- --coverage
```

`coverage` ディレクトリー以下に諸々出力される。`coverage/lcov-report/index.html` をブラウザーで開いて見てみよう。

### リポジトリーには上げないよ

自動生成される系のファイルを上げてはいけない。これも例外ではない。

`.gitignore` に追加しておく。

### CodeClimateと連携

ちょっと面倒。

TravisCIで試験時に coverage 出力し、その結果を CodeClimate へ送信する、という流れ。

1. CodeClimate連携キーを取得
2. TravisCIに連携キーを設定
3. `.travis.yml` で連携するコードを記述
4. 以降は自動的に更新される

キーが必要なので、まず CodeClimate で対象プロジェクトの "Repo Settings" → "Analysis" 以下 "Test coverage" → "Use this ID with Code Climate's test reporters" のIDを控えておく。

続いてTravisCIで対象プロジェクトの設定で Environment Variables を追加する。

- Name: `CC_TEST_REPORTER_ID`
- Value: 前述のID

さらに `.travis.yml` の設定を更新。

- `before_script` と `after_script` を挿入
- 試験スクリプトに `-- --coverage` を追加。

```yaml
language: node_js
node_js:
  - "10"
before_script:
  - curl -L https://codeclimate.com/downloads/test-reporter/test-reporter-latest-linux-amd64 > ./cc-test-reporter
  - chmod +x ./cc-test-reporter
  - ./cc-test-reporter before-build
script:
  - npm run type-check
  - npm run lint
  - npm run test -- --coverage
after_script:
  - ./cc-test-reporter after-build --exit-code $TRAVIS_TEST_RESULT

```

これをリポジトリーに上げると、TravisCI が反応して試験を実行、同時に coverage 出力して CodeClimate へ送信、そこで記録される。

### その他のサービス

- [Coveralls - Test Coverage History & Statistics  
https://coveralls.io/](https://coveralls.io/)

## GitHub Pagesへ自動デプロイする

`release` ブランチ更新時、GitHub Pagesも自動で更新するようにする。

### GitHub Pages とは

- [GitHub Pages | Websites for you and your projects, hosted directly from your GitHub repository. Just edit, push, and your changes are live.  
https://pages.github.com/](https://pages.github.com/)
- `gh-pages` ブランチに `git push` すると、自動的に `https://<username>.github.io/<reponame>/` で公開される
- 独自ドメインも使える
- RewriteRule的なものはないのでSPAには不向き（hashbang `#!` でならいける）

### GitHub Pagesを手動で設定

一度やってみる。

1. コンソールでビルド。例： `react-scripts build`
2. 出力ディレクトリーへ移動。このディレクトリーは `.gitignore` されているはず。例： `build/`
3. そのディレクトリー内でGitを設定
    - `git init`
    - `git add .`
    - `git commit --message pages`
    - `git branch -m gh-pages` ← 必ず `gh-pages` にする
    - `git remote add origin git@github.com:xxxxxx/yyyyyy.git` ← プロジェクトのトップページからコピペ
    - `git push -u origin gh-pages`
4. GitHubのリポジトリーのページで、Settings → Options （最初の画面）→ 
GitHub Pages の節を確認
    - 緑色の " Your site is published at https://..." が出てくるまで待つ
5. URLを開くと出てくる（やったぜ）
6. プロジェクトの Website に設定しとこう

これをCIで設定して自動化する。

なお **`gh-pages` ブランチのコミットはこの後上書き更新される**ので、コミットメッセージは適当で良い。

### 本番用ビルド

設定は既にあるが、`config/webpack.config.js` を自分のに合わせて更新する必要がある。

ここの `'/hannya-roll'` を自分のリポジトリー名に合わせて変更する。

```diff
- const publicPath = isProduction ? '/hannya-roll' : '';
+ const publicPath = isProduction ? '/20190116-ci' : '';
```

そして `NODE_ENV=production` を設定して出力。

```bash
$ NODE_ENV=production npm run build
```

minifyとかされる。

### TravisCI の設定

TravisCI は GitHub Pages用の設定を持っているので、Git コマンドを延々と設定ファイルに書く必要がない。（そっちが良ければ自前で書いてもいい。）

- [GitHub Pages Deployment - Travis CI  
https://docs.travis-ci.com/user/deployment/pages/](https://docs.travis-ci.com/user/deployment/pages/)
- 他に [Firebase](https://docs.travis-ci.com/user/deployment/firebase/) や [Heroku](https://docs.travis-ci.com/user/deployment/heroku/) 等々へのデプロイも対応。文書読め

まずはトークンの設定。先のリンク先の説明通り、[Personal Access Tokens](https://github.com/settings/tokens) でトークンを生成し、Travis CIのプロジェクトページで `GITHUB_TOKEN` という名前の環境変数として設定する。

続いて `.travis.yml` を編集。

- `script` に `npm run build` を追加
- `deploy` を追加

```yaml
language: node_js
node_js:
  - "10"
before_script:
  - curl -L https://codeclimate.com/downloads/test-reporter/test-reporter-latest-linux-amd64 > ./cc-test-reporter
  - chmod +x ./cc-test-reporter
  - ./cc-test-reporter before-build
script:
  - npm run type-check
  - npm run lint
  - npm run test -- --coverage
  - NODE_ENV=production npm run build
after_script:
  - ./cc-test-reporter after-build --exit-code $TRAVIS_TEST_RESULT
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GITHUB_TOKEN
  keep-history: true
  local-dir: "dist"
  on:
    branch: master

```

### リリース用デプロイ

先の `.travis.yml` の設定で、最後の `branch: master` を変更して、指定のブランチ更新時にのみデプロイを行うようにする。

```yaml
deploy:
  on:
    branch: release
…
```

## おわり

- 2019-03-16
- Frog Office
- 高梨ギンペイ
- [自動で徳を積むための試験とCIハンズオン | Frog](https://frogagent.com/event/testtools-ci-workshop/)
