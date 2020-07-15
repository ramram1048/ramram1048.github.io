---
layout: post
title: "vanillajs single-page-app(SPA) 개발 환경 구성"
categories: js
---

# 잡담

일전에 게임 플러그인에 사용할 single page app을 React 프레임워크로 만들었던 적이 있었다. 게임에서 발생하는 네트워크 패킷을 외부 프로그램이 읽어오면 내 어플리케이션이 event listener로 받아와 parsing하고 화면에 보여주는 간단한 어플리케이션이었다. 약 1년간 학업에 집중한다는 핑계로 개발을 멈추고 있었는데, 요즘 졸업을 하고 시간도 남고 미련도 남아서 이 프로그램을 개선해보기로 마음을 먹게 됐다.

일단 작업을 하려 마음을 먹었을때 답답했던 점은 너무 초보일 때 만들었던 프로그램이라 소스가 정말 엉망이었다는 점이었다. 모든 코드가 하나의 파일에 들어있었고 이미지 파일 관리도 엉망이었다. 모든 이미지가 내 git repository에 들어있었어서 github pages에서 제공하는 무료 트래픽을 금방 초과하기 일쑤였다.

결국 새로 만들기로 마음을 먹었는데... 여기서 **굳이 React를 쓸 필요가 없을 것 같음 + 개인적인 공부 욕심**이 생겨서 vanilla js를 이용해 새로 짜보기로 했다.

사용하게될 웹페이지가 앞서 언급했던 외부 프로그램의 인 앱 브라우저에서만 동작하므로, 해당 프로그램에 설치된 chrome의 특정 버전상에서만 동작하게 된다. 따라서 이 chrome 버전만을 target으로 컴파일하고 싶었고, 이를 위해 babel을 도입하고 webpack과 연동해 사용하기로 했다. 또 학교에서 배운 eslint를 실제 프로젝트에 적용해본 경험이 없었기 때문에 eslint 또한 적용시켜보기로 했다.

배포는 이전과 마찬가지로 Github Pages를 이용하려 했는데, ``gh-pages`` 모듈을 이용하지 않고 스스로 배포 자동화를 할 방법이 없을까 하고 찾다가 Travis CI를 이용해 자동화할 수 있었다. 

<br />

# 이 글의 목표

1. 프로젝트 생성
2. webpack으로 SPA 구현
3. 개발서버로 webpack-dev-server 이용 + hot reload 적용
4. babel과 연동해 target browser 설정
5. 개발환경으로 eslint 적용

<br />

# 1. 프로젝트 생성

**프로젝트에 사용된 예제 완성본은 [https://github.com/ramram1048/vanilla-spa](https://github.com/ramram1048/vanilla-spa)에서 확인할 수 있다.

우선 빈 폴더를 하나 만들고 npm과 git 초기 설정을 진행하자. 터미널에 아래 명령어들을 입력한다.

```
npm init -y
```

<br />

```
.
├-- /src
|   ├-- /styles
|   |   └-- main.css
|   ├-- /view
|   |   ├-- header.ejs
|   |   ├-- index.ejs
|   |   └-- main.ejs
|   ├-- app.js
|   └-- module.js
├-- .gitignore
└-- package.json
```

폴더 구조는 어찌되던 상관 없지만 본 예제에서는 위와 같이 구성하였다. 코드 작업은 `/src` 폴더에서 이루어지며 `app.js`가 webpack의 entry point가 되고, `/src/view` 폴더의 `.ejs` 파일들이 화면을 구성하는 HTML DOM을 표현하게 된다.

<br />

`index.ejs`를 VS code에서 열고 빈 화면에 컨트롤+space를 누르면 `HTML sample`을 불러올 수 있다. 그대로 불러온 뒤 css와 js를 불러오는 라인만 지워주자. 

`head`의 필드를 적당히 수정해주고 `noscript` tag를 추가하고 `header.ejs`와 `main.ejs`를 불러오는 코드를 삽입하자. `.ejs`를 불러오는 코드는 webpack으로 컴파일할 시 ejs 문법(include)이 아닌 webpack에서 요구하는 문법을 사용해야하는데, 아래 코드 예시와 같이 `<%= require('html-loader!./*.ejs') %>`의 형태로 불러와야 한다.

최종 `index.ejs`의 코드는 아래와 같이 완성된다.

```html
<!-- index.ejs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset='utf-8'>
  <meta http-equiv='X-UA-Compatible' content='IE=edge'>
  <title>Page Title</title>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
</head>
<body>
  <noscript>You need to enable JavaScript to run this app.</noscript>
  <header>
    <%= require('html-loader!./header.ejs') %>
  </header>
  <%= require('html-loader!./main.ejs') %>
</body>
</html>
```

<br />

파일들이 합쳐지는 모습을 확인하기 위해 `header.ejs`와 `main.ejs`, `main.css`, `app.js`, `module.js`에 적당히 내용을 입력해보자.

```html
<!-- header.ejs -->
<nav>
  저는 header.ejs입니다~
</nav>

<!-- main.ejs -->
<div>
  저는 main.ejs요~
</div>
```

```css
/* main.css */
html {
  background-color: yellow
}
```

```js
// app.js
import './styles/main.css'
import module from './module'

alert('저는 app.js요~')
module('저는 module.js요~')

// module.js
module.exports = (text) => console.log(text)
```

<br />

# 2. webpack으로 SPA 구현

webpack을 적용하기 위해 아래 패키지들을 devDependencies로 설치한다.

```
npm i --save-dev webpack webpack-cli html-webpack-plugin html-loader css-loader style-loader
```

- `webpack`, `webpack-cli`: 각각 webpack과 webpack CLI이다.
- `html-webpack-plugin`: webpack이 entry point로 새 html 파일을 생성하게 해주는 플러그인이다.
- `html-loader`: webpack이 html 파일을 불러올 수 있게 하는 플러그인이다. 우리 프로젝트에서는 앞에서 `index.ejs` 파일이 `header.ejs`, `main.ejs`를 불러오는 데에 사용하고 있다.
- `css-loader`: webpack이 css 파일을 불러올 수 있게 하는 플러그인이다.
- `style-loader`: webpack이 불러온 css 파일을 js파일에 style형태로 포트할 수 있게 해주는 플러그인이다. 잘은 모르겠는데 이거랑 같이 없으면 css 스타일이 적용되지 않는다...

<br />

webpack 설정 파일을 만들자. 본인은 프로젝트 최상단에 `webpack.config.js`라는 이름으로 만들었다.

```js
// webpack.config.js
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  mode: 'production',
  target: 'web',
  entry: [
    `${__dirname}/src/App.js`,
  ],
  output: {
    filename: '[name].[hash].js',
    path: `${__dirname}/dist`,
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: `${__dirname}/src/view/index.ejs`,
    }),
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader',
          'css-loader',
        ],
      },
    ],
  },
};
```

- `mode`: `production` or `development`
- `target`: `web` or `node`. 해당 어플리케이션은 node 서버로 운영될 것이 아니라 웹페이지 어플리케이션으로 사용될 것이므로 `web`으로 설정했다.
- `entry`: entry point. 최상단의 js파일이라 생각하면 편할 것 같다..
- `output`: 컴파일 결과물이 위치할 폴더. 이름과 위치는 자유롭게 입력해도 된다.
- `plugins`: 사용할 플러그인. 우리 예제에서는 HtmlWebpackPlugin을 사용하고 있다. 
  - HtmlWebpackPlugin: 앞에서 설명한 대로, entry point로 새 html 파일을 생성하도록 하는 플러그인이다. 이때 사용할 html의 template를 지정할 수 있는데 우리가 생성했던 `index.ejs`를 사용하도록 경로를 지정해주자.
- `module`: 파일을 불러올 때 사용하는 모듈. 우리 예제에서는 css파일을 불러오기 위해 `style-loader`와 `css-loader`를 사용하고 있다.

더 자세한 설명은 [webpack documentation](https://webpack.js.org/configuration/)을 참고하자. 튜토리얼이 상세하게 되어있어서 따라다니면 금방 익숙해진다.

<br />

이제 이대로 build해보자. 터미널에

```
webpack --config webpack.config.js
```

를 입력하자. `--config` 뒤의 인자는 webpack 설정 파일을 지정하는데 혹시 다른 위치에 저장했다면 다른 위치를 입력해주자.

파일이 별로 크지 않아 컴파일도 금방 된다. 컴파일이 완료되면 설정 파일에서 `output`으로 지정했던 폴더에 `index.html`이 들어있다. 이 파일을 열어보면 `app.js`에서 입력했던 `alert()`과 `header.ejs`, `main.ejs`에서 입력한 내용들이 웹페이지에 보이고, `main.css`에서 입력한 `background-color: yellow`도 적용된 모습을 확인할 수 있다.

![webpack으로 컴파일 된 모습](https://res.cloudinary.com/djhv99xj7/image/upload/v1594803460/jekyll/image-20200715154652772_tu5uia.png)

<br />

# 3. 개발서버로 webpack-dev-server 이용 + hot reload 적용

매번 파일을 수정할 때마다 컴파일 명령어를 입력하고 새로고침하는 것이 상당히 귀찮으므로 hot-reload를 이용하면 편리하다. 처음에는 live-server를 이용했었는데 이것도 상당히 귀찮아서 그냥 webpack-dev-server를 설치해 사용하기로 했다.

아래 패키지들을 devDependencies로 설치하자.

```
npm i --save-dev webpack-dev-server chokidar
```

- `webpack-dev-server`: webpack에서 제공하는 개발서버 모듈으로 [express](https://expressjs.com/)를 이용한다. 자체 express 서버를 활용하고자 하는 경우 `webpack-dev-middleware`를 사용할 수도 있다고 하는데 나는 그냥 이걸 사용하기로 했다.
- `chokidar`: node.js의 fs.watch, fs.watchFile, FSEvents와 같은 tracking 모듈들을 보완할 수 있는 패키지라고 한다~~(잘 모름)~~. 그냥 `webpack-dev-server`를 이용할 때 ejs파일을 수정해도 hot-reloading이 되지 않길래 이용하게 됐다. (https://github.com/paulmillr/chokidar)

<br />

개발서버용 webpack config파일을 만들어보자. 본인은 프로젝트 최상단에 `webpack.config.dev.js`라는 파일을 만들었다.

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { watch } = require('chokidar');

module.exports = {
  mode: 'development',
  devtool: 'inline-source-map',
  devServer: {
    before: (app, server) => {
      watch([
        `${__dirname}/src/view`,
      ]).on('all', () => {
        server.sockWrite(server.sockets, 'content-changed')
      })
    },
    contentBase: `${__dirname}/dist`,
  },
  entry: [
    `${__dirname}/src/App.js`,
  ],
  output: {
    filename: '[name].[hash].js',
    path: `${__dirname}/dist`,
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: `${__dirname}/src/view/index.ejs`,
    }),
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          'style-loader',
          'css-loader',
        ],
      },
    ],
  },
};

```

기존 설정파일이었던 `webpack.config.js`와의 차이점은

- `mode`: `development`로 설정되었다. 
- `devtool`: 핫 리로드 시 컴파일 에러 등을 웹 콘솔로도 출력하게 해준다.
- `devServer`: 개발서버에서 사용할 설정들이 들어간다.
  - `before`: 개발서버에 사용할 middleware를 다른 middleware들보다 먼저 작동시킨다. ([링크](https://webpack.js.org/configuration/dev-server/#devserverbefore))
    우리 예제에서는 chokidar가 `/src/view`의 파일이 변한 경우 개발서버에 `content-changed`라는 메시지를 보내도록 설정했다.
  - `contentBase`: serve할 파일이 들어있을 위치를 나타낸다. `output`에서 지정한 경로가 설정되야 한다.

<br />

개발서버를 구동할 명령어를 `package.json`에 지정해주자.

```json
{
  // ...
  "scripts": {
    "start": "webpack-dev-server --config webpack.config.dev.js --hot --inline --watch --progress",
      // ...
  },
  // ...
}
```

`script` 항목에 `"start": "webpack-dev-server --config webpack.config.dev.js --hot --inline --watch --progress"`를 지정해준다.

각 option은

- `config`: 사용할 webpack 설정파일의 경로이다. 필자는 최상단에 `webpack.config.dev.js`라는 이름으로 저장했지만 다른 경로에 저장할 경우 parameter만 바꿔주면 된다.
- ``hot``: hot reloading을 사용한다는 표시이다.
- `inline`: live reloading에 필요한 devServer 옵션이다.
- `watch`: 파일이 변경되는 것을 감시한다.
- `progress`: 컴파일 %를 보여준다. 필요없을 경우 빼도 된다.

<br />

이제 터미널에 

```
npm start
```

를 입력하면 개발서버가 실행된다. 기본 포트는 8080이므로 `http://localhost:8080`으로 접속하면 개발서버의 모습을 확인할 수 있다.

![hot reloading이 적용된 모습](https://res.cloudinary.com/djhv99xj7/image/upload/v1594803460/jekyll/image-20200715162018879_skrsdd.png)

콘솔창을 열어보면 [HMR], [WDS]로 시작하는 메시지들이 추가로 출력된 것을 확인할 수 있다. 얘네가 개발서버에서의 요청에 반응해 hot-reloading이나 hard reloading을 실행해준다.

이렇게 개발서버 콘솔을 켜두고 브라우저에 개발서버 앱을 띄워둔 상태로 파일을 변경하면 변경된 사항이 자동으로 갱신된다.

<br />

# 4. babel과 연동해 target browser 설정

필자의 프로젝트는 특정 브라우저 버전이 필요하기 때문에 babel을 이용했지만, 이외에도 여러 이유로 babel을 사용할 것이라 생각한다. babel을 연동해보자.

아래 패키지들을 devDependencies로 설치하자.

```
npm i --save-dev @babel/core @babel/cli @babel/preset-env babel-loader
```

- `@babel/core`, `@babel/cli`: 각각 모듈과 cli interface
- `@babel/preset-env`: babel에서 컴파일하는 기본 규칙같은것
- `babel-loader`: webpack에서 babel을 불러오는 플러그인

<br />

babel 설정 파일을 만들자. 프로젝트 최상단에 `babel.config.js` 파일을 만든다.

```js
// babel.config.js
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        targets: {
          chrome: '75',
          node: 'current',
        },
      },
    ],
  ],
};
```

규칙으로 `@babel/preset-env`를 사용하고 target으로 `chrome@75`를 사용하는 예시이다. 더 다양한 target을 이용하려 하거나 최신을 이용하려 하는 등 다른 옵션을 적용하려면 [공식 documentation](https://babeljs.io/docs/en/configuration)을 참고하자.

<br />

이제 webpack build 시에 babel이 연동되도록 설정하자.

```js
// webpack.config.js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel-loader',
      },
      // ...
    ]
  }
}
```

`module.rules` 필드에 위와 같이 `.js`파일에 `babel-loader`를 사용하도록 지정하면 된다.

<br />

# 5. 개발환경으로 eslint 적용

webpack에 eslint를 적용해 컴파일할 수도 있지만 그냥 VS code에 적용하고 저장 시 linter가 동작하는 방법을 사용하기로 했다. eslint에도 여러 규칙이 있는데 본인은 `eslint-config-airbnb-base`를 사용하기로 했다.

아래 명령어를 입력해 필요한 package들을 devDependencies로 설치한다.

```
npx install-peerdeps --dev eslint-config-airbnb-base
```

<br />

eslint 설정파일 `.eslintrc.json`을 프로젝트 최상단에 만든다.

```json
{
  "env": {
    "browser": true,
    "es2020": true
  },
  "extends": [
    "airbnb-base"
  ],
  "parserOptions": {
    "ecmaVersion": 11,
    "sourceType": "module"
  },
  "rules": {
    "semi": "off",
    "brace-style": ["error", "stroustrup", { "allowSingleLine" : true }],
    "indent": ["error", 2],
    "no-bitwise": "off",
    "no-plusplus": "off",
    "comma-dangle": "off",
    "linebreak-style": "off"
  },
  "globals": {
    "addOverlayListener": false,
    "removeOverlayListener": false,
    "callOverlayHandler": false,
    "startOverlayEvents": false
  }
}
```

- `env`: 파일이 사용될 환경.
- `extends`: 프리셋을 지정할 수 있다. 본인은 `airbnb-base`를 이용했다.
- `parserOptions`: 코드 버전을 설정할 수 있다 ([링크](https://eslint.org/docs/user-guide/configuring#specifying-parser-options)). 
  - ecmaVersion: 사용할 ECMAScript syntax의 버전을 입력한다. 2020년판은 11이다.
  - sourceType: `script` or `module`
- `rules`: 프리셋에 지정된 rule 중 내 마음에 들지 않는 것은 여기서 새로 정의할 수 있다. 예를 들어 나는 js를 사용할 때 세미콜론을 쓰지 않는 것이 습관이었는데, eslint를 적용하고 세미콜론을 입력해야한다는 에러가 엄청나게 발생했다. 내 습관을 고쳐야하나 싶었는데 어차피 개인 프로젝트이기도 하고 그냥 `semi` 옵션을 `off`로 설정해 해결했다.
  `semi` 이외에도 자기 입맛대로 여러 rule들을 수정할 수 있다.
- `globals`: cdn으로 불러온 모듈을 사용하려 하는 경우 eslint에서 인식하지 못해서 에러 처리한다. 이런 에러들을 해당 모듈들을 `globals`에 넣는 것으로 표시하지 않게 했다.

<br />

VS code에서 eslint의 에러 메시지를 확인하려면 VS code eslint extension을 설치해야 한다. 

https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint

설치하고 나면 코드에서 기존에는 보지 못하던 사소한 에러 메시지들도 보이기 시작한다.

<br />

문법에 맞게 코드를 하나하나 수정하면 상당히 번거로운데, 이를 해소하기 위해 webpack에서 컴파일하거나 VS code에서 저장 시 자동으로 수정해주는 방법이 있다. 나는 후자를 이용하기로 해서 후자를 소개해보도록 하겠다. 저장할 때마다 자동으로 코드가 예쁘게 맞춰지니 상당히 편하지만, 단점으로 rule을 제대로 설정하지 못하면 내가 의도하지 않은 대로 자동 저장이 되서 rule 설정을 잘 해줘야한다는 점이 있는 것 같다.

![vscode 자동저장 시 eslint 적용 설정](https://res.cloudinary.com/djhv99xj7/image/upload/v1594803550/jekyll/image-20200715171144012_mtkm8h.png)

위 그림에서 빨간색 표시를 한대로 따라가자. `Settings(Ctil+,)` > `save` 검색 > `Editor: Code Actions On Save` 항목의 `Edit in settings.json` 클릭하면 `settings.json`을 수정할 수 있게 된다. 여기서 `editor.codeActionsOnSave` 필드에 `"source.fixAll.selint": true`를 추가해주자.

설정이 완료되면 에러 메시지가 있는 파일을 저장하면 수정할 수 있는 에러는 자동으로 수정이 되는 모습을 확인할 수 있다.

<br />

# 끝?

추가적으로 png 등의 이미지 파일을 불러오려면 `file-loader`와 같은 webpack 모듈을 설치하고, webpack의 dev config와 prod config를 `webpack-merge`와 같은 모듈로 중복 없이 다룰 수도 있다.

또 배포를 해야 하는데... 나는 `gh-pages` 모듈을 이용하는 대신 스스로 해보고싶어서 다른 방법을 찾았는데 사실 `gh-pages`를 사용해서 나쁠 이유는 하나도 없고 아주 편리하므로 [gh-pages](https://github.com/tschaub/gh-pages)를 사용할 것을 권한다.

<br />

예제 내용은 github repository에서도 확인할 수 있다.

주소: [https://github.com/ramram1048/vanilla-spa](https://github.com/ramram1048/vanilla-spa)