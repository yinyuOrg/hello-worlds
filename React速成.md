# React 速成｜12 章笔记直达实战

> 严格说 React 不是一门新语言，而是 **JavaScript 的 UI 库**。有 JS/TS 基础的人，核心难点不在语法，而在 **组件思维、状态管理、单向数据流**。

## 第1章 React 快速认知

**一、React 与其他方案的核心差异**

1. **声明式 UI**：你描述“界面长什么样”，React 负责把它画出来；不用像 jQuery 那样手动操作 DOM
2. **组件化**：页面由独立、可复用的小组件拼装而成，一个组件 = 一个函数
3. **单向数据流**：数据自顶向下通过 props 传递，子组件不能直接改父组件数据
4. **虚拟 DOM**：React 先在内存中比较差异，再最小化更新真实 DOM

**二、React 程序固定结构**

```tsx
// App.tsx
function App() {
  return <h1>Hello React!</h1>;
}

export default App;
```

```tsx
// main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

- `App`：根组件，所有 UI 从这里开始
- `ReactDOM.createRoot`：创建根渲染节点
- `React.StrictMode`：开发模式下帮你检查潜在问题的工具

**三、必守规则**

- 组件名必须**大驼峰**，如 `UserCard`，不能写 `usercard`
- JSX 只能返回**一个根元素**（可用 `<></>` 片段包裹）
- `class` 要写成 `className`，`for` 要写成 `htmlFor`
- JSX 里的 `{}` 只能放表达式，不能放 `if`/`for` 等语句

**四、React 组件执行流程【要懂】**

状态/Props 变化 → React 重新调用组件函数 → 生成新的虚拟 DOM → Diff 比较 → 最小化更新真实 DOM

**口诀：组件拼装，状态驱动。**

**React 实战场景**：把一个后台管理页面拆成 `Sidebar` + `Header` + `Main` + `Footer`，每个区域各自负责自己的 UI 和状态。

---

## 第2章 JSX 语法

**一、JSX 的本质**

JSX 不是 HTML，它是 JavaScript 的语法扩展，编译后会被转换成 `React.createElement` 调用。

```tsx
// 你写的 JSX
const element = <h1 className="title">Hello</h1>;

// 编译后大致等价于
const element = React.createElement('h1', { className: 'title' }, 'Hello');
```

**二、JSX 核心规则**

| 规则 | 说明 | 示例 |
| --- | --- | --- |
| 单根元素 | 必须有一个外层标签包裹 | `<div>...</div>` 或 `<></>` |
| 插值表达式 | 用 `{}` 嵌入 JS 表达式 | `{name}`、`{1 + 2}` |
| 属性命名 | 驼峰式，`className` 代替 `class` | `<div className="box">` |
| 自闭合标签 | 无子元素必须闭合 | `<img src="..." />` |

**三、JSX 里能写什么**

```tsx
function Greeting({ name }: { name: string }) {
  const isAdmin = true;

  return (
    <div className="greeting">
      <h1>你好，{name}</h1>
      <p>{isAdmin ? '管理员' : '普通用户'}</p>
      <p>当前时间：{new Date().toLocaleDateString()}</p>
    </div>
  );
}
```

- `{}` 里可以放：变量、函数调用、三元表达式、数组 `map` 结果
- `{}` 里**不能放**：`if` 语句、`for` 循环、对象声明（但可以在外部算好再插入）

**四、样式写法**

```tsx
// 行内样式：注意是对象，不是字符串
const style = { color: 'red', fontSize: '16px' };

function Box() {
  return <div style={style}>红色文字</div>;
}

// 或者直接写
function Box2() {
  return <div style={{ color: 'red' }}>红色文字</div>;
}
```

**口诀：JS 里写 HTML，表达式要加 `{}`。**

**React 实战场景**：用 JSX 渲染一个用户信息卡片，把头像、昵称、角色状态通过 props 和条件表达式一次性拼出来。

---

## 第3章 组件

**一、组件的核心概念**

组件是 React 应用的基本单位，本质上是一个接收 props、返回 JSX 的函数。

```tsx
// Button.tsx
function Button({ text }: { text: string }) {
  return <button>{text}</button>;
}

export default Button;
```

**二、组件拆分原则**

| 原则 | 说明 |
| --- | --- |
| 单一职责 | 一个组件只做一件事 |
| 复用优先 | 重复出现的 UI 提取成组件 |
| 命名即意图 | `UserAvatar` 比 `Img` 更有意义 |

**三、组件组合示例**

```tsx
function UserCard({ name, age }: { name: string; age: number }) {
  return (
    <div className="user-card">
      <h2>{name}</h2>
      <p>年龄：{age}</p>
    </div>
  );
}

function App() {
  return (
    <div>
      <UserCard name="张三" age={20} />
      <UserCard name="李四" age={25} />
    </div>
  );
}
```

**四、props 是只读的**

```tsx
function Counter({ count }: { count: number }) {
  // 错误：props 不能修改
  // count = count + 1;

  return <div>{count}</div>;
}
```

> 冷知识：React 组件就像纯函数，给定同样的 props，应该渲染出同样的 UI。

**口诀：组件是函数，UI 靠返回。**

**React 实战场景**：把一个复杂的商品详情页拆成 `ProductImage`、`ProductInfo`、`PriceTag`、`AddToCartButton` 等小组件，方便独立维护。

---

## 第4章 Props

**一、Props 是什么**

Props（properties）是父组件向子组件传递数据的方式，可以理解为函数的参数。

```tsx
// 父组件
function App() {
  return <Avatar src="/logo.png" alt="Logo" size={64} />;
}

// 子组件
function Avatar({ src, alt, size }: { src: string; alt: string; size: number }) {
  return <img src={src} alt={alt} width={size} height={size} />;
}
```

**二、Props 常见写法**

| 写法 | 说明 | 示例 |
| --- | --- | --- |
| 解构接收 | 推荐，直接拿到字段 | `({ name })` |
| 设置默认值 | 解构时赋值 | `({ count = 0 })` |
| children | 组件标签中间的内容 | `<Card>内容</Card>` |

```tsx
// 默认值
function Button({ text = '确认' }: { text?: string }) {
  return <button>{text}</button>;
}

// children
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}

function App() {
  return (
    <Card>
      <h2>标题</h2>
      <p>这是卡片内容</p>
    </Card>
  );
}
```

**三、Props 是只读的【重点】**

子组件不能修改 props。想让子组件影响父组件，必须传一个回调函数过去。

**口诀：Props 向下传，数据单向流。**

**React 实战场景**：封装一个通用 `Modal` 组件，通过 `title`、`visible`、`onClose`、`children` 等 props 控制内容和行为。

---

## 第5章 State

**一、State 是什么**

State 是组件内部的可变数据，当 state 变化时，React 会自动重新渲染组件。

```tsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>当前计数：{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}
```

**二、useState 使用规则**

| 规则 | 说明 |
| --- | --- |
| 数组解构 | `const [state, setState] = useState(initialValue)` |
| 初始值 | 只在首次渲染时生效 |
| 更新函数 | 调用后触发重新渲染 |

**三、State 不可变更新【重点】**

不要直接修改 state，应该创建新值。

```tsx
// 错误：直接修改原数组
const [list, setList] = useState([1, 2, 3]);
list.push(4);       // ❌

// 正确：创建新数组
setList([...list, 4]);  // ✅
```

```tsx
// 错误：直接修改对象
const [user, setUser] = useState({ name: '张三', age: 20 });
user.age = 21;      // ❌

// 正确：创建新对象
setUser({ ...user, age: 21 });  // ✅
```

**四、函数式更新**

当新状态依赖旧状态时，用函数式更新更可靠。

```tsx
<button onClick={() => setCount(c => c + 1)}>+1</button>
```

**口诀：State 不可变，更新用函数。**

**React 实战场景**：购物车数量加减、表单输入实时同步、Tab 切换高亮当前选项。

---

## 第6章 事件处理

**一、React 事件和 DOM 事件的区别**

- React 事件是**合成事件（SyntheticEvent）**，兼容性更好
- 命名用驼峰：`onClick`、`onChange`、`onSubmit`
- 事件处理函数写成 JSX 表达式

```tsx
function Button() {
  function handleClick() {
    alert('点击了按钮');
  }

  return <button onClick={handleClick}>点击我</button>;
}
```

**二、传递参数**

```tsx
function ItemList() {
  const items = ['A', 'B', 'C'];

  function handleClick(name: string) {
    console.log('点击了：', name);
  }

  return (
    <ul>
      {items.map(item => (
        <li key={item} onClick={() => handleClick(item)}>
          {item}
        </li>
      ))}
    </ul>
  );
}
```

**三、事件对象与阻止默认行为**

```tsx
function Form() {
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();      // 阻止表单默认提交刷新
    console.log('提交表单');
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">提交</button>
    </form>
  );
}
```

**口诀：事件驼峰名，传参用箭头。**

**React 实战场景**：按钮提交、列表项点击选中、输入框实时搜索、文件上传触发。

---

## 第7章 条件渲染

**一、三种条件渲染方式**

```tsx
function Greeting({ isLogin }: { isLogin: boolean }) {
  // 方式1：if/else 提前返回
  if (!isLogin) {
    return <p>请先登录</p>;
  }

  // 方式2：三元表达式
  return <p>{isLogin ? '欢迎回来' : '请先登录'}</p>;
}

function Message({ count }: { count: number }) {
  // 方式3：&& 短路渲染
  return (
    <div>
      {count > 0 && <p>你有 {count} 条未读消息</p>}
    </div>
  );
}
```

**二、条件渲染常见坑**

| 坑 | 说明 | 示例 |
| --- | --- | --- |
| `&&` 接 0 | `0 && <X />` 会渲染出 `0` | 用 `count > 0 && ...` |
| `&&` 接空字符串 | `'' && <X />` 会渲染空字符串 | 用 `name ? <X /> : null` |
| 三元套太多 | 超过两层可读性差 | 抽成组件或用 if |

**口诀：`&&` 防 falsy，`? :` 做二选一。**

**React 实战场景**：登录后显示用户头像，未登录显示“登录”按钮；VIP 用户显示专属标签，普通用户隐藏。

---

## 第8章 列表渲染

**一、核心方法：map**

```tsx
function TodoList() {
  const todos = [
    { id: 1, text: '学习 React', done: false },
    { id: 2, text: '写组件', done: true },
  ];

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.done ? '✅' : '⬜'} {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**二、key 的重要性【必背】**

- 每个列表项都需要一个稳定的 `key`
- key 帮助 React 识别哪些项变了、增删改位置
- **不要用数组索引作为 key**（除非列表从不增删改顺序）

```tsx
// 错误：用 index 当 key
{todos.map((todo, index) => <li key={index}>{todo.text}</li>)}

// 正确：用唯一 id 当 key
{todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
```

**三、列表过滤/排序**

```tsx
function NumberList() {
  const numbers = [5, 2, 8, 1, 9];
  const sorted = [...numbers].sort((a, b) => a - b);

  return (
    <ul>
      {sorted.map(n => <li key={n}>{n}</li>)}
    </ul>
  );
}
```

**口诀：列表必加 key，稳定是底线。**

**React 实战场景**：渲染商品列表、评论列表、消息通知列表，配合过滤条件实时更新显示。

---

## 第9章 useEffect

**一、useEffect 的作用**

用于处理副作用：数据请求、订阅、手动操作 DOM、定时器等。

```tsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<{ name: string } | null>(null);

  useEffect(() => {
    // 副作用函数
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));

    // 可选的清理函数
    return () => {
      console.log('组件卸载或 userId 变化时清理');
    };
  }, [userId]); // 依赖数组

  if (!user) return <p>加载中...</p>;

  return <p>用户名：{user.name}</p>;
}
```

**二、依赖数组规则**

| 依赖数组 | 执行时机 |
| --- | --- |
| 不传 | 每次渲染后都执行（很少用） |
| `[]` | 只在组件挂载时执行一次 |
| `[a, b]` | 依赖变化时执行 |

**三、常见错误**

```tsx
// 错误：忘记把 count 加入依赖数组
const [count, setCount] = useState(0);
useEffect(() => {
  console.log(count);
}, []); // 警告：count 变化不会重新执行

// 错误：依赖数组里放了对象/数组，导致每次都变
useEffect(() => {
  // do something
}, [{ id: 1 }]); // 每次渲染都是新对象，无限触发
```

**口诀：依赖写全，清理别忘。**

**React 实战场景**：页面加载时请求用户数据、监听窗口尺寸变化、设置定时器倒计时、WebSocket 连接与断开。

---

## 第10章 表单与受控组件

**一、受控组件**

表单元素的值由 React 的 state 控制，输入框的显示内容由 state 决定。

```tsx
import { useState } from 'react';

function LoginForm() {
  const [username, setUsername] = useState('');

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    setUsername(e.target.value);
  }

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log('提交：', username);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={username}
        onChange={handleChange}
        placeholder="请输入用户名"
      />
      <button type="submit">登录</button>
    </form>
  );
}
```

**二、受控 vs 非受控**

| 类型 | 值由谁控制 | 获取值方式 | 适用场景 |
| --- | --- | --- | --- |
| 受控组件 | React state | `state` 变量 | 需要实时校验、联动 |
| 非受控组件 | DOM 自身 | `useRef` | 简单表单、文件上传 |

**三、多个字段的表单**

```tsx
function RegisterForm() {
  const [form, setForm] = useState({ username: '', password: '' });

  function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  }

  return (
    <form>
      <input name="username" value={form.username} onChange={handleChange} />
      <input name="password" value={form.password} onChange={handleChange} />
    </form>
  );
}
```

**口诀：受控组件 value + onChange 配对。**

**React 实战场景**：登录/注册表单、搜索框实时联想、配置面板保存前校验。

---

## 第11章 组件通信

**一、父子组件通信**

父传子用 props，子传父用回调函数。

```tsx
// 父组件
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>父组件 count：{count}</p>
      <Child count={count} onAdd={() => setCount(c => c + 1)} />
    </div>
  );
}

// 子组件
function Child({ count, onAdd }: { count: number; onAdd: () => void }) {
  return (
    <div>
      <p>子组件收到：{count}</p>
      <button onClick={onAdd}>+1</button>
    </div>
  );
}
```

**二、兄弟组件通信**

兄弟组件不能直接通信，需要把状态提升到最近的公共父组件。

```tsx
function App() {
  const [text, setText] = useState('');

  return (
    <div>
      <Input onChange={setText} />
      <Preview text={text} />
    </div>
  );
}
```

**三、跨层级通信：Context**

当层级很深时，用 Context 避免层层传递 props。

```tsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext('light');

function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Toolbar />
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  return <ThemedButton />;
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button>{theme}</button>;
}
```

**口诀：父子靠 props，深层用 Context。**

**React 实战场景**：主题切换（深色/浅色模式）、用户登录状态全局共享、多语言切换。

---

## 第12章 Hooks 规则

**一、Hooks 是什么**

Hooks 是 React 16.8 引入的函数，让你在函数组件里使用 state、副作用等能力。

常用 Hooks：

| Hook | 作用 |
| --- | --- |
| `useState` | 状态管理 |
| `useEffect` | 副作用处理 |
| `useContext` | 跨层级共享数据 |
| `useRef` | 获取 DOM 引用或保存不变值 |
| `useMemo` | 缓存计算结果 |
| `useCallback` | 缓存函数引用 |

**二、Hooks 两大铁律**

1. **只在函数组件或自定义 Hook 的最顶层调用**

```tsx
function Demo() {
  const [count, setCount] = useState(0);  // ✅ 顶层

  if (count > 0) {
    // ❌ 错误：不能在 if 里调用 Hook
    const [flag] = useState(false);
  }

  return <div>{count}</div>;
}
```

2. **只在 React 函数组件或自定义 Hook 中调用**

```tsx
// ❌ 错误：在普通函数里调用 Hook
function getData() {
  const [data] = useState(null);
}
```

**三、自定义 Hook**

把可复用的状态逻辑抽成自定义 Hook。

```tsx
import { useState, useEffect } from 'react';

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}

function App() {
  const width = useWindowWidth();
  return <p>窗口宽度：{width}</p>;
}
```

**口诀：顶层调用，只在函数组件用。**

**React 实战场景**：封装 `useLocalStorage`、`useFetch`、`useDebounce` 等自定义 Hook，在多个组件间复用逻辑。

---

## 附录：React 避坑速查表

1. 组件名没有大驼峰 → 渲染报错
2. JSX 返回多个根元素 → 用 `<></>` 或 `<div>` 包裹
3. 直接修改 state → 用 `setState` 或展开运算符创建新值
4. 用数组索引当 key → 列表增删改顺序时 UI 异常
5. `useEffect` 依赖写漏 → 拿不到最新状态，产生闭包旧值
6. `&&` 短路接 `0` 或 `''` → 页面显示奇怪的数字或空白
7. 事件处理函数写成 `onClick={handleClick()}` → 会立即执行，应写 `onClick={handleClick}`
8. `useState` 初始值是函数调用 → 每次渲染都会执行，应用 `useState(() => expensive())`
9. Context 滥用导致无关组件重渲染 → 必要时拆分 Context 或用状态管理库
10. 在 `useEffect` 里 async 函数直接写 `async` → 不能返回 Promise，应内部再声明 async 函数

---

**到这里，12 章过完：前 6 章你会了 React 基础写法，后 6 章你知道了组件之间怎么协作、副作用怎么处理。下一站：开个最小项目（Todo List → 天气查询 → 后台管理），在实战里把知识焊死。**
