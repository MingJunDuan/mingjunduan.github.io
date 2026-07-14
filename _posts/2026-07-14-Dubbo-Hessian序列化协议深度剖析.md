---
layout: post
title: "Dubbo RPC 中 Hessian 序列化协议深度剖析 —— 从原理到线上问题"
author: "Inela"
---

在 Dubbo RPC 调用中，对象是怎么从 Consumer 的内存变成 Provider 的内存里的？中间那一段二进制字节流到底长什么样？当你面对 "Cannot deserialize"、"enum 反序列化后变成 null"、"泛型集合类型丢失" 这些线上问题时，答案都藏在序列化层。

本文从 Dubbo 源码层面，拆解 Hessian 2.0 序列化协议的全链路，覆盖对象/集合/枚举/异常/引用的编解码细节，并汇集生产环境中最常见的 8 个典型问题及排查方案。

---

## 一、为什么 Dubbo 选择 Hessian

### 1.1 Dubbo 序列化层的定位

在 Dubbo 的架构中，序列化层处于 **Codec 层** 的核心位置：

```
┌─────────────────────────────────────────────────────────┐
│                     Dubbo RPC 调用链路                      │
│                                                         │
│  Consumer                              Provider         │
│  ┌─────────┐    serialize     ┌─────────┐              │
│  │ Request │ ───────────────► │  bytes  │              │
│  │ Object  │                  │ (网络传输)│              │
│  └─────────┘                  └────┬────┘              │
│                                    │ deserialize       │
│                                    ▼                    │
│                              ┌─────────┐              │
│                              │ Request │              │
│                              │ Object  │              │
│                              └─────────┘              │
│                                                         │
│  Serialization 接口 (SPI, name="serialization")          │
│  ┌──────────────┬──────────────┬────────────────────┐  │
│  │  hessian2    │   fastjson2  │     protobuf       │  │
│  │  (默认)       │              │                    │  │
│  └──────────────┴──────────────┴────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Dubbo 通过 SPI 机制加载序列化实现：

```java
// Dubbo 源码: org.apache.dubbo.common.serialize.Serialization
@SPI("hessian2")  // 默认扩展名为 hessian2
public interface Serialization {
    byte getContentTypeId();
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;
    ObjectInput deserialize(URL url, InputStream input) throws IOException;
}
```

`@SPI("hessian2")` 意味着 —— 如果没人显式配置序列化协议，Dubbo 默认就用 Hessian 2。

### 1.2 Hessian 的设计哲学

Hessian 是 Caucho 公司（Resin 应用服务器的开发商）设计的一种二进制 Web 服务协议。它的核心设计目标有三个：

| 设计目标 | 含义 | 实现方式 |
|---------|------|---------|
| **紧凑二进制** | 比 XML/SOAP 小得多 | tag-byte + 变长整数编码 |
| **自描述** | 二进制流包含类型信息，不需要预先编译 IDL | 类名、字段名写入流中 |
| **跨语言** | 理论上有 Java/PHP/Python/C++ 等多语言实现 | 定义了语言无关的二进制格式规范 |

与常见序列化方案的横向对比：

```
┌──────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│              │ Hessian2 │ Fastjson │ Kryo     │ Protobuf │ Java原生  │
├──────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ 格式         │ 二进制    │ 文本      │ 二进制    │ 二进制    │ 二进制    │
│ 自描述       │ ✅       │ ✅       │ ❌        │ ❌        │ ✅       │
│ 跨语言       │ ✅       │ ❌        │ ❌        │ ✅       │ ❌        │
│ 无需IDL      │ ✅       │ ✅       │ ✅       │ ❌        │ ✅       │
│ 序列化体积   │ 小       │ 大       │ 很小      │ 最小      │ 大       │
│ 序列化速度   │ 中等     │ 中等     │ 快        │ 很快      │ 慢       │
│ 字段增删兼容  │ 较好     │ 较好     │ 差        │ 好        │ 差       │
└──────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

Hessian 在 **自描述 + 紧凑 + 兼容性** 三者之间取了一个折中，这是 Dubbo 选择它作为默认方案的根本原因：RPC 场景下，你不可能要求所有微服务都预先协商好一份 `.proto` 文件。

### 1.3 Hessian 1.x vs Hessian 2.0

Dubbo 使用的是 **Hessian 2.0**（Dubbo 2.x 早期曾用过 Hessian 1.x，但早已废弃）。两者的关键差异：

| 特性 | Hessian 1.x | Hessian 2.0 |
|------|------------|-------------|
| 紧凑性 | 一般 | 更好（引入压缩整数、引用机制） |
| 对象编码 | 类似 XML 结构的二进制映射 | 类定义 + 对象实例分离（`C` + `O`） |
| 引用机制 | 不支持 | 支持值引用和类引用 |
| 字符串编码 | 定长 | 分块（chunked） |
| Map/Object 区分 | 严格区分 | `M` tag 具有双重语义 |

**核心升级**：Hessian 2.0 引入了类似数据库中"表结构 + 数据行"的分离思想——先发类定义（字段名列表），再发对象实例（字段值），避免了每个对象都重复发送字段名。这是它比 1.x 紧凑得多的根本原因。

---

## 二、Hessian 协议基础：二进制格式全景

### 2.1 Tag-Byte 驱动的编码模型

Hessian 的二进制格式是**地图式编码**：先写 tag（标记这是什么类型），再写数据。反序列化时按 tag 分派到对应的解析器。

核心 tag 一览：

```
┌──────┬──────┬────────────────────────────────┐
│ Tag  │ 值   │ 含义                           │
├──────┼──────┼────────────────────────────────┤
│ N    │ 0x4e │ null                           │
│ T    │ 0x54 │ true                           │
│ F    │ 0x46 │ false                          │
│ I    │ 0x49 │ int (定长4字节)                 │
│ L    │ 0x4c │ long (定长8字节)                │
│ D    │ 0x44 │ double (定长8字节)              │
│ S    │ 0x53 │ String (定长或分块)             │
│ B    │ 0x42 │ binary 字节数组                │
│ M    │ 0x4d │ Map (type 可能为空)            │
│ H    │ 0x48 │ Map 的 key-value 对结束标记     │
│ C    │ 0x43 │ 类定义（字段名列表）            │
│ O    │ 0x4f │ 对象实例（按字段顺序放值）       │
│ R    │ 0x52 │ 引用（ref，指向已出现的对象）     │
│ 0x55-5f │    │ 紧凑 int/long（单字节表示）     │
│ 0x80-0xbf │  │ 紧凑 long（2字节表示）         │
│ 0xc0-0xcf │  │ 紧凑 int（3字节表示）          │
│ 0xd0-0xd7 │  │ 紧凑 int（4字节表示）          │
└──────┴──────┴────────────────────────────────┘
```

整数的紧凑编码是 Hessian 压缩体积的关键手段之一。它根据数值大小使用 1~5 个字节 — 这比 Java 原生的 `writeInt()` 固定 4 字节要高效得多。

### 2.2 一个直观的对比：Hessian vs JSON

以 `Person { name: "张三", age: 25 }` 为例，看看两种编码的本质区别：

```
JSON (文本, 31 bytes):
  {"name":"张三","age":25}

Hessian 2.0 (二进制, ~20 bytes):
  C                       ← 类定义开始
    \x06 com.Person        ← 类名
    \x92                   ← 字段数量 = 2
    \x04 name              ← 字段1名
    \x03 age               ← 字段2名
  O                       ← 对象实例开始
    \x91                   ← 类定义引用 #1
    \x06 张三              ← name 字段值
    \x5a                   ← age 字段值 (紧凑int = 25)
```

Hessian 的"类定义 + 对象实例"分离设计意味着：同一个类的 N 个实例，类定义只发一次，后面都是引用。

---

## 三、序列化过程：Java 对象 → 二进制流

### 3.1 Hessian2Output 整体架构

```
┌────────────────────────────────────────────────────────────┐
│                 Hessian2Output 架构                         │
│                                                            │
│  调用入口 writeObject(obj)                                  │
│       │                                                    │
│       ▼                                                    │
│  ┌─────────────────┐                                       │
│  │ SerializerFactory │ ──► 查找 Serializer                  │
│  └────────┬────────┘                                       │
│           │ 根据类型分发                                     │
│           ▼                                                │
│  ┌──────────────────────────────────────────────────┐     │
│  │ UnsafeSerializer    (普通 POJO)                   │     │
│  │ BeanSerializer      (有 getter 的 JavaBean)       │     │
│  │ StringSerializer    (String)                      │     │
│  │ MapSerializer       (Map/EnumMap/Properties)      │     │
│  │ CollectionSerializer(List/Set/Array)              │     │
│  │ EnumSerializer      (enum)                       │     │
│  │ ThrowableSerializer (Exception)                  │     │
│  │ BigDecimalSerializer, DateSerializer, ...        │     │
│  └──────────────────────────────────────────────────┘     │
│           │                                                │
│           ▼                                                │
│  ┌──────────────────────────────────────────────┐         │
│  │ _refs: IdentityHashMap  (引用追踪)            │         │
│  │ _classRefs: HashMap     (类定义引用追踪)       │         │
│  │ _os: OutputStream        (底层输出)            │         │
│  └──────────────────────────────────────────────┘         │
└────────────────────────────────────────────────────────────┘
```

`SerializerFactory` 是核心调度器，它根据 Java 类型选择合适的 `Serializer`。工厂内部维护了一份类型到序列化器的映射表，首字母大写保证了线程查找的安全性。

### 3.2 基本类型序列化

#### int / long —— 紧凑编码

```java
// Hessian2Output 源码简化
public void writeInt(int value) throws IOException {
    if (INT_DIRECT_MIN <= value && value <= INT_DIRECT_MAX) {
        // -16 ~ 47: 单字节编码 [0x80, 0xbf]
        _os.write(value + 0x90);
    } else if (INT_SHORT_MIN <= value && value <= INT_SHORT_MAX) {
        // -0x800 ~ 0x7ff: 2字节编码 [0xc0, 0xcf] + byte
        _os.write(0xc8 + (value >> 8));
        _os.write(value);
    } else if (INT_3BYTE_MIN <= value && value <= INT_3BYTE_MAX) {
        // -0x40000 ~ 0x3ffff: 3字节编码 [0xd0, 0xd7] + 2 bytes
        _os.write(0xd4 + (value >> 16));
        _os.write(value >> 8);
        _os.write(value);
    } else {
        // 其他: 5字节 'I' + 4 bytes (大端)
        _os.write('I');
        _os.write(value >> 24);
        _os.write(value >> 16);
        _os.write(value >> 8);
        _os.write(value);
    }
}
```

关键示例：

```
值 = 0      →  0x90  (1字节)       // 比 Java 节省 3 字节
值 = 25     →  0xa9  (1字节)       // 0x90 + 25
值 = 100    →  0xf4  (1字节)
值 = 0x800  →  [0xd0, 0x08, 0x00]  (3字节)
值 = Integer.MAX_VALUE → [0x49, 0x7f, 0xff, 0xff, 0xff] (5字节)
```

Dubbo RPC 中的方法参数大多是 0~100 范围内的 int 值，紧凑编码能节省大量空间。

#### String —— Chunked 分块编码

Hessian 2.0 对字符串采用**分块传输**，每块最大 65535 字节（16 位长度前缀）：

```java
// Hessian2Output 源码简化
public void writeString(String value, int offset, int length) {
    // 小字符串：直接写
    if (length <= SIZE) {
        _os.write('S');          // tag: String 定长模式
        _os.write(length >> 8);  // 长度高字节
        _os.write(length);       // 长度低字节
    } else {
        // 大字符串：分块写
        while (length > 0) {
            int sublen = Math.min(length, SIZE);
            _os.write('s');      // tag: String 分块模式  (小写 s!)
            _os.write(sublen >> 8);
            _os.write(sublen);
            _os.write(value, offset, sublen);
            length -= sublen;
            offset += sublen;
        }
        _os.write('S');          // 结束标记
    }
}
```

这里有一个容易忽略的细节：**大写 `S` 和小写 `s` 是不同的 tag**。`S`（0x53）是完整的定长字符串或结束标记，`s`（0x73）是分块片段。反序列化时如果混了这两个 tag 的处理逻辑就会出 bug。

#### null / boolean / Date / BigDecimal

```java
// null
writeNull() → _os.write('N');       // 永远 1 字节

// boolean
writeBoolean(true)  → _os.write('T');  // 1 字节
writeBoolean(false) → _os.write('F');  // 1 字节

// java.util.Date —— 序列化为 UTC 毫秒时间戳
writeObject(new Date()) →
  'M'           // Map tag（Hessian 将 Date 当 Map 序列化）
  '\x04' + type // "java.util.Date"
  ':'           // 毫秒值（紧凑 long）
  结束 'Z'

// BigDecimal —— 序列化为字符串表示
writeObject(new BigDecimal("123.45")) →
  'S'           // String tag
  '\x00\x06'    // 长度 6
  "123.45"      // 字符串内容
```

`BigDecimal` 序列化为 String 这一点在 Hessian 社区中引发过争议——如果 BigDecimal 来自一个高精度计算结果（比如 0.1 + 0.2 的内部表示），写成 String 时可能丢失精度。Dubbo 通过特殊处理规避了这个问题。

### 3.3 对象序列化：类定义 + 实例分离

这是 Hessian 2.0 最核心的设计。以这个类为例：

```java
public class User {
    private String name;
    private int age;
    // getter/setter 省略
}

User user = new User("张三", 25);
```

序列化输出（十六进制 + 注释）：

```
C                         ← 类定义开始 (Class Definition)
  0x0b                    ← 类名长度 11
  "com.example.User"       ← 全限定类名
  0x92                    ← 字段数量 = 2
  0x04 "name"             ← 字段1: 长度4, 名称"name"
  0x03 "age"              ← 字段2: 长度3, 名称"age"

O                         ← 对象实例开始 (Object Instance)
  0x91                    ← 类定义引用编号 #1 (紧凑int = 1)
  0x06 "张三"             ← 字段1的值 (String, 长度6)
  0xa9                    ← 字段2的值 (紧凑int = 25)
```

**类定义引用**（`0x91`）是关键优化：指向之前发出的第 1 个类定义。如果流中再次出现 `com.example.User` 实例，就不需要再发 `C ...` 了，直接 `O 0x91 ...`。

Serializer 的核心代码（简化）：

```java
// UnsafeSerializer.writeObject()
public void writeObject(Object obj, AbstractHessianOutput out) {
    // 1. 写入类定义 (C tag)，如果已经写过了就跳过
    int ref = out.writeObjectBegin(cl.getName());
    if (ref < -1) {
        // 首次：写入类定义
        for (Field field : fields) {
            out.writeString(field.getName());
        }
        out.writeMapEnd();  // H tag
        ref = out.writeObjectBegin(cl.getName());
        // 写字段值
        for (Field field : fields) {
            out.writeObject(field.get(obj));
        }
    } else if (ref == -1) {
        // 已写过类定义，直接写实例数据
        for (Field field : fields) {
            out.writeObject(field.get(obj));
        }
    }
    // ref >= 0: 已写过完全相同的对象实例，写引用
}
```

这里 `writeObjectBegin` 的返回值含义：

| 返回值 | 含义 |
|--------|------|
| `< -1` | 类定义未出现，需要先发 `C` 再发 `O` |
| `-1` | 类定义已出现，直接发 `O`（无需再发字段名） |
| `>= 0` | 同一对象已出现过，直接发 `R` 引用 |

### 3.4 引用追踪机制

Hessian 通过 `IdentityHashMap` 追踪已序列化的对象：

```java
// Hessian2Output 源码
private IdentityHashMap<Object, Integer> _refs = new IdentityHashMap<>();

public boolean addRef(Object object) {
    if (_refs == null) _refs = new IdentityHashMap<>();
    Integer newRef = _refs.size();
    Integer ref = _refs.put(object, newRef);
    if (ref != null) {
        // 这个对象之前已经被序列化过
        writeRef(ref);  // 写 R tag + 引用编号
        return true;    // 上层不再序列化字段值
    }
    return false;
}
```

**示例：循环引用**

```java
class Node {
    String name;
    Node next;
}

Node a = new Node("A");
Node b = new Node("B");
a.next = b;
b.next = a;  // 环形引用！
```

序列化输出：
```
C "Node" 0x02  "name" "next"     ← 类定义
O 0x91                            ← a 实例
  S "A"                            ← a.name
  O 0x91                           ← b 实例 (嵌套在 a.next)
    S "B"                           ← b.name
    R 0x90                          ★ 引用回 a! (指向流中第 0 个实例)
```

如果没有引用机制，环形引用会死循环。Hessian 的 `R` tag 完美解决了这个问题，反序列化时能正确还原环。

### 3.5 集合序列化

#### List

```java
List<String> list = Arrays.asList("A", "B");
```

序列化输出：
```
V                           ← 类型定义开始（V = 0x56）
  M                         ← Map tag，因为 Hessian 用 Map 表示 type
    S "type"                ← key: "type"
    S "[string"             ← value: 类型描述
    Z                       ← end of map
  M "java.util.ArrayList"  ← Map 表示 java.util.ArrayList 类型
    Z

  S "A"                      ← 元素0
  S "B"                      ← 元素1
Z                          ← end of list
```

**关键发现**：Hessian 用 `V`（Variable List，未定型列表）tag 开始列表，用 `Z` 结束。中间的类型信息也用 Map 来传。

#### Map

```java
Map<String, Integer> map = new HashMap<>();
map.put("key1", 1);
```

序列化输出：
```
H                           ← Map 开始（H = 0x48，等同于 M）
  S "key1"                   ← key
  0x91                        ← value (紧凑int = 1)
Z                           ← Map 结束
```

### 3.6 枚举序列化

```java
enum Color { RED, GREEN, BLUE }
Color c = Color.GREEN;
```

序列化输出：
```
M                           ← Map tag
  S "name"                   ← key
  S "GREEN"                  ← value (枚举常量名)
  Z
```

**Hessian 按 `name()` 序列化枚举，不是 `ordinal()`**。这意味着：
- JDK 的 `Enum.name()` 返回源码中的常量名
- 如果常量改名（如 `RED` → `RED_COLOR`），反序列化端按新 name 找不到旧值，返回 **null**

这是生产环境中枚举反序列化的头号陷阱。

### 3.7 异常序列化

Hessian 对 `Throwable` 有特殊处理：

```java
// ThrowableSerializer
public void writeObject(Object obj, AbstractHessianOutput out) {
    Throwable e = (Throwable) obj;
    // 把异常像普通对象一样序列化其字段（message, cause, stackTrace...）
    // 序列化器强行把异常的字段提取出来，逐个写
}
```

关键：**Hessian 不保留异常的完整栈信息**。它序列化 `detailMessage`、`cause`、`stackTrace` 这三个字段，但 `stackTrace` 经过跨语言处理后可能丢失部分精度。

---

## 四、反序列化过程：二进制流 → Java 对象

### 4.1 Hessian2Input 整体架构

反序列化是序列化的逆过程，核心是一个**按 tag 分派的状态机**：

```
┌─────────────────────────────────────────────────────────────┐
│                   Hessian2Input 反序列化流程                   │
│                                                             │
│  readObject()                                               │
│      │                                                      │
│      ▼                                                      │
│  读取 1 byte → tag                                           │
│      │                                                      │
│      ├─ 'N' → 返回 null                                     │
│      ├─ 'T'/'F' → 返回 Boolean                              │
│      ├─ 'I' → readInt()                                     │
│      ├─ 'L' → readLong()                                    │
│      ├─ 'D' → readDouble()                                  │
│      ├─ 'S'/'s' → readString()                              │
│      ├─ 'C' → readObjectDefinition() → 下一步继续读          │
│      ├─ 'O' → readObjectInstance()                          │
│      ├─ 'R' → readRef() → 返回已缓存的引用对象               │
│      ├─ 'M'/'H' → readMap()                                 │
│      ├─ 'V' → readList()                                    │
│      ├─ 0x80-0xbf → 紧凑 int/long                           │
│      ├─ 0xc0-0xcf → 紧凑 int (2-3字节)                       │
│      ├─ 0xd0-0xd7 → 紧凑 int (2-4字节)                       │
│      └─ ...                                                 │
│                                                             │
│  辅助结构:                                                    │
│  _refs: ArrayList<Object>     (值引用列表)                    │
│  _classDefs: ArrayList<ClassDef> (类定义列表)                 │
│  _types: ArrayList<String>     (类型名列表)                   │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 基本类型反序列化

紧凑整数的反序列化需要精确理解编码规则：

```java
// Hessian2Input 源码简化
public int readInt() throws IOException {
    int tag = read();
    switch (tag) {
        case 'I':  // 0x49: 定长 int
            return (read() << 24) + (read() << 16) + (read() << 8) + read();

        case 'L':  // 0x4c: 实际上是 long，但适合 int 时直接转
            return (int) ((long) read() << 56 + ...)

        case 0xd8: case 0xd9: case 0xda: case 0xdb:
        case 0xdc: case 0xdd: case 0xde: case 0xdf:
            // [0xd8, 0xdf]: 3字节 int
            is = read();  // 第二个字节
            return ((tag - 0xd4) << 16) + (is << 8) + read();

        case 0xc0: ... case 0xcf:
            // [0xc0, 0xcf]: 2字节 int
            return ((tag - 0xc8) << 8) + read();

        case 0x80: ... case 0xbf:
            // [0x80, 0xbf]: 单字节 int
            return tag - 0x90;

        default:
            throw error("expected integer at " + ...);
    }
}
```

### 4.3 对象反序列化：类定义解析

```java
// readObjectDefinition() 核心逻辑
private void readObjectDefinition() {
    String type = readString();        // 类名，如 "com.example.User"
    int len = readInt();              // 字段数量

    String[] fieldNames = new String[len];
    for (int i = 0; i < len; i++) {
        fieldNames[i] = readString();  // 逐个读字段名
    }

    ClassDef def = new ClassDef(type, fieldNames);
    _classDefs.add(def);              // 缓存类定义
}
```

`ClassDef` 存储了类名和字段名列表，后续每个 `O` tag 的对象实例都引用它：

```java
// readObjectInstance() 核心逻辑
private Object readObjectInstance(ClassDef def) throws IOException {
    String type = def.getType();        // "com.example.User"
    Class<?> cl = loadClass(type);     // 通过 Class.forName 加载

    // 实例化策略选择
    Object obj;
    if (UnsafeAllocator.isAvailable()) {
        obj = UnsafeAllocator.create().newInstance(cl);  // 绕过构造器
    } else {
        obj = cl.newInstance();  // 调用无参构造器
    }

    _refs.add(obj);  // 加入引用表，支持后续 R tag 引用

    // 按字段顺序反序列化
    Serializer ser = findSerializer(cl);
    for (int i = 0; i < def.getFields().length; i++) {
        String fieldName = def.getFields()[i];
        Object value = readObject();   // 递归读字段值
        ser.setField(obj, fieldName, value);
    }

    return obj;
}
```

### 4.4 实例化策略：UnsafeAllocator vs 构造器

这是一个非常值得关注的细节。Hessian 反序列化时有两个实例化路径：

```java
// 策略选择逻辑
if (UnsafeAllocator.isAvailable()) {
    // 路径1: 使用 sun.misc.Unsafe 直接分配内存
    // 优点: 不依赖无参构造器，类没有无参构造器也能反序列化
    // 缺点: 可能绕过构造器中的初始化逻辑
    obj = UnsafeAllocator.create().newInstance(cl);
} else {
    // 路径2: 通过反射调用无参构造器
    // 要求: 类必须有 public/protected 的无参构造器
    obj = cl.getDeclaredConstructor().newInstance();
}
```

**UnsafeAllocator 的隐患**：如果你的类在构造器中做了关键初始化（比如给一个字段赋默认值），反序列化后这个初始化可能没有执行。举个例子：

```java
public class Config {
    private Map<String, String> props;

    public Config() {
        this.props = new ConcurrentHashMap<>();  // 构造器中初始化
        this.props.put("timeout", "3000");       // 设置默认值
    }
}

// 如果 Hessian 通过 Unsafe 实例化 Config，
// 构造器不会执行！props 为 null！
// 后续 setField("props", someMap) 没问题，
// 但如果流中没有 props 字段的数据（版本不兼容），props 就一直是 null
```

### 4.5 引用解析

```java
// readRef() 核心逻辑
private Object readRef() {
    int ref = readInt();          // 引用编号
    return _refs.get(ref);        // 从引用表中取出之前反序列化的对象
}
```

环形引用的还原：反序列化端首先创建 a 对象放入 `_refs[0]`，然后在反序列化 b 时发现 `R 0x90`（引用编号 0），直接从 `_refs[0]` 取回 a 对象赋给 `b.next`。最后 `a.next = b`，环就还原了。

### 4.6 集合反序列化

```java
// readList() 核心逻辑
private Object readList() {
    int tag = read();
    if (tag == 'M') readType();     // 读类型描述
    if (tag == 't') readString();   // "type"
    if (tag == 'V') readString();   // value
    if (tag == 'z') readMapEnd();

    // 确定目标集合类
    String type = ...;  // 从流中解析出的类型名
    Collection list;
    if (type.equals("java.util.ArrayList")) {
        list = new ArrayList();
    } else {
        list = (Collection) Class.forName(type).newInstance();
    }

    _refs.add(list);

    // 逐个读元素
    while ((tag = read()) != 'Z') {
        list.add(readObject());
    }
    return list;
}
```

### 4.7 泛型擦除带来的挑战

这是 Dubbo + Hessian 组合中最大的坑之一。Hessian 协议 **不携带 Java 泛型信息**：

```java
// Provider 端
public List<User> getUsers() {
    return Arrays.asList(new User("张三", 25));
}

// Hessian 序列化: V [M [S "type" S "[com.example.User" Z] ... Z]
// 类型信息 "[com.example.User" 在流中

// 但 Consumer 端收到的 Response 中，方法返回值类型是
// java.lang.reflect.Type → ParameterizedType → List<User>
// 而 Hessian 反序列化出来的实际对象是
// java.util.ArrayList<Map> !!!
```

**为什么变成 `ArrayList<Map>` 了？** 因为 Hessian 先把对象反序列化到中间格式（list 元素是 Map），然后 Dubbo 根据方法签名的泛型信息再做一次类型转换：

```java
// Dubbo 源码: DecodeableRpcResult
// 反序列化后，对返回值做类型兼容转换
Object value = Hessian2ObjectInput.readObject();
// value 实际类型可能是 ArrayList<HashMap>
// 方法签名期望 List<User>

// Dubbo 的修复: 使用 TypeUtils 做强制类型转换
value = TypeUtils.cast(value, returnType);
// 将 HashMap 转为 User 对象（反射 setter）
```

---

## 五、Hessian 的高级特性

### 5.1 引用（Ref）机制深入

Hessian 有三种引用：

| 引用类型 | Tag | 含义 | 示例场景 |
|---------|-----|------|---------|
| **值引用** | `R` | 引用之前出现过的 Java 对象实例 | 环形引用 `a.next = b; b.next = a` |
| **类定义引用** | 紧凑 int | 引用之前发送过的类定义 | 同一类的第 2 个实例不需要再发 `C` |
| **类型引用** | 紧凑 int | 引用之前出现过的类型名 | `List<User>` 中 `User` 出现多次 |

三种引用的存储结构是分离的：

```java
// Hessian2Output 中
_refs       // IdentityHashMap<Object, Integer> — 值引用
_classRefs  // HashMap<String, Integer> — 类定义引用
_typeRefs   // HashMap<String, Integer> — 类型名引用
```

### 5.2 类定义缓存与线程安全

`SerializerFactory` 内部维护了类定义的缓存，但这里的线程安全策略值得注意：

```java
// SerializerFactory 的关键缓存
private ConcurrentHashMap<String, Serializer> _cachedSerializerMap
    = new ConcurrentHashMap<>();

private ConcurrentHashMap<String, Deserializer> _cachedDeserializerMap
    = new ConcurrentHashMap<>();

private WeakHashMap<ClassLoader, HashMap<String, ClassDef>> _classDefs;
```

`_classDefs` 使用 `WeakHashMap` —— key 是 `ClassLoader`，当 ClassLoader 被 GC 回收时，对应的类定义缓存自动清理。这对 Dubbo 这种有自定义 ClassLoader 的场景非常重要。

### 5.3 压缩整数编码深度解析

Hessian 的整数编码是一种变长编码（Variable Length Encoding），类似于 Protocol Buffers 的 Varint，但编码表不同：

```
数值范围                             编码形式
─────────────────────────────────────────────────────
[-16, 47]                           [0x80, 0xbf] 即 0x90 + value
[-2048, 2047]                       [0xc0, 0xcf] + 1 byte
[-262144, 262143]                   [0xd0, 0xd7] + 2 bytes
[-536870912, 536870911]             0x49('I') + 4 bytes
其他 long                           0x4c('L') + 8 bytes
```

具体编码公式：

```java
// 编码规则实现
if (-16 <= v && v <= 47) {
    write(0x90 + v);           // 单字节
} else if (-0x800 <= v && v <= 0x7ff) {
    write(0xc8 + (v >> 8));    // tag + 高字节
    write(v);                   // 低字节
} else if (-0x40000 <= v && v <= 0x3ffff) {
    write(0xd4 + (v >> 16));   // tag + 高字节
    write(v >> 8);              // 中字节
    write(v);                   // 低字节
} else {
    write('I');                 // 定长 int tag
    write(v >> 24);
    write(v >> 16);
    write(v >> 8);
    write(v);
}
```

**为什么 `0x90` 是基准值？** 因为 Hessian 的 tag 空间需要避免与已有 tag 冲突。0x80 是所有紧凑 int 的起始区间，0x90 恰好让编码后的最小结果 `0x80`（-16），最大单字节结果 `0xbf`（47），完美位于区间内。

### 5.4 Chunked String

大字符串（超过 32768 字节）被分成多个 chunk：

```
s 0x80 0x00 {前 32768 字节数据}   ← chunk 1
s 0x80 0x00 {后 32768 字节数据}   ← chunk 2
s 0x00 0x10 {最后 16 字节数据}    ← chunk 3
S                                  ← 结束标记 (大写 S)
```

每个 chunk 的前缀是 `s`（0x73）+ 2 字节长度。最后以大写 `S`（0x53）收尾。

这种设计的好处是：反序列化端不需要预先知道字符串总长就可以逐块分配 buffer，适合内存受限的场景。

### 5.5 Map/Object 类型混淆

这是 Hessian 2.0 的一个设计"特性"——`M` tag 既用于 Map 也用于封装类型信息：

```
普通 Map:  H {key-value pairs} Z
类型化 Map: M {type info} Z {key-value pairs} Z
Date 序列化: M {type info} Z {millis value} Z
```

反序列化时，Hessian 根据 type 信息判断最终要创建什么对象：如果 type 是 `java.util.Date`，就解析 key-value 中的毫秒值创建 Date；如果 type 是 `java.util.HashMap`，就创建 HashMap。

这种"一 tag 多义"的设计增加了反序列化器的复杂度，但换来了协议的紧凑性——不需要为每种带类型的值单独设计 tag。

### 5.6 Whitelist/Blacklist 安全机制

反序列化安全是 Dubbo 的重中之重。如果攻击者构造恶意字节流，可能触发任意代码执行（gadget chain）。Dubbo 的防线：

```java
// Dubbo 源码: ClassUtils
// 反序列化前检查类名是否在白名单/黑名单中
private static boolean isAllowClass(Class<?> cl) {
    // 1. 检查系统黑名单（已知危险类）
    if (CLASS_BLACKLIST.contains(cl.getName())) return false;

    // 2. 检查用户配置的白名单
    String whitelist = System.getProperty("dubbo.security.serialization.whitelist");
    if (whitelist != null) {
        return whitelist.contains(cl.getName());
    }

    // 3. 默认允许常见 JDK 类和用户类
    return cl.getName().startsWith("java.")
        || cl.getName().startsWith("javax.")
        || cl.getName().startsWith("com.example.");
}

// 配置方式
// dubbo.properties
// dubbo.security.serialization.whitelist=com.example.dto.*,com.example.model.*
```

---

## 六、典型问题 & 排查实战

### 问题 1：Cannot deserialize XXX

**症状**：
```
com.alibaba.com.caucho.hessian.io.HessianProtocolException:
  unknown code for readObject at 0x50 (...)
```

**根因**：序列化端和反序列化端的类结构不一致。

最常见的场景：
```java
// Provider 端（新版本）
public class User {
    private String name;
    private int age;
    private String email;  // ← 新增字段
}

// Consumer 端（旧版本）
public class User {
    private String name;
    private int age;
    // 没有 email 字段
}
```

当 Provider 返回的 `User` 包含 `email` 字段时，Consumer 端的类定义只有 `name` 和 `age`。Hessian 按类定义的字段数读值，读到 `email` 这个多出来的字段时，字节流的位置会对不齐。

**排查思路**：
1. 对比两端类的字段列表差异：`diff <(javap -p com.example.User) <(ssh remote "javap -p com.example.User")`
2. 检查是否缺少 `serialVersionUID`
3. 查看 Dubbo 日志中是否打印了 `"deserialize error"` 的堆栈

**解决方案**：
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // ← 必须加

    private String name;
    private int age;
    // 新增字段放最后，Hessian 按顺序读，多余的会跳过的
}
```

**注意**：Hessian 实际上不完全依赖 `serialVersionUID`（这是 Java 原生序列化的概念），但它按字段顺序匹配，新增字段放在类定义的最后可以减少版本兼容问题。

### 问题 2：枚举反序列化返回 null

**症状**：
```java
// 序列化时 Color.GREEN → 字节流 → 反序列化后变成 null
Color color = rpcService.getColor();  // 返回 null !!
```

**根因**：Hessian 按 `Enum.name()` 序列化枚举常量，反序列化时调用 `Enum.valueOf(Class, name)`。如果枚举常量改了名字：

```java
// 旧版本
enum Color { RED, GREEN, BLUE }

// 新版本（GREEN → GREEN_COLOR）
enum Color { RED, GREEN_COLOR, BLUE }
// 序列化发的是 "GREEN"，反序列化端找不到 "GREEN"，返回 null
```

**排查思路**：
1. 确认枚举类的变更历史（`git log --oneline -- Color.java`）
2. 在反序列化处打断点，看 `Enum.valueOf()` 的返回值
3. 确认是否修改过常量名

**解决方案**：

方案一：保留旧常量名 + `@Deprecated`：
```java
enum Color {
    RED,
    @Deprecated
    GREEN,        // 保留旧名称
    GREEN_COLOR,  //  新名称（但内部用这个）
    BLUE
}
```

方案二：自定义 Dubbo 序列化器：
```java
public class ColorSerializer extends EnumSerializer {
    @Override
    public void writeObject(Object obj, AbstractHessianOutput out) {
        // 按 ordinal 序列化，而不是 name
        Enum<?> e = (Enum<?>) obj;
        out.writeInt(e.ordinal());
    }
}
// 注册: SerializerFactory.addFactory(...)
```

### 问题 3：泛型集合反序列化类型丢失

**症状**：
```java
// Provider 返回 List<User>
public List<User> getUsers() { ... }

// Consumer 收到后
List<User> users = rpcService.getUsers();
User first = users.get(0);  // ClassCastException: HashMap cannot be cast to User!
```

**根因**：Hessian 不携带 Java 泛型信息。反序列化后 List 的元素实际上是 `HashMap`，Dubbo 需要用方法签名的泛型信息做 `TypeUtils.cast()` 转换，但这个转换过程可能失败。

**排查思路**：
1. 打印 Hessian 反序列化后的中间对象类型：
```java
// Dubbo Filter 中
Object raw = invocation.getArguments()[0];
System.out.println(raw.getClass());  // 看看实际类型是什么
```

2. 检查泛型是否嵌套太深（如 `Map<String, List<User>>`），Hessian 处理多层泛型容易出问题

3. 确认是否开启了泛化调用（`generic=true`）

**解决方案**：

方案一：Dubbo 3.x 中配置泛化调用：
```xml
<dubbo:reference id="userService"
    interface="com.example.UserService"
    generic="true" />
```
用 `GenericService` 方式调用，自己处理类型转换。

方案二：避免复杂泛型嵌套，改用确定的 DTO 封装：
```java
// 不要这样
Map<String, List<User>> getUsersByDept();

// 改为这样
UserListResponse getUsersByDept();  // UserListResponse 内部封装 Map
```

方案三：升级序列化协议（如使用 Protobuf）。

### 问题 4：BigDecimal 精度丢失

**症状**：
```java
// Provider 端
BigDecimal amount = new BigDecimal("123.45");
// 到了 Consumer 端
// amount.toString() → "123.45000000000000001"
```

**根因**：低版本 Hessian（如 Dubbo 2.5.x 内置的版本）将 BigDecimal 序列化为 `double`（使用 `D` tag），而不是 `String`。`double` 类型无法精确表示 `123.45`。

新版本 Hessian 修复了这个问题，改用 `String` 序列化 BigDecimal。但如果 Consumer 和 Provider 的 Dubbo 版本不一致，可能一边用 `double` 一边用 `String`。

**排查思路**：
1. 检查两端的 Dubbo 版本：`grep 'dubbo' pom.xml`
2. 抓包看 Hessian 字节流中 BigDecimal 的 tag 是 `D` 还是 `S`
3. 确认 BigDecimal 在小数点后很多位的场景

**解决方案**：
```java
// 业务代码中显式控制传输格式
public class AmountDTO implements Serializable {
    private String amountStr;  // 用 String 传金额，不用 BigDecimal

    @Transient  // 不序列化
    private BigDecimal amount;

    // 手动转换
    public BigDecimal getAmount() {
        if (amount == null && amountStr != null) {
            amount = new BigDecimal(amountStr);
        }
        return amount;
    }
}
```

### 问题 5：反序列化安全漏洞

**症状**：服务被安全扫描发现有反序列化漏洞（CVE 相关）。

**根因**：恶意构造的 Hessian 字节流可以触发服务端的 gadget chain（如 Commons-Collections 的 `InvokerTransformer`）。

**排查思路**：
1. 检查 Dubbo 版本是否有已知反序列化漏洞
2. 检查是否开启了安全白名单
3. 扫描 classpath 中是否有已知危险库（如 Commons-Collections 3.x）

**解决方案**：

```properties
# dubbo.properties
# 方案1: 开启白名单
dubbo.security.serialization.whitelist=com.example.dto.*
# 方案2: 开启黑名单
dubbo.security.serialization.blacklist=org.apache.commons.collections.*
```

```java
// 方案3: 编程式配置
@Bean
public ProtocolConfig protocolConfig() {
    ProtocolConfig config = new ProtocolConfig();
    config.setSerialization("hessian2");
    config.setSerializationSecurityCheck(true);
    return config;
}
```

Dubbo 2.7.9+ 默认开启了反序列化安全检查，如果还在用旧版本请尽快升级。

### 问题 6：Hessian 与 Hessian2 不兼容

**症状**：
```
com.alibaba.com.caucho.hessian.io.HessianProtocolException:
  expected hessian reply ('H' x48) but got 'c' (x63)
```

**根因**：Consumer 端配置了 `hessian`，Provider 端配置了 `hessian2`（或反过来）。Hessian 1.x 和 Hessian 2.0 的二进制格式完全不同。

**排查思路**：
1. 检查两端的 Dubbo 协议配置
2. 抓包看前几个字节的 tag 是否匹配

**解决方案**：

显式统一配置：
```xml
<!-- Provider -->
<dubbo:protocol name="dubbo" serialization="hessian2" />

<!-- Consumer -->
<dubbo:reference id="xxx" serialization="hessian2" />
```

Dubbo 2.6+ 版本默认使用 `hessian2`，但如果有老旧的消费者可能仍用 `hessian`。

### 问题 7：大量对象序列化 OOM

**症状**：
```java
// 一次 RPC 调用返回 10 万条记录
List<Order> orders = orderService.queryAll();
// Consumer 端 OOM: Java heap space
```

**根因**：Hessian 的引用追踪机制（`IdentityHashMap`）会在序列化过程中持有所有已序列化对象的引用，直到 `reset()` 被调用。一次返回 10 万个对象，`_refs` Map 中就会累计 10 万个 entry。

**排查思路**：
1. 检查 Heap Dump 中 `IdentityHashMap` 的占用
2. 检查是否有超大数据量的 RPC 调用
3. 检查是否调用了 `Hessian2Output.reset()`

**解决方案**：

```java
// 方案1: 分页传输
List<Order> queryOrders(int pageNo, int pageSize);

// 方案2: 流式传输（Dubbo 3.x）
// 使用 StreamObserver / ServerStream

// 方案3: 手动 reset
Hessian2Output out = new Hessian2Output(outputStream);
for (int i = 0; i < orders.size(); i++) {
    out.writeObject(orders.get(i));
    if (i % 100 == 0) {
        out.reset();  // 每 100 个对象清理一次引用表
    }
}
```

### 问题 8：跨语言调用失败

**症状**：用 PHP/Python 写的 Hessian 客户端调用 Java Dubbo 服务，返回的数据解析失败。

**根因**：虽然 Hessian 号称跨语言，但不同语言的 Hessian 库实现质量参差不齐：

| 语言 | 库 | 问题 |
|------|-----|------|
| PHP | hessianphp | 不支持 Hessian 2.0 的 `C`+`O` 分离编码 |
| Python | python-hessian | 不支持引用的环形对象 |
| Go | gohessian | 不处理 Java `Throwable` 的特殊格式 |
| C++ | hessian-cpp | 维护停滞，不支持 Java 8+ 的新类型 |

另外，Hessian 序列化的是 Java 全限定类名（如 `com.example.User`），其他语言无法直接加载 Java 类。所以"跨语言"实际上只是格式兼容，语义上并不互通。

**解决方案**：
- 如果确实有跨语言需求，切换到 **Protobuf**（有成熟的多语言支持）或者使用 **JSON/HTTP** 协议
- 如果必须用 Hessian，在 Java 端把返回类型限定为基本类型 + String + Map/List

### 问题 9：Provider/Consumer 字段增删导致兼容性问题

**症状**：

```
Provider 升级后 Consumer 调不通，或者 Consumer 拿到的对象某些字段为 null。
但诡异的是：不报错，不抛异常，数据静默丢失。
```

**根因深度分析**：

先理解 Hessian 反序列化的核心循环：

```java
// Hessian2Input.readObjectInstance() 核心逻辑
for (int i = 0; i < def.getFields().length; i++) {
    String fieldName = def.getFields()[i];  // ← 来自流的字段名
    Object value = readObject();            // ← 从流中消费一个值
    ser.setField(obj, fieldName, value);    // ← 按字段名匹配本地类的字段
}
```

**关键：循环次数 = 流中的字段数，不是本地类的字段数。** 每轮循环都从流中消费一个值，流位置绝对不会错位。匹配靠字段名（`setField` 内部按 name 查找），不以顺序为准。

以下逐一分析四种增删场景：

---

**场景 A：Provider 多了字段，Consumer 还是旧类**

```
流中: C "User" 3 fields: [name, age, email]     ← Provider 定义
本地: class User { name, age }                   ← Consumer 类定义
```

反序列化过程：

```
i=0: fieldName="name"  → readObject() = "张三"  → obj.setName("张三")   ✅ 匹配
i=1: fieldName="age"   → readObject() = 25      → obj.setAge(25)        ✅ 匹配
i=2: fieldName="email" → readObject() = "z@e"   → 本地类无 email 字段   ⚠️ 值被读出来但丢弃
```

✅ **协议安全**。但 Consumer 拿不到 Provider 新增的 `email` 数据。

---

**场景 B：Provider 少了字段，Consumer 还是旧类**

```
流中: C "User" 2 fields: [name, age]            ← Provider 删了 email
本地: class User { name, age, email }            ← Consumer 旧类
```

```
i=0: fieldName="name"  → obj.setName("张三")     ✅
i=1: fieldName="age"   → obj.setAge(25)          ✅
循环结束 (流中只有2个字段)
obj.email = null  ← 没被赋值，保持默认值
```

✅ **协议安全**。但 Consumer 代码如果调了 `user.getEmail().isEmpty()` → **NPE**。这不是反序列化的错，是业务代码做了下游的 null 不安全操作。

---

**场景 C：Consumer 发请求时字段比 Provider 多**

```
Consumer 序列化: C "User" 3 fields [name, age, email]
Provider 反序列化: 本地只有 name, age
```

Provider 从流中读 3 个值但只填充前 2 个，多余字段丢弃。Provider 的 `email` 字段为 null。

✅ **协议安全**。同样注意 Provider 业务代码的空指针保护。

---

**场景 D：Consumer 发请求时字段比 Provider 少**

```
Consumer 序列化: C "User" 2 fields [name, age]
Provider 反序列化: 本地有 name, age, email
```

Provider 从流中读 2 个值，`email` 保持默认值 null。

✅ **协议安全**。

---

**汇总矩阵**：

```
┌────────────────────┬─────────────────────────┬──────────────────┐
│ 场景                │ Consumer 看到的效果       │ 是否抛异常        │
├────────────────────┼─────────────────────────┼──────────────────┤
│ Provider +字段      │ 新字段值为 null           │ ❌ 不抛           │
│ Provider -字段      │ 被删字段值为 null          │ ❌ 不抛，业务可能NPE│
│ Consumer 入参 +字段  │ Provider 拿不到新字段     │ ❌ 不抛           │
│ Consumer 入参 -字段  │ Provider 拿到 null       │ ❌ 不抛           │
└────────────────────┴─────────────────────────┴──────────────────┘
```

---

**真正会炸的三个场景**：

> ⚠️ **改字段类型**
> ```java
> // 旧: private int age;
> // 新: private String age;  ← 类型变 String
> ```
> 流中是 `S "25"`（String tag），本地类是 `int` → Hessian 尝试 `Integer.parseInt("25")` 可能成功（小数字），但如果是 `S "abc"` 直接炸 `NumberFormatException`。类型不兼容时**不保证成功**。

> ⚠️ **改字段名**
> ```java
> // 旧: private String userName;
> // 新: private String username;  ← 大小写变了
> ```
> Hessian 按字段名精确匹配：`userName` ≠ `username`。旧 Consumer 的 `userName` 为 null，新 Provider 发的 `username` 数据被丢弃。**数据静默丢失，最难排查！**

> ⚠️ **枚举常量改名**（见问题 2）

---

**排查思路**：

1. **对比类定义差异**：
   ```bash
   # 对比两端类的字段列表
   diff <(javap -p com.example.User) <(ssh provider-host "javap -p com.example.User")
   ```

2. **在 Dubbo Filter 中打印反序列化后的对象**：
   ```java
   @Activate(group = CONSUMER)
   public class DeserializeCheckFilter implements Filter {
       @Override
       public Result invoke(Invoker<?> invoker, Invocation invocation) {
           Result result = invoker.invoke(invocation);
           Object value = result.getValue();
           // 反射遍历所有字段，检查是否有意外 null
           for (Field f : value.getClass().getDeclaredFields()) {
               f.setAccessible(true);
               if (f.get(value) == null) {
                   log.warn("字段 {} 反序列化后为 null, 可能是版本不兼容", f.getName());
               }
           }
           return result;
       }
   }
   ```

3. **加序列化版本号**（非 `serialVersionUID`，而是业务字段）：
   ```java
   public class User implements Serializable {
       private int schemaVersion = 2;  // 手动维护版本号
       private String name;
       private int age;
       private String email;  // v2 新增
   }
   // Provider 端判断版本号，做兼容处理
   ```

---

**最佳实践**：

```
新增字段:  末尾追加 → 不影响现有字段序号 → 两端按名字匹配 ✅
删除字段:  先 @Deprecated 过渡一个版本 → 确认无消费者使用 → 再物理删除
修改类型:  ❌ 禁止。新增字段用新类型，旧字段保留并 @Deprecated
修改名称:  ❌ 禁止。等于删了旧的+加了新的，数据静默丢失
枚举常量:  只增不删不改名，废弃的加 @Deprecated
```

一个安全的演进示例：

```java
// === v1 ===
public class OrderDTO implements Serializable {
    private Long orderId;
    private BigDecimal amount;
}

// === v2 (安全演进) ===
public class OrderDTO implements Serializable {
    private Long orderId;
    private BigDecimal amount;

    // v2 新增: 都是末尾追加，不影响老消费者
    private String remark;           // 新增字段，老消费者忽略
    private Integer discountType;    // 新增字段，老消费者忽略
}

// === v3 (安全演进) ===
public class OrderDTO implements Serializable {
    private Long orderId;
    private BigDecimal amount;

    @Deprecated  // ← 标记废弃但不删除
    private String remark;          // 老消费者可能还在用

    private Integer discountType;

    // v3 新增: 替代 remark 的新字段
    private String orderNote;       // 新字段，老消费者拿 null
}
```

---

## 七、性能优化实战

### 7.1 使用 writeReplace / readResolve

这两个方法是 Java 原生序列化的钩子，Hessian 同样支持：

```java
public class SingletonConfig implements Serializable {
    private static final SingletonConfig INSTANCE = new SingletonConfig();

    // 序列化时用代理对象代替
    private Object writeReplace() {
        return new ConfigProxy(this.name, this.value);
        // 代理对象更简单，字段更少
    }

    // 反序列化后还原为单例
    private Object readResolve() {
        return INSTANCE;  // 始终返回同一个实例
    }
}
```

这适用于单例对象、缓存对象等不需要完整传输的场景。

### 7.2 reset() 的正确时机

```java
// 错误: 从不 reset，引用表无限增长
Hessian2Output out = ...;
for (Object item : hugeList) {
    out.writeObject(item);
}

// 正确: 定期 reset
Hessian2Output out = ...;
int count = 0;
for (Object item : hugeList) {
    out.writeObject(item);
    if (++count % 500 == 0) {
        out.reset();       // 清理引用表
        out.resetReferences(); // 清理类引用表  ← 容易遗漏!
    }
}
```

**关键**：`reset()` 清的是值引用表，`resetReferences()` 清的是类定义引用表。如果只调用 `reset()`，类定义累积过多也会导致内存问题。

### 7.3 泛化调用 vs 强类型调用的开销

```java
// 强类型调用: 序列化/反序列化直接到目标类型
User user = userService.getUser(1L);
// 序列化: User对象 → 二进制
// 反序列化: 二进制 → User对象 (一次完成)

// 泛化调用: 中间多一层 Map 转换
GenericService svc = (GenericService) userService;
Object result = svc.$invoke("getUser", new String[]{"long"}, new Object[]{1L});
// 序列化: Map → User(反射填充) → 二进制
// 反序列化: 二进制 → User → Map(反射提取)
// 多了两次反射操作!
```

泛化调用的额外开销大约在 10-20%，在频繁调用场景下应尽量用强类型调用。

### 7.4 序列化大小对比

实际测试（一个包含 10 个字段的简单 DTO）：

```
┌──────────────────┬──────────────┬───────────────────┐
│ 序列化方式         │ 大小 (bytes) │ 相对 Hessian2     │
├──────────────────┼──────────────┼───────────────────┤
│ Java 原生         │ 892          │ 2.3x              │
│ JSON (Fastjson)   │ 365          │ 0.94x             │
│ Hessian2          │ 387          │ 1.0x (基线)       │
│ Kryo (优化后)     │ 178          │ 0.46x             │
│ Protobuf          │ 142          │ 0.37x             │
└──────────────────┴──────────────┴───────────────────┘
```

Hessian2 的体积大约是 Java 原生的 40%，但比 Kryo 和 Protobuf 大约一倍。如果你的服务对带宽敏感，可以考虑切换序列化协议。

---

## 八、总结与建议

### 8.1 Hessian 的适用场景

```
推荐使用 Hessian:                 考虑其他方案:
├─ 纯 Java 微服务集群             ├─ 有跨语言调用需求 → Protobuf
├─ 字段变更频繁（自描述）          ├─ 超高性能要求 → Kryo
├─ 不想维护 .proto 文件           ├─ 需要强 Schema 校验 → Protobuf
├─ Dubbo 默认，改动成本低          ├─ 数据量超大 → Kryo + 压缩
└─ 团队对二进制协议了解有限        └─ 安全敏感场景 → 需额外配置白名单
```

### 8.2 Dubbo 3.x 的演进方向

Dubbo 3.x 引入了 **Triple 协议**（基于 gRPC），默认使用 **Protobuf** 序列化。这标志着 Dubbo 的核心序列化战略正在从 Hessian 转向 Protobuf。但 Hessian 作为存量系统的默认方案，仍将长期存在。

### 8.3 生产环境 Checklist

在关键服务上线前，建议逐项确认：

```
□ 所有传输 DTO 实现了 Serializable 接口
□ 敏感类已加入反序列化白名单
□ 枚举类检查是否有常量名变更历史
□ BigDecimal 字段确认传输格式（String vs double）
□ Consumer 与 Provider 的 Dubbo 版本一致
□ 序列化协议显式配置为 hessian2（非 hessian）
□ 大数据量接口已做分页（单次不超过 1000 条）
□ 没有需要跨语言调用的场景使用 Hessian
□ 泛型集合类型已在方法签名中完整声明
```

---

## 参考资源

- [Hessian 2.0 Serialization Specification](http://hessian.caucho.com/doc/hessian-serialization.html) — Caucho 官方规范文档
- [Dubbo Serialization 文档](https://dubbo.apache.org/zh/docs/references/protocols/rest/)
- [Hessian2Output 源码](https://github.com/apache/dubbo/blob/3.2/dubbo-serialization/dubbo-serialization-hessian2/src/main/java/org/apache/dubbo/common/serialize/hessian2/Hessian2ObjectOutput.java) — Dubbo 中的 Hessian 序列化实现
