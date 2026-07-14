# 06 · 序列化机制（serialization/）

OpenMM 提供基于 XML 的序列化，用于把 `System`/`Force`/`Integrator`/`State` 等对象保存为可读 XML，或从中恢复。这是 OpenMM 的标准交换格式，Python 端 `XmlSerializer` 也基于此。

## 6.1 目录结构

```
serialization/
├── include/openmm/serialization/
│   ├── SerializationProxy.h       # 抽象代理基类
│   ├── SerializationNode.h        # 通用属性树节点
│   ├── XmlSerializer.h            # 对外模板门面
│   └── *Proxy.h                   # 每个可序列化类型一个代理头
└── src/
    ├── SerializationProxy.cpp     # 全局注册表（type_info→proxy, typeName→proxy）
    ├── SerializationNode.cpp
    ├── XmlSerializer.cpp          # XML 读写（用 irrXML 解析）
    ├── SerializationProxyRegistration.cpp  # ~50 个 registerProxy 调用，DLL 构造器自注册
    ├── SystemProxy.cpp、NonbondedForceProxy.cpp、...（每个类型一个）
    ├── TabulatedFunctionProxies.cpp
    ├── dtoa.cpp、g_fmt.cpp        # 快速 double 格式化
```

## 6.2 核心抽象

### 6.2.1 SerializationProxy（代理模式）

**头**：`serialization/include/openmm/serialization/SerializationProxy.h`

每个可序列化类型配一个代理，构造时传入 `typeName`，实现两个纯虚：
```cpp
class SerializationProxy {
    SerializationProxy(string typeName);
    virtual void serialize(const void* object, SerializationNode& node) const = 0;
    virtual void* deserialize(const SerializationNode& node) const = 0;
    static void registerProxy(const type_info& type, SerializationProxy* proxy);
    static const SerializationProxy& getProxy(const type_info& type);
    static const SerializationProxy& getProxyByName(const string& name);
};
```

### 6.2.2 SerializationNode（属性树）

**头**：`serialization/include/openmm/serialization/SerializationNode.h`

通用属性包/树节点，支持类型化属性（int/double/string/bool）+ 子节点。是 C++ 对象与 XML 之间的中间表示。

多态反序列化的关键：`SerializationNode::decodeObject<Force>()` 读子节点的 `type` 属性，查代理注册表，调对应 `deserialize`。

### 6.2.3 全局注册表

`SerializationProxy.cpp` 维护两张 map：
- `type_info` → proxy（序列化时按对象动态类型查）
- `typeName`(string) → proxy（反序列化时按 XML `type` 属性查）

### 6.2.4 自注册

`SerializationProxyRegistration.cpp` 定义 `registerSerializationProxies()`（约 50 个 `registerProxy` 调用），作为共享库构造器自动执行：
- Linux：`__attribute__((constructor))`
- Windows：`DllMain(DLL_PROCESS_ATTACH)`

所以只要进程链接了 OpenMM 序列化库，核心类型的代理就自动可用。插件（amoeba/drude）加载时再追加自己的代理。

## 6.3 XmlSerializer —— 对外门面

**头**：`serialization/include/openmm/serialization/XmlSerializer.h`
**实现**：`serialization/src/XmlSerializer.cpp`

模板静态方法：
```cpp
template<class T>
static void serialize(T& object, string rootName, ostream&);

template<class T>
static T* deserialize(istream&);

template<class T>
static T* clone(T& object);   // 内存序列化→反序列化，跳过 XML，更快
```

`serialize` 流程：
1. `getProxy(typeid(*object))` 取代理
2. `proxy.serialize(obj, node)` 填属性树
3. `node["type"] = proxy.getTypeName()` 盖类型戳
4. 写 XML（用 `libraries/irrxml/`）

`deserialize` 流程：
1. 用 irrXML 解析 XML → `SerializationNode`
2. 读 `type` 属性 → `getProxyByName`
3. `proxy.deserialize(node)` → `T*`

## 6.4 System 的序列化

`serialization/src/SystemProxy.cpp`：
- 记录版本、OpenMM 版本、默认周期盒向量
- 粒子（质量 + 可选 `VirtualSite` 子节点：`TwoParticleAverageSite` 等）
- 约束
- `Forces` 子节点：每个 `Force` 一个节点，多态序列化（由各 Force 自己的代理处理）
- 反序列化时 `force.decodeObject<Force>()` 按类型戳分发到对应代理

`Integrator` 不属于 `System`，单独由 `*IntegratorProxy.cpp` 序列化（记录步长、参数、CustomIntegrator 计算块等）。

## 6.5 关键文件

- `serialization/include/openmm/serialization/SerializationProxy.h`
- `serialization/include/openmm/serialization/SerializationNode.h`
- `serialization/include/openmm/serialization/XmlSerializer.h`
- `serialization/src/SerializationProxyRegistration.cpp`（自注册）
- `serialization/src/SystemProxy.cpp`、`serialization/src/XmlSerializer.cpp`

## 6.6 NPU 适配启示

序列化层通常**无需为 NU 适配做改动**——它只持久化 API 层对象（System/Force/Integrator/State），与平台无关。仅当你新增了自定义 Force/Integrator（如 NPU 专属力场）时，才需写对应 `*Proxy` 并在 `SerializationProxyRegistration.cpp` 注册。可参照 amoeba 的 `serialization/` 子目录。

下一篇 [07-libraries.md](07-libraries.md) 盘点第三方库。
