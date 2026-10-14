# IDA Pro 准备指引

> **IDA Pro 是商业软件，不随本包分发。** 你需要自备授权，或使用免费替代方案。

---

## 方案 A：已有 IDA Pro 授权（推荐，功能最全）

### 支持版本
IDA Pro **8.4+ / 9.x**（含 Hex-Rays 反编译器）

### 安装后需要做的

> **关于 Python 解释器（重要）**：网上大量教程写 `<IDA安装目录>\python311\python.exe`，
> 但 **IDA 9.5 起安装目录下通常已不再附带 `python.exe`** —— `<IDA安装目录>\python\` 只是
> IDAPython 的脚本库（`idaapi.py` / `ida_hexrays.py` …），并非解释器。
> 因此请**先确认解释器真实位置**，再决定用哪一个：

```powershell
# 探测顺序：IDA 自带 Python → 系统 Python
Get-ChildItem "<IDA安装目录>" -Recurse -Filter python.exe -ErrorAction SilentlyContinue
python --version   # 回退方案：系统 Python
```

1. **安装 `ida-pro-mcp` 与 `idapro` 绑定**（用上一步确认的解释器）：
   ```powershell
   & "<PYTHON>" -m pip install ida-pro-mcp
   ```

2. **激活 idalib（仅模式 A 需要，极易遗漏）**：`idapro` 必须先知道 IDA 装在哪，
   否则 `import idapro` 会直接失败。运行 IDA 自带的激活脚本：
   ```powershell
   & "<PYTHON>" "<IDA安装目录>\idalib\python\py-activate-idalib.py" -d "<IDA安装目录>"
   ```
   它会把路径写入 `%APPDATA%\Hex-Rays\IDA Pro\ida-config.json`。
   （也可改用环境变量 `IDADIR`，优先级更高。）

3. **运行 `install.ps1`** —— 它会自动探测 IDA 路径并写入 `mcp.json`。
   若探测失败，手工填 `~/.pi/agent/mcp.json`（见下方【两种运行模式】）。

4. **使用方式**：在 pi 里说「用 IDA 分析这个文件」，`ida-reverse` Skill 会自动唤醒 IDA 并挂载 MCP。

> ⚠️ **端口说明（常见误区）**：只有**模式 B**（IDA GUI 内运行 `server.py` 插件）才监听
> `http://127.0.0.1:13337`。**模式 A（`--stdio`）根本不监听任何端口** —— 它通过标准输入输出
> 通信，supervisor 的 HTTP 端口默认是 `8745`（仅在非 `--stdio` 时才启用）。
> 因此**不要**在模式 A 下去探测 13337，那会永远探测不到。

### 两种运行模式（IDA 9.x 起）

`install-ida.ps1` 会**优先探测 IDA 9.x 原生模块模式**，失败才回退到传统脚本模式。

#### 模式 A：IDA 9.x 原生 `idalib_supervisor`（推荐，无需 GUI 常驻）

IDA 9.4+ 把 MCP 服务做成了可独立运行的 `idalib` 模块，由 supervisor 拉起，
**不需要手工打开 IDA 图形界面**：

```json
"ida": {
  "command": "<PYTHON>",
  "args": ["-m", "ida_pro_mcp.idalib_supervisor", "--stdio"],
  "env": { "PYTHONIOENCODING": "utf-8" },
  "transport": "stdio",
  "lifecycle": "eager",
  "requestTimeoutMs": 180000
}
```

> `<PYTHON>` 是**任意能 `import ida_pro_mcp` 的 Python 3.11+ 解释器**（系统 Python 亦可），
> 不要求是 IDA 自带的那个 —— supervisor 会用 `sys.executable` 自行拉起 worker。

**实测工具数**：默认 **70** 个；加 `--unsafe` 为 **73** 个（多出 `py_eval` / `py_exec_file` /
`diff_before_after`）。

> ⚠️ **模式 A 的能力边界**：无头 `idalib` 提供完整的反编译 / 类型恢复 / 交叉引用 / 签名能力，
> 但 **不暴露 `dbg_*` 调试器工具**（22 个断点、单步、寄存器类工具）。
> 需要**动态调试**时必须改用模式 B（打开 IDA GUI + `server.py`）。

#### 模式 B：传统 `server.py` 脚本模式（IDA 8.4 ~ 9.3，或需要 GUI 调试时）

```json
"ida": {
  "command": "<PYTHON>",
  "args": ["<PYTHON_SITE_PACKAGES>\\ida_pro_mcp\\server.py"],
  "env": { "PYTHONIOENCODING": "utf-8" },
  "transport": "stdio",
  "lifecycle": "eager",
  "requestTimeoutMs": 180000
}
```

> 该模式由 IDA GUI 内的插件承载，**监听 `http://127.0.0.1:13337`**，`ida-reverse` Skill
> 会轮询该端口等待就绪。只有此模式才需要保持 IDA 图形界面开启。

> 不确定用哪种？直接跑 `powershell -File .\setup\install-ida.ps1`，脚本会自动判定并打印
> 实际选用的模式（`配置为 IDA 原生模块运行模式` 或 `配置为 server.py 脚本模式`）。

#### 其他 Agent 的配置示例（DSH / Codex 风格）

```yml
- insert:
    - id: ida-mcp
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: ida
        transport: stdio
        command: '<PYTHON>'
        args:
          - '-m'
          - 'ida_pro_mcp.idalib_supervisor'
          - '--stdio'
        env:
          PYTHONIOENCODING: utf-8
        toolCallTimeoutMs: 180000
```

> **DSH 用户注意**：DSH 会清洗子进程环境变量 —— 凡名字匹配 `/KEY|PASSWORD|SECRET|TOKEN/i`
> 或 `DSH_*` 的变量都会被丢弃。所以**不要**指望用 `env:` 传 `IDADIR`（`DSH_*` 同理）；
> 请改用 `py-activate-idalib.py` 写入 `ida-config.json` 的方式，或把路径直接写进 `command`。

### 购买渠道
- 官方：https://hex-rays.com/ida-pro
- 免费版 IDA Free（无 Hex-Rays 反编译）：https://hex-rays.com/ida-free

---

## 方案 B：免费替代（无 IDA 授权时）

**核心结论：seep MCP 的 8 个 Radare2 工具已覆盖大部分场景，无需 IDA 也能干活。**

| 能力 | IDA MCP | 免费替代 |
| :--- | :--- | :--- |
| 快速侦察（架构/壳/熵/字符串） | — | ✅ `seep_r2_info` / `seep_r2_strings` |
| 反汇编（含交叉引用） | ✅ | ✅ `seep_r2_disasm` |
| 反编译为类 C 伪代码 | ✅ Hex-Rays | ✅ `seep_r2_decompile` |
| 二进制差分 | — | ✅ `seep_r2_diff` |
| 汇编 ↔ 机器码 | — | ✅ `seep_r2_asm` |
| 函数表 / 导入导出 | ✅ | ✅ `seep_r2_functions` |
| 结构体恢复 | ✅ | ⚠️ Ghidra |
| 类型推断 | ✅ | ⚠️ Ghidra |

### 推荐组合

```
基础层：seep MCP（r2 八件套）      ← 脚本自动装，开箱可用
增强层：Ghidra（免费，含反编译）    ← 需要时下载
可选层：IDA Free（无反编译）        ← 需要 GUI 时
```

### Ghidra 安装

```powershell
# 需先装 JDK 17+
winget install EclipseAdoptium.Temurin.17.JDK
# 下载 Ghidra
# https://github.com/NationalSecurityAgency/ghidra/releases
```

> `Tool/kb/tools/skills/mcp/GhidraMCP/` 提供了 Ghidra 桥接 MCP，可按需注册。

---

## 方案 C：只用 Radare2（最轻量）

**完全跳过 IDA**，只依赖 seep MCP：

```powershell
# install.ps1 已自动装 radare2 到 tools/radare2/
# 直接用这些工具：
#   seep_r2_info / _strings / _functions / _disasm / _decompile / _diff / _asm
```

对以下任务**足够**：
- 壳检测与熵分析
- 字符串与端点提取
- 授权判定点定位
- 补丁点确认（配合 `seep_r2_asm` 验字节）
- Android SO 分析

**不足**：
- 复杂 C++ 逆向的类型恢复
- 大规模交叉引用图谱
- 大型二进制的交互式分析

---

## 三种方案对比

| | 方案 A（IDA Pro） | 方案 B（免费组合） | 方案 C（仅 r2） |
| :--- | :--- | :--- | :--- |
| **成本** | 商业授权 | 免费 | 免费 |
| **反编译** | ✅ Hex-Rays（最强） | ⚠️ Ghidra（次之） | ⚠️ r2 伪代码 |
| **结构体恢复** | ✅ | ⚠️ 部分 | ❌ |
| **上手成本** | 低（自动联动） | 中 | 最低 |
| **覆盖场景** | 全部 | 90% | 70% |

> **建议**：先用**方案 C** 跑通流程，遇到真正需要深度反编译的目标再上方案 A。
