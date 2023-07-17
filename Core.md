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

