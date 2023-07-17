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
