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
