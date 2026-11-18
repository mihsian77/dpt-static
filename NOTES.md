# DPT 静态分析

DPT 加固 APK 的静态分析参考。用于不运行样本的前提下，定位数据段、还原被清空的方法体、
校验还原结果。

## 环境

| 依赖 | 用途 |
|---|---|
| JDK 8+ | 读写 dex |
| `nm` / `readelf` | 取 ELF 符号与构建标记 |
| jadx / apktool | 解资源、看外壳结构 |

## 加壳后的结构

| 位置 | 内容 |
|---|---|
| `classes*.dex` | 外壳，保留方法与签名，方法体为空或填充指令 |
| `assets/OoooooOooo` | 抽出的指令数据段 |
| `assets/d_shell_data_001` | 壳配置，含入口类、签名校验、开关位 |
| `lib/**.so` | 解密材料的出处，随版本变化 |

## 分析步骤

1. **识别版本**：从配置或 so 的构建标记判断，决定后续按哪套布局解析
2. **解配置段**：取得入口类、dex 分块索引与开关位
3. **定位数据段**：按 dex 分块，读块头的方法数与索引表
4. **回填指令**：按索引把指令写回对应方法体
5. **修 dex 头**：重算 adler32 校验和与 sha1 签名
6. **校验**：逐方法比对签名与指令序列

## 数据段格式

每个 dex 的指令按三段组织，对应 `dpt/model/DexCode.java`：

| 字段 | 类型 | 说明 |
|---|---|---|
| methodCount | short | 该 dex 的方法数 |
| insnsIndex | int[] | 每个方法的指令起始偏移 |
| insns | list | 指令体，按 code item 拼接 |

旧版样本的块头以长度字段打头，字段顺序与新版不同，解析时不能按同一偏移硬读。

## 密钥链路

对应 `dpt/util/CryptoUtils.java`：

1. 从 so 的构建标记与数据符号取 key 材料
2. `HMAC-SHA256(randomKey, keyMaterial)` 得 32 字节 AES-256 key
3. 该 key 解配置段
4. 每条指令的 RC4 key：`aesKey(32B) || methodIdx(LE u32)`

配置字段见 `dpt/config/ShellConfig.java`。`risk_check_flags` 是位开关，运行时读取；
`insnsCryptKey` 不写入 JSON，仅内存使用。

## 校验

- 方法签名一致
- 指令序列一致
- dex 头重算正确（adler32 + sha1）

任一项不一致按失败处理，不做部分成功的判断。

## 版本对照

配置字段逐 tag 比对（`ShellConfig.java`），指令加密对照 `CryptoUtils.java`：

| 版本 | 配置字段变化 | 指令加密 |
|---|---|---|
| v2.19.0 | 含 `insns_xor_key`（随机 int，写入配置） | XOR，key 取自配置 |
| v2.20.0 | 去掉 `insns_xor_key` | RC4，key = AES-256 key + methodIdx(LE) |
| v2.21.0 | 新增 `junk_cls_name` | 同上 |
| v2.21.1 | 新增 `app_pkg_name` | 同上 |
| v2.22.0 / v2.22.1 | 字段不变 | 同上 |

`methodCount` / `insnsIndex` / `insns` 三段结构在所有版本中一致，差异只在配置字段和
指令加密方式。

未核实项：旧版数据段的块头是否存在长度打头的变体，上游 Java 侧看不到序列化实现，
需要对照 native 代码才能确认。

## 许可

MIT。引用的代码版权归原作者。
