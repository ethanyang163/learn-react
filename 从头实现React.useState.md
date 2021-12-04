# 从头实现 React.useState

## 根据 useState 的定义，写出基础框架

```javascript
function useState(initVal) {
  let _val = initVal;
  const state = _val;
  const setState = (newVal) => {
    _val = newVal;
  };
  return [state, setState];
}

const [count, setCount] = useState(1);
console.log(count);
setCount(2);
console.log(count);
```

输出:

```
1
1
```

## 重新暴露内部变量

在上面的代码中，`setCount` 修改了内部变量的值，我们把`state`变成一个 getter 函数，重新把内部变量暴露出来。

```javascript
function useState(initVal) {
  let _val = initVal;
  const state = () => _val; // 最容易的一种方式
  const setState = (newVal) => {
    _val = newVal;
  };
  return [state, setState];
}

const [count, setCount] = useState(1);
console.log(count()); // updated
setCount(2);
console.log(count()); // updated
```

输出:

```
1
2
```

## 使用立即执行函数引入 React 模块

```javascript
const React = (function () {
  function useState(initVal) {
    let _val = initVal;
    const state = () => _val;
    const setState = (newVal) => {
      _val = newVal;
    };
    return [state, setState];
  }

  return { useState };
})();

const [count, setCount] = React.useState(1);
console.log(count());
setCount(2);
console.log(count());
```

输出:

```
1
2
```

## 定义一个`HelloWorld`组件来使用`useState`:

```javascript
function HelloWorld() {
  const [count, setCount] = React.useState(1);

  return {
    render: () => console.log(count), //模拟JSX
    click: () => setCount(count + 1), // 模拟点击
  };
}
```

## 教会 React 如何渲染一个组件

```javascript
const React = (function () {
  function useState(initVal) {
    let _val = initVal;
    const state = () => _val;
    const setState = (newVal) => {
      _val = newVal;
    };
    return [state, setState];
  }

  // ===新加一个render函数
  function render(Component) {
    const C = Component();
    C.render();
    return C;
  }
  // ===

  return { useState, render };
})();
```

## 从新组织一下目前的代码，并使用 React.render 方法

```javascript
const React = (function () {
  let _val; // 使_val变成React的私有变量
  function useState(initVal) {
    const state = _val || initVal; // 通过useState来访问_val
    const setState = (newVal) => {
      _val = newVal;
    };
    return [state, setState];
  }

  function render(Component) {
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();

function HelloWorld() {
  const [count, setCount] = React.useState(1);

  return {
    render: () => console.log(count),
    click: () => setCount(count + 1),
  };
}

var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
```

输出:

```
1
2
3
4
5
```

到目前为止，看上去已经比较完美了。

## 组件中使用多个`state`

```javascript
function HelloWorld() {
  const [count, setCount] = React.useState(1);
  const [text, setText] = React.useState("apple"); //增加一个state

  return {
    render: () => console.log({ count, text }),
    click: () => setCount(count + 1),
    type: (word) => setText(word), // 增加一个设置方法
  };
}

var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.type("banana");
var App = React.render(HelloWorld);
```

输出:

```
{count: 1, text: "apple"}
{count：2, text： 2}
{count: "apple", text: "apple"}
```

这是由于我们从头到尾都只有一个变量`_val`，接下来我们来解决这个问题，我们引入`hooks`数组和索引`idx`, 每调用一次`useState`,`idx`自增 1。

## 引入`hooks`数组和当前索引`idx`

```javascript
const React = (function () {
  let hooks = []; // updated
  let idx = 0; // updated
  function useState(initVal) {
    const state = hooks[idx] || initVal; //_val被hooks[idx]替代
    const setState = (newVal) => {
      hooks[idx] = newVal; //_val被hooks[idx]替代
    };
    idx++; // 每调用一次useState,idx自增1
    return [state, setState];
  }

  function render(Component) {
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();
```

输出：

```
{count: 1, text: "apple"}
{count：2, text： "apple"}
{count: "pear", text: "apple"}
```

看输出还是没有达到我们预期的效果。

## 重置`idx`

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const setState = (newVal) => {
      hooks[idx] = newVal;
    };
    idx++;
    return [state, setState];
  }

  function render(Component) {
    idx = 0; //每次render的时候重置idx
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
```

输出：

```
{count: 1, text: "apple"}
{count: 1, text: "apple"}
{count: 1, text: "apple"}
```

这还是不对，setState 是在 React.render 之后执行，那个时候`idx`已经被重置为 0。

## 锁住`idx`的值

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const _idx = idx; // updated
    const setState = (newVal) => {
      hooks[_idx] = newVal;
    };
    idx++;
    return [state, setState];
  }

  function render(Component) {
    idx = 0;
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();
```

输出：

```
{count: 1, text: "apple"}
{count: 2, text: "apple"}
{count: 2, text: "pear"}
```

## 最终使用不到 50 行代码完成了 React.useState()和 demo

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const _idx = idx; // updated
    const setState = (newVal) => {
      hooks[_idx] = newVal;
    };
    idx++;
    return [state, setState];
  }

  function render(Component) {
    idx = 0;
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();

function HelloWorld() {
  const [count, setCount] = React.useState(1);
  const [text, setText] = React.useState("apple");

  return {
    render: () => console.log({ count, text }),
    click: () => setCount(count + 1),
    type: (word) => setText(word),
  };
}

var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.type("banana");
var App = React.render(HelloWorld);
```

输出：

```
{count: 1, text: "apple"}
{count: 2, text: "apple"}
{count: 2, text: "pear"}
```
