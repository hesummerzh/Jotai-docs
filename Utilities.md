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
