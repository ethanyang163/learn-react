# 从头实现 React.useState

## 什么是闭包

带状态的函数

练习闭包：

```javascript
function getAdd() {
  let foo = 1;
  return function () {
    foo = foo + 1;
    return foo;
  };
}

const add = getAdd();
console.log(add());
console.log(add());
console.log(add());
console.log(add());
console.log(add());
```

输出:

```javascript
2;
3;
4;
5;
6;
```

有一个比较简洁的写法：

```javascript
const add = (function getAdd() {
  let foo = 1;
  return function () {
    foo = foo + 1;
    return foo;
  };
})();
console.log(add());
console.log(add());
console.log(add());
console.log(add());
console.log(add());
```

这个就是立即执行函数 IIFE

## 开始实现 useState

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

事实上，`setCount` 修改了内部变量的值，我们需要重新把内部变量暴露出来。对上面的代码稍作修改，把`state`变成一个 getter 函数，这实际上是 node 中 hooks 的一种实现方式。

```javascript
function useState(initVal) {
  let _val = initVal;
  const state = () => _val;
  const setState = (newVal) => {
    _val = newVal;
  };
  return [state, setState];
}

const [count, setCount] = useState(1);
console.log(count());
setCount(2);
console.log(count());
```

输出:

```
1
2
```

接下来，我们使用立即执行函数来引入 React 模块：

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

我们定义一个`HelloWorld`组件来使用`useState`:

```javascript
function HelloWorld() {
    const [count, setCount] = React.useState(1);

    return {
        render: () => console.log(count);
        click: () => setCount(count+1);
    }
}
```

同时教会 React 如何来渲染一个组件：

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

  function render(Component) {
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();
```

我们来重新组织一下目前的代码：

```javascript
const React = (function () {
  let _val;
  function useState(initVal) {
    const state = initVal || _val;
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
        render: () => console.log(count);
        click: () => setCount(count+1);
    }
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

一切都很完美，直到`HelloWorld`中有多个状态

```javascript
function HelloWorld() {
  const [count, setCount] = React.useState(1);
  const [text, setText] = React.useState("apple");

  return {
    render: () => console.log(count),
    click: () => setCount(count + 1),
    type: (word) => setText(word),
  };
}

var App = React.render(HelloWorld);
App.click();
var App = React.render(HelloWorld);
App.type();
var App = React.render(HelloWorld);
```

输出:

```
{count: 1, text: "apple"}
{count：2, text： 2}
{count: "apple", text: "apple"}
```

这是由于我们从头到尾都只有一个变量`_val`，为了解决这个问题，我们引入`hooks`数组和索引`idx`, 每调用一次`useState`,`idx`自增 1。

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

  function render(Component) {
    const C = Component();
    C.render();
    return C;
  }

  return { useState, render };
})();
```

我们来重新组织一下目前的代码：

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const setState = (newVal) => {
      initVal = newVal;
    };
    idx++;
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

这是由于`idx`一直在增加没有重置造成的，我们在`render`的时候将`idx`重置为 0。

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const setState = (newVal) => {
      initVal = newVal;
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
{count: 1, text: "apple"}
{count: 1, text: "apple"}
```

由于`setState`是异步的,在`render`之后执行，那个时候`idx`已经被重置为 0。可以使用闭包来锁住`idx`：

```javascript
const React = (function () {
  let hooks = [];
  let idx = 0;
  function useState(initVal) {
    const state = hooks[idx] || initVal;
    const _idx = idx;
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
