# Storage
## atomWithStorage
Ref: [https://github.com/pmndrs/jotai/pull/394](https://github.com/pmndrs/jotai/pull/394)

`atomWithStorage`函数为React创建一个原子`localStorage`或`sessionStorage`，为React Native创建一个原子`AsyncStorage`。

参数

**key（必填）**：唯一字符串，在与localStorage、sessionStorage或AsyncStorage同步状态时用作密钥。

**initialValue（必填）**：原子的初始值。

**storage（可选）**：具有以下方法的对象：
+ **getItem(key,initialValue)（必填）**：从存储空间读取一个项目，或者返回到初始值。
+ **setItem(key, value) （必填）**：将项目保存到存储空间
+ **removeItem(key) （必填）**：从存储空间中删除项目
+ **subscribe(key, callback, initialValue) （可选）**：订阅外部存储更新的方法。

如果未指定，默认存储实现使用`localStorage`进行存储/检索，使用`JSON.stringify()`/`JSON.parse()`进行序列化/反序列化，并订阅存储事件以实现跨标签同步。

### 服务器端渲染
任何依赖于存储原子（例如 `className` 或`样式`prop）值的 JSX 标记在服务器上呈现时都将使用初始值（因为服务器上不可用 `localStorage` 和 `sessionStorage`。
这意味着，如果用户的`存储值`与`初始值`不同，那么最初以HTML形式提供给用户浏览器的内容与React在补水过程中预期的内容将不匹配。
针对此问题的建议解决方法是，通过将其封装在自定义 [<ClientOnly>](https://www.joshwcomeau.com/react/the-perils-of-rehydration/#abstractions) 封装器中，仅在客户端呈现依赖于存储值的内容，该封装器仅在补水后呈现。其他解决方案在技术上是可行的，但需要在初始值与存储值交换时进行短暂的 "闪烁"，这会导致不愉快的用户体验，因此建议采用此解决方案。

### 从存储器中删除项目

如果您想从存储空间中删除一个项目，创建的 `atomWithStorage` 在写入时接受 `RESET` 符号。
具体用法请参见下面的示例：
```
import { useAtom } from 'jotai'
import { atomWithStorage, RESET } from 'jotai/utils'

const textAtom = atomWithStorage('text', 'hello')

const TextBox = () => {
  const [text, setText] = useAtom(textAtom)

  return (
    <>
      <input value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={() => setText(RESET)}>Reset (to 'hello')</button>
    </>
  )
}
```

如果需要，您也可以根据以前的值进行有条件的重置。
如果您希望在前值满足条件时清除`localStorage中`的键值，这将特别有用。
下面举例说明这种用法，只要前一个`值`为真，就清除`visible`
```
import { useAtom } from 'jotai'
import { atomWithStorage, RESET } from 'jotai/utils'

const isVisibleAtom = atomWithStorage('visible', false)

const TextBox = () => {
  const [isVisible, setIsVisible] = useAtom(isVisibleAtom)

  return (
    <>
      { isVisible && <h1>Header is visible!</h1> }
      <button onClick={() => setIsVisible((prev) => prev ? RESET : true))}>Toggle visible</button>
    </>
  )
}
```
### React-Native实现

您可以使用任何实现`getItem`、`setItem`和`removeItem`的库。比方说，您可以使用社区提供的标准`AsyncStorage`

```
import { atomWithStorage, createJSONStorage } from 'jotai/utils'
import AsyncStorage from '@react-native-async-storage/async-storage'

const storage = createJSONStorage(() => AsyncStorage)
const content = {} // anything JSON serializable
const storedAtom = atomWithStorage('stored-key', content, storage)
```
> 注意事项
> 如果使用任何类型的 `AsyncStorage`，atom 值可能会变成异步。因此，如果您要更新值，您可能需要解决promise

```
const countAtom = atomWithStorage('count-key', 0, anyAsyncStorage)
const Component = () => {
  const [count, setCount] = useAtom(countAtom)
  const increment = () => {
    setCount(async (promiseOrValue) => (await promiseOrValue) + 1)
  }
  // ...
}
```
## 验证存储值
要为您的存储原子添加运行时验证，您需要创建一个自定义的存储实现。
下面是一个利用Zod验证存储在`localStorage`中的值并进行跨标签同步的示例

```
import { atomWithStorage, createJSONStorage } from 'jotai/utils'
import { z } from 'zod'

const myNumberSchema = z.number().int().nonnegative()

const storedNumberAtom = atomWithStorage('my-number', 0, {
  getItem(key, initialValue) {
    const storedValue = localStorage.getItem(key)
    try {
      return myNumberSchema.parse(JSON.parse(storedValue ?? ''))
    } catch {
      return initialValue
    }
  },
  setItem(key, value) {
    localStorage.setItem(key, JSON.stringify(value))
  },
  removeItem(key) {
    localStorage.removeItem(key)
  },
  subscribe(key, callback, initialValue) {
    if (
      typeof window === 'undefined' ||
      typeof window.addEventListener === 'undefined'
    ) {
      return
    }
    window.addEventListener('storage', (e) => {
      if (e.storageArea === localStorage && e.key === key) {
        let newValue
        try {
          newValue = myNumberSchema.parse(JSON.parse(e.newValue ?? ''))
        } catch {
          newValue = initialValue
        }
        callback(newValue)
      }
    })
  },
})
```
