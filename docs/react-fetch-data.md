---
id: react-fetch-data
title: 在React中加载数据
sidebar_label: 加载数据
---

## 准备

在学习 React 中加载数据之前，需要掌握一些基础知识，如下：

- [JSX、React 组件](react-getting-started.md)
- [useState](https://zh-hans.reactjs.org/docs/hooks-state.html)
- [useEffect](https://zh-hans.reactjs.org/docs/hooks-effect.html)
- [useReducer](https://zh-hans.reactjs.org/docs/hooks-reference.html#usereducer)
- [使用 Axios 调用 API](https://github.com/axios/axios)
- [ES2015: async/await 语法](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function)
- [自定义 Hook](https://zh-hans.reactjs.org/docs/hooks-custom.html)

使用[create-react-app](https://facebook.github.io/create-react-app/)创建一个空的 React 项目，并在`index.css`中添加上[primitive.css](assets/css/primitive.css)样式。

## 目标

通过本教程可掌握使用 React Hooks 如何加载数据，包括数据加载、加载指示器、错误提示、取消数据加载以及自定义数据加载 hook 以达到复用数据加载处理逻辑的目的。

本文将介绍大家实现一个简单的数据加载 demo：使用[Hacker News API](https://hn.algolia.com/api)获取科技界中的流行文章。

[在 Github 上查看源码](https://github.com/sinoui/react-fetch-data-tutorial)

[查看效果](https://sinoui.github.io/react-fetch-data-tutorial/)

注意：React 即将推出的 Suspense 更适合做数据加载。等 React 推出 Suspense 之后，再补充文章以说明。

## 使用 React Hooks 加载数据

首先预览一下 Hacher News API 的数据结构：

![Hacher News API的数据结构](assets/images/hacker-news-api-view.png)

红框中的字段是我们这次 demo 需要用到的字段。

接着我们实现一下展示文章列表。打开`App.js`文件，定义一个`data`状态，代表从[Hacker News API](https://hn.algolia.com/api)获取到的文章结果。在页面上我们展现出每篇文章的标题、作者，并且可点击查看文章。

```jsx
import React, { useState } from 'react';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  return (
    <div className="container">
      <ul>
        {data.hits.map((article) => (
          <li key={article.objectID}>
            <a href={article.url}>
              {article.title}（作者：{article.author}）
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
```

我们即将用[axios](https://github.com/axios/axios)加载数据，当然你也可以用其他的库或者 Fetch API 加载数据。我们需要在项目中通过`yarn add axios`命令行添加`axios`依赖。我们需要在`App`组件创建时就去加载数据，而加载数据属于“副作用”，需要在 effect hook 中执行数据加载：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  useEffect(async () => {
    const result = await axios(
      'http://hn.algolia.com/api/v1/search?query=react',
    );

    setData(result.data);
  });

  return (
    <div className="container">
      <ul>
        {data.hits.map((article) => (
          <li key={article.objectID}>
            <a href={article.url}>
              {article.title}（作者：{article.author}）
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
```

当你启动上面代码后，发现页面会进入一个死循环，会不停地发送请求。这是因为 effect hook 不仅仅会在组件创建时（准确说是组件 mount 后）被调用，在组件每次渲染时也会被调用（组件状态发生变化就会引起组件重新渲染）。**我们只需要在组件 mount 时加载数据**。我们给`useEffect()`函数的第二参数设置为`[]`，即可让这个 effect hook 只在组件 mount 时被调用，这样就修复了无限发送请求的缺陷：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  useEffect(async () => {
    const result = await axios(
      'http://hn.algolia.com/api/v1/search?query=react',
    );

    setData(result.data);
  }, []);

  return (
    <div className="container">
      <ul>
        {data.hits.map((article) => (
          <li key={article.objectID}>
            <a href={article.url}>
              {article.title}（作者：{article.author}）
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
```

现在启动项目看看页面效果吧。

![Hacker News文章列表](assets/images/react-fetch-data-1.png)

在页面上按下`F12`键打开 devtools，然后打开`console`页签，你会看到一个警告信息：**Effect callbacks are synchronous to prevent race conditions. Put the async function inside:...**。这个警告告诉我们，咱们不能在`useEffect`中用`async`函数。我们稍微调整一下代码，以消除这个警告：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  useEffect(() => {
    async function fetchData() {
      const result = await axios(
        'http://hn.algolia.com/api/v1/search?query=react',
      );

      setData(result.data);
    }

    fetchData();
  }, []);

  return (
    <div className="container">
      <ul>
        {data.hits.map((article) => (
          <li key={article.objectID}>
            <a href={article.url}>
              {article.title}（作者：{article.author}）
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
}

export default App;
```

遵循[关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns)原则，我们创建一个`ArticleList`组件，将文章列表展现代码放入到这个组件中，如下：

`ArticleList.js`:

```jsx
import React from 'react';

function ArticleList(props) {
  const { articles } = props;

  return (
    <ul>
      {articles.map((article) => (
        <li key={article.objectID}>
          <a href={article.url}>
            {article.title}（作者：{article.author}）
          </a>
        </li>
      ))}
    </ul>
  );
}

export default ArticleList;
```

`App.js`:

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  useEffect(() => {
    async function fetchData() {
      const result = await axios(
        'http://hn.algolia.com/api/v1/search?query=react',
      );

      setData(result.data);
    }

    fetchData();
  }, []);

  return (
    <div className="container">
      <ArticleList articles={data.hits} />
    </div>
  );
}

export default App;
```

我们将文章展现的代码提取到`ArticleList`组件中，这个过程称之为[提取组件](https://zh-hans.reactjs.org/docs/components-and-props.html#extracting-components)。我们将这种通过调整代码以提升代码可读性的行为，称之为[重构](https://baike.baidu.com/item/%E9%87%8D%E6%9E%84/2182519?fr=aladdin)。重构应发生在我们编码过程中的时时刻刻，当我们发现代码很难被阅读时，就是我们停下来思考如何让它变得更清晰并付诸行动的时候。在重构中的说法就是，一旦你发现代码有坏味道，别放过它。

## 如何再次执行 effect hook？

刚刚实现的文章列表，只会在组件 mount 时加载数据。这次我们需要添加一个查询功能，默认查询的是`react`相关的文章，我们可以通过查询功能查询其他相关文章，如查询`redux`、`angular`等相关的文章。这就需要我们在查询关键词发生变化时再次执行发送请求的 effect hook。

首先，我们创建一个查询表单组件`QueryForm.js`，这个查询表单会在提交表单时调用`handleSubmit`属性，并将`query`传递给`handleSubmit()`。

```jsx
import React, { useState } from 'react';

function QueryForm(props) {
  const [query, setQuery] = useState('react');

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault();
        props.handleSubmit(query);
      }}
    >
      <input
        type="text"
        value={query}
        onChange={(event) => setQuery(event.target.value)}
      />
      <input type="submit" value="查询" />
    </form>
  );
}

export default QueryForm;
```

我们在`App.js`中，加载`QueryForm.js`：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';

function App() {
  const [data, setData] = useState({
    hits: [],
  });

  useEffect(() => {
    async function fetchData() {
      const result = await axios(
        'http://hn.algolia.com/api/v1/search?query=react',
      );

      setData(result.data);
    }

    fetchData();
  }, []);

  return (
    <div className="container">
      <QueryForm />
      <ArticleList articles={data.hits} />
    </div>
  );
}

export default App;
```

`App`组件需要维护查询关键字这个状态，取名为`search`，当`QueryForm`提交时，更新`search`状态，如下所示：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';

function App() {
  const [data, setData] = useState({
    hits: [],
  });
  const [search, setSearch] = useState('react');

  useEffect(() => {
    async function fetchData() {
      const result = await axios(
        'http://hn.algolia.com/api/v1/search?query=react',
      );

      setData(result.data);
    }

    fetchData();
  }, []);

  return (
    <div className="container">
      <QueryForm handleSubmit={setSearch} />
      <ArticleList articles={data.hits} />
    </div>
  );
}

export default App;
```

打开页面，在输入框中输入`redux`，然后点击查询，你会发现文章列表纹丝不动。这是因为`useEffect(fn, [])`的第二个参数是`[]`，导致这个 effect hook 只会在组件 mount 时被执行，而组件更新时不会被调用。为了让 effect hook 在`search`状态发生变化时也执行一次，我们需要将`search`状态放在第二个参数的数组中，即`useEffect(fn, [search])`：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';

function App() {
  const [data, setData] = useState({
    hits: [],
  });
  const [search, setSearch] = useState('react');

  useEffect(() => {
    async function fetchData() {
      const result = await axios(
        `http://hn.algolia.com/api/v1/search?query=${search}`,
      );

      setData(result.data);
    }

    fetchData();
  }, [search]);

  return (
    <div className="container">
      <QueryForm handleSubmit={setSearch} />
      <ArticleList articles={data.hits} />
    </div>
  );
}

export default App;
```

打开页面，在输入框中输入`redux`或者任何你想输入的文字，点击查询按钮，你的文章列表就会在很短的时间内发生变化。效果如下：

![Hacker News文章列表](assets/images/react-fetch-data-2.png)

`QueryForm`中的`query`状态与`App`中的`search`状态在初始和表单提交时是一样的值。有时，这样的情况可能令人感到困惑，因为都是表单查询关键字（有细微区别），而且在某些时刻是一样的值。我们其实是想在`QueryForm`提交查询时更改一下发送请求的`url`，所以为何我们不将`App`中的`search`状态替换成`url`状态呢？

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';

function App() {
  const [data, setData] = useState({
    hits: [],
  });
  const [url, setUrl] = useState(
    'http://hn.algolia.com/api/v1/search?query=react',
  );

  useEffect(() => {
    async function fetchData() {
      const result = await axios(url);

      setData(result.data);
    }

    fetchData();
  }, [url]);

  const handleSubmit = (query) => {
    setUrl(`http://hn.algolia.com/api/v1/search?query=${query}`);
  };

  return (
    <div className="container">
      <QueryForm handleSubmit={handleSubmit} />
      <ArticleList articles={data.hits} />
    </div>
  );
}

export default App;
```

调整完代码后，在页面上试试搜索`redux`，看看文章列表是否发生了变化。

以上代码实现了对 effect hook 执行的精确控制。我们在更新查询关键字然后点击查询时，会导致`App`组件的`url`发生变化，而 effect hook 的第二个参数是`[url]`，这样，effect hook 就会在这个时刻再次被执行，从而达到再次发送请求的效果。

## 加载状态指示器（加载中）

让我们继续前进：在加载数据时显示加载状态提示器。

首先创建一个`LoadingIndicator`组件，表示加载状态提示器：

`LoadingIndicator.js`：

```jsx
import React from 'react';

function LoadingIndicator() {
  return <div>加载中...</div>;
}

export default LoadingIndicator;
```

在`App`组件中定义一个`isLoading`状态，用来表示是否正在加载数据。在发送请求之前和之后更新`isLoading`状态。

`App.js`：

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';
import LoadingIndicator from './LoadingIndicator';

function App() {
  const [data, setData] = useState({
    hits: [],
  });
  const [url, setUrl] = useState(
    'http://hn.algolia.com/api/v1/search?query=react',
  );
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      const result = await axios(url);

      setData(result.data);
      setIsLoading(false);
    }

    fetchData();
  }, [url]);

  const handleSubmit = (query) => {
    setUrl(`http://hn.algolia.com/api/v1/search?query=${query}`);
  };

  return (
    <div className="container">
      <QueryForm handleSubmit={handleSubmit} />
      {isLoading ? <LoadingIndicator /> : <ArticleList articles={data.hits} />}
    </div>
  );
}

export default App;
```

打开页面预览一下效果，输入新的查询关键字，点击查询，你会看到会出现“加载中...”提示，等加载完成后，提示消失，显示文章列表。

## 错误提示

如何处理请求数据的错误呢？与加载提示器一样，加一个`error`状态来处理。

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';
import LoadingIndicator from './LoadingIndicator';

function App() {
  const [data, setData] = useState({
    hits: [],
  });
  const [url, setUrl] = useState(
    'http://hn.algolia.com/api/v1/search?query=react',
  );
  const [isLoading, setIsLoading] = useState(false);
  const [isError, setIsError] = useState(false);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      setIsError(false);

      try {
        const result = await axios(url);

        setData(result.data);
      } catch (error) {
        setIsError(true);
      }
      setIsLoading(false);
    }

    fetchData();
  }, [url]);

  const handleSubmit = (query) => {
    setUrl(`http://hn.algolia.com/api/v1/search?query=${query}`);
  };

  return (
    <div className="container">
      <QueryForm handleSubmit={handleSubmit} />
      {isError && <div>加载数据失败。</div>}
      {isLoading ? <LoadingIndicator /> : <ArticleList articles={data.hits} />}
    </div>
  );
}

export default App;
```

你可以打开 devtools，在`Network`面板上将网络设置为`offline`，输入新的关键字，点击查询，页面上就会出现错误提示。

## 自定义获取数据的 hook

如果现在让我们再开发一个类似的数据获取类的程序，我们应该怎么做？跟上面介绍的思路基本一致，关于数据获取的代码也基本一样。所以，我们可以将数据获取的代码提炼出来以复用。这个章节会向大家介绍如何自定义获取数据的 hook，以达到复用的目的。另外，使用自定义 hook，也能提升代码的可读性和可维护性，有利于将 UI 与逻辑处理分离开。

要想将数据获取放到自定义 hook 中，先自定义一个 hook，命名为`useHackerNewsApi`，然后将与数据获取相关的状态和 effect hook 迁移到`useHackerNewsApi`中，最后将`data`、`isLoading`、`isError`和更新 url 的方法（这个方法会在`App`的`handleSubmit()`中调用）返回给`App`组件。代码如下：

`useHackerNewsApi.js`:

```js
import { useState, useEffect } from 'react';
import axios from 'axios';

function useHackerNewsApi() {
  const [data, setData] = useState({
    hits: [],
  });
  const [url, setUrl] = useState(
    'http://hn.algolia.com/api/v1/search?query=react',
  );
  const [isLoading, setIsLoading] = useState(false);
  const [isError, setIsError] = useState(false);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      setIsError(false);

      try {
        const result = await axios(url);

        setData(result.data);
      } catch (error) {
        setIsError(true);
      }
      setIsLoading(false);
    }

    fetchData();
  }, [url]);

  function doFetch(url) {
    setUrl(url);
  }

  return { data, isLoading, isError, doFetch };
}

export default useHackerNewsApi;
```

`App.js`:

```jsx
import React from 'react';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';
import LoadingIndicator from './LoadingIndicator';
import useHackerNewsApi from './useHackerNewsApi';

function App() {
  const { data, isLoading, isError, doFetch } = useHackerNewsApi();

  const handleSubmit = (query) => {
    doFetch(`http://hn.algolia.com/api/v1/search?query=${query}`);
  };

  return (
    <div className="container">
      <QueryForm handleSubmit={handleSubmit} />
      {isError && <div>加载数据失败。</div>}
      {isLoading ? <LoadingIndicator /> : <ArticleList articles={data.hits} />}
    </div>
  );
}

export default App;
```

打开页面看看效果，确保应用能够正常运行。

我们还不能在其他场景下复用`useHackerNewsApi`，因为这个自定义 hook 中的初始 url 和初始数据都是与`Hacker News API`有关的。我们可以创建一个更通用的 hook，它会接受`初始url`和`初始数据`两个参数，其他逻辑处理与`useHackerNewsApi`基本一致，这个自定义 hook 取名为`useDataApi`。代码如下：

`useDataApi.js`:

```jsx
import { useState, useEffect } from 'react';
import axios from 'axios';

function useDataApi(initialUrl, initialData) {
  const [data, setData] = useState(initialData);
  const [url, setUrl] = useState(initialUrl);
  const [isLoading, setIsLoading] = useState(false);
  const [isError, setIsError] = useState(false);

  useEffect(() => {
    async function fetchData() {
      setIsLoading(true);
      setIsError(false);

      try {
        const result = await axios(url);

        setData(result.data);
      } catch (error) {
        setIsError(true);
      }
      setIsLoading(false);
    }

    fetchData();
  }, [url]);

  function doFetch(url) {
    setUrl(url);
  }

  return { data, isLoading, isError, doFetch };
}

export default useDataApi;
```

`App.js`:

```jsx
import React from 'react';
import ArticleList from './ArticleList';
import QueryForm from './QueryForm';
import LoadingIndicator from './LoadingIndicator';
import useDataApi from './useDataApi';

function App() {
  const { data, isLoading, isError, doFetch } = useDataApi(
    'http://hn.algolia.com/api/v1/search?query=react',
    {
      hits: [],
    },
  );

  const handleSubmit = (query) => {
    doFetch(`http://hn.algolia.com/api/v1/search?query=${query}`);
  };

  return (
    <div className="container">
      <QueryForm handleSubmit={handleSubmit} />
      {isError && <div>加载数据失败。</div>}
      {isLoading ? <LoadingIndicator /> : <ArticleList articles={data.hits} />}
    </div>
  );
}

export default App;
```

打开页面看看效果，确保应用能够正常运行。

经过提炼后的`useDataApi` hook 非常具有复用性，可以用在其他获取数据的场景中。你可以将`useDataApi`分享给团队中的其他同事，或者发布到 npm 中分享给需要的人。

## 使用 useReducer 加载数据

我们在程序中用了三个状态钩子（state hook）来管理我们的数据获取状态：数据、加载中和错误状态。这些状态都是由自己的状态钩子管理，但是却会由同一个原因导致同时更新多个状态。如你所见，这三个状态的更新都发生在数据获取方法中。在数据加载前，同时调用`setIsError(false)`和`setIsLoading(true)`，在数据加载完成后，同时调用`setData()`和`setIsLoading(false)`，在数据加载失败时同时调用`setIsError(true)`和`setIsLoading(false)`。这个章节会介绍使用 reducer hook 来同时管理这三个状态数据。

Reducer hook 返回一个状态对象`state`和一个改变状态对象的函数。该函数（称为**dispatch**函数）采用具有**类型（type）**和可选的**有效负载（payload）**的**操作（action）**。所有这些信息都在实际的 reducer 函数中用于从先前的状态、动作的有效负载和类型中提取新状态。让我们看看它在代码中时如何工作的：

`useDataApi.js`:

```js
import { useState, useEffect, useReducer } from 'react';
import axios from 'axios';

function dataFetchReducer(state, action) {
  // 稍后补充
}

function useDataApi(initialUrl, initialData) {
  const [url, setUrl] = useState(initialUrl);

  const [state, dispatch] = useReducer(dataFetchReducer, {
    isLoading: false,
    isError: false,
    data: initialData,
  });

  // 剩下部分代码稍后补充

  return { ...state, doFetch };
}
```

reducer hook 将 reducer 函数和初始状态对象作为参数。在我们的例子中，我们的初始化状态对象中包含了`data`、`isLoading`和`isError`。

接下来，我们在开始加载、加载完成和加载失败这三个时刻通过`dispatch`方法，发送合适的`action`。`action`是具有**type**属性和可选的用来承载额外数据的**payload**属性组成的对象。我们定义三个操作类型（action type）：`FETCH_INIT`代表开始加载、`FETCH_SUCCESS`代表加载成功、`FETCH_FAILURE`代表加载失败。如下所示：

`useDataApi.js`:

```js
import { useState, useEffect, useReducer } from 'react';
import axios from 'axios';

function dataFetchReducer(state, action) {
  // 稍后补充
}

function useDataApi(initialUrl, initialData) {
  const [url, setUrl] = useState(initialUrl);

  const [state, dispatch] = useReducer(dataFetchReducer, {
    isLoading: false,
    isError: false,
    data: initialData,
  });

  useEffect(() => {
    async function fetchData() {
      dispatch({ type: 'FETCH_INIT' });

      try {
        const result = await axios(url);

        dispatch({ type: 'FETCH_SUCCESS', payload: result.data });
      } catch (error) {
        dispatch({ type: 'FETCH_FAILURE' });
      }
    }

    fetchData();
  }, [url]);

  const doFetch = (url) => {
    setUrl(url);
  };

  return { ...state, doFetch };
}
```

通过`dispatch`函数发送的操作(action)都会传递给`dataFetchReducer`函数。在`dataFetchReducer()`中，可以根据`action`的类型产生新的状态对象返回。代码如下：

```js
const dataFetchReducer = (state, action) => {
  switch (action.type) {
    case 'FETCH_INIT':
      return { ...state, isLoading: true, isError: false };
    case 'FETCH_SUCCESS':
      return {
        ...state,
        isLoading: false,
        isError: false,
        data: action.payload,
      };
    case 'FETCH_FAILURE':
      return { ...state, isLoading: false, isError: true };
    default:
      return state;
  }
};
```

完整的`useDataApi`如下：

```js
import { useState, useEffect, useReducer } from 'react';
import axios from 'axios';

function dataFetchReducer(state, action) {
  switch (action.type) {
    case 'FETCH_INIT':
      return {
        ...state,
        isError: false,
        isLoading: true,
      };
    case 'FETCH_SUCCESS':
      return {
        ...state,
        isLoading: false,
        isError: false,
        data: action.payload,
      };
    case 'FETCH_FAILURE':
      return {
        ...state,
        isError: true,
        isLoading: false,
      };
    default:
      return state;
  }
}

function useDataApi(initialUrl, initialData) {
  const [state, dispatch] = useReducer(dataFetchReducer, {
    isLoading: false,
    isError: false,
    data: initialData,
  });
  const [url, setUrl] = useState(initialUrl);

  useEffect(() => {
    const fetchData = async () => {
      dispatch({ type: 'FETCH_INIT' });

      try {
        const result = await axios(url);

        dispatch({ type: 'FETCH_SUCCESS', payload: result.data });
      } catch (error) {
        dispatch({ type: 'FETCH_FAILURE' });
      }
    };

    fetchData();
  }, [url]);

  const doFetch = (url) => {
    setUrl(url);
  };

  return { ...state, doFetch };
}

export default useDataApi;
```

打开页面看看效果，确保应用能够正常运行。

## 在 Effect hook 中取消数据加载

在 React 中有一个通用问题：在组件销毁时组件的状态仍然被修改。比如在页面切换时，组件就会被销毁。我们的应用程序也会面临这样的情况：如果在数据加载过程中组件被销毁了，但是 api 请求却没有被取消，当 api 请求成功后，仍然会修改状态。我们可以给数据加载 effect hook 添加一个可以取消请求的函数作为返回值，这个函数会在组件销毁时被调用，具体详情参见[需要清除的 effect](https://zh-hans.reactjs.org/docs/hooks-effect.html#%E9%9C%80%E8%A6%81%E6%B8%85%E9%99%A4%E7%9A%84-effect)。我们的代码如下：

`useDataApi.js`:

```js
import { useState, useEffect, useReducer } from 'react';
import axios from 'axios';

function dataFetchReducer(state, action) {
  switch (action.type) {
    case 'FETCH_INIT':
      return {
        ...state,
        isError: false,
        isLoading: true,
      };
    case 'FETCH_SUCCESS':
      return {
        ...state,
        isLoading: false,
        isError: false,
        data: action.payload,
      };
    case 'FETCH_FAILURE':
      return {
        ...state,
        isError: true,
        isLoading: false,
      };
    default:
      return state;
  }
}

function useDataApi(initialUrl, initialData) {
  const [state, dispatch] = useReducer(dataFetchReducer, {
    isLoading: false,
    isError: false,
    data: initialData,
  });
  const [url, setUrl] = useState(initialUrl);

  useEffect(() => {
    let didCancel = false;
    const fetchData = async () => {
      dispatch({ type: 'FETCH_INIT' });

      try {
        const result = await axios(url);

        if (!didCancel) {
          dispatch({ type: 'FETCH_SUCCESS', payload: result.data });
        }
      } catch (error) {
        if (!didCancel) {
          dispatch({ type: 'FETCH_FAILURE' });
        }
      }
    };

    fetchData();

    return () => {
      didCancel = true;
    };
  }, [url]);

  const doFetch = (url) => {
    setUrl(url);
  };

  return { ...state, doFetch };
}

export default useDataApi;
```

我们在加载数据的 effect 中添加了一个`didCancel`变量，当组件销毁时，会调用`effect`的清除函数，将`didCancel`设置为`ture`，这样就可以阻止在组件销毁后 api 请求成功时仍然修改状态的情况，就相当于取消了数据加载。

> 注意：
>
> 文中提到**effect 的清除函数会在组件销毁时被调用**，这种说法不太准确，有兴趣的同学可以参考一下 Dan Abranmov 的[useEffect 完整指南](https://overreacted.io/zh-hans/a-complete-guide-to-useeffect/#%E9%82%A3effect%E4%B8%AD%E7%9A%84%E6%B8%85%E7%90%86%E5%8F%88%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%E5%91%A2%EF%BC%9F)。

这就是我们这篇文章的全部了。我们的技术文章查询应用就完成了。

## 总结

这篇文章详细地介绍了如何使用 react hooks 加载数据，不仅能让大家够掌握加载数据的技能，同时也能通过实战让大家加深对 react hooks 的理解，特别是 state hook 和 effect hook。

本篇文章也介绍了 reducer hook 的用法。reducer hook 是[redux](https://redux.js.org/)的简化版本，可用于处理复杂的状态逻辑。

[在 Github 上查看源码](https://github.com/sinoui/react-fetch-data-tutorial)

[查看效果](https://sinoui.github.io/react-fetch-data-tutorial/)

如果这篇文章有不正确或者不明确的地方，请[告知我们](https://github.com/sinoui/sinoui-guide/issues)（添加 github issue）。

## 参考文章

- [How to fetch data with React Hooks?](https://www.robinwieruch.de/react-hooks-fetch-data/)
- [How to fetch data in React](https://www.robinwieruch.de/react-hooks-fetch-data/)
- [JSX、React 组件](react-getting-started.md)
- [useState](https://zh-hans.reactjs.org/docs/hooks-state.html)
- [useEffect](https://zh-hans.reactjs.org/docs/hooks-effect.html)
- [useReducer](https://zh-hans.reactjs.org/docs/hooks-reference.html#usereducer)
- [使用 Axios 调用 API](https://github.com/axios/axios)
- [redux](https://redux.js.org/)
