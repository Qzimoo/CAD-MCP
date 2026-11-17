# CAD-MCP 试用指南（Windows PC 操作版）

> 对象：GitHub 仓库 `daobataotie/CAD-MCP`（v2.0.0，MIT 协议）。它是一个 MCP 服务，通过 Windows COM 接口控制本机 CAD 软件，让 Codex / ZCode 等 AI agent 用自然语言画图、查图、改图。
> 本指南依据仓库 `README_zh.md`、`src/config.json`、`src/server.py`、`src/cad_controller.py` 核实整理，2026-11-17。

---

## 1. 前置条件

| 项 | 要求 |
|---|---|
| 操作系统 | Windows（必须，依赖 pywin32 + COM，Mac/Linux 跑不了） |
| CAD 软件 | AutoCAD、浩辰CAD（GCAD/GstarCAD）或中望CAD（ZWCAD）任一，已正常安装并能手动打开 |
| Python | 3.10 或更高版本，安装时勾选「Add Python to PATH」 |
| AI 客户端 | Codex 或 ZCode（任一支持 MCP 的客户端即可） |

依赖（`requirements.txt`）：`pywin32>=306`、`mcp>=1.12,<2`、`pydantic>=2.0`、`Pillow>=10.0`。

---

## 2. 安装

在 Windows 上打开命令行（cmd 或 PowerShell），执行：

```bat
git clone https://github.com/daobataotie/CAD-MCP.git C:\cad-mcp
cd C:\cad-mcp
pip install -r requirements.txt
```

（没有 git 也可以在 GitHub 页面下载 ZIP 解压到 `C:\cad-mcp`。pip 下载慢可加 `-i https://pypi.tuna.tsinghua.edu.cn/simple` 用清华镜像。）

可选：也可以 `pip install -e .` 把它装成命令行工具，之后用 `cad-mcp` 命令启动。

### 改一处配置（按需）

配置文件在 `C:\cad-mcp\src\config.json`，关键项：

- `cad.type`：默认 `"AUTOCAD"`。用浩辰填 `GCAD`，中望填 `ZWCAD`。
- `cad.progid`：默认留空，服务会从注册表自动探测 CAD 版本。**如果电脑上装了多个版本的 AutoCAD、连错了版本**，就按 README 示例填版本化 ProgID（如 `AutoCAD.Application.25.1`）。
- `cad.startup_wait_time`：CAD 启动等待秒数，默认 20。电脑慢、CAD 冷启动久就调大到 40。
- `output.directory`：保存图纸的默认目录，默认 `./output`（相对于服务启动目录）。

---

## 3. 启动顺序（重要）

推荐顺序：**先开 CAD，再连服务**。

1. 先手动打开 AutoCAD，新建一张空白图，停在绘图界面。
   （服务本身也会尝试附着已运行的 CAD、附着不到时自动拉起一个新实例，但先手动打开最稳，还能亲眼看着它画。）
2. 再启动 AI 客户端（Codex / ZCode），由客户端按下面的配置自动拉起 MCP 服务（stdio 模式），不用手动运行 `server.py`。
3. 第一次让 agent 画图前，先让它执行 `list_layers` 或 `screenshot`，确认连接通了、它能「看见」当前图纸。

### 备选：HTTP 模式（多客户端共享 / 手动启动时用）

```bat
cd C:\cad-mcp
python src\server.py --transport streamable-http --port 8000
```

服务起后，客户端改为连接 `http://127.0.0.1:8000/mcp`。手动调试也可以用官方检查工具：

```bat
npx -y @modelcontextprotocol/inspector python C:\cad-mcp\src\server.py
```

---

## 4. 客户端 MCP 配置

仓库只给了一份通用 JSON 配置（原为 Claude Desktop / Cursor 等客户端的格式），Codex 与 ZCode 的配置文件格式与字段名以各自客户端文档为准——**把下面的 `command` 和 `args` 两个值按客户端 MCP 配置填入命令和参数即可，args 里的路径换成你机器上的实际路径**：

```json
{
    "mcpServers": {
        "CAD": {
            "command": "python",
            "args": [
                "C:\\cad-mcp\\src\\server.py"
            ]
        }
    }
}
```

- **Codex**：在其 MCP 服务器配置中新增一个名为 `CAD` 的服务，命令填 `python`，参数填 `C:\cad-mcp\src\server.py`（按客户端 MCP 配置填入命令和参数）。
- **ZCode**：在其 MCP 服务器配置中新增一个名为 `CAD` 的服务，命令填 `python`，参数填 `C:\cad-mcp\src\server.py`（按客户端 MCP 配置填入命令和参数）。
- 若客户端支持填 URL 而非命令，改用上面的 HTTP 模式，地址填 `http://127.0.0.1:8000/mcp`。

配好后重启客户端，让它问一句「列出当前图纸的图层」，能返回图层列表就是通了。

---

## 5. 它能干什么（工具概览，共 35 个）

- **基本绘图（9）**：直线、圆、圆弧、椭圆、多段线、矩形、文本、填充、线性标注
- **查询（4）**：图层列表、实体列表（带可编辑句柄）、实体属性、CAD 窗口截图
- **编辑（11）**：删除、移动、旋转、缩放、复制、镜像、偏移、线性/环形阵列、撤销/重做
- **图层与图纸（10）**：建图层、切换当前图层、新建/打开/关闭/保存图纸（DWG/DXF）、建块/插块/列块、任意 CAD 命令直通（`send_command`，工具不支持的操作可以用它兜底）
- **遗留接口（1）**：自然语言命令处理

注意它的定位是「基础图元 + 编辑 + 管理」，没有现成的法兰、阀门、泵符号库；复杂符号要让 agent 用基本图元拼，或先做成图块反复插入。

---

## 6. 第一个测试任务建议（冶金工艺小图）

连通后，把下面这段原样发给 Codex / ZCode 试跑（离心泵平面符号 + 一段带标注的管道单线）：

> 在当前空白图纸中依次完成：
> 1. 新建 4 个图层：`EQUIP`（设备）、`PIPE`（管道）、`CENTER`（中心线）、`ANNO`（标注文字）；
> 2. 在 `CENTER` 层，以 (0,0) 为中心画一条长 1200 的水平中心线和一条长 1200 的垂直中心线；
> 3. 在 `EQUIP` 层，以 (0,0) 为圆心画直径 600 的圆作为离心泵泵壳的平面符号，再同心画一个直径 200 的小圆表示叶轮；
> 4. 在 `PIPE` 层，从 (300,0) 沿 +X 方向画一条长 2400 的管道单线到 (2700,0)，再从 (0,300) 沿 +Y 方向画一条长 1200 的管道单线到 (0,1500)；
> 5. 在 `ANNO` 层，用标注标出水平管道长度 2400，并加文字「P-101 离心泵」放在泵符号下方 (0,-500) 处，字高 120；
> 6. 执行截图确认效果，再把图纸保存为 `test_pump.dwg`。

**验收方法**：让 agent 依次执行 `list_layers`（应有 4 个新图层）、`list_entities`（逐个核对圆、线、标注的坐标尺寸）、`screenshot`（肉眼看图）。三步都对，说明「画—查—验」闭环可用，再考虑往真实施工图上放。

---

## 7. 常见坑

1. **权限不一致连不上 COM**：AutoCAD 如果是「以管理员身份运行」，而客户端/服务是普通权限（或反过来），COM 附着会失败。两边用同一权限级别启动。
2. **连错 CAD 版本**：装了多个版本 AutoCAD 时自动探测可能连到旧版。在 `src/config.json` 里把 `cad.progid` 填成你要用的版本化 ProgID。
3. **首次连接超时**：CAD 冷启动慢，默认只等 20 秒。把 `startup_wait_time` 调到 40 再试。
4. **stdio 模式下手动运行 `server.py` 看起来「卡住」**：这是正常的，它在等客户端从标准输入发指令。日常使用让客户端自动拉起即可；想手动验证就用 HTTP 模式或 MCP Inspector。
5. **连续画复杂图时个别命令丢失/错位**：命令间默认延迟 0.5 秒（`command_delay`），老机器或大图可调到 1.0。
6. **保存路径找不到**：`output.directory` 的 `./output` 是相对服务启动目录的，不是相对图纸目录；找不到文件先去 `C:\cad-mcp\output` 看。
7. **pywin32 安装报错**：多为 Python 版本与 pywin32 不匹配或缺 C++ 运行库，换官方 Python 3.11/3.12 并升级 pip 后重装。
8. **先拿空图练手**：agent 的编辑、删除是真操作。正式施工图上试之前先复制一份备份图，或只在测试图上跑，确认风格（图层、字高、标注样式）符合你的出图习惯再放行。

---

## 8. 小结

- 它解决的是「让 AI 直接在本机 AutoCAD 里落图元、查实体、改图」的问题，适合施工图里重复性强的放线、标注、符号插入类工作。
- 它的上限取决于图块库和你的指令颗粒度：先把常用设备符号做成图块，agent 插块 + 连线 + 标注的效率会明显高于每次现拼。
- 出图质量与签字责任仍在人：按你的现有分工，AI 出草图、你校核后再交付。
