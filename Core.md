# 核心

## atom<div id="Atom"/>

该功能是创建一个原子配置。我们称之为“atom config”，因为它只是一个定义，它还没有一个值。如果上下文清楚，我们也可以称它为“原子”。
原子配置是一个不可变的对象。原子配置对象不保存值。
原子值存在于存储区中。要创建一个原始原子（配置），您只需要提供一个初始值。
```
import { atom } from 'jotai'

const priceAtom = atom(10)
const messageAtom = atom('hello')
const productAtom = atom({ id: 12, name: 'good stuff' })
```

您还可以创建派生原子。我们有三种模式：

- Read-only atom 只读原子 
- Write-only atom 只写原子 
- Read-Write atom 读写原子
   
为了创建派生原子，我们传递一个读取函数和一个可选的写入函数。

```
const readOnlyAtom = atom((get) => get(priceAtom) * 2)
const writeOnlyAtom = atom(
  null, // 通常第一个参数为`null`
  (get, set, update) => {
    // update 是我们收到的用于更新该原子的任何单个值
    set(priceAtom, get(priceAtom) - update.discount)
  }
)
const readWriteAtom = atom(
  (get) => get(priceAtom) * 2,
  (get, set, newPrice) => {
    set(priceAtom, newPrice / 2)
    // 可同时设置多个原子
  }
)

```

`get` 在读取函数中是读取原子值。它是反应性的，并跟踪读取依赖项。

`get` 在写入函数中也是读取原子值，但它不被跟踪。此外，它无法读取 Jotai v1 API 中未解析的异步值。

`set` 在写入函数中是写入原子值。它将调用目标原子的写入函数。

### 关于在渲染函数中创建原子的注意事项
可以在任何地方创建原子配置，但是参考平等很重要。它们也可以动态创建。要创建渲染函数的原子，需要useMemo或useref才能获得稳定的参考。如果对使用useMemo或useRef进行记忆有疑问，请使用useMemo。否则，它可能会引起无限循环.
```
const Component = ({ value }) => {
  const valueAtom = useMemo(() => atom({ value }), [value])
  // ...
}
```
## Signatures 

```
// primitive atom
function atom<Value>(initialValue: Value): PrimitiveAtom<Value>

// read-only atom
function atom<Value>(read: (get: Getter) => Value | Promise<Value>): Atom<Value>

// writable derived atom
function atom<Value, Update>(
  read: (get: Getter) => Value | Promise<Value>,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>
): WritableAtom<Value, Update>

// write-only derived atom
function atom<Value, Update>(
  read: Value,
  write: (get: Getter, set: Setter, update: Update) => void | Promise<void>
): WritableAtom<Value, Update>
```
- `initialValue`: 原子在其值更改之前将返回的初始值。
- `read`: 是每次重新渲染时都会调用的函数。`read`的签名是`(get) => Value | Promise<Value>`，`get`是一个接收原子配置并返回其存储在Provider中的值的函数，如下所述。依赖关系会被跟踪，因此如果`get`至少被用于一个原子一次，那么只要原子值发生变化，`read`就会被重新评估。
- `write`: 为了更好地描述，`useAtom()[1]`是一个主要用于修改原子值的函数，每当我们调用 `useAtom()[1]` 返回值的第二个值时，它就会被调用。该函数在原始原子中的默认值将改变该原子的值。`write` 的签名是 `(get, set, update) => void | Promise<void>`。`get` 与上面描述的签名类似，但它不跟踪依赖关系。`set`是一个函数，它接收一个原子配置和一个新值，然后更新Provider中的原子值。

```
const primitiveAtom = atom(initialValue)
const derivedAtomWithRead = atom(read)
const derivedAtomWithReadWrite = atom(read, write)
const derivedAtomWithWriteOnly = atom(null, write)
```

有两种原子：可写原子和只读原子。原始原子总是可写的。如果指定了 ，则派生原子是可写的。原始原子的等价于`React.useState`的`setState`.

## `debugLabel`属性 
创建的原子配置可以具有可选属性。调试标签用于在调试中显示原子。有关详细信息，请参阅[<u>调试指南</u>](https://jotai.org/docs/guides/debugging)。

>note
>虽然调试标签不必是唯一的，但通常建议使它们可区分。

## `onMount`属性

创建的原子配置可以具有可选属性`onMount`。 `onMount`是一个函数，它接收`setAtom`函数并返回`onUnmount`函数。

该函数`onMount`在提供程序中首次使用 `atom` 时调用，在不再使用时调用。在某些边缘情况下，可以卸载原子，然后立即安装。
```
const anAtom = atom(1)
anAtom.onMount = (setAtom) => {
  console.log('atom is mounted in provider')
  setAtom(c => c + 1) // increment count on mount
  return () => { ... } // return optional onUnmount function
}
```

调用函数`setAtom`将调用原子的`write`.自定义`write`允许更改行为。

```const countAtom = atom(1)
const derivedAtom = atom(
  (get) => get(countAtom),
  (get, set, action) => {
    if (action.type === 'init') {
      set(countAtom, 10)
    } else if (action.type === 'inc') {
      set(countAtom, (c) => c + 1)
    }
  }
)
derivedAtom.onMount = (setAtom) => {
  setAtom({ type: 'init' })
}
```
## 高级接口 <div id="Advanced"/>
自Jotai v2以来，`read`函数具有第二个参数。

### `options.signal`

它使用[<u>AbortController<u/>](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)，因此您可以中止异步函数。中止在启动新计算（调用`read`函数）之前触发。

如何使用它： 
```
const readOnlyDerivedAtom = atom(async (get, { signal }) => {
  // 使用信号中止函数
})

const writableDerivedAtom = atom(
  async (get, { signal }) => {
    // 使用信号中止函数
  },
  (get, set, arg) => {
    // ...
  }
)
```
`signal`值为[<u>AbortController<u/>](https://developer.mozilla.org/en-US/docs/Web/API/AbortController)。您可以检查布尔值，或使用`abort`事件`addEventListener`。

对于`fetch`用例，我们可以简单地通过`signal`.

有关用法，请参阅以下`fetch`示例。

```
import { Suspense } from 'react'
import { atom, useAtom } from 'jotai'

const userIdAtom = atom(1)
const userAtom = atom(async (get, { signal }) => {
  const userId = get(userIdAtom)
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/users/${userId}?_delay=2000`,
    { signal }
  )
  return response.json()
})

const Controls = () => {
  const [userId, setUserId] = useAtom(userIdAtom)
  return (
    <div>
      User Id: {userId}
      <button onClick={() => setUserId((c) => c - 1)}>Prev</button>
      <button onClick={() => setUserId((c) => c + 1)}>Next</button>
    </div>
  )
}

const UserName = () => {
  const [user] = useAtom(userAtom)
  return <div>User name: {user.name}</div>
}

const App = () => (
  <>
    <Controls />
    <Suspense fallback="Loading...">
      <UserName />
    </Suspense>
  </>
)

export default App
```
# useAtom

`useAtom`钩子是读取状态中的原子值。状态可以看作是原子配置和原子值的弱映射。

`useAtom`钩子以元组的形式返回原子值和更新函数，就像 React 的`useState`一样。它需要一个用 `atom()`创建原子配置。

初始状态下，原子没有相关值。只有通过 `useAtom` 使用原子后，初始值才会存储在状态中。如果原子是派生原子，则调用读取函数计算初始值。当原子不再被使用时，即所有使用原子的组件都已卸载，原子配置也不复存在，状态中的值将被垃圾回收。

```const [value, setValue] = useAtom(anAtom)```

`setValue`只接受一个参数，该参数将传递给原子写入函数的第三个参数。行为取决于写入函数的实现方式。

>注意：
>如 atom 部分所述，您必须注意处理 atom 的引用，否则可能会进入无限循环
```
const stableAtom = atom(0)
const Component = () => {
  const [atomValue] = useAtom(atom(0)) // 这将导致无限循环
  const [atomValue] = useAtom(stableAtom) // This is fine
  const [derivedAtomValue] = useAtom(
    useMemo(
      // This is also fine
      () => atom((get) => get(stableAtom) * 2),
      []
    )
  )
}
```
>请记住，React负责调用您的组件。这意味着它必须是幂等的，可以被多次调用。即使没有道具或原子发生变化，您也会经常看到额外的重新渲染。没有提交的额外重新渲染是一种预期行为。它实际上是React 18中useReducer的默认行为。

## 特性
```
// primitive or writable derived atom
function useAtom<Value, Update>(
  atom: WritableAtom<Value, Update>,
  options?: { store?: Store }
): [Value, SetAtom<Update>]

// read-only atom
function useAtom<Value>(
  atom: Atom<Value>,
  options?: { store?: Store }
): [Value, never]
```
useAtom 钩子用于读取存储在提供程序中的原子值。它以元组的形式返回原子值和更新函数，就像 useState 一样。它需要一个使用 创建的原子配置。最初，提供程序中不存储任何值。第一次通过 使用 `useAtom` 时，它将在提供程序中添加一个初始值。如果原子是派生原子，则执行读取函数以计算初始值。当不再使用原子时，这意味着卸载使用它的所有组件，并且原子配置不再存在，该值将从提供程序中删除。

```const [value, setValue] = useAtom(anAtom)```

`setValue`需要一个参数，该参数将被传递给原子的 writeFunction 的第三个参数。行为取决于 writeFunction 的实现方式。

### 原子依赖如何工作 

首先，让我们解释一下。在当前的实现中，每次调用“read”函数时，我们都会刷新依赖项和依赖项。例如，如果 A 依赖于 B，则表示 B 是 A 的依赖关系，A 是 B 的依赖关系。

```const uppercaseAtom = atom((get) => get(textAtom).toUpperCase())```

读取函数是原子的第一个参数。依赖项最初将为空。第一次使用时，我们运行读取函数并知道`uppercaseAtom`取决于`textAtom`,`textAtom`依赖于`uppercaseAtom`。因此，`uppercaseAtom`添加到`textAtom`的依赖项中。当我们重新运行 read 函数时（因为它的依赖项`textAtom`已更新），将再次创建依赖项，在这种情况下是相同的。然后，我们删除过时的依赖项并替换为最新的依赖项。

### 原子可以按需创建

虽然这里的基本示例显示了在组件之外全局定义原子，但对于我们可以在何时何地创建原子没有任何限制。只要我们记住原子是由它们的对象参照身份标识的，我们就可以随时创建它们。

如果在渲染函数中创建原子，则通常需要使用类似`useRef` 或`useMemo`用于记忆的钩子。否则，每次组件呈现时都会重新创建原子。

您可以创建一个原子并将其与另一个原子用`useState`一起存储，甚至可以存储在另一个原子中。请参阅[问题 #5 ](https://github.com/pmndrs/jotai/issues/5)中的示例。

您可以将原子缓存在全局的某个位置。请参阅此[示例](https://twitter.com/dai_shi/status/1317653548314718208)或[该示例](https://github.com/pmndrs/jotai/issues/119#issuecomment-706046321)。

## useAtomValue 

```const countAtom = atom(0)

const Counter = () => {
  const setCount = useSetAtom(countAtom)
  const count = useAtomValue(countAtom)

  return (
    <>
      <div>count: {count}</div>
      <button onClick={() => setCount(count + 1)}>+1</button>
    </>
  )
}
```
与`useSetAtom`钩子类似，`useAtomValue`允许您访问只读原子。

## useSetAtom 
```
const switchAtom = atom(false)

const SetTrueButton = () => {
  const setCount = useSetAtom(switchAtom)
  const setTrue = () => setCount(true)

  return (
    <div>
      <button onClick={setTrue}>Set True</button>
    </div>
  )
}

const SetFalseButton = () => {
  const setCount = useSetAtom(switchAtom)
  const setFalse = () => setCount(false)

  return (
    <div>
      <button onClick={setFalse}>Set False</button>
    </div>
  )
}

export default function App() {
  const state = useAtomValue(switchAtom)

  return (
    <div>
      State: <b>{state.toString()}</b>
      <SetTrueButton />
      <SetFalseButton />
    </div>
  )
}
```
如果您需要在不读取原子的情况下更新原子的值，您可以使用`useSetAtom()`
当需要考虑性能问题时，这一点尤其有用，因为`const [, setValue] = useAtom(valueAtom)`会在每次`valueAtom`更新时引起不必要的重新显示.

# Store


