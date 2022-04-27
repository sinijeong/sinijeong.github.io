---
layout: post
title: E1. 투두리스트 개발기 - Mono-Repo구축(w.npm workspaces + carco)
---

## Multi-Repo와 Mono-Repo란?

Multi(다중) + Rrepo(저장소) 와 Mono(단일) + Repo(저장소) 라는 뜻 그대로 각각 복수 Repository와 단일 Repository를 의미한다. 바퀴, 핸들, 안장의 Repository를 각각 개별적으로 구성하는 것은 Multi-Repo형식이다. 자전거라는 하나의 Repository가 핸들, 안장 Packge 및 바퀴 Repository를 포함하는 것은 Mono-Repo방식이다.

{% highlight bash %}
// Multi-repo

바퀴-Repo
핸들-Repo
안장-Repo

// Mono-repo

자전거-Repo
├── 바퀴-Repo
│ ├── 체인-package
│ └── 타이어-package
├── 핸들-package
└── 안장-package
{% endhighlight %}

<br/>

Mono-Repo는 하나의 Repository를 사용하기 때문에 패키지 간의 코드 공유가 쉬워지고 중첩 코드와 node module을 줄일 수 있어 큰 프로젝트에서 사용할 경우 복잡성을 줄일 수 있다.

<br/>

## Npm workspaces

npm V7부터 workspaces를 지원하여 mono-repo 프로젝트를 제작할 수 있습니다. 최상위 루트에 package.json에 workspaces를 정의합니다.

{% highlight js %}
{
    "name": "root",
    "private": true,
    "workspaces": [
    "workspace-a",
    "workspace-b",
    ]
}
{% endhighlight %}

<br/>

루트 폴더에서 npm install을 실행하면 packages와 packages/plugins 하위에 위치한 폴더는 전부 최상단에 위치한 node_modules에 심볼릭 링크로 연결됩니다. 

{% highlight bash %}
.
├── node_modules
│   ├── workspace-a -> ../workspace-a
│   └── workspace-b -> ../workspace-b
├── package-lock.json
├── package.json
├── workspace-a
│   └── package.json
└── workspace-b
    └── package.json
{% endhighlight %}
node_modules 폴더는 전체 폴더를 통틀어 최상단에만 위치하게 됩니다.

<br/>

## 폴더 구조

{% highlight bash %}
.
├── node_modules
├── package-lock.json
├── package.json
└── packages
    ├── app
    │   └── package.json
    └── plugins
        ├── todo
        │   └── package.json
        └── ui-core
			└── package.json
{% endhighlight %}

<br/>

## 시작해보자!

todo패지키의 기본 골계를 작성하는 것부터 시작합니다.

plugins/todo/package.json
{% highlight js %}
{
  "name": "@todolist/plugin-todo",
  "private": true,
  "version": "0.0.1",
  "main": "src/index.ts"
}
{% endhighlight %}

<br/>

plugins/todo/src/index.ts
{% highlight ts %}
export * from './types';


export { default } from './components/TodoList/TodoList';
{% endhighlight %}

<br/>

plugins/todo/src/types.ts
{% highlight ts %}
export type Todo = {
  id: string;
  title: string;
  checked: boolean;
};
{% endhighlight %}

<br/>

plugins/todo/src/components/TodoList.tsx
{% highlight tsx %}
import React, { FC } from 'react';
import { Todo } from '../../types';

type TodoListProps = {
  todos: Todo[];
};

const TodoList: FC<TodoListProps> = ({ todos }) => {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input type='checkbox' defaultChecked={todo.checked} />
          <p>{todo.title}</p>
        </li>
      ))}
    </ul>
  );
};

export default TodoList;
{% endhighlight %}

<br/>

다음으로 위에서 제작한 todo 패키지를 사용할 데모 앱을 만들어야 합니다. create-react-app를 사용하여 CRA앱을 생성합니다.

생성된 CRA앱에 Todolist를 렌더링하는 기본적인 코드를 작성합니다.

<br/>

packages/app/src/index.ts
{% highlight tsx %}
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';

ReactDOM.render(<App />, document.getElementById('root'));
};
{% endhighlight %}

<br/>

packages/app/src/App.tsx
{% highlight tsx %}
import React from 'react';
import TodoList from '@todolist/plugin-todo';

const todos = [
  { id: 'a', title: 'a', checked: true },
  { id: 'b', title: 'b', checked: false },
];


function App() {
  return <TodoList todos={todos} />;
}

export default App;
};
{% endhighlight %}

<br/>

하지만 CRA앱을 실행하니 에러를 보여줍니다.

![Error message]({{ site.baseurl }}/assets/img/todolist_series_1/error_message.png)


오류메시지는 Todo패지키에서 Unexpected token이 발견되었다고 알려줍니다. 원인은 타입스크립트를 컴파일하지 않고 그대로 가져와 사용했기 때문입니다. 이를 해결하기위해서 babel을 사용해 Todo패키지를 컴파일 한 뒤 app에서 컴파일된 자바스크립트 코드를 가져와 사용해야 합니다.

<br/>

하지만 매번 코드가 변경될때마다 재컴파일을 하는 일은 매우 번거롭습니다. 우리는 CRA을 사용하기에 eject하지 않고 자동으로 컴파일을 할 수 있는 방법을 찾아야 합니다.

<br/>

### 해결 사항

1. 자동으로 @todolist/plugin-todo 패키지를 컴파일해야 한다.
2. CRA를 eject하지 않아야 한다.

<br/>

### 해결책은 craco

craco는 CRA에서 eject를 하지 않고도 리액트의 webpack을 수정할 수 있게 도와주는 라이브러리입니다. 우선 app 폴더에 craco를 설치하겠습니다.

{% highlight bash %}
$ npm install @craco/craco
{% endhighlight %}

<br/>

다음, package.json 내의 실행 스크립트를 react-scripts에서 craco로 교체합니다.
{% highlight js %}
...
"scripts": {
    "start": "craco start",
    "build": "craco build",
    "test": "craco test"
  },
...
{% endhighlight %}

<br/>

마지막으로 app 폴더에 craco.config.js를 작성해줍니다.

{% highlight js %}
const path = require('path');
const { getLoader, loaderByName } = require('@craco/craco');

module.exports = {
  webpack: {
    configure: (webpackConfig) => {
      const { isFound, match } = getLoader(
        webpackConfig,
        loaderByName('babel-loader')
      );
      if (isFound) {
        const include = Array.isArray(match.loader.include)
          ? match.loader.include
          : [match.loader.include];
        match.loader.include = include.concat(
          path.join(__dirname, '../plugins')
        );
      }
      return webpackConfig;
    },
  },
};
{% endhighlight %}
 include로 컴파일 범위를 지정해 준 것이 보이시나요? plugins의 하위 폴더를 전부 include 배열에 추가했기에 include 배열에 속한 파일은 전부 babel-loader로 컴파일 될 것입니다.

<br/>

다시 CRA 앱을 실행해보니 이번엔 정상적으로 작동하네요😄
![Success Screen]({{ site.baseurl }}/assets/img/todolist_series_1/screen.png)

<br/>

### 마무리

다음 포스팅에선 투두리스트를 구현해보도록 하겠습니다.

<br/>

### Series: 투두리스트 개발기 

***[E1. 투두리스트 개발기 - Mono-Repo구축(w.npm workspaces + carco)](/2022/04/24/todolist-series-01.html)***<br>
[E2. 투두리스트 개발기 - 기능 구현](/2022/04/26/todolist-series-02.html)
