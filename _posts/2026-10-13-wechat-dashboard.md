---
layout:     post
title:      "公众号数据看板 v2：从零开始，小白也能搭起来"
subtitle:   " \"从拿 AppSecret、配 IP 白名单，到本地采集、同步线上、每天自动更新，一步一验收\""
date:       2026-10-13 12:00:00
author:     "Yellowsoy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 分享
    - Python
    - AI编程
    - 数据看板
    - 微信公众号
---

> **更新说明**：本文最初发布于 2026-10-13，讲「本地 Python + JSON + 单文件 HTML」的零成本方案。
> 用了两周后暴露出一堆问题（密码裸奔、回收站不同步、发布日期错标……），于是有了 v2。
> 本文已按 v2 **整体重写，并改写成从零开始的完整教程**：不管你有没有写过代码，照着做就能跑通。
> 完整源码开源在 GitHub（见第五节）。

想要一个能**自动更新**的公众号数据看板：顶部一排 KPI、阅读分享趋势、粉丝增长、来源构成、文章榜单和明细表。而要求很朴素：**不花钱、不买服务器、不装数据库、不依赖云服务**。

这篇文章从"微信后台该点哪里"开始讲起，一直到"每天自动更新、手机上也能看"，每一步都给出**验收标准**——做完一步自己验证一步，不用猜。

* * *

## 目录

01. 先定目标：做到什么程度算成功（OKR 验收清单）
02. v2 相比 v1 改了什么
03. 先搞懂几个名词（小白必看）
04. 方案原理：为什么必须"本地采集"
05. 开工前：把源码和 Python 准备好
06. 微信侧配置：权限、AppSecret、access_token、IP 白名单
07. 本地部署：一步一步（Windows / macOS / Linux）
08. 把数据送到线上端
09. 设置每天自动同步
10. 排障大全（几乎所有你可能遇到的问题）
11. 备份、迁移与恢复
12. 进阶：v2 的五个关键改造（含代码）

* * *

## 一、先定目标：做到什么程度算成功（OKR 验收清单）

先说清楚终点在哪，免得做到一半不知道对不对。

**O（目标）**：在自己的电脑上跑起一个**会自动更新**的公众号数据看板，并且手机、其他电脑打开链接也能看到同一份数据。

| | 关键结果 | 验收标准（做到了打勾） |
| --- | --- | --- |
| **KR1** | 微信侧打通 | 拿到 AppID / AppSecret；手动请求能返回一个 `access_token`（而不是 40164） |
| **KR2** | 本地采集跑通 | `sync.py` 执行完不报错；目录下出现 `store.json` 和 `data.js`，文章数 > 0 |
| **KR3** | 看板能看到 | 打开页面：KPI 有数字、趋势图有曲线、文章明细表有行 |
| **KR4** | 线上也能看 | 线上地址打开看到同样的数据；删一篇文章，手机上也消失 |
| **KR5** | 自动跑起来 | 定时任务配好，连续 3 天不用管；漏跑一天能自己补上 |

**开工前的自检表**（有一项不满足就先解决它）：

- [ ] 公众号是**已认证**的订阅号或服务号（个人订阅号不行）
- [ ] 你是管理员，或者已被绑为**长期运营者**
- [ ] 电脑能上网，装了 Python 3.7 以上
- [ ] 有 20 分钟不被打断的时间

> 第一项卡死最多人：**数据统计接口只对已认证账号开放**。没认证的话，后面全部白做，先去认证。

## 二、v2 相比 v1 改了什么

| 问题 | v1 | v2 |
| --- | --- | --- |
| 密码写在前端 JS，F12 可见 | `var PWD = '...'` | 挪到服务端，校验后下发 8 小时令牌 |
| 回收站不同步（这台删了那台还在） | 存本机浏览器 | 云端状态，全设备同步 |
| 只能单机看 | 手动拷文件夹 | 增加线上部署版，本地采完自动推送 |
| 周统计只有一种口径 | 固定"近 7 天" | 近 7 天滚动 / 本周（周一至周日）可切换 |
| 老文章发布日期错标 | `setdefault` 写死首次日期 | 按文章日报取最早日期重算 |
| 趋势图拉长看不清 | 无缩放 | 底部比例尺拖拽 + 滚轮/双指缩放 |
| 布局固定 | 无 | 模块可拖拽调位置和大小 |

一句话：**v2 没放弃"零成本"，只是把"谁来验证身份、数据存在哪"重新安排了一下。**

## 三、先搞懂几个名词（小白必看）

名词 | 通俗解释
--- | ---
**AppID（开发者ID）** | 公众号的"身份证号"，`wx` 开头，公开可查，不保密
**AppSecret（开发者密码）** | 公众号的"密码"，**绝不能泄露**，用它换 access_token
**access_token（访问令牌）** | 拿 AppID + AppSecret 换来的"临时通行证"，**有效期 7200 秒（2 小时）**，调数据接口必须带它
**IP 白名单** | 微信只允许白名单里的 IP 去**换取 token**，不在名单里直接拒绝
**公网 IP** | 你家宽带在互联网上的"门牌号"，和内网 `192.168.x.x` 不是一回事
**datacube** | 微信官方数据统计接口组，取阅读、分享、粉丝等
**UV / PV** | UV = 阅读**人数**（同一个人看多次算 1）；PV = 阅读**次数**。**微信自 2025-11-01 起只保留 UV，取消了 PV**

## 四、方案原理：为什么必须"本地采集"

先把三条看似能偷懒的路堵死，你才会死心塌地按本文做：

**1. 纯静态 HTML 直接调微信接口？不行**
浏览器跨域被拦（CORS）；AppSecret 写在前端等于公开；而且页面跑在每个访客自己的网络里，IP 不固定，没法加白名单。

**2. Cloudflare Workers 免费版？不行**
免费版出口 IP 是动态共享地址，每天在变，无法加白名单。

**3. GitHub Actions 代跑？也不行**
同样是一大段动态 IP 段。

### 那"线上版"是怎么回事？

这是 v2 最关键的设计：**线上版不采集，只负责"展示 + 状态同步"。**

```
   ┌────────────┐   每天定时    ┌──────────────────────┐
   │  定时任务    │ ──────────▶ │      sync.py         │
   └────────────┘               │  (token 中控 + 采集)  │
                                └──────────┬───────────┘
                                           │ 写入
                                 ┌─────────▼─────────┐
                                 │  store.json（数据库）│
                                 └─────────┬─────────┘
                                           │ 生成
                                 ┌─────────▼─────────┐
                                 │      data.js       │
                                 └─────────┬─────────┘
                          ┌────────────────┴────────────────┐
                          │ 读取                        自动推送 │
                 ┌────────▼────────┐            ┌──────────▼─────────┐
                 │ 本地 index.html  │            │ 线上 index.html     │
                 │ （自己电脑看）    │            │ （手机/别的电脑也能看）│
                 └─────────────────┘            └──────────┬─────────┘
                                                           │ /api/state
                                                 ┌─────────▼─────────┐
                                                 │ 回收站 / 重命名 / 布局 │
                                                 └───────────────────┘
```

采集只能在你本机跑（因为 AppSecret 和白名单都在你手上），采完生成 `data.js` 推到线上。线上只是个"能给别人看的展示层"。

三个数据文件各司其职：

| 文件 | 作用 |
| --- | --- |
| `store.json` | 本地累积库：文章字典、日报、删除台账。**这是你最该备份的文件** |
| `data.js` | 看板渲染数据（KPI、趋势、文章明细） |
| `state.json`（线上） | 云端共享状态：回收站、重命名、布局 |

## 五、开工前：把源码和 Python 准备好

### 1. 拿到源码（两种方式，选一个）

**方式 A：下载 ZIP（推荐给完全不熟命令行的）**

1. 打开 https://github.com/Yellowsoy/gzh-data-dashboard
2. 点绿色按钮 **Code → Download ZIP**
3. 解压，得到一个 `gzh-data-dashboard-main` 文件夹

**方式 B：git clone（熟悉命令行的）**

```bash
git clone https://github.com/Yellowsoy/gzh-data-dashboard.git
```

目录长这样，你只需要关心 `local-deployment`（本地版）和 `workbuddy-deployment`（线上版）：

```
gzh-data-dashboard/
├── README.md
├── LICENSE                       # MIT 协议
├── local-deployment/             # ← 先用这个
│   ├── index.html                # 看板页面
│   ├── serve.py                  # 本地服务
│   ├── sync.py                   # 采集脚本（核心）
│   ├── config.example.json       # 配置模板
│   ├── run.sh / stop.sh          # 一键启动 / 停止
│   └── vendor/                   # echarts / xlsx 离线库
└── workbuddy-deployment/         # ← 第八节才用到
    ├── index.html
    ├── server.py
    └── vendor/
```

### 2. 确认 Python 装好了

打开终端（Windows 用「命令提示符」或 PowerShell；Mac/Linux 用「终端」），执行：

```bash
python3 --version      # Mac / Linux
python --version       # Windows
```

看到 `Python 3.7.x` 以上就行。没装的话：

- **Windows**：去 https://www.python.org/downloads/ 下载，**安装时务必勾选 "Add Python to PATH"**（漏了会很麻烦）
- **macOS**：`brew install python3`，或官网下载
- **Linux / 统信 UOS**：一般自带；没有就 `sudo apt install python3`

> **验收点**：`python3 --version` 能打印版本号，不是"不是内部或外部命令"。

### 3. 建一个固定目录，把文件放进去

新建一个文件夹，路径**不要带中文和空格**，之后也别再挪动：

- Windows：`D:\wechat-dashboard`
- Mac / Linux：`~/wechat-dashboard`

把 `local-deployment` 里的**全部内容**（含 `vendor` 文件夹）拷进去。

> **验收点**：目录里能看到 `sync.py`、`serve.py`、`index.html`、`config.example.json`、`vendor/`。

## 六、微信侧配置：权限、AppSecret、access_token、IP 白名单

这一节是全文最容易卡住的地方，慢慢来。

### 6.1 关于"成为开发者"：你需要什么权限

微信现在没有单独的"开发者"按钮了，实际起作用的是**身份**：

| 身份 | 能做什么 |
| --- | --- |
| **管理员** | 公众号注册时的微信号，权限最高：能重置 AppSecret、改 IP 白名单、绑运营者 |
| **长期运营者** | 管理员在后台绑定后，基本等同管理员（可重置 AppSecret、改白名单） |
| **短期运营者** | 有期限，权限同上但会过期 |

**如果你不是管理员**，让管理员这么做：

1. 管理员登录 https://mp.weixin.qq.com
2. **设置与开发 → 人员设置 → 绑定运营者**
3. 选「长期运营者」，输入你的微信号，管理员扫码确认，你在手机上点确认
4. 之后你就能自己重置 AppSecret、改白名单了

**如果一时找不到管理员**，退而求其次：让管理员把 AppSecret 重置后发给你，并帮忙填好 IP 白名单。注意 AppSecret 等同于密码，别在群里明文发。

### 6.2 获取 AppID 与 AppSecret（跟着点）

1. 电脑浏览器打开 https://mp.weixin.qq.com，管理员微信扫码登录
2. 左侧菜单拉到最下：**设置与开发 → 基本配置**
3. 页面中部「开发者ID」区域：
   - **AppID**：直接显示，`wx` 开头，点「复制」存好
   - **AppSecret**：默认隐藏。点右边「重置」（有的显示「生成」）→ 管理员微信扫码 → **立刻复制保存**
4. 保存后关掉窗口就看不到了（只能再重置一次，而重置会让旧的立刻失效）

> **验收点**：你手上有两串字符——`wx` 开头的 AppID，和 32 位的 AppSecret。

### 6.3 access_token 是什么，怎么验证它真的能拿到

你不会"手动去拿令牌"，脚本会自动换。但**开工前建议手动测一次**，能提前暴露 AppSecret 填错、白名单没配对等问题，比等脚本报错好排查。

**Mac / Linux**，终端执行（换成你自己的值）：

```bash
curl "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=你的AppID&secret=你的AppSecret"
```

**Windows PowerShell**：

```powershell
(Invoke-WebRequest "https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=你的AppID&secret=你的AppSecret").Content
```

**也可以直接用浏览器**访问上面那个 URL（把中文替换掉）。

三种结果：

| 返回 | 含义 | 怎么办 |
| --- | --- | --- |
| `{"access_token":"57_xxxx...","expires_in":7200}` | ✅ 成功 | 继续下一步 |
| `{"errcode":40164,"errmsg":"invalid ip 1.2.3.4, not in whitelist"}` | IP 不在白名单 | 去 6.4 加白名单，注意报错里那个 IP 就是你当前的公网 IP |
| `{"errcode":40013,"errmsg":"invalid appid"}` 或 `40001` | AppID / AppSecret 填错 | 回到 6.2 重新复制，注意别多复制空格 |

关于令牌，记住三点：

1. **有效期 2 小时**，过期要重新换。脚本会把它缓存到 `token.json`，快过期才去换，不用你管
2. **不要每次调用都去换**。频繁换取会触发日调用上限，正确做法是"中控"——全局只维护一个令牌
3. **换取令牌这一步才校验 IP 白名单**，后面调数据接口不校验

> **验收点（KR1）**：上面那条请求能返回一个 `access_token` 字符串。这是整个项目最关键的一道门槛，过了它后面基本顺畅。

### 6.4 IP 白名单：11 条注意事项

在**同一页面**（基本配置）往下找「IP白名单」，点「修改」，把你的公网 IP 填进去。

**怎么查自己的公网 IP**：浏览器打开 https://ip.cn 或 https://ip.sb ；命令行 `curl ifconfig.me`；或者直接看 6.3 里 40164 报错里那个 IP——最准。

填的时候注意这些（每一条都是真踩过的坑）：

1. **填公网出口 IP，不是内网 IP**。`192.168.x.x`、`10.x.x.x`、`172.16-31.x.x`、`127.0.0.1` 统统无效
2. **家庭宽带 IP 会变**。重启光猫、运营商回收、隔几天都可能换。变了就报 40164，按提示更新即可
3. **单位网络可能多出口**。有负载均衡的话，每次请求的出口 IP 可能不一样——症状是"明明加了白名单，还是偶尔报错"。解决办法：把报错里出现过的 IP 逐个加进去，或者加一段 CIDR（如 `1.2.3.0/24`）
4. **格式**：一行一个；支持 `1.2.3.4` 和 `1.2.3.4/24`；**不支持** `1.2.3.*` 这种通配符
5. **生效有延迟**：保存后约 1–2 分钟，极少数情况更久。刚改完立刻重试失败，等两分钟再说
6. **修改需要管理员或长期运营者扫码**
7. **改白名单会影响所有用这个 AppSecret 的地方**。如果还有别的系统在用，别删掉原来的条目，只做新增
8. **换网络要重加**：手机热点、出差住酒店、开 VPN——出去的 IP 都变了。**开着 VPN 时出去的是 VPN 的 IP**
9. **线上版服务器不需要加白名单**。因为线上不调微信接口，只做展示（很多人白白加了一堆服务器 IP）
10. **白名单只卡"换 token"这一步**。数据接口本身不校验 IP，所以只要 token 拿到了，后面不会被 IP 拦
11. **填错不会提前警告**。只有真正请求时才会报 40164，所以务必用 6.3 的方式实测一次

> **验收点**：白名单保存 2 分钟后，重跑 6.3 的请求，能拿到 `access_token`。

## 七、本地部署：一步一步

### 7.1 创建 config.json

在 `wechat-dashboard` 目录里，把 `config.example.json` **复制一份**，改名为 `config.json`，然后用记事本 / VS Code / 文本编辑器打开，填入你的信息：

```json
{
  "appid": "wx你的AppID",
  "secret": "你的AppSecret",
  "lookback_days": 7,
  "article_detail_days": 30,
  "upstream": "",
  "upstream_pwd": "你的线上管理密码",
  "upstream_auto_push": true
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `appid` | 6.2 拿到的 AppID |
| `secret` | 6.2 拿到的 AppSecret |
| `lookback_days` | 按"统计日"往回补几天，日常保持 `7` |
| `article_detail_days` | 按"发表日"取多少天内发表的文章，默认 `30` |
| `upstream` | 线上站点地址，**先留空**（第八节再填） |
| `upstream_pwd` | 线上管理密码，先留空 |
| `upstream_auto_push` | 采集后是否自动推线上，先留 `true` |

> **安全**：`config.json` 含密钥，**不要发给任何人、不要提交到 Git**（仓库的 `.gitignore` 已排除它）。

### 7.2 第一次采集

**Mac / Linux**：

```bash
cd ~/wechat-dashboard
python3 sync.py
```

**Windows**（命令提示符）：

```bat
cd /d D:\wechat-dashboard
python sync.py
```

首次建议多回溯一些历史，比如 90 天：

```bash
python3 sync.py --days 90
```

你会看到一串日志：取 token → 逐日采集 → 补标题 → 生成 data.js。**首次可能要跑几分钟，耐心等它自己结束。**

> **验收点（KR2）**：目录下出现了 `store.json` 和 `data.js`，日志里没有红色报错，最后打印的统计里文章数大于 0。

### 7.3 打开看板

**推荐方式**（功能最全）：启动本地服务

```bash
python3 serve.py           # Windows 用 python serve.py
```

终端会打印 `http://127.0.0.1:8000`，浏览器打开它就行。Mac/Linux 也可以直接 `./run.sh`。

**备用方式**：直接双击 `index.html`。能看，但"更新数据""删除"这类需要服务端的功能会提示不可用。

> **验收点（KR3）**：页面顶部 KPI 有数字、趋势图有曲线、文章明细表有行。**如果是空的，说明上一没采到数据，回去看 7.2 的日志。**

### 7.4 认识顶栏的「锁定」按钮

页面默认**锁定（只读）**，任何人打开都只能看。点顶栏最右的「锁定」，输入管理密码才能解锁管理功能。

| 状态 | 能做什么 |
| --- | --- |
| **锁定**（默认） | 看图表和明细；点「更新数据」拉取服务端最新版本；临时调布局（仅本次有效） |
| **已解锁**（输密码后） | 上传数据、回收站、删除/还原、重命名、保存为默认布局、同步到线上 |

- 密码**只存在服务端**，页面源码里没有；解锁后下发 8 小时令牌，关掉页面即失效
- 再点一次「锁定」立即锁回去
- **部署前务必改掉默认密码**：本地版改 `serve.py` 里的 `ACCESS_PWD`，线上版设置环境变量 `ADMIN_PWD`

### 7.5 这一步的完整验收清单

- [ ] `config.json` 填好且能拿到 token（KR1）
- [ ] `sync.py` 跑完无报错，`store.json` / `data.js` 已生成（KR2）
- [ ] 页面 KPI、趋势图、明细表都有数据（KR3）
- [ ] 点「锁定」输密码能解锁，刷新后回到锁定状态

## 八、把数据送到线上端

做到第七节，看板只有你自己电脑能看。这一节把它变成"手机也能开"。

### 8.1 先部署线上版

1. 取 `workbuddy-deployment/` 里的全部文件（`index.html`、`server.py`、`vendor/`）
2. 传到任意能跑 Python 的服务器/云平台
3. 启动：`python3 server.py`（监听 `PORT` 环境变量，缺省 3000，绑定 `0.0.0.0`）
4. **设置管理密码**：`export ADMIN_PWD=你的密码` 后再启动（务必不要用默认占位值）
5. 浏览器打开线上地址，能看到页面（此时还没数据，等本地推过来）

> 线上版**不需要**配 IP 白名单、不需要 AppSecret——它不调微信接口。

### 8.2 让本地采完自动推上去（推荐）

编辑本地的 `config.json`：

```json
{
  "upstream": "https://你的线上地址",
  "upstream_pwd": "你的线上管理密码",
  "upstream_auto_push": true
}
```

之后这些操作**会自动同步，不用你管**：

- 采集成功 → 自动推一次数据
- 上传数据文件 → 自动推一次
- 删除 / 还原 / 重命名 → 自动推回收站和命名状态

顶栏「数据更新时间」后面有个「线上：已同步 / 未同步」标记，**点它可以立即手动同步一次**（需密码）。

### 8.3 另外两种上传方式

**方式二：在线上页面直接上传**

1. 打开线上地址 → 点「锁定」输密码解锁
2. 点「更新数据 → 选择数据文件」
3. 选中本地生成的 `data.js` → 上传

**方式三：直接把 `data.js` 拷到线上目录**（有服务器访问权限时最快），覆盖 `data.js` 即可。

### 8.4 怎么确认同步成功了

1. 看本地顶栏：**「线上：已同步」**
2. 打开线上页面，看数据生成时间是否变成刚才的
3. 本地删一篇文章 → 线上刷新 → 这篇文章也消失了（说明回收站状态同步了）

如果显示「未同步」，鼠标悬停在标记上会看到失败原因。最常见的是**本机暂时访问不了外网**（单位内网限制）——数据已经存在本机，恢复网络后点标记重试即可。每次同步结果都记在 `sync.log` 里。

> **验收点（KR4）**：线上地址能看到和本地一样的数据；本地删一篇，手机上也消失。

### 8.5 线上端的小提醒

- 别人正在看线上页面时，检测到数据更新会在底部弹「数据已更新 → 立即刷新 / 稍后」，由他自己决定何时刷新，不会打断操作
- **微信里直接打开链接可能被拦**（部分平台网关按 UA 拦截微信内置浏览器），用系统浏览器打开
- `upstream_auto_push` 设成 `false` 可关掉自动同步，需要时再手动点

## 九、设置每天自动同步

### Mac / Linux（cron）

```bash
crontab -e
```

加入一行（路径换成你自己的）：

```cron
30 8 * * * cd /Users/你的名字/wechat-dashboard && /usr/bin/python3 sync.py >> sync.log 2>&1
```

### Windows（任务计划程序）

1. 开始菜单搜「任务计划程序」→ 创建基本任务
2. 触发器：每天，时间 8:30
3. 操作：启动程序
   - 程序：`python`（或完整路径 `C:\Python311\python.exe`）
   - 参数：`sync.py`
   - 起始位置：`D:\wechat-dashboard`
4. 勾选「不管用户是否登录都要运行」

### 统信 UOS / 国产 Linux

同 cron，先确认定时服务开着：

```bash
sudo systemctl status cron     # 没开就 sudo systemctl enable --now cron
```

> **验收点（KR5）**：第二天 8:30 之后打开看板，数据日期变成了昨天。**故意关一天电脑**，第三天再看——漏跑的日子应该被自动补上（脚本默认往回补 7 天）。

## 十、排障大全（几乎所有你可能遇到的问题）

### 环境与安装

**Q：`python3 --version` 提示"不是内部或外部命令"？**
Windows 装 Python 时没勾 "Add Python to PATH"。重装一次并勾上，或手动把 Python 目录加进环境变量。

**Q：Windows 上脚本报编码错误（UnicodeDecodeError / gbk）？**
文件路径或目录名带中文/空格。把整个文件夹挪到纯英文路径，如 `D:\wechat-dashboard`。

**Q：端口被占用？**
不用管，服务会自动顺延到 8001–8020 并打印实际地址。想固定就 `./run.sh --port 8080`，或在 `config.json` 里加 `"serve_port": 8080`。

**Q：双击 `index.html` 页面是空的？**
还没采集数据。先跑一次 `python3 sync.py`。另外文件直开模式下部分功能（删除、上传）不可用，请用 `python3 serve.py` 启动服务访问。

### 凭据与权限

**Q：报 `40001` / `40013`？**
AppSecret 填错或 token 失效。核对 `config.json`（注意别多复制空格），脚本会自动重取 token。

**Q：重置 AppSecret 后老系统也挂了？**
重置会让旧值立即失效。如果还有别的系统在用，需要同步更新它们。

**Q：报 `48001`？**
接口未授权。需要公众号**已认证**且具备对应权限。数据接口不受影响的，只是标题可能取不到——可以在明细表标题旁点「命名」手动补。

**Q：我不是管理员，也绑不了运营者？**
只能请管理员代劳：重置 AppSecret 发给你 + 帮忙填 IP 白名单。

### 网络与白名单

**Q：报 `40164 / not in whitelist`？**
IP 不在白名单。脚本会打印你当前的公网 IP，把它加进白名单，等 1–2 分钟生效后重跑。详见 6.4。

**Q：明明加了白名单，还是偶尔报错？**
网络多出口（负载均衡），每次出去的 IP 不同。把报错里出现过的 IP 逐个加进去，或加 CIDR 段。

**Q：换网络 / 重启后不能用了？**
公网 IP 变了。跑一次脚本，按提示把新 IP 加进去。

**Q：开着 VPN 能用吗？**
出去的 IP 是 VPN 的，要加 VPN 的出口 IP。建议采集时关掉 VPN。

### 采集与数据

**Q：为什么没有阅读次数（PV）？**
微信自 2025-11-01 起下线了「阅读次数」，只保留「阅读人数」。历史次数可在公众平台后台「数据统计」里看。

**Q：为什么没有更早的历史数据？**
微信不提供批量导出历史，新接口数据从 **2025-11-01** 起。从现在开始每天采，数据会越攒越多。

**Q：报 `45009 reach max api daily quota limit`？**
接口日调用量到顶（按接口分别算，每天 0 点重置）。脚本会暂停该接口的剩余扫描、不会反复重试，未完成的日期**下次运行自动继续**。后台「接口权限模板」可查实时调用量，认证账号每月有 10 次清零机会。

**Q：报 `47009 / this api is offline`？**
老接口已下线。用本版脚本，别改回旧接口名。

**Q：有些文章标题显示"未获取到标题（xxx）"？**
脚本会通过「已发布消息列表」自动补全，包括 2025-11-01 之前的老文章。如果仍显示：
1. 发布列表接口无权限（48001）→ 只能点「命名」手动补
2. 该文章确实不在发布列表（未发布 / 已删 / 转载）→ 同上手动补
3. 当天被限流 → **下次运行自动补扫**，不用管。多次不行就 `python3 sync.py --full` 强制全量重扫

**Q：删掉的文章，重新采集后又冒出来了？**
不会。删除记录写在 `store.json` 的台账里，生成数据时始终过滤。想找回就去回收站恢复。

**Q：发布日期和实际不符？**
v2 已按「文章日报最早日期」校正。若仍不对，跑一次 `python3 sync.py` 触发重算（日志会打印修正了多少篇）。

### 页面与显示

**Q：图表不显示 / 页面白屏？**
`vendor/` 文件夹没一起拷过去（echarts 和 xlsx 是离线库）。确认目录里有 `vendor/echarts.min.js`。

**Q：提示"未解锁或登录已过期"？**
令牌 8 小时过期，点顶栏「锁定」重新输密码。

**Q：手机上布局乱了？**
窄屏下不提供布局编辑，会用响应式默认布局。想调整请用电脑。

**Q：导出 Excel 中文乱码？**
用 Excel 直接打开；WPS 一般正常。若仍有问题，检查系统区域设置是否为中文。

### 线上与同步

**Q：本机上传的数据，线上看不到？**
看顶栏「线上：」标记是否为「已同步」。显示「未同步」时悬停看原因，点标记可手动重试（需密码）。本机访问不了外网时，数据仍完整保存在本机，联网后重试即可。

**Q：微信里打不开线上链接？**
网关按 UA 拦截了微信内置浏览器，用系统浏览器打开。

**Q：线上页面一直显示旧数据？**
线上页面需要刷新。检测到更新时底部会弹提示，点「立即刷新」即可。

## 十一、备份、迁移与恢复

**要备份的只有两个文件**：

| 文件 | 为什么 |
| --- | --- |
| `config.json` | 含 AppID / AppSecret，丢了要重新重置 |
| `store.json` | 全部历史累积数据，丢了要从头再采 |

其余（`data.js`、`token.json`、`sync.log`）都能重新生成。

**换机迁移**：

1. 把整个文件夹连同上面两个文件拷到新机器
2. 装好 Python
3. **在新机器上重新配 IP 白名单**（IP 变了，这一步最容易忘）
4. 跑一次 `sync.py` 验证

**恢复误删**：去回收站还原即可，删除记录一直在 `store.json` 里，不会因为重新采集而丢失。

## 十二、进阶：v2 的五个关键改造（含代码）

如果你只想用，前面十一节就够了。想改代码的看这里。

### 改造 1：密码挪到服务端，前端不留密码

v1 最严重的问题。v2 由服务端比对密码，通过则下发随机令牌，写操作只认令牌。

```python
# server.py
import secrets, threading, time

PWD = os.environ.get("ADMIN_PWD", "change-me")   # 部署前必须改
TOKEN_TTL = 8 * 60 * 60
_TOKENS = {}
_TOKEN_LOCK = threading.Lock()

def issue_token():
    with _TOKEN_LOCK:
        now = time.time()
        for k in [k for k, exp in _TOKENS.items() if exp < now]:
            _TOKENS.pop(k, None)                 # 顺手清理过期令牌
        tok = secrets.token_hex(24)
        _TOKENS[tok] = now + TOKEN_TTL
        return tok

def check_token(tok):
    if not tok:
        return False
    with _TOKEN_LOCK:
        exp = _TOKENS.get(tok)
        return bool(exp and exp > time.time())
```

```python
if path == "/api/auth":
    if body.get("pwd") != PWD:
        time.sleep(0.4)                          # 轻微延迟，增加暴力尝试成本
        self._json({"ok": False, "message": "密码不正确"}, code=403)
        return
    tok = issue_token()
    self._json({"ok": True, "token": tok, "expires_in": TOKEN_TTL})
    return

if path in ("/api/state", "/api/data"):
    if not check_token(body.get("token")):
        self._json({"ok": False, "message": "未解锁或登录已过期，请重新验证"}, code=403)
        return
```

### 改造 2：上行同步改用令牌，失效自动重试

```python
# serve.py
_UP_TOKEN_CACHE = {"key": "", "token": "", "exp": 0}

def _upstream_token(conf):
    key = conf["url"] + "|" + conf["pwd"]
    now = time.time()
    if _UP_TOKEN_CACHE["key"] == key and _UP_TOKEN_CACHE["exp"] > now + 60:
        return _UP_TOKEN_CACHE["token"]                  # 缓存未过期直接复用
    j = _http_post_json(conf["url"] + "/api/auth", {"pwd": conf["pwd"]}, timeout=30)
    if j.get("ok") and j.get("token"):
        ttl = int(j.get("expires_in") or 28800)
        _UP_TOKEN_CACHE.update({"key": key, "token": j["token"], "exp": now + ttl})
        return j["token"]
    raise RuntimeError("线上密码验证失败：%s" % (j.get("message") or "未知原因"))

def _up_post(conf, api_path, payload, timeout=30):
    """上行请求：凭临时 token 调线上接口；token 失效自动刷新重试一次。"""
    payload = dict(payload)
    payload.pop("pwd", None)                             # 绝不明文带密码
    payload["token"] = _upstream_token(conf)
    j = _http_post_json(conf["url"] + api_path, payload, timeout=timeout)
    if not j.get("ok"):
        msg = str(j.get("message") or "")
        if "过期" in msg or "未解锁" in msg:
            _UP_TOKEN_CACHE["exp"] = 0                   # 强制刷新
            payload["token"] = _upstream_token(conf)
            j = _http_post_json(conf["url"] + api_path, payload, timeout=timeout)
    return j
```

### 改造 3：周发文数 / 周阅读数，两种口径

```javascript
function weekRange() {
  if (!dates.length) return null;
  var base = dates[dates.length - 1];              // 最新数据日
  var b = new Date(base + 'T00:00:00');
  var s = new Date(base + 'T00:00:00');
  if (weekMode === 'mon') {
    var wd = b.getDay();                           // 0 = 周日
    s.setDate(b.getDate() - (wd === 0 ? 6 : wd - 1));   // 回退到本周一
  } else {
    s.setDate(b.getDate() - 6);                    // 含今天往前 7 天
  }
  return { start: isoDate(s), end: base };
}
```

### 改造 4：发布日期校正（修掉错标的老 bug）

v1 用 `setdefault` 写入首发日期，**只写一次、永不更新**，补扫历史后老文章会被错标成最近几天。

```python
# sync.py
def fix_first_dates(store):
    """按「文章日报」重算所有文章的首发日期。

    老版本（setdefault）把首次写入的日期固定下来，补扫历史数据后
    也不会更新。权威口径：文章在某统计日有数据 → 取最早的那天为发布日期。
    """
    fixed = 0
    daily = store.get("article_daily", {}) or {}
    arts = store.setdefault("articles", {})
    for d in sorted(daily.keys()):
        for m in daily[d]:
            a = arts.get(m)
            if a is None:
                continue
            if not a.get("first_date") or d < a["first_date"]:
                a["first_date"] = d
                fixed += 1
    if fixed:
        log("  发布日期校正：%d 篇按文章日报取最早日期修正" % fixed)
```

### 改造 5：趋势图加比例尺

```javascript
grid: { left: 48, right: 14, top: 30, bottom: 58 },   // 底部留 58px 给滑块
dataZoom: [
  { type: 'inside', xAxisIndex: 0, zoomOnMouseWheel: true,
    moveOnMouseMove: true, moveOnMouseWheel: false },
  { type: 'slider', xAxisIndex: 0, height: 20, bottom: 6,
    fillerColor: 'rgba(37,99,235,.12)',
    handleStyle: { color: '#fff', borderColor: '#2563eb' },
    labelFormatter: function (v) { return String(v).slice(5); } }  // 只显示 MM-DD
],
```

---

## 写在最后

v2 的代码量比 v1 翻了一倍多，但**"零成本"这条底线没破**：依然不买服务器、不装数据库、不依赖云服务，Python 端依然只用标准库。

回头看，最值钱的经验不是某段代码，而是这三条：

1. **动手前先把平台边界搞清楚**——哪些是接口能力问题，哪些是网络环境问题。分清楚这个，就不会在"为什么不行"上反复打转。
2. **凡是和身份有关的判断，一律放服务端**。前端做权限校验本质上等于没做——v1 的密码裸奔就是这么来的。
3. **给每一步定一个可验证的验收点**。本文那张 OKR 表不是形式主义：它让你在任何一步卡住时，都能立刻知道是"哪一步没过"，而不是对着一个不工作的系统发呆。

源码在 [GitHub](https://github.com/Yellowsoy/gzh-data-dashboard)（MIT），有问题欢迎提 Issue。希望这篇能帮你少走几段弯路。
