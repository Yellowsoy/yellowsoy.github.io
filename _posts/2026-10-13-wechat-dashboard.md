---
layout:     post
title:      "公众号数据看板：从零开始，小白也能搭起来"
subtitle:   " \"零成本、不装数据库；附全部完整源码，复制即可运行\""
date:       2026-10-13 12:00:00
permalink:  /2026/10/13/wechat-dashboard/
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

> 想要一个能**自动更新**的公众号数据看板：顶部一排 KPI、阅读分享趋势、粉丝增长、来源构成、文章榜单和明细表。
> 而要求很朴素：**不花钱、不买服务器、不装数据库、不依赖云服务**。
>
> 这篇文章从"微信后台该点哪里"讲起，一直到"每天自动更新、手机上也能看"，**每一步都给出验收标准**，
> 并且**附上全部完整源码**（本地版 6 个文件 + 线上版 1 个文件），照着复制就能跑通。
>
> **源码仓库（建议先收藏，本文所有代码均出自这里）**
> **https://github.com/Yellowsoy/gzh-data-dashboard**
> 本文全部代码与该仓库 `main` 分支**逐字一致**；仓库 18 个文件中每一份在本文的对应位置，见文末「附录 A」的对照表。

* * *

## 目录

01. 先定目标：做到什么程度算成功（验收清单）
02. 先搞懂几个名词（小白必看）
03. 方案原理：本地采集 + 线上展示
04. 开工前：资质、Python、源码
05. 微信侧配置：权限、AppSecret、access_token、IP 白名单
06. 本地部署（**完整源码 6 个文件**）
07. 线上部署（**完整源码**）：把数据送到线上端
08. 设置每天自动同步
09. 排障大全（几乎所有你可能遇到的问题）
10. 备份、迁移与恢复
11. 附录 A：仓库其余 6 个文件的完整内容 + 全部文件对照表
12. 附录 B：两个离线库文件（vendor/）的获取方式

* * *

## 一、先定目标：做到什么程度算成功（验收清单）

先说清终点在哪，免得做到一半不知道对不对。

**目标（O）**：在自己的电脑上跑起一个**会自动更新**的公众号数据看板，并且手机、其他电脑打开链接也能看到同一份数据。

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
- [ ] 有 30 分钟不被打断的时间

> 第一项卡死最多人：**数据统计接口只对已认证账号开放**。没认证的话，后面全部白做，先去认证。

## 二、先搞懂几个名词（小白必看）

名词 | 通俗解释
--- | ---
**AppID（开发者ID）** | 公众号的"身份证号"，`wx` 开头，公开可查，不保密
**AppSecret（开发者密码）** | 公众号的"密码"，**绝不能泄露**，用它换 access_token
**access_token（访问令牌）** | 拿 AppID + AppSecret 换来的"临时通行证"，**有效期 7200 秒（2 小时）**，调数据接口必须带它
**IP 白名单** | 微信只允许白名单里的 IP 去**换取 token**，不在名单里直接拒绝
**公网 IP** | 你家宽带在互联网上的"门牌号"，和内网 `192.168.x.x` 不是一回事
**datacube** | 微信官方数据统计接口组，取阅读、分享、粉丝等
**UV / PV** | UV = 阅读**人数**（同一个人看多次算 1）；PV = 阅读**次数**。**微信自 2025-11-01 起只保留 UV，取消了 PV**

## 三、方案原理：本地采集 + 线上展示

先把三条看似能偷懒的路堵死，你才会死心塌地按本文做：

1. **纯静态 HTML 直接调微信接口？不行**——浏览器跨域被拦（CORS）；AppSecret 写在前端等于公开；而且页面跑在每个访客自己的网络里，IP 不固定，没法加白名单。
2. **Cloudflare Workers 免费版？不行**——免费版出口 IP 是动态共享地址，每天在变，无法加白名单。
3. **GitHub Actions 代跑？也不行**——同样是一大段动态 IP 段。

### 那"线上版"是怎么回事？

这是最关键的设计：**线上版不采集，只负责"展示 + 状态同步"。**

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

## 四、开工前：资质、Python、源码

### 1. 拿到源码

本文会把全部源码贴出来（第六、七节），**直接复制保存即可**，不需要下载。

想省事也可以直接取仓库：

- 打开 https://github.com/Yellowsoy/gzh-data-dashboard → 绿色按钮 **Code → Download ZIP**
- 或命令行：`git clone https://github.com/Yellowsoy/gzh-data-dashboard.git`

仓库里 `local-deployment/` 是本地版，`workbuddy-deployment/` 是线上版，**两版共用同一个 `index.html`**。

### 2. 确认 Python 装好了

打开终端（Windows 用「命令提示符」或 PowerShell；Mac/Linux 用「终端」）：

```bash
python3 --version      # Mac / Linux
python --version       # Windows
```

看到 `Python 3.7.x` 以上就行。没装的话：

- **Windows**：https://www.python.org/downloads/ 下载，**安装时务必勾选 "Add Python to PATH"**
- **macOS**：`brew install python3`，或官网下载
- **Linux / 统信 UOS**：一般自带；没有就 `sudo apt install python3`

> **验收点**：`python3 --version` 能打印版本号，而不是"不是内部或外部命令"。

### 3. 建一个固定目录

新建文件夹，路径**不要带中文和空格**，之后也别再挪动：

- Windows：`D:\wechat-dashboard`
- Mac / Linux：`~/wechat-dashboard`

后面第六节的 6 个文件全部放在这里。

## 五、微信侧配置：权限、AppSecret、access_token、IP 白名单

这一节最容易卡住，慢慢来。

### 5.1 关于"成为开发者"：你需要什么权限

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

**如果一时找不到管理员**：让管理员把 AppSecret 重置后发给你，并帮忙填好 IP 白名单。AppSecret 等同于密码，别在群里明文发。

### 5.2 获取 AppID 与 AppSecret（跟着点）

1. 电脑浏览器打开 https://mp.weixin.qq.com，管理员微信扫码登录
2. 左侧菜单拉到最下：**设置与开发 → 基本配置**
3. 页面中部「开发者ID」区域：
   - **AppID**：直接显示，`wx` 开头，点「复制」存好
   - **AppSecret**：默认隐藏。点右边「重置」（有的显示「生成」）→ 管理员微信扫码 → **立刻复制保存**
4. 保存后关掉窗口就看不到了（只能再重置一次，而重置会让旧的立刻失效）

> **验收点**：你手上有两串字符——`wx` 开头的 AppID，和 32 位的 AppSecret。

### 5.3 access_token 是什么，怎么验证它真的能拿到

你不会"手动去拿令牌"，脚本会自动换。但**开工前建议手动测一次**，能提前暴露 AppSecret 填错、白名单没配对等问题，比等脚本报错好排查。

**Mac / Linux**：

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
| `{"errcode":40164,"errmsg":"invalid ip 1.2.3.4, not in whitelist"}` | IP 不在白名单 | 去 5.4 加白名单，报错里那个 IP 就是你当前的公网 IP |
| `{"errcode":40013,"errmsg":"invalid appid"}` 或 `40001` | AppID / AppSecret 填错 | 回到 5.2 重新复制，注意别多复制空格 |

关于令牌，记住三点：

1. **有效期 2 小时**，过期要重新换。脚本会缓存到 `token.json`，快过期才去换，不用你管
2. **不要每次调用都去换**。频繁换取会触发日调用上限，正确做法是"中控"——全局只维护一个令牌
3. **换取令牌这一步才校验 IP 白名单**，后面调数据接口不校验

> **验收点（KR1）**：上面那条请求能返回一个 `access_token` 字符串。这是整个项目最关键的一道门槛，过了它后面基本顺畅。

### 5.4 IP 白名单：11 条注意事项

在**同一页面**（基本配置）往下找「IP白名单」，点「修改」，把你的公网 IP 填进去。

**怎么查自己的公网 IP**：浏览器打开 https://ip.cn 或 https://ip.sb ；命令行 `curl ifconfig.me`；或者直接看 5.3 里 40164 报错里那个 IP——最准。

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
11. **填错不会提前警告**。只有真正请求时才会报 40164，所以务必用 5.3 的方式实测一次

> **验收点**：白名单保存 2 分钟后，重跑 5.3 的请求，能拿到 `access_token`。

## 六、本地部署（完整源码 6 个文件）

下面 6 个文件**全部放在同一个目录**（第四节建好的那个），文件名必须一字不差。

文件 | 作用 | 需要你改吗
--- | --- | ---
`config.json` | 配置：AppID / AppSecret / 同步设置 | ✅ **必须填**
`sync.py` | 采集脚本（核心） | 不用改
`serve.py` | 本地服务：提供接口、调度采集、同步线上 | 不用改（建议改默认密码）
`index.html` | 看板页面 | 不用改
`run.sh` | 一键启动（Mac/Linux） | 不用改
`stop.sh` | 一键停止（Mac/Linux） | 不用改

> Windows 用户不需要 `run.sh` / `stop.sh`，直接执行 `python serve.py` 即可。

### 文件 1：`config.json`

**这是唯一需要你改动的文件**。复制下面内容，把 AppID、AppSecret 换成你自己的：

```json
{
  "_说明": "复制本文件为 config.json 并填入你自己的公众号凭据；config.json 含密钥，已被 .gitignore 排除，切勿提交或外发",
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

字段 | 说明
--- | ---
`appid` | 5.2 拿到的 AppID
`secret` | 5.2 拿到的 AppSecret
`lookback_days` | 按"统计日"往回补几天，日常保持 `7`
`article_detail_days` | 按"发表日"取多少天内发表的文章，默认 `30`
`upstream` | 线上站点地址，**先留空**（第七节再填）
`upstream_pwd` | 线上管理密码，先留空
`upstream_auto_push` | 采集后是否自动推线上，先留 `true`

> **安全**：`config.json` 含密钥，**不要发给任何人、不要提交到 Git**。

### 文件 2：`run.sh`（一键启动，Mac / Linux）

```bash
#!/usr/bin/env bash
# 微信公众号数据看板 - 一键启动（统信 UOS / Linux）
# 用法：
#   ./run.sh            启动看板服务（默认 http://127.0.0.1:8000），并尝试打开浏览器
#   ./run.sh --port 8080  指定端口
#   ./run.sh --no-browser 不自动打开浏览器
#   ./run.sh --sync-only  只跑一次采集（不启动服务），等价于旧版行为
#
# 页面上的「更新数据」按钮会在服务端子进程运行 sync.py，无需再手动执行本脚本。

set -u
cd "$(dirname "$0")"

if ! command -v python3 >/dev/null 2>&1; then
  echo "未检测到 python3。统信 UOS 一般自带；若没有，请在「应用商店」安装或执行：sudo apt install python3"
  exit 1
fi

PYVER=$(python3 -c 'import sys;print("%d.%d"%sys.version_info[:2])')
echo "使用 Python 版本：$PYVER"

if [ "${1:-}" = "--sync-only" ]; then
  shift
  python3 sync.py "$@"
  exit $?
fi

python3 serve.py "$@"
```

### 文件 3：`stop.sh`（一键停止，Mac / Linux）

```bash
#!/usr/bin/env bash
# 微信公众号数据看板 - 停止服务（统信 UOS / Linux）
# 优先级：1) serve.pid 精确停止  2) 兜底按 /proc 精确匹配 argv（python + serve.py + cwd 在本目录）
# 用法：./stop.sh
set -u
cd "$(dirname "$0")"

stopped=""

# 方式一：PID 文件（serve.py 启动时写入，退出时自动清理）
if [ -f serve.pid ]; then
  PID=$(cat serve.pid 2>/dev/null)
  if [ -n "${PID:-}" ] && kill -0 "$PID" 2>/dev/null; then
    kill "$PID" 2>/dev/null
    for _ in 1 2 3 4 5 6 7 8 9 10; do
      kill -0 "$PID" 2>/dev/null || break
      sleep 0.3
    done
    if kill -0 "$PID" 2>/dev/null; then
      kill -9 "$PID" 2>/dev/null
    fi
    stopped="$PID"
  fi
  rm -f serve.pid
fi

# 方式二：兜底扫描（PID 文件丢失时）
# 精确匹配 argv：解释器为 python 且某一参数恰为 serve.py（或以其结尾的绝对路径），
# 且进程 cwd 就是本目录。避免误伤命令行里只提到 serve.py 的进程
# （例如 bash -c "... serve.py ..."、vim serve.py、grep serve.py 等）。
if [ -z "$stopped" ]; then
  HERE=$(pwd)
  for p in /proc/[0-9]*; do
    pid=${p#/proc/}
    [ "$pid" = "$$" ] && continue
    [ "$pid" = "$PPID" ] && continue
    [ -r "$p/cmdline" ] || continue
    # 以 NUL 分隔逐项读取 argv
    argv0=""; hit=0
    while IFS= read -r -d '' item; do
      if [ -z "$argv0" ]; then argv0=$item; fi
      case "$item" in
        serve.py|*/serve.py) hit=1 ;;
      esac
    done < "$p/cmdline" 2>/dev/null
    [ "$hit" = "1" ] || continue
    # argv[0] 必须是 python 解释器
    case "$argv0" in
      *python*) ;;
      *) continue ;;
    esac
    # cwd 必须在本目录
    cwd=$(readlink -f "$p/cwd" 2>/dev/null)
    [ "$cwd" = "$HERE" ] || continue
    kill "$pid" 2>/dev/null && stopped="$stopped $pid"
  done
  # 等 3 秒让优雅退出（atexit 清理 PID 文件）完成
  for _ in 1 2 3 4 5 6 7 8 9 10; do
    alive=0
    for pid in $stopped; do
      kill -0 "$pid" 2>/dev/null && alive=1
    done
    [ "$alive" = "0" ] && break
    sleep 0.3
  done
fi

if [ -n "$stopped" ]; then
  echo "看板服务已停止（PID:$stopped）"
else
  echo "未发现运行中的看板服务"
fi
```

### 文件 4：`sync.py`（采集脚本，核心）

> 这是整个项目最关键的文件，负责换令牌、调微信接口、合并数据、生成 `data.js`。**原样复制即可，不用改。**

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
微信公众号数据看板 - 本地采集脚本（方案A，新接口版）
- 仅使用 Python 标准库，无需 pip 安装任何依赖
- 自动获取并缓存 access_token（中控，有效期 7200 秒）
- 新口径：仅「阅读人数 UV」，不再提供「阅读次数 PV」（2025-11-01 起）
- 数据来源：
  · getarticletotaldetail  按发表日取标题/链接 + 累计轨迹（还原每日增量）
  · getarticleread / share  按统计日取每篇文章阅读(含来源)/分享
  · getbizsummary           按统计日取账号级汇总（点赞/在看/留言/收藏等）
  · getusersummary / cumulate  粉丝增减 / 累计
- 累积合并到 store.json，生成 data.js 供 index.html 直接读取

标题采集策略（2026-10 修复，v4）：
- 每次运行都扫描最近 article_detail_days 天的「发表日」（拿新文章标题，新的日期优先）；
- 另有一个逐日台账 title_scan_dates，记录每个发表日的扫描结果：
  成功的日期不会重复扫；失败的日期（限流/系统繁忙/网络抖动）下次自动补扫，
  不再使用"只跑一次"的 full_scan_done 标记，避免标题永久缺失；
- 2025-11-01 之前发表的老文章：datacube 不返回其标题（数据从 2025-11-01 起），
  但它们至今仍会被 getarticleread 返回。v3 起用「获取已发布的消息列表」
  （freepublish/batchget，内容接口、无日期限制）补全这些老文章的标题：
  该接口返回的 news_item[].url 中含 mid（图文消息id）与 idx（篇目序号），
  官方约定 msgid = 图文消息id_index，因此 msgid = mid + "_" + idx 可直接对上；
  若 url 是短链（不带 mid/idx），则跟随 302 跳转拿到完整 URL 再解析；
- 45009（日调用量上限，按接口分别计算）：立即停止该接口的剩余扫描（当天重试
  无意义），已扫日期不重复、未扫日期明天自动继续；标题不受影响（发布列表会
  补全），累计轨迹数据会在随后几天自动补齐。也可以在公众平台后台
  「接口权限模板」查看实时调用量，或使用清零操作（每月 10 次）当天跑完；
- 以上接口都拿不到时，看板中显示"未获取到标题"，可点「命名」手动补上（最后兜底）。

删除与命名持久化（2026-10，v5）：
- 文章删除为「软删除」：store.json 的 deleted 台账记录删除时的指标快照，
  data.js 输出时过滤这些文章（重新采集不会让它们复活），回收站中可恢复
  （瞬间还原）或彻底删除（从 store 中物理移除）；
- 手动命名可存到服务端 store.json 的 manual_titles（serve.py 提供接口），
  浏览器 localStorage 作为 file:// 直开时的回退；
- serve.py 提供本地服务：静态看板 + 手动更新（跑本脚本）+ 删除/恢复/彻底删除/
  命名等 API，run.sh 一键启动。

适用：统信 UOS / 任意 Debian 系 Linux（Python 3.6+）
"""

import os
import sys
import re
import json
import time
from datetime import datetime, timedelta

try:
    from urllib.request import Request, urlopen
    from urllib.parse import urlencode, urlparse, parse_qs
    from urllib.error import URLError, HTTPError
except ImportError:
    print("当前 Python 版本过低，请使用 Python 3.6 及以上版本。")
    sys.exit(1)

# ----------------------------------------------------------------------
# 路径
# ----------------------------------------------------------------------
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(BASE_DIR, "config.json")
TOKEN_PATH = os.path.join(BASE_DIR, "token.json")
STORE_PATH = os.path.join(BASE_DIR, "store.json")
DATAJS_PATH = os.path.join(BASE_DIR, "data.js")

API_BASE = os.environ.get("WECHAT_API_BASE", "https://api.weixin.qq.com")

# 按「统计日」逐日采集的接口
DAILY_ENDPOINTS = {
    "article_read": "/datacube/getarticleread",    # 每篇文章当日阅读（含来源）
    "article_share": "/datacube/getarticleshare",  # 每篇文章当日分享
    "biz_summary": "/datacube/getbizsummary",      # 账号级当日汇总
    "user_summary": "/datacube/getusersummary",    # 粉丝增减
    "user_cumulate": "/datacube/getusercumulate",  # 累计粉丝
}

# 按「发表日」采集的接口（含标题/链接与累计轨迹）
ARTICLE_DETAIL_KEY = "article_total_detail"
ARTICLE_DETAIL_ENDPOINT = "/datacube/getarticletotaldetail"

# 获取已发布的消息列表（内容接口，无日期限制）：补全老文章标题的关键来源
PUBLISH_LIST_ENDPOINT = "/cgi-bin/freepublish/batchget"
PUBLISH_PAGE = 20          # 官方分页上限
PUBLISH_MAX_PAGES = 500    # 安全上限（约 1 万条消息）

# 累计口径中可做“每日增量”的指标
ADD_METRICS = [
    "read_user", "share_user", "zaikan_user", "like_user",
    "comment_count", "collection_user", "read_subscribe_user",
]

# 新接口数据起点（早于此日期无数据，且接口对之前日期返回 61501）
EARLIEST_DATE = "2025-11-01"

# 采集时值得重试的“瞬时错误”（系统繁忙 / 频率限制，稍后通常会自己恢复）
TRANSIENT_ERRCODES = (-1, 45011)

# 日调用量上限（官方：reach max api daily quota limit）。
# 配额按接口分别计算、每天 0 点重置；当天重试无意义，应停扫等明天，
# 未完成的日期由 title_scan_dates 台账在下次运行自动续上。
DAILY_QUOTA_ERRCODE = 45009


# 工具
def log(msg):
    print("[%s] %s" % (datetime.now().strftime("%H:%M:%S"), msg))


def load_json(path, default):
    if not os.path.exists(path):
        return default
    try:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return default


def save_json(path, data):
    tmp = path + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    os.replace(tmp, path)


def http_request(url, body=None, timeout=15):
    data = None
    headers = {"Accept": "application/json", "User-Agent": "wechat-dashboard/1.0"}
    if body is not None:
        data = json.dumps(body).encode("utf-8")
        headers["Content-Type"] = "application/json"
    req = Request(url, data=data, headers=headers,
                  method="POST" if body is not None else "GET")

    last_err = None
    for attempt in range(3):
        try:
            with urlopen(req, timeout=timeout) as resp:
                return json.loads(resp.read().decode("utf-8"))
        except HTTPError as e:
            try:
                return json.loads(e.read().decode("utf-8"))
            except Exception:
                last_err = "HTTP %s: %s" % (e.code, e.reason)
        except URLError as e:
            last_err = "网络错误: %s" % e.reason
        except Exception as e:
            last_err = "请求异常: %s" % e
        time.sleep(1.5 * (attempt + 1))
    return {"errcode": -99, "errmsg": last_err or "请求失败"}


# ----------------------------------------------------------------------
# access_token 中控
# ----------------------------------------------------------------------
def fetch_new_token(appid, secret):
    url = API_BASE + "/cgi-bin/token?" + urlencode({
        "grant_type": "client_credential",
        "appid": appid,
        "secret": secret,
    })
    result = http_request(url)
    if result.get("errcode") == 40164 or "not in whitelist" in str(result.get("errmsg", "")):
        msg = result.get("errmsg", "")
        mm = re.search(r"invalid ip\s+([0-9.]+)", msg)
        ip = mm.group(1) if mm else ""
        print("\n================ IP 白名单错误 ================")
        print("微信拒绝了取 token 的请求，因为本机出口 IP 不在白名单。")
        if ip:
            print("本机当前出口公网 IP：%s" % ip)
        print("请登录「微信公众平台 → 设置与开发 → 基本配置 → IP 白名单」")
        print("把上面的 IP 加进去（支持单 IP 或 8.8.8.8/24 这种 IP 段，不支持 8.8.8.* 写法）。")
        print("保存（约 1–2 分钟生效）后重新运行本脚本。")
        print("==============================================\n")
        sys.exit(2)
    if result.get("access_token"):
        token_info = {
            "access_token": result["access_token"],
            "expires_at": int(time.time()) + int(result.get("expires_in", 7200)) - 300,
        }
        save_json(TOKEN_PATH, token_info)
        return token_info["access_token"]
    errcode = result.get("errcode")
    errmsg = result.get("errmsg")
    if errcode in (40013, 40125):
        print("AppID 或 AppSecret 不正确，请核对 config.json。（错误码 %s）" % errcode)
    else:
        print("获取 access_token 失败：%s %s" % (errcode, errmsg))
    sys.exit(2)


def get_access_token(appid, secret, force=False):
    token_info = None if force else load_json(TOKEN_PATH, None)
    if token_info and token_info.get("access_token") and \
            int(time.time()) < int(token_info.get("expires_at", 0)):
        return token_info["access_token"]
    log("access_token 已失效或不存在，重新获取 …")
    return fetch_new_token(appid, secret)


# ----------------------------------------------------------------------
# 数据结构
# ----------------------------------------------------------------------
def new_store():
    return {
        "generated_at": "",
        # 发表日期 -> "ok" 或 "err:<错误码>"：标题扫描台账。
        # 已成功的日期不再重复扫描；失败的日期下次自动补扫（自愈）。
        "title_scan_dates": {},
        # getarticletotaldetail 返回过 48001（无接口权限）时为 true，
        # 此时跳过逐日补扫（重试无意义），加 --full 可强制再试。
        "title_api_unauthorized": False,
        # freepublish/batchget（发布列表）最近一次运行结果：
        # "" | "ok" | "unauthorized"(48001) | "error"
        "publish_list_status": "",
        # 软删除台账（v5）：msgid -> {title, pubdate, deleted_at, read_user, ...}
        # 删除只影响看板展示（data.js 输出时过滤），store 内数据保留，
        # 因此「恢复」可瞬间还原；重新采集不会让已删除文章复活。
        "deleted": {},
        # 服务端持久化的手动命名（v5）：msgid -> title。
        # 浏览器不可用时（file:// 直开）仍回退 localStorage。
        "manual_titles": {},
        "articles": {},        # msgid -> {title, content_url, first_date}
        "article_total": {},   # msgid -> 最新累计快照
        "article_daily": {},   # stat_date -> msgid -> 当日指标
        "biz_daily": {},       # stat_date -> 账号级当日指标
        "user_daily": {},      # stat_date -> {new_user, cancel_user, cumulate_user}
    }


def ensure(d, key):
    return d.setdefault(key, {})


def upgrade_store(store):
    """兼容旧版 store.json：把 full_scan_done 迁移为逐日台账。"""
    if not isinstance(store, dict):
        return new_store()
    base = new_store()
    for k, v in base.items():
        store.setdefault(k, v)
    if "full_scan_done" in store:
        # 旧版只做了“一次性全扫”：迁移时视为全部未登记，
        # 下次运行会自动补扫一遍（代价约几分钟，只为把标题补齐）。
        del store["full_scan_done"]
    if not isinstance(store.get("title_scan_dates"), dict):
        store["title_scan_dates"] = {}
    if not isinstance(store.get("deleted"), dict):
        store["deleted"] = {}
    if not isinstance(store.get("manual_titles"), dict):
        store["manual_titles"] = {}
    # v5 起下线「合集」：旧 store 里的合集数据直接丢弃
    store.pop("collections", None)
    return store


# ----------------------------------------------------------------------
# 合并：按统计日
# ----------------------------------------------------------------------
def merge_article_read(store, rows):
    for r in rows:
        d, m = r.get("ref_date"), r.get("msgid")
        if not d or not m:
            continue
        a = ensure(store["articles"], m)
        # 发布日期取最早出现的那天（老版本 setdefault 一旦写入不再更新，
        # 会导致补扫历史后仍错标为最近日期，这里统一按最小日期修正）
        if not a.get("first_date") or d < a["first_date"]:
            a["first_date"] = d
        detail = r.get("detail", {}) or {}
        day = ensure(ensure(store["article_daily"], d), m)
        if "read_user" in detail:
            day["read_user"] = detail["read_user"]
        src = {}
        for s in detail.get("read_user_source", []) or []:
            scene = s.get("scene_desc")
            if scene and scene != "全部":
                src[scene] = s.get("user_count", 0)
        if src:
            day["read_source"] = src


def merge_article_share(store, rows):
    for r in rows:
        d, m = r.get("ref_date"), r.get("msgid")
        if not d or not m:
            continue
        detail = r.get("detail", {}) or {}
        if "share_user" in detail:
            ensure(ensure(store["article_daily"], d), m)["share_user"] = detail["share_user"]


def merge_biz_summary(store, rows):
    for r in rows:
        d = r.get("ref_date")
        if not d:
            continue
        detail = r.get("detail", {}) or {}
        day = ensure(store["biz_daily"], d)
        for k in ("read_user", "share_user", "zaikan_user", "like_user",
                  "comment_count", "collection_user", "redirect_ori_page_user",
                  "send_page_count"):
            if k in detail:
                day[k] = detail[k]
        src = {}
        for s in detail.get("read_user_source", []) or []:
            scene = s.get("scene_desc")
            if scene and scene != "全部":
                src[scene] = s.get("user_count", 0)
        if src:
            day["read_source"] = src


def merge_user_summary(store, rows):
    agg = {}
    for r in rows:
        d = r.get("ref_date")
        if not d:
            continue
        a = agg.setdefault(d, {"new_user": 0, "cancel_user": 0})
        a["new_user"] += int(r.get("new_user", 0) or 0)
        a["cancel_user"] += int(r.get("cancel_user", 0) or 0)
    for d, val in agg.items():
        day = ensure(store["user_daily"], d)
        day["new_user"] = val["new_user"]
        day["cancel_user"] = val["cancel_user"]


def merge_user_cumulate(store, rows):
    for r in rows:
        d = r.get("ref_date")
        if d and "cumulate_user" in r:
            ensure(store["user_daily"], d)["cumulate_user"] = r["cumulate_user"]


# ----------------------------------------------------------------------
# 合并：按发表日（getarticletotaldetail）
# ----------------------------------------------------------------------
def merge_article_total_detail(store, rows):
    for r in rows:
        pub, m = r.get("ref_date"), r.get("msgid")
        if not pub or not m:
            continue
        art = ensure(store["articles"], m)
        if r.get("title"):
            art["title"] = r["title"]
        if r.get("content_url"):
            art["content_url"] = r["content_url"]
        if not art.get("first_date") or pub < art["first_date"]:
            art["first_date"] = pub

        details = sorted(r.get("detail_list", []) or [],
                         key=lambda x: x.get("stat_date", ""))
        if not details:
            continue

        prev = {k: 0 for k in ADD_METRICS}
        prev_src = {}
        for dl in details:
            sd = dl.get("stat_date")
            if not sd:
                continue
            day = ensure(ensure(store["article_daily"], sd), m)
            for k in ADD_METRICS:
                if k in dl:
                    cur = int(dl[k] or 0)
                    inc = cur - prev[k]
                    if inc < 0:
                        inc = 0
                    day[k] = inc
                    prev[k] = cur
            cur_src = {}
            for s in dl.get("read_user_source", []) or []:
                scene = s.get("scene_desc")
                if scene and scene != "全部":
                    cur_src[scene] = int(s.get("user_count", 0) or 0)
            inc_src = {}
            for scene, cur in cur_src.items():
                inc = cur - prev_src.get(scene, 0)
                if inc < 0:
                    inc = 0
                if inc:
                    inc_src[scene] = inc
            if inc_src:
                day["read_source"] = inc_src
            prev_src = cur_src

        # 最新累计快照
        last = details[-1]
        snap = ensure(store["article_total"], m)
        snap["stat_date"] = last.get("stat_date")
        for k in ("read_user", "share_user", "zaikan_user", "like_user",
                  "comment_count", "collection_user", "read_subscribe_user",
                  "read_finish_rate", "read_avg_activetime"):
            if k in last:
                snap[k] = last[k]
        src = {}
        for s in last.get("read_user_source", []) or []:
            scene = s.get("scene_desc")
            if scene and scene != "全部":
                src[scene] = int(s.get("user_count", 0) or 0)
        if src:
            snap["read_source"] = src


DAILY_MERGERS = {
    "article_read": merge_article_read,
    "article_share": merge_article_share,
    "biz_summary": merge_biz_summary,
    "user_summary": merge_user_summary,
    "user_cumulate": merge_user_cumulate,
}


# ----------------------------------------------------------------------
# 调用
# ----------------------------------------------------------------------
def call_endpoint(store, token, key, endpoint, date):
    """调用单个接口一天。

    返回 (状态, 行数)：
      ("ok", n)      成功（n 可能为 0，表示当天无数据）
      ("retry", 0)   token 失效，需要换 token 后重试
      ("quota", 0)   日调用量上限（45009）：当天重试无意义，调用方应停扫等明天
      ("error", 0)   失败（已记入台账，下次自动补扫 / 已打日志）
    """
    url = API_BASE + endpoint + "?" + urlencode({"access_token": token})
    body = {"begin_date": date, "end_date": date}

    result = None
    # 瞬时错误（系统繁忙/频率限制）原地退避重试，避免把好日期误记成失败
    for attempt in range(3):
        result = http_request(url, body)
        errcode = result.get("errcode", 0)
        if errcode in TRANSIENT_ERRCODES:
            if attempt < 2:
                time.sleep(1.0 * (attempt + 1))
                continue
        break

    errcode = result.get("errcode", 0)
    if errcode == 0:
        rows = result.get("list", []) or []
        if key in DAILY_MERGERS:
            DAILY_MERGERS[key](store, rows)
        elif key == ARTICLE_DETAIL_KEY:
            merge_article_total_detail(store, rows)
        return ("ok", len(rows))
    if errcode in (40001, 42001):
        return ("retry", 0)

    if errcode == DAILY_QUOTA_ERRCODE:
        # 日调用量上限：按接口分别计算，当天不会再恢复。
        # 不重试、不记台账，交给调用方停止该接口剩余扫描，明天自动继续。
        return ("quota", 0)

    if key == ARTICLE_DETAIL_KEY and errcode == 48001:
        # 无接口权限：逐日重试没有意义，标记后跳过补扫（--full 可强制再试）
        if not store.get("title_api_unauthorized"):
            store["title_api_unauthorized"] = True
            print("\n================ 标题接口无权限 ================")
            print("getarticletotaldetail 返回 48001（接口未授权）。")
            print("本脚本仍会正常采集阅读/分享/粉丝数据，但拿不到文章标题，")
            print("看板中会显示“未获取到标题”，可点击「命名」手动补上。")
            print("如确认公众号已认证且具备数据接口权限，可运行")
            print("python3 sync.py --full 强制重试一次。")
            print("================================================\n")
        return ("error", 0)

    # 接口下线 / 数据未生成 / 日期问题 / IP 白名单 → 静默跳过（与老版本一致）
    if errcode in (47009, 47011, 61503, 61500, 61501, 40164):
        if errcode == 40164:
            log("接口 %s 提示 IP 不在白名单（40164）。" % endpoint)
        # 标题扫描遇到“数据未生成”时记入台账，下次重试（可能是当天数据没出来）
        if key == ARTICLE_DETAIL_KEY and errcode == 61503:
            return ("error", 0)
        return ("ok", 0)

    log("接口 %s 在 %s 返回：%s %s（已记入台账，下次运行自动补扫）"
        % (key, date, errcode, result.get("errmsg")))
    return ("error", 0)


def fetch_publish_date(store, token, date, appid, secret, sleep_s=0.12):
    """扫描一个「发表日」的标题/累计数据，并更新台账。"""
    status, n = call_endpoint(store, token, ARTICLE_DETAIL_KEY,
                              ARTICLE_DETAIL_ENDPOINT, date)
    if status == "retry":
        token = get_access_token(appid, secret, force=True)
        status, n = call_endpoint(store, token, ARTICLE_DETAIL_KEY,
                                  ARTICLE_DETAIL_ENDPOINT, date)
    ledger = store.setdefault("title_scan_dates", {})
    if status == "quota":
        # 日配额用完：不动台账——没扫过的日期保持待扫（明天继续），
        # 已成功的日期也不降级。
        pass
    else:
        ledger[date] = "ok" if status == "ok" else "err"
    time.sleep(sleep_s)
    return token, status


def date_range(today, n):
    ds = [(today - timedelta(days=i + 1)).strftime("%Y-%m-%d") for i in range(n)]
    return sorted(set(ds))


# ----------------------------------------------------------------------
# 标题补全：获取已发布的消息列表（freepublish/batchget）
# ----------------------------------------------------------------------
def parse_mid_idx(url):
    """从文章 URL 中解析 (mid, idx)。

    mid 即「图文消息id」，idx 即篇目序号；官方文档明确
    msgid = 图文消息id_index，因此 msgid = mid + "_" + idx。
    兼容 appmsgid / itemidx 写法；解析不到返回 None。
    """
    if not url:
        return None
    try:
        q = parse_qs(urlparse(url.strip()).query)
    except Exception:
        return None
    mid = (q.get("mid") or q.get("appmsgid") or [""])[0].strip()
    idx = (q.get("idx") or q.get("itemidx") or [""])[0].strip()
    if not mid:
        return None
    return mid, (idx or "1")


def resolve_short_url(url, timeout=12):
    """跟随 302 跳转，拿到短链背后的完整文章 URL。

    有些 news_item[].url 是不带 mid/idx 的短链（mp.weixin.qq.com/s/<code>），
    urllib 默认会自动跟随重定向，resp.geturl() 即最终 URL。
    """
    if not url:
        return None
    try:
        req = Request(url.strip(), method="GET",
                      headers={"User-Agent": "wechat-dashboard/1.0"})
        with urlopen(req, timeout=timeout) as resp:
            return resp.geturl() or url
    except Exception as e:
        log("    短链解析失败（%s）：%s" % (url, e))
        return None


def sync_publish_list(store, token, appid, secret):
    """用「获取已发布的消息列表」补全缺失的文章标题/链接。

    - 该接口是内容接口，不受 2025-11-01 数据起点限制，是补全老文章标题的关键来源；
    - news_item[].url 中的 mid/idx 与 msgid 前两段一一对应（官方：msgid=图文消息id_index）；
    - 只填充看板已知、但缺标题的 msgid，不会凭发布列表新增文章记录；
    - 48001（无权限）时打印指引并续跑，不影响其他数据采集。

    返回 (补上标题的篇数, 可能更新过的 token)。
    """
    # 1) 汇总看板已知、但没有标题的 msgid
    need = set()
    for m, a in (store.get("articles") or {}).items():
        if not (a or {}).get("title"):
            need.add(m)
    for d, arts in (store.get("article_daily") or {}).items():
        for m in (arts or {}):
            if not (store["articles"].get(m) or {}).get("title"):
                need.add(m)
    if not need:
        return 0, token

    log("开始通过「已发布消息列表」补全标题：待补全 %s 篇 …" % len(need))
    base = API_BASE + PUBLISH_LIST_ENDPOINT
    filled = 0
    offset = 0
    total = None
    pages = 0
    warned = False

    while True:
        url = base + "?" + urlencode({"access_token": token})
        body = {"offset": offset, "count": PUBLISH_PAGE, "no_content": 0}
        result = http_request(url, body)
        errcode = result.get("errcode", 0)
        if errcode in (40001, 42001):
            token = get_access_token(appid, secret, force=True)
            result = http_request(base + "?" + urlencode({"access_token": token}), body)
            errcode = result.get("errcode", 0)
        if errcode != 0:
            if errcode == DAILY_QUOTA_ERRCODE:
                store["publish_list_status"] = "quota"
                if not warned:
                    warned = True
                    log("「已发布消息列表」接口今日调用量已达上限（45009），"
                        "本次跳过标题补全；明天运行自动继续。")
                break
            if errcode == 48001:
                store["publish_list_status"] = "unauthorized"
                if not warned:
                    warned = True
                    print("\n================ 发布列表接口无权限 ================")
                    print("freepublish/batchget 返回 48001（接口未授权）。")
                    print("老文章（2025-11-01 前发表）的标题将无法自动补全，")
                    print("看板中会显示“未获取到标题”，可点「命名」手动补上。")
                    print("====================================================\n")
            else:
                store["publish_list_status"] = "error"
                if not warned:
                    warned = True
                    log("「已发布消息列表」接口返回 %s %s，本次跳过标题补全。"
                        % (errcode, result.get("errmsg")))
            break

        if total is None:
            total = int(result.get("total_count", 0) or 0)
            store["publish_list_status"] = "ok"
        items = result.get("item", []) or []
        if not items:
            break
        pages += 1
        for it in items:
            content = it.get("content") or {}
            for news in content.get("news_item", []) or []:
                title = (news.get("title") or "").strip()
                # 真实接口 news_item 同时含 url 与 content_url，个别账号只返回其中一个
                url = (news.get("url") or news.get("content_url") or "").strip()
                if not title and not url:
                    continue
                mid_idx = parse_mid_idx(url)
                final_url = url
                if not mid_idx and url:
                    # url 是短链：跟随 302 跳转后再解析
                    final = resolve_short_url(url)
                    if final:
                        final_url = final
                        mid_idx = parse_mid_idx(final)
                if not mid_idx:
                    continue
                msgid = "%s_%s" % mid_idx
                if msgid in need and title:
                    art = ensure(store["articles"], msgid)
                    art["title"] = title
                    if final_url:
                        art["content_url"] = final_url
                    art["title_src"] = "publish_list"
                    need.discard(msgid)
                    filled += 1
        if not need:
            break
        offset += len(items)
        if offset >= total or len(items) < PUBLISH_PAGE or pages >= PUBLISH_MAX_PAGES:
            break
        time.sleep(0.15)

    if filled:
        log("发布列表标题补全完成：补上 %s 篇（剩余 %s 篇缺标题）。"
            % (filled, len(need)))
    elif need:
        log("发布列表未补到标题（仍有 %s 篇缺标题）。" % len(need))
    return filled, token


# ----------------------------------------------------------------------
# 输出：data.js（应用手动命名 + 过滤软删除 + 附回收站快照）
# ----------------------------------------------------------------------
SNAP_FIELDS = ("read_user", "share_user", "zaikan_user", "like_user",
               "comment_count", "collection_user", "read_subscribe_user")


def snapshot_article(store, m):
    """取一篇文章的展示快照（供删除台账/回收站使用）。"""
    art = store.get("articles", {}).get(m, {})
    snap = store.get("article_total", {}).get(m, {})
    out = {"id": m,
           "title": art.get("title") or "",
           "pubdate": art.get("first_date") or "",
           "content_url": art.get("content_url") or ""}
    for k in SNAP_FIELDS:
        out[k] = snap.get(k, 0) or 0
    return out


def fix_first_dates(store):
    """按「文章日报」重算所有文章的首发日期。

    老版本（setdefault）把首次写入的日期固定下来，补扫历史数据后
    也不会更新，导致 47 篇老文章被错标为最近几天的日期。
    权威口径：文章在某统计日有数据 → 取最早的那天为发布日期。
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


def build_payload(store):
    """store.json -> data.js 载荷：
    - 应用 manual_titles（手动命名优先，title_src=manual）
    - 过滤 deleted 台账中的文章（重扫不复活）
    - 附 deleted_articles 回收站快照（含删除时的指标）
    """
    deleted = store.get("deleted", {}) or {}
    manual = store.get("manual_titles", {}) or {}
    payload = dict(store)
    payload.pop("deleted", None)
    payload["manual_titles"] = manual

    articles = {}
    for m, art in (store.get("articles", {}) or {}).items():
        if m in deleted:
            continue
        art = dict(art)
        if m in manual and manual[m]:
            art["title"] = manual[m]
            art["title_src"] = "manual"
        articles[m] = art
    payload["articles"] = articles

    payload["article_total"] = {m: v for m, v in
                                (store.get("article_total", {}) or {}).items()
                                if m not in deleted}
    daily = {}
    for sd, rows in (store.get("article_daily", {}) or {}).items():
        daily[sd] = {m: v for m, v in rows.items() if m not in deleted}
    payload["article_daily"] = daily

    bin_list = []
    for m, rec in deleted.items():
        item = dict(rec) if isinstance(rec, dict) else {}
        item["id"] = m
        item.setdefault("title", "")
        item.setdefault("pubdate", "")
        item.setdefault("deleted_at", "")
        for k in SNAP_FIELDS:
            item.setdefault(k, 0)
        bin_list.append(item)
    bin_list.sort(key=lambda x: x.get("deleted_at", ""), reverse=True)
    payload["deleted_articles"] = bin_list
    return payload


def write_data_js(store):
    payload = build_payload(store)
    payload["generated_at"] = store.get("generated_at", "")
    out = (
        "window.DATA = " + json.dumps(payload, ensure_ascii=False) + ";\n"
        + "window.DATA_GENERATED_AT = "
        + json.dumps(store.get("generated_at", ""), ensure_ascii=False) + ";\n"
    )
    tmp = DATAJS_PATH + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        f.write(out)
    os.replace(tmp, DATAJS_PATH)
    return payload


def main():
    config = load_json(CONFIG_PATH, {})
    appid = (config.get("appid") or "").strip()
    secret = (config.get("secret") or "").strip()
    lookback = int(config.get("lookback_days", 7))
    detail_days = int(config.get("article_detail_days", 30))

    if "--days" in sys.argv:
        idx = sys.argv.index("--days")
        try:
            val = int(sys.argv[idx + 1])
            lookback = val
            detail_days = max(detail_days, val)
        except Exception:
            pass

    if not appid or not secret or appid.startswith("wx在"):
        print("请先编辑 config.json，填入你的 appid 和 secret。")
        sys.exit(1)

    log("开始采集：统计日回溯 %s 天，发表日回溯 %s 天。" % (lookback, detail_days))
    store = upgrade_store(load_json(STORE_PATH, new_store()))

    token = get_access_token(appid, secret)
    today = datetime.now().date()
    yesterday = today - timedelta(days=1)
    yesterday_s = yesterday.strftime("%Y-%m-%d")

    # 1) 最近 N 个「发表日」：新文章的标题/链接 + 累计轨迹（每次必跑）。
    #    新的日期排在前面：万一遇到日调用量上限（45009），先保住最近发表文章的标题。
    pub_dates = [d for d in date_range(today, detail_days) if d <= yesterday_s]
    pub_dates.sort(reverse=True)
    detail_quota = False      # getarticletotaldetail 今日配额是否已用完
    n_pub = 0
    for i, date in enumerate(pub_dates):
        token, status = fetch_publish_date(store, token, date, appid, secret)
        if status == "quota":
            detail_quota = True
            log("getarticletotaldetail 今日调用量已达上限（45009），发表日期扫描暂停。")
            log("  已扫描的日期不会重复；剩余 %s 个日期将在下次运行自动继续"
                "（配额每天 0 点重置，无需干预）。" % (len(pub_dates) - i))
            break
        if status == "ok":
            n_pub += 1
    if not detail_quota:
        log("最近 %s 天发表内容明细完成" % len(pub_dates))

    # 1.1) 逐日补扫：只扫台账里没成功过的日期（首次≈全量；失败日期自愈重试）
    force_full = "--full" in sys.argv
    earliest = datetime.strptime(EARLIEST_DATE, "%Y-%m-%d").date()
    start = today - timedelta(days=365)
    if start < earliest:
        start = earliest
    all_dates = []
    cur = start
    while cur <= yesterday:
        all_dates.append(cur.strftime("%Y-%m-%d"))
        cur += timedelta(days=1)

    ledger = store.setdefault("title_scan_dates", {})
    pending = [d for d in all_dates if force_full or ledger.get(d) != "ok"]
    if force_full:
        log("--full：强制重新扫描全部发表日期。")
    if pending and store.get("title_api_unauthorized") and not force_full:
        log("标题接口此前返回过 48001（无权限），跳过 %s 个日期的补扫；"
            "加 --full 可强制重试。" % len(pending))
        pending = []
    if detail_quota and pending:
        # 同一个接口，配额已在第 1 步用完，补扫今天不可能成功
        log("标题接口今日配额已用完（45009），跳过剩余 %s 个待补扫日期；"
            "明天运行自动继续。" % len(pending))
        pending = []
    if pending:
        log("开始标题补扫：%s 至 %s（共 %s 天，期间请勿中断）"
            % (pending[0], pending[-1], len(pending)))
        for i, date in enumerate(pending):
            token, status = fetch_publish_date(store, token, date, appid, secret,
                                               sleep_s=0.05)
            if status == "quota":
                detail_quota = True
                log("标题补扫遇到日调用量上限（45009），已暂停。")
                log("  本次完成 %s/%s，剩余 %s 个日期将在下次运行自动继续。"
                    % (i, len(pending), len(pending) - i))
                break
            if (i + 1) % 30 == 0:
                log("  补扫进度 %s/%s" % (i + 1, len(pending)))
        if not detail_quota:
            log("标题补扫完成。")
    elif not store.get("title_api_unauthorized") and not detail_quota:
        log("标题台账已是最新，无需补扫。")

    # 2) 按统计日：每日端点（45009 按接口分别计算：只跳过中招的那个接口）
    stat_dates = date_range(today, lookback)
    n_stat = 0
    quota_eps = set()
    for date in stat_dates:
        for key, endpoint in DAILY_ENDPOINTS.items():
            if key in quota_eps:
                continue
            status, n = call_endpoint(store, token, key, endpoint, date)
            if status == "retry":
                token = get_access_token(appid, secret, force=True)
                status, n = call_endpoint(store, token, key, endpoint, date)
            if status == "quota":
                quota_eps.add(key)
                log("接口 %s 今日调用量已达上限（45009），跳过其剩余日期；"
                    "明天运行自动继续。" % key)
                continue
            if isinstance(n, int):
                n_stat += n
            time.sleep(0.12)
        log("  %s 完成" % date)

    # 2.1) 发布列表补全：老文章（2025-11-01 前发表）标题的权威来源。
    #      放在每日采集之后：getarticleread 每天都会“新发现”仍有阅读的老文章，
    #      补全必须覆盖这些刚入库的记录。
    #      （内容接口，无日期限制；url 中 mid/idx 与 msgid 一一对应）
    filled, token = sync_publish_list(store, token, appid, secret)

    fix_first_dates(store)   # 保存前统一校正发布日期
    store["generated_at"] = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    save_json(STORE_PATH, store)
    write_data_js(store)

    # 3) 标题健康检查：解释“为什么还有文章没有标题”
    deleted_ids = set(store.get("deleted", {}) or {})
    known_ids = (set(store["articles"]) | set(store["article_total"])) - deleted_ids
    for d in store["article_daily"]:
        known_ids |= (set(store["article_daily"][d]) - deleted_ids)
    manual = store.get("manual_titles", {}) or {}
    untitled = [m for m in known_ids
                if not (store["articles"].get(m, {}).get("title") or manual.get(m))]
    failed_dates = [d for d, v in ledger.items() if v != "ok"]
    log("采集完成：发表明细 %s 天、每日记录 %s 条%s。"
        % (n_pub, n_stat, "、发布列表补标题 %s 篇" % filled if filled else ""))
    log("标题情况：共 %s 篇文章，其中 %s 篇没有标题。" % (len(known_ids), len(untitled)))
    if untitled:
        if store.get("publish_list_status") == "unauthorized":
            print("  · 原因：发布列表接口无权限（48001），无法自动补全老文章标题；"
                  "可在看板中「命名」手动补上。")
        elif store.get("publish_list_status") == "quota":
            print("  · 原因：发布列表接口今日调用量已达上限（45009），"
                  "明天运行会自动补全。")
        elif store.get("publish_list_status") == "error":
            print("  · 原因：发布列表接口调用失败（详见上方日志），"
                  "下次运行会自动重试补全。")
        elif failed_dates:
            print("  · 原因：%s 个发表日采集失败（限流/网络等原因），"
                  "下次运行会自动补扫。" % len(failed_dates))
        print("  · 2025-11-01 之前发表的老文章：若发布列表也没补上（未发布过/已删除/无权限），"
              "微信不提供其标题，看板中显示为“未获取到标题”，点「命名」可手动补上。")
    if detail_quota:
        print("  · getarticletotaldetail 今日配额已用完（45009）：剩余发表日期明天自动补扫；"
              "标题以发布列表补全为准，累计轨迹数据随后几天自动补齐。")
        print("  · 如需当天跑完：公众平台后台「接口权限模板」可查看实时调用量，"
              "认证账号每月有 10 次清零机会。")
    log("已写入 store.json / data.js，用浏览器打开 index.html 即可。")


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n已取消。")
        sys.exit(1)
    except BrokenPipeError:
        pass
```

### 文件 5：`serve.py`（本地服务）

> 提供看板页面、写接口（删除/重命名/上传）、调度采集、同步线上。
> **部署前请把 `ACCESS_PWD` 改成你自己的密码**（默认只是占位符）。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
微信公众号数据看板 - 本地服务（v5，仅标准库）
- 静态提供 index.html / data.js（config.json、token.json、store.json 不对外）
- POST /api/sync        手动更新：子进程运行 sync.py（单航班，不会并发重入）
- GET  /api/status      更新任务状态 + 日志尾部 + 数据概况
- POST /api/delete      软删除文章 {ids:[...]}（进回收站，可恢复）
- POST /api/restore     从回收站恢复 {ids:[...]}
- POST /api/purge       彻底删除 {ids:[...]}（物理移除，不可恢复）
- POST /api/rename      手动命名 {id, title}（存 store.json，全设备生效）
- POST /api/unrename    取消手动命名 {id}
- GET  /api/config      config.json 是否已配置（不返回任何密钥内容）
- POST /api/data        上传数据文件内容（需 pwd），写入 data.js 并同步到线上
- POST /api/push        手动把本机数据同步到线上站点（需 pwd）
- GET  /api/upstream    线上同步配置与最近一次同步结果
- POST /api/upstream    修改线上同步配置（需 pwd）

用法：
    python3 serve.py [--port 8000] [--no-browser]
"""

import os
import sys
import json
import time
import signal
import atexit
import secrets
import socket
import threading
import tempfile
import subprocess
from datetime import datetime
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse
from urllib.request import urlopen, Request

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, BASE_DIR)

import sync  # noqa: E402  复用 store 读写 / data.js 生成逻辑

CONFIG_PATH = os.path.join(BASE_DIR, "config.json")
STORE_PATH = os.path.join(BASE_DIR, "store.json")
LOG_PATH = os.path.join(BASE_DIR, "sync.log")
PID_PATH = os.path.join(BASE_DIR, "serve.pid")

# 写操作密码（删除/还原/彻底删除/重命名/采集更新）。内网开放后，接口同样需要密码。
# 页面不再保存密码：POST /api/auth 由本服务比对后下发临时 token，写操作凭 token 放行。
# 默认值仅为占位符，部署前必须修改：推荐设置环境变量 ADMIN_PWD，或直接改下面这一行。
ACCESS_PWD = os.environ.get("ADMIN_PWD", "change-me")
TOKEN_TTL = 8 * 60 * 60
_TOKENS = {}

def _issue_token():
    tok = secrets.token_hex(24)
    _TOKENS[tok] = time.time() + TOKEN_TTL
    return tok

def _check_token(tok):
    if not isinstance(tok, str) or not tok:
        return False
    exp = _TOKENS.get(tok)
    return bool(exp) and exp > time.time()

# 不对外提供的文件（含密钥/中间态/脚本自身）
BLOCKED_NAMES = {"config.json", "token.json", "store.json", "sync.py",
                 "serve.py", "run.sh", "stop.sh", "install.sh", "sync.log",
                 "store.json.tmp", "data.js.tmp", "serve.pid"}
SAFE_EXTS = {".html", ".js", ".css", ".svg", ".png", ".jpg", ".jpeg",
             ".webp", ".ico", ".woff", ".woff2", ".txt", ".md"}

# ----------------------------------------------------------------------
# 更新任务（单航班）
# ----------------------------------------------------------------------
_job_lock = threading.Lock()
_job = {
    "running": False,
    "started_at": "",
    "finished_at": "",
    "returncode": None,
    "error": "",
}


def _job_snapshot():
    with _job_lock:
        return dict(_job)


def _read_log_tail(n=4000):
    try:
        with open(LOG_PATH, "r", encoding="utf-8", errors="replace") as f:
            f.seek(0, os.SEEK_END)
            size = f.tell()
            f.seek(max(0, size - n))
            return f.read()
    except Exception:
        return ""


LOG_MAX_BYTES = 2 * 1024 * 1024   # sync.log 超过 2MB 触发滚动
LOG_KEEP_LINES = 3000             # 滚动后保留最近 3000 行


def _rotate_log():
    """sync.log 过大时截断到最近 N 行（本地桌面长期部署，避免日志无限增长）。"""
    try:
        if not os.path.isfile(LOG_PATH) \
                or os.path.getsize(LOG_PATH) < LOG_MAX_BYTES:
            return
        with open(LOG_PATH, "r", encoding="utf-8", errors="replace") as f:
            lines = f.readlines()
        with open(LOG_PATH, "w", encoding="utf-8") as f:
            f.write("===== 日志滚动 %s（保留最近 %d 行）=====\n"
                    % (_now(), LOG_KEEP_LINES))
            f.writelines(lines[-LOG_KEEP_LINES:])
    except Exception:
        pass


def _sync_worker():
    """后台线程：子进程跑 sync.py，全程记录日志。"""
    try:
        _rotate_log()
        with open(LOG_PATH, "a", encoding="utf-8") as lf:
            lf.write("\n===== 手动更新 %s =====\n" % _now())
            lf.flush()
            proc = subprocess.Popen(
                [sys.executable, os.path.join(BASE_DIR, "sync.py")],
                stdout=lf, stderr=subprocess.STDOUT, cwd=BASE_DIR)
            rc = proc.wait()
        # 采集成功后自动把新数据同步到线上（可关：config.json 的 upstream_auto_push = false）
        if rc == 0:
            try:
                if _upstream_conf()["auto"]:
                    push_to_upstream("采集完成")
            except Exception as e:
                _log_push("自动同步异常：%s" % e)
        with _job_lock:
            _job.update({"running": False, "finished_at": _now(),
                         "returncode": rc})
    except Exception as e:  # 进程启动失败等
        try:
            with open(LOG_PATH, "a", encoding="utf-8") as lf:
                lf.write("更新任务异常：%s\n" % e)
        except Exception:
            pass
        with _job_lock:
            _job.update({"running": False, "finished_at": _now(),
                         "returncode": -1, "error": str(e)})


def _run_sync_job():
    """触发一次更新（单航班）。已在运行时返回 False。"""
    with _job_lock:
        if _job["running"]:
            return False
        _job.update({"running": True, "started_at": _now(),
                     "finished_at": "", "returncode": None, "error": ""})
    threading.Thread(target=_sync_worker, daemon=True).start()
    return True


def _now():
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


# ----------------------------------------------------------------------
# 数据变更（删除 / 恢复 / 彻底删除 / 命名）
# ----------------------------------------------------------------------
def _load_store():
    store = sync.upgrade_store(sync.load_json(STORE_PATH, sync.new_store()))
    return store


def _save_and_refresh(store):
    store["generated_at"] = _now()
    sync.save_json(STORE_PATH, store)
    sync.write_data_js(store)


def api_delete(ids):
    store = _load_store()
    done, already, unknown = [], [], []
    for m in ids:
        if not m:
            continue
        if m in store["deleted"]:
            already.append(m)
        elif m in store["articles"] or m in store["article_total"] \
                or any(m in rows for rows in store["article_daily"].values()):
            rec = sync.snapshot_article(store, m)
            rec["deleted_at"] = _now()
            store["deleted"][m] = rec
            done.append(m)
        else:
            unknown.append(m)
    _save_and_refresh(store)
    _push_state_async("删除文章")
    return {"ok": True, "deleted": done, "already": already,
            "unknown": unknown}


def api_restore(ids):
    store = _load_store()
    done = []
    for m in ids:
        if m in store["deleted"]:
            del store["deleted"][m]
            done.append(m)
    _save_and_refresh(store)
    _push_state_async("恢复文章")
    return {"ok": True, "restored": done}


def api_purge(ids):
    store = _load_store()
    done = []
    for m in ids:
        hit = False
        store["deleted"].pop(m, None)
        store["manual_titles"].pop(m, None)
        if m in store["articles"]:
            del store["articles"][m]
            hit = True
        if m in store["article_total"]:
            del store["article_total"][m]
            hit = True
        for sd in list(store["article_daily"].keys()):
            if m in store["article_daily"][sd]:
                del store["article_daily"][sd][m]
                hit = True
        if hit:
            done.append(m)
    _save_and_refresh(store)
    _push_state_async("彻底删除")
    return {"ok": True, "purged": done}


def api_rename(mid, title):
    store = _load_store()
    title = (title or "").strip()
    if title:
        store["manual_titles"][mid] = title[:120]
    else:
        store["manual_titles"].pop(mid, None)
    _save_and_refresh(store)
    _push_state_async("重命名")
    return {"ok": True, "title": title}


# ----------------------------------------------------------------------
# 本机 → 线上 同步（数据与回收站状态推送到线上站点，线上所有页面随之更新）
# ----------------------------------------------------------------------
DEFAULT_UPSTREAM = ""   # 留空则不同步；在 config.json 的 upstream 填入你的线上地址

_push_lock = threading.Lock()
_push_state = {"running": False, "last_at": "", "last_ok": None, "last_message": ""}


def _upstream_conf():
    """读取线上同步配置；未配置时使用默认线上地址，默认自动同步。"""
    try:
        cfg = sync.load_json(CONFIG_PATH, {}) or {}
    except Exception:
        cfg = {}
    url = str(cfg.get("upstream") or DEFAULT_UPSTREAM).strip().rstrip("/")
    pwd = str(cfg.get("upstream_pwd") or ACCESS_PWD).strip()
    auto = cfg.get("upstream_auto_push")
    return {"url": url, "pwd": pwd, "auto": True if auto is None else bool(auto)}


def load_data_dict():
    """解析本机 data.js，返回数据字典。"""
    class _W(object):
        pass
    w = _W()
    try:
        with open(os.path.join(BASE_DIR, "data.js"), "r", encoding="utf-8") as f:
            src = f.read()
        # data.js 为 JSON 直出，可能含 false/true/null，需映射为 Python 字面量
        exec(compile(src, "data.js", "exec"),
             {"window": w, "false": False, "true": True, "null": None})
    except Exception:
        return None
    return getattr(w, "DATA", None)


def write_data_js_from_upload(data, gen=""):
    """把上传的数据直接写成本机 data.js（覆盖），返回生成时间。"""
    gen = gen or data.get("generated_at") or _now()
    data = dict(data)
    data["generated_at"] = gen
    text = ("window.DATA = " + json.dumps(data, ensure_ascii=False) + ";\n"
            + 'window.DATA_GENERATED_AT = "' + str(gen) + '";\n')
    fd, tmp = tempfile.mkstemp(dir=BASE_DIR, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(text)
        os.replace(tmp, os.path.join(BASE_DIR, "data.js"))
    except Exception:
        try:
            os.unlink(tmp)
        except OSError:
            pass
        raise
    return gen


def _http_post_json(url, obj, timeout=60):
    req = Request(url, data=json.dumps(obj, ensure_ascii=False).encode("utf-8"),
                  headers={"Content-Type": "application/json"}, method="POST")
    with urlopen(req, timeout=timeout) as r:
        raw = r.read().decode("utf-8", "replace")
    try:
        return json.loads(raw)
    except Exception:
        return {"ok": False, "message": "响应解析失败"}


def _log_push(msg):
    try:
        with open(LOG_PATH, "a", encoding="utf-8") as lf:
            lf.write("%s  [上行同步] %s\n" % (_now(), msg))
    except Exception:
        pass


# 线上 token 缓存：推送前用线上密码换取临时 token，过期自动刷新
_UP_TOKEN_CACHE = {"key": "", "token": "", "exp": 0}


def _upstream_token(conf):
    key = conf["url"] + "|" + conf["pwd"]
    now = time.time()
    if _UP_TOKEN_CACHE["key"] == key and _UP_TOKEN_CACHE["exp"] > now + 60:
        return _UP_TOKEN_CACHE["token"]
    j = _http_post_json(conf["url"] + "/api/auth",
                        {"pwd": conf["pwd"]}, timeout=30)
    if j.get("ok") and j.get("token"):
        ttl = int(j.get("expires_in") or 28800)
        _UP_TOKEN_CACHE.update({"key": key, "token": j["token"],
                                "exp": now + ttl})
        return j["token"]
    raise RuntimeError("线上密码验证失败：%s" % (j.get("message") or "未知原因"))


def _up_post(conf, api_path, payload, timeout=30):
    """上行请求：凭临时 token 调线上接口；token 失效自动刷新重试一次。"""
    payload = dict(payload)
    payload.pop("pwd", None)
    payload["token"] = _upstream_token(conf)
    j = _http_post_json(conf["url"] + api_path, payload, timeout=timeout)
    if not j.get("ok"):
        msg = str(j.get("message") or "")
        if "过期" in msg or "未解锁" in msg:
            _UP_TOKEN_CACHE["exp"] = 0
            payload["token"] = _upstream_token(conf)
            j = _http_post_json(conf["url"] + api_path, payload,
                                timeout=timeout)
    return j


def upstream_snapshot():
    conf = _upstream_conf()
    return {
        "configured": bool(conf["url"]),
        "url": conf["url"],
        "auto_push": conf["auto"],
        "running": _push_state.get("running", False),
        "last_push_at": _push_state.get("last_at", ""),
        "last_push_ok": _push_state.get("last_ok", None),
        "last_push_message": _push_state.get("last_message", ""),
    }


def push_to_upstream(reason="", with_data=True, with_state=True):
    """把本机数据与回收站状态推送到线上站点。返回 (是否全部成功, 说明)。"""
    with _push_lock:
        if _push_state["running"]:
            return False, "上一次同步尚未结束"
        _push_state["running"] = True
    try:
        conf = _upstream_conf()
        if not conf["url"]:
            return False, "未配置线上地址"
        msgs, ok_all = [], True
        if with_data:
            data = load_data_dict()
            if not data:
                ok_all = False
                msgs.append("读取本机 data.js 失败")
            else:
                try:
                    j = _up_post(conf, "/api/data", {
                        "data": data,
                        "generated_at": data.get("generated_at", ""),
                    }, timeout=180)
                    if j.get("ok"):
                        msgs.append("数据已同步（%d 篇）" % len(data.get("articles", {}) or {}))
                    else:
                        ok_all = False
                        msgs.append("数据同步失败：%s" % (j.get("message") or "未知原因"))
                except Exception as e:
                    ok_all = False
                    msgs.append("数据同步失败：%s" % e)
        if with_state:
            try:
                store = sync.load_json(STORE_PATH, {}) or {}
                deleted = {}
                for k, v in (store.get("deleted") or {}).items():
                    v = v or {}
                    deleted[str(k)] = {"title": v.get("title", ""),
                                       "deleted_at": v.get("deleted_at", "")}
                j = _up_post(conf, "/api/state", {
                    "rev": int(time.time() * 1000),
                    "deleted": deleted,
                    "titles": store.get("manual_titles") or {},
                }, timeout=30)
                if j.get("ok"):
                    msgs.append("回收站状态已同步")
                else:
                    msgs.append("回收站状态同步失败：%s" % (j.get("message") or "未知原因"))
            except Exception as e:
                msgs.append("回收站状态同步失败：%s" % e)
        msg = "；".join(msgs) or "无内容"
        _push_state.update({"last_at": _now(), "last_ok": ok_all, "last_message": msg})
        _log_push("%s → %s（%s）" % (reason or "手动", conf["url"], msg))
        return ok_all, msg
    finally:
        with _push_lock:
            _push_state["running"] = False


def _push_state_async(reason):
    """删除/恢复/彻底删除/命名等轻量变更：后台线程只推回收站状态，不阻塞页面操作。"""
    def _run():
        try:
            if _upstream_conf()["auto"]:
                push_to_upstream(reason, with_data=False, with_state=True)
        except Exception as e:
            _log_push("状态同步异常：%s" % e)
    threading.Thread(target=_run, daemon=True).start()


# ----------------------------------------------------------------------
# HTTP 处理
# ----------------------------------------------------------------------
class Handler(BaseHTTPRequestHandler):
    server_version = "WechatDashboard/5.0"
    protocol_version = "HTTP/1.1"

    # ---- 工具 ----
    def _send_json(self, obj, code=200):
        body = json.dumps(obj, ensure_ascii=False).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self.send_header("X-Content-Type-Options", "nosniff")
        self.end_headers()
        self.wfile.write(body)

    def _read_body(self):
        try:
            n = int(self.headers.get("Content-Length", 0) or 0)
        except ValueError:
            n = 0
        if n <= 0 or n > 24 * 1024 * 1024:   # 放宽：上传数据文件可能较大
            return {}
        raw = self.rfile.read(n)
        try:
            return json.loads(raw.decode("utf-8"))
        except Exception:
            return {}

    def log_message(self, fmt, *args):  # 静默默认访问日志
        pass

    # ---- GET ----
    def do_GET(self):
        path = urlparse(self.path).path
        if path == "/api/status":
            job = _job_snapshot()
            store = sync.load_json(STORE_PATH, {})
            self._send_json({
                "job": job,
                "log_tail": _read_log_tail(),
                "generated_at": store.get("generated_at", ""),
                "articles": len(store.get("articles", {}) or {}),
                "deleted": len(store.get("deleted", {}) or {}),
                "server_time": _now(),
                "upstream": upstream_snapshot(),
            })
            return
        if path == "/api/upstream":
            self._send_json(upstream_snapshot())
            return
        if path == "/api/config":
            cfg = sync.load_json(CONFIG_PATH, {})
            appid = (cfg.get("appid") or "").strip()
            self._send_json({
                "configured": bool(appid) and not appid.startswith("wx在"),
                "has_appid": bool(appid),
            })
            return
        self._serve_static(path)

    # ---- POST ----
    def do_POST(self):
        path = urlparse(self.path).path
        body = self._read_body()
        # 管理密码验证：密码只在本服务比对，成功下发临时 token
        if path == "/api/auth":
            if (body or {}).get("pwd") != ACCESS_PWD:
                time.sleep(0.4)
                self._send_json({"ok": False, "message": "密码不正确"}, code=403)
                return
            self._send_json({"ok": True, "token": _issue_token(),
                             "expires_in": TOKEN_TTL})
            return
        if path == "/api/logout":
            _TOKENS.pop((body or {}).get("token") or "", None)
            self._send_json({"ok": True})
            return
        # 写操作必须携带服务端下发的临时 token
        if path in ("/api/sync", "/api/delete", "/api/restore",
                    "/api/purge", "/api/rename", "/api/unrename",
                    "/api/data", "/api/push", "/api/upstream"):
            if not _check_token((body or {}).get("token")):
                self._send_json({"ok": False, "message": "未解锁或登录已过期，请重新验证"}, code=403)
                return
        try:
            if path == "/api/sync":
                started = _run_sync_job()
                if not started:
                    self._send_json({"ok": False, "running": True,
                                     "message": "更新已在进行中"}, code=409)
                else:
                    self._send_json({"ok": True, "started": True}, code=202)
                return
            if path == "/api/delete":
                ids = body.get("ids") or []
                if not isinstance(ids, list) or not ids:
                    self._send_json({"ok": False, "message": "ids 不能为空"},
                                    code=400)
                    return
                self._send_json(api_delete([str(i) for i in ids]))
                return
            if path == "/api/restore":
                ids = body.get("ids") or []
                self._send_json(api_restore([str(i) for i in ids]))
                return
            if path == "/api/purge":
                ids = body.get("ids") or []
                self._send_json(api_purge([str(i) for i in ids]))
                return
            if path == "/api/rename":
                mid = (body.get("id") or "").strip()
                if not mid:
                    self._send_json({"ok": False, "message": "id 不能为空"},
                                    code=400)
                    return
                self._send_json(api_rename(mid, body.get("title") or ""))
                return
            if path == "/api/unrename":
                mid = (body.get("id") or "").strip()
                self._send_json(api_rename(mid, ""))
                return
            if path == "/api/data":
                data = body.get("data")
                ok_shape = isinstance(data, dict) and data.get("articles") \
                    and data.get("article_daily") and data.get("biz_daily") \
                    and data.get("user_daily")
                if not ok_shape:
                    self._send_json({"ok": False,
                                     "message": "数据校验未通过（缺少必要字段）"},
                                    code=400)
                    return
                try:
                    gen = write_data_js_from_upload(data, body.get("generated_at") or "")
                except Exception as e:
                    self._send_json({"ok": False,
                                     "message": "写入数据文件失败：%s" % e}, code=500)
                    return
                pushed, pmsg = None, ""
                try:
                    if _upstream_conf()["auto"]:
                        pushed, pmsg = push_to_upstream("上传数据")
                except Exception as e:
                    pushed, pmsg = False, "同步异常：%s" % e
                self._send_json({"ok": True, "generated_at": gen,
                                 "articles": len(data.get("articles") or {}),
                                 "pushed": pushed, "push_message": pmsg})
                return
            if path == "/api/push":
                ok, msg = push_to_upstream("手动同步")
                self._send_json({"ok": ok, "message": msg,
                                 "push_message": msg,
                                 "pushed_at": _push_state.get("last_at", "")})
                return
            if path == "/api/upstream":
                try:
                    cfg = sync.load_json(CONFIG_PATH, {}) or {}
                except Exception:
                    cfg = {}
                if "upstream" in body:
                    cfg["upstream"] = str(body.get("upstream") or "").strip().rstrip("/")
                if "upstream_pwd" in body:
                    cfg["upstream_pwd"] = str(body.get("upstream_pwd") or "").strip()
                if "upstream_auto_push" in body:
                    cfg["upstream_auto_push"] = bool(body.get("upstream_auto_push"))
                try:
                    sync.save_json(CONFIG_PATH, cfg)
                except Exception as e:
                    self._send_json({"ok": False, "message": "保存配置失败：%s" % e},
                                    code=500)
                    return
                self._send_json({"ok": True, "upstream": upstream_snapshot()})
                return
            self._send_json({"ok": False, "message": "未知接口"}, code=404)
        except Exception as e:
            self._send_json({"ok": False, "message": "服务器错误：%s" % e},
                            code=500)

    # ---- 静态文件 ----
    def _serve_static(self, path):
        if path in ("/", ""):
            path = "/index.html"
        # 防目录穿越
        rel = os.path.normpath(path).lstrip("/\\")
        full = os.path.abspath(os.path.join(BASE_DIR, rel))
        if not full.startswith(BASE_DIR + os.sep) and full != BASE_DIR:
            self._send_json({"ok": False, "message": "forbidden"}, code=403)
            return
        name = os.path.basename(full)
        if name in BLOCKED_NAMES:
            self._send_json({"ok": False, "message": "forbidden"}, code=403)
            return
        ext = os.path.splitext(name)[1].lower()
        if ext not in SAFE_EXTS or not os.path.isfile(full):
            self._send_json({"ok": False, "message": "not found"}, code=404)
            return
        ctype = {".html": "text/html; charset=utf-8",
                 ".js": "application/javascript; charset=utf-8",
                 ".css": "text/css; charset=utf-8",
                 ".svg": "image/svg+xml",
                 ".png": "image/png", ".jpg": "image/jpeg",
                 ".jpeg": "image/jpeg", ".webp": "image/webp",
                 ".ico": "image/x-icon", ".txt": "text/plain; charset=utf-8",
                 ".md": "text/markdown; charset=utf-8",
                 ".woff": "font/woff", ".woff2": "font/woff2"}.get(
                     ext, "application/octet-stream")
        try:
            with open(full, "rb") as f:
                data = f.read()
        except Exception:
            self._send_json({"ok": False, "message": "not found"}, code=404)
            return
        self.send_response(200)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(data)))
        self.send_header("X-Content-Type-Options", "nosniff")
        if ext in (".html", ".js", ".css"):
            self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(data)


# ----------------------------------------------------------------------
# 本地部署辅助：实例探测 / 端口自适应 / PID / 信号 / 静默断连
# ----------------------------------------------------------------------
def find_existing_instance(port):
    """探测端口上是否已有本服务的实例（GET /api/status 返回 job 字段）。"""
    try:
        with urlopen("http://127.0.0.1:%d/api/status" % port, timeout=1.5) as r:
            if r.status != 200:
                return False
            data = json.loads(r.read().decode("utf-8", "replace"))
            return isinstance(data, dict) and "job" in data
    except Exception:
        return False


def pick_port(preferred, strict=False):
    """首选端口被占用（且非本服务实例）时，顺延尝试 8001-8020。

    strict=True（用户在命令行显式指定 --port）时不顺延：返回 None，
    由调用方明确报错——显式端口被静默更换会破坏用户的书签/预期。
    """
    candidates = [preferred] if strict else \
        [preferred] + [p for p in range(8001, 8021) if p != preferred]
    for p in candidates:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            try:
                s.bind(("127.0.0.1", p))
            except OSError:
                continue
            return p
    return None


def write_pid_file():
    try:
        with open(PID_PATH, "w", encoding="utf-8") as f:
            f.write(str(os.getpid()))
    except Exception:
        pass


def remove_pid_file():
    try:
        if os.path.isfile(PID_PATH):
            with open(PID_PATH, "r", encoding="utf-8") as f:
                if f.read().strip() == str(os.getpid()):
                    os.remove(PID_PATH)
    except Exception:
        pass


def _open_browser_later(url):
    try:
        import webbrowser
        threading.Thread(target=lambda: (time.sleep(0.8),
                                         webbrowser.open(url)),
                         daemon=True).start()
    except Exception:
        pass


class Server(ThreadingHTTPServer):
    """静默客户端提前断开（页面刷新/关闭时的常见噪音，本地部署无需刷屏）。"""
    daemon_threads = True

    def handle_error(self, request, client_address):
        e = sys.exc_info()[1]
        if isinstance(e, (BrokenPipeError, ConnectionResetError,
                          ConnectionAbortedError)):
            return
        super().handle_error(request, client_address)


def _lan_ip():
    """取本机内网 IP（不会真的发包）。"""
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        s.connect(("10.255.255.255", 1))
        return s.getsockname()[0]
    except Exception:
        return ""
    finally:
        s.close()


def main():
    args = sys.argv[1:]
    preferred = 8000
    explicit_port = False
    cfg = sync.load_json(CONFIG_PATH, {})
    cfg_port = cfg.get("serve_port")
    if isinstance(cfg_port, int) and 1024 < cfg_port < 65536:
        preferred = cfg_port
    if "--port" in args:
        try:
            preferred = int(args[args.index("--port") + 1])
            explicit_port = True
        except Exception:
            pass
    # 双击启动器/再次运行 run.sh 时，若已有实例则只打开浏览器，不重复起服务
    if "--no-browser" not in args and find_existing_instance(preferred):
        print("看板已在运行：http://127.0.0.1:%d/（本次仅打开浏览器）" % preferred, flush=True)
        _open_browser_later("http://127.0.0.1:%d/" % preferred)
        time.sleep(2.5)
        return
    port = pick_port(preferred, strict=explicit_port)
    if port is None:
        print("端口 %d 被占用。可换端口：./run.sh --port 8080，"
              "或先停止已运行的服务（./stop.sh）" % preferred, flush=True)
        sys.exit(1)
    if port != preferred:
        print("端口 %d 被其他程序占用，已自动改用 %d。" % (preferred, port), flush=True)

    srv = Server(("0.0.0.0", port), Handler)   # 对内网开放，手机/其他电脑可访问
    url = "http://127.0.0.1:%d/" % port
    lan_ip = _lan_ip()
    if lan_ip:
        print("局域网访问地址（手机/其他电脑连同一 Wi-Fi 后打开）：http://%s:%d/" % (lan_ip, port), flush=True)

    def _shutdown(signum, frame):
        # 在子线程里 shutdown，避免信号回调中直接停造成死锁
        threading.Thread(target=srv.shutdown, daemon=True).start()

    signal.signal(signal.SIGTERM, _shutdown)
    try:
        signal.signal(signal.SIGHUP, _shutdown)
    except (ValueError, AttributeError, OSError):
        pass

    write_pid_file()
    atexit.register(remove_pid_file)

    print("看板已启动：%s" % url, flush=True)
    print("按 Ctrl+C 或运行 ./stop.sh 停止。"
          "数据更新请使用页面上的「更新数据」按钮。", flush=True)
    if "--no-browser" not in args:
        _open_browser_later(url)
    try:
        srv.serve_forever()
    except KeyboardInterrupt:
        pass
    finally:
        try:
            srv.server_close()
        except Exception:
            pass
        remove_pid_file()
        print("\n已停止。", flush=True)


if __name__ == "__main__":
    main()
```

### 文件 6：`index.html`（看板页面）

> 这个文件**本地版和线上版完全共用**，内容较长（含全部图表与交互逻辑，依赖同目录 `vendor/` 下的两个库文件）。
> 原样复制保存，不要改动。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>微信公众号数据看板</title>
<link rel="icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 32 32'%3E%3Crect width='32' height='32' rx='4' fill='%239c1f1f'/%3E%3Cpath d='M7 21l5-5 4 4 8-9' stroke='white' stroke-width='3' fill='none' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E" />
<!-- 图表与 Excel 导出库：本地内置，不依赖外网 -->
<script src="vendor/echarts.min.js" defer></script>
<script src="vendor/xlsx.full.min.js" defer></script>
<script src="data.js"></script>
<style>
/* ============ 设计令牌：党政机关风格 ============
   中性色为主，红色作为唯一强调色；无毛玻璃、无模糊、无常驻动画 */
:root {
  --font: -apple-system, BlinkMacSystemFont, "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "Source Han Sans SC", sans-serif;
  --red: #9c1f1f;
  --red-deep: #7d1818;
  --red-soft: #f7ecec;
  --gold: #b8860b;
  --ink: #1c2126;
  --ink-2: #4d5661;
  --ink-3: #7b8593;
  --line: #e2e6ea;
  --line-2: #eff2f5;
  --bg: #f4f5f7;
  --card: #ffffff;
  --thead: #f8f9fa;
  --hover: #f6f7f9;
  --up: #c0392b;   /* 增长：红（国内惯例） */
  --down: #1a7f37; /* 下降：绿 */
  --radius: 4px;
  --shadow: 0 1px 2px rgba(16,24,40,.05);
  --ease: cubic-bezier(.16,.84,.44,1);
}
* { box-sizing: border-box; margin: 0; padding: 0; }
html { -webkit-text-size-adjust: 100%; }
body {
  font-family: var(--font);
  background: var(--bg);
  color: var(--ink);
  font-size: 14px;
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
}
b, .num, .kpi-v, td.num { font-variant-numeric: tabular-nums; }

/* ============ 顶栏 ============ */
.topbar {
  background: var(--card);
  border-bottom: 1px solid var(--line);
  border-top: 3px solid var(--red);
  padding: 12px 20px;
  display: flex; align-items: center; gap: 14px;
  position: sticky; top: 0; z-index: 30;
}
.brand { display: flex; align-items: center; gap: 10px; min-width: 0; }
.brand .mark {
  width: 3px; height: 30px; background: var(--red); border-radius: 2px; flex: none;
}
.brand h1 { font-size: 17px; font-weight: 600; letter-spacing: .01em; white-space: nowrap; }
.brand h1 span { color: var(--red); }
.brand .sub { font-size: 12px; color: var(--ink-3); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.topbar .spacer { flex: 1; }
.meta { font-size: 12px; color: var(--ink-3); white-space: nowrap; }
.meta b { color: var(--ink-2); font-weight: 600; }
.flag {
  display: inline-block; font-size: 11px; color: var(--red-deep);
  background: var(--red-soft); border: 1px solid #ecd9d9;
  border-radius: 3px; padding: 0 6px; margin-left: 6px;
}
.tools { display: flex; gap: 8px; flex: none; }

/* ============ 按钮 ============ */
.btn {
  display: inline-flex; align-items: center; gap: 6px;
  height: 32px; padding: 0 12px;
  border: 1px solid var(--line); border-radius: var(--radius);
  background: var(--card); color: var(--ink-2);
  font-size: 13px; font-family: var(--font); cursor: pointer;
  transition: background .12s var(--ease), border-color .12s var(--ease), color .12s var(--ease);
  white-space: nowrap;
}
.btn:hover { background: var(--hover); border-color: #cfd6dd; color: var(--ink); }
.btn:active { background: #eceff2; }
.btn:disabled { opacity: .45; cursor: default; }
.btn.primary { background: var(--red); border-color: var(--red); color: #fff; }
.btn.primary:hover { background: var(--red-deep); border-color: var(--red-deep); color: #fff; }
.btn.danger { color: var(--red); border-color: #e8cfcf; }
.btn.danger:hover { background: var(--red-soft); border-color: #dcbcbc; color: var(--red-deep); }
.btn.sm { height: 26px; padding: 0 9px; font-size: 12px; }
.btn.icon { padding: 0; width: 32px; justify-content: center; }
.btn.on { background: var(--red-soft); border-color: #dcbcbc; color: var(--red-deep); }
/* 管理员门禁：未解锁时隐藏全部管理入口（回收站、删除、重命名、批量操作、保存布局） */
body:not(.admin) #btn-recycle,
body:not(.admin) #sel-delete,
body:not(.admin) th.act,
body:not(.admin) td.act,
body:not(.admin) th.pick,
body:not(.admin) td.pick,
body:not(.admin) #btn-layout-save,
body:not(.admin) #btn-layout-reset { display: none !important; }
/* 锁定 / 解锁 状态区分：解锁后用主色高亮，图标同步切换 */
#btn-lock { transition: background .18s ease, border-color .18s ease, color .18s ease; }
#btn-lock.on svg { transform: translateY(-1px); }
#btn-lock svg { transition: transform .18s ease; }
.btn svg { width: 15px; height: 15px; flex: none; }
#recycle-badge {
  width: 7px; height: 7px; border-radius: 50%;
  background: var(--red); margin-left: 3px; vertical-align: 2px;
}
.exp-opts { display: grid; gap: 8px; }
.exp-opts label {
  display: flex; align-items: center; gap: 8px;
  padding: 8px 12px; border: 1px solid var(--line); border-radius: var(--radius);
  font-size: 13px; color: var(--ink-2); cursor: pointer;
}
.exp-opts label:hover { background: var(--hover); }
.exp-opts label:has(input:checked) { border-color: #dcbcbc; background: var(--red-soft); color: var(--red-deep); }
.exp-opts input { accent-color: var(--red); }
.exp-dates { display: flex; align-items: center; gap: 8px; margin-top: 10px; }
.exp-dates input {
  flex: 1; min-width: 0; height: 34px; padding: 0 10px;
  border: 1px solid var(--line); border-radius: var(--radius);
  font-size: 13px; font-family: var(--font); color: var(--ink);
}

/* ============ 板块栅格 ============ */
.board {
  display: grid; grid-template-columns: repeat(12, minmax(0, 1fr));
  gap: 14px; padding: 16px 20px 28px; align-items: start;
}
.mod {
  background: var(--card); border: 1px solid var(--line); border-radius: var(--radius);
  box-shadow: var(--shadow); contain: layout style;
  min-width: 0;
}
.mod-head {
  display: flex; align-items: center; gap: 10px;
  padding: 11px 14px; border-bottom: 1px solid var(--line-2);
}
.mod-head h2 {
  font-size: 14px; font-weight: 600; color: var(--ink);
  display: flex; align-items: center; gap: 7px;
}
.mod-head h2::before {
  content: ""; width: 3px; height: 13px; background: var(--red); border-radius: 2px; flex: none;
}
.mod-head .note { font-size: 12px; color: var(--ink-3); }
.mod-head .spacer { flex: 1; }
.mod-body { padding: 14px; }
.editing .mod { outline: 1px dashed #d9b7b7; outline-offset: 2px; }
.editbar { display: none; gap: 6px; }
.editing .editbar { display: inline-flex; }
.editbar .btn { height: 24px; padding: 0 7px; font-size: 12px; }
/* 趋势图时间缩放：头部快捷范围 + 底部比例尺（拖动滑块自由缩放） */
.zoombar { display: flex; align-items: center; gap: 4px; flex: none; }
.zoombar .zbtn { height: 24px; padding: 0 8px; font-size: 12px; }
.zoombar .zbtn.on { background: var(--red-soft); border-color: #dcbcbc; color: var(--red-deep); }
.zoombar .wk-label { font-size: 12px; color: var(--ink-3); margin-right: 2px; }
.editing .zoombar { display: none; }
.mod[data-pid="trend"] .mod-head { flex-wrap: wrap; row-gap: 6px; }
/* 布局调整提示条 */
.layoutbar {
  display: none; align-items: center; gap: 10px; flex-wrap: wrap;
  padding: 8px 20px; font-size: 12px; color: var(--red-deep);
  background: var(--red-soft); border-bottom: 1px solid #ecd9d9;
}
.editing .layoutbar { display: flex; }
.layoutbar .tip { display: flex; align-items: center; gap: 6px; }
.layoutbar .tip svg { width: 14px; height: 14px; flex: none; }
.layoutbar .spacer { flex: 1; }
.layoutbar .btn { height: 28px; padding: 0 10px; font-size: 12px; }
@media (max-width: 900px) {
  .layoutbar { padding: 8px 14px; }
}

/* ============ 拖拽编辑（位置 + 大小） ============ */
.editing .mod { position: relative; }
.editing .mod-head { cursor: grab; touch-action: none; }
.editing .editbar .btn { cursor: pointer; }
body.dnd-active, body.dnd-active * { user-select: none !important; cursor: grabbing !important; }
.mod.drag-src { opacity: .38; }
.dnd-ghost {
  position: fixed; z-index: 90; pointer-events: none;
  display: flex; align-items: center; gap: 8px;
  background: var(--card); border: 1px solid var(--red); border-radius: var(--radius);
  box-shadow: 0 10px 28px rgba(16, 24, 40, .18);
  padding: 10px 14px; font-size: 13px; font-weight: 600; color: var(--ink); opacity: .92;
}
.dnd-ghost::before { content: ""; width: 3px; height: 13px; background: var(--red); border-radius: 2px; flex: none; }
.rs-handle {
  display: none; position: absolute; right: 0; bottom: 0; z-index: 30;
  width: 24px; height: 24px; cursor: nwse-resize; touch-action: none;
  border-bottom-right-radius: var(--radius);
  background:
    linear-gradient(135deg, transparent 46%, #c4ccd4 46%, #c4ccd4 54%, transparent 54%),
    linear-gradient(135deg, transparent 66%, #c4ccd4 66%, #c4ccd4 74%, transparent 74%),
    linear-gradient(135deg, transparent 86%, #c4ccd4 86%, #c4ccd4 94%, transparent 94%);
}
.editing .rs-handle { display: block; }
.rs-handle:hover { background-color: var(--red-soft); }
.rs-badge {
  position: fixed; z-index: 95; pointer-events: none;
  background: #26292e; color: #fff; font-size: 12px; padding: 3px 9px; border-radius: 4px;
  transform: translate(14px, 14px); white-space: nowrap;
}

/* ============ KPI ============ */
.kpis { display: grid; grid-template-columns: repeat(7, minmax(0, 1fr)); gap: 0; }
.kpi { padding: 14px 16px; border-right: 1px solid var(--line-2); }
.kpi:last-child { border-right: none; }
.kpi-l { font-size: 12px; color: var(--ink-3); display: flex; align-items: center; gap: 5px; }
.kpi-l svg { width: 13px; height: 13px; color: var(--ink-3); }
.kpi-v { font-size: 25px; font-weight: 600; line-height: 1.25; margin-top: 2px; letter-spacing: -.01em; }
.kpi-d { font-size: 12px; color: var(--ink-3); }
.kpi-v.up { color: var(--up); }
.kpi-v.down { color: var(--down); }

/* ============ 图表 ============ */
.chart { width: 100%; }
.chart-empty {
  display: flex; align-items: center; justify-content: center;
  height: 100%; min-height: 140px; color: var(--ink-3); font-size: 13px;
}

/* ============ 筛选条 ============ */
.filters {
  display: flex; align-items: center; gap: 10px; flex-wrap: wrap;
  padding: 10px 14px; border-bottom: 1px solid var(--line-2); background: #fcfcfd;
}
.field { display: flex; align-items: center; gap: 6px; font-size: 12px; color: var(--ink-2); }
.field input[type="date"], .field input[type="text"] {
  height: 30px; padding: 0 9px; border: 1px solid var(--line); border-radius: var(--radius);
  background: #fff; color: var(--ink); font-size: 13px; font-family: var(--font);
}
.field input[type="text"] { width: 210px; }
.field input:focus { outline: 2px solid rgba(156,31,31,.35); outline-offset: 0; border-color: var(--red); }
.filters .spacer { flex: 1; }
.count { font-size: 12px; color: var(--ink-3); }

/* ============ 表格 ============ */
.tablewrap { overflow-x: auto; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
thead th {
  background: var(--thead); color: var(--ink-2); font-weight: 600; font-size: 12px;
  text-align: left; padding: 9px 10px; border-bottom: 1px solid var(--line);
  white-space: nowrap; position: sticky; top: 0; z-index: 2;
}
thead th.sortable { cursor: pointer; user-select: none; }
thead th.sortable:hover { color: var(--red); }
thead th .ar { color: var(--red); font-size: 10px; margin-left: 2px; }
tbody td { padding: 9px 10px; border-bottom: 1px solid var(--line-2); vertical-align: middle; }
tbody tr:hover { background: var(--hover); }
tbody tr.sel { background: var(--red-soft); }
td.num, th.num { text-align: right; font-variant-numeric: tabular-nums; white-space: nowrap; }
td.title a { color: var(--ink); text-decoration: none; }
td.title a:hover { color: var(--red); text-decoration: underline; }
td.title { max-width: 420px; }
td.title .t { display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
td.title .manual { color: var(--red); font-size: 11px; margin-left: 4px; }
td.act { text-align: right; white-space: nowrap; }
td.act .btn { margin-left: 6px; }
input[type="checkbox"].cb { width: 15px; height: 15px; accent-color: var(--red); cursor: pointer; }
.empty { padding: 40px 14px; text-align: center; color: var(--ink-3); font-size: 13px; }
.empty .t { color: var(--ink-2); font-weight: 600; margin-bottom: 4px; }
.pager { display: flex; align-items: center; gap: 10px; padding: 10px 14px; border-top: 1px solid var(--line-2); }
.pager .spacer { flex: 1; }
.pager .info { font-size: 12px; color: var(--ink-3); }

/* 选择操作条 */
.selbar {
  display: flex; align-items: center; gap: 10px;
  padding: 8px 14px; background: var(--red-soft); border-bottom: 1px solid #ecd9d9;
  font-size: 13px; color: var(--red-deep);
}
.selbar .spacer { flex: 1; }

/* ============ 弹窗 ============ */
.overlay {
  position: fixed; inset: 0; z-index: 60;
  display: none; align-items: center; justify-content: center;
  background: rgba(16,24,40,.38); padding: 20px;
}
/* 密码框与确认框必须始终压在其他弹窗（如回收站）之上 */
#auth-overlay { z-index: 100; }
#confirm-overlay { z-index: 110; }
.overlay.open { display: flex; }
.modal {
  background: #fff; border-radius: var(--radius); width: min(460px, 100%);
  box-shadow: 0 12px 40px rgba(16,24,40,.22); max-height: 88vh; display: flex; flex-direction: column;
}
.modal header {
  display: flex; align-items: center; gap: 10px;
  padding: 13px 16px; border-bottom: 1px solid var(--line-2);
}
.modal header h3 { font-size: 15px; font-weight: 600; }
.modal header .spacer { flex: 1; }
.xbtn {
  width: 26px; height: 26px; border: none; background: none; cursor: pointer;
  color: var(--ink-3); border-radius: 3px; display: grid; place-items: center;
}
.xbtn:hover { background: var(--line-2); color: var(--ink); }
.xbtn svg { width: 15px; height: 15px; }
.mbody { padding: 16px; overflow: auto; }
.mfoot {
  display: flex; gap: 8px; justify-content: flex-end;
  padding: 12px 16px; border-top: 1px solid var(--line-2);
}
.hint { font-size: 13px; color: var(--ink-2); line-height: 1.7; }
.note-s { font-size: 12px; color: var(--ink-3); line-height: 1.7; }
.inrow { display: flex; gap: 8px; margin-top: 10px; }
.inrow input {
  flex: 1; min-width: 0; height: 34px; padding: 0 10px;
  border: 1px solid var(--line); border-radius: var(--radius);
  font-size: 14px; font-family: var(--font); color: var(--ink);
}
.inrow input:focus { outline: 2px solid rgba(156,31,31,.35); outline-offset: 0; border-color: var(--red); }
.err { color: var(--red); font-size: 12px; margin-top: 8px; display: none; }
.chk { display: flex; align-items: center; gap: 6px; margin-top: 10px; font-size: 12px; color: var(--ink-3); cursor: pointer; }
.linkbtn {
  border: none; background: none; padding: 0; cursor: pointer;
  font-size: 12px; color: var(--ink-3); text-decoration: underline; font-family: var(--font);
}
.linkbtn:hover { color: var(--red); }

/* 回收站列表 */
.blist { max-height: 46vh; overflow: auto; border: 1px solid var(--line); border-radius: var(--radius); }
.bitem {
  display: flex; align-items: center; gap: 10px;
  padding: 9px 12px; border-bottom: 1px solid var(--line-2); font-size: 13px;
}
.bitem:last-child { border-bottom: none; }
.bitem .bt { flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.bitem .bd { font-size: 12px; color: var(--ink-3); flex: none; }
.tabs { display: flex; gap: 2px; padding: 0 16px; border-bottom: 1px solid var(--line-2); }
.tabs button {
  border: none; background: none; cursor: pointer; font-family: var(--font);
  font-size: 13px; color: var(--ink-3); padding: 9px 12px;
  border-bottom: 2px solid transparent;
}
.tabs button:hover { color: var(--ink); }
.tabs button.on { color: var(--red); border-bottom-color: var(--red); font-weight: 600; }

/* ============ 提示 ============ */
.toasts { position: fixed; right: 20px; bottom: 20px; z-index: 80; display: flex; flex-direction: column; gap: 8px; }
.toast {
  display: flex; align-items: center; gap: 8px;
  background: #fff; border: 1px solid var(--line); border-left: 3px solid var(--red);
  border-radius: var(--radius); box-shadow: 0 4px 16px rgba(16,24,40,.12);
  padding: 9px 14px; font-size: 13px; color: var(--ink); max-width: 340px;
}
.toast.ok { border-left-color: var(--down); }
.toast.err { border-left-color: var(--up); }
.toast svg { width: 15px; height: 15px; flex: none; }
.toast.ok svg { color: var(--down); }
.toast.err svg { color: var(--up); }
.toast.info svg { color: var(--ink-3); }

/* ============ 响应式 ============ */
@media (max-width: 1180px) {
  .kpis { grid-template-columns: repeat(4, minmax(0, 1fr)); }
  .kpi { border-bottom: 1px solid var(--line-2); }
}
@media (max-width: 900px) {
  .board { grid-template-columns: minmax(0, 1fr); padding: 12px; }
  .mod { grid-column: span 1 !important; }
  .kpis { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .topbar { flex-wrap: wrap; row-gap: 8px; padding: 10px 14px; }
  .tools { flex-wrap: wrap; justify-content: flex-end; width: 100%; order: 2; }
  .topbar .spacer { display: none; }
  .meta { order: 3; width: 100%; }
  .field input[type="text"] { width: 150px; }
  td.title { max-width: 220px; }
  .mod-body { padding: 10px; }
  .zoombar { order: 3; width: 100%; justify-content: flex-end; }
}
@media (max-width: 480px) {
  .brand h1 { font-size: 15px; }
  .brand .sub { display: none; }
  .btn { height: 30px; padding: 0 10px; font-size: 12px; }
  .board { padding: 10px; gap: 10px; }
  .kpi { padding: 12px 10px; }
  .kpi-v { font-size: 20px; }
  .mod-head { padding: 10px 12px; flex-wrap: wrap; }
  .modal { width: calc(100vw - 24px); }
}
@media (prefers-reduced-motion: reduce) {
  * { transition: none !important; animation: none !important; }
}
</style>
</head>
<body>

<header class="topbar">
  <div class="brand">
    <span class="mark" aria-hidden="true"></span>
    <div>
      <h1>微信公众号数据看板</h1>
      <div class="sub">阅读 · 分享 · 粉丝 · 文章管理</div>
    </div>
  </div>
  <div class="spacer"></div>
  <div class="meta">数据更新时间：<b id="gen-time">—</b><span id="upstream-tag" style="display:none;margin-left:10px;cursor:pointer" title="点击立即同步到线上">· 线上：<b id="upstream-state">—</b></span></div>
  <div class="tools">
    <button class="btn primary" id="btn-update" title="上传数据文件（需密码）">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><path d="M17 8l-5-5-5 5"/><path d="M12 3v12"/></svg>
      更新数据
    </button>
    <button class="btn" id="btn-export" title="导出 Excel（需密码）">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><path d="M14 2v6h6"/><path d="M8 13h8M8 17h5"/></svg>
      导出
    </button>
    <button class="btn" id="btn-recycle" title="回收站">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M8 6V4a1 1 0 0 1 1-1h6a1 1 0 0 1 1 1v2"/><path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6"/></svg>
      回收站<span id="recycle-badge" style="display:none"></span>
    </button>
    <button class="btn" id="btn-layout" title="调整布局（无需密码，仅本次查看有效；需长期生效可保存为默认布局，保存时需密码）">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg>
      布局
    </button>
    <button class="btn" id="btn-lock" title="点击输入管理员密码，解锁上传数据、回收站等管理功能">
      <svg id="lock-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/></svg>
      <span id="lock-text">锁定</span>
    </button>
  </div>
</header>

<div class="layoutbar" id="layoutbar">
  <span class="tip">
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"/><path d="M12 16v-4"/><path d="M12 8h.01"/></svg>
    <span>当前为<strong>临时布局</strong>：<strong>拖动板块标题栏换位置，拖右下角手柄调大小</strong>，仅本次查看有效，关闭或重新打开页面后恢复默认。</span>
  </span>
  <div class="spacer"></div>
  <button class="btn primary" id="btn-layout-save">保存为默认布局（需密码）</button>
  <button class="btn" id="btn-layout-reset">恢复默认布局（需密码）</button>
  <button class="btn" id="btn-layout-cancel">放弃调整</button>
</div>

<main class="board" id="board">

  <section class="mod" data-pid="kpis" style="grid-column: span 12">
    <div class="mod-head">
      <h2>关键指标</h2>
      <span class="note" id="kpi-note"></span>
      <div class="spacer"></div>
      <div class="zoombar" id="week-zoom" role="group" aria-label="周统计口径">
        <span class="wk-label">周口径</span>
        <button class="btn sm zbtn on" data-week="roll7">近7天</button>
        <button class="btn sm zbtn" data-week="mon">本周</button>
      </div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>
    <div class="kpis" id="kpis"></div>
  </section>

  <section class="mod" data-pid="trend" style="grid-column: span 8">
    <div class="mod-head">
      <h2>阅读与分享趋势</h2>
      <span class="note">按天统计</span>
      <div class="spacer"></div>
      <div class="zoombar" id="trend-zoom" role="group" aria-label="趋势图时间范围">
        <button class="btn sm zbtn on" data-zoom="all">全部</button>
        <button class="btn sm zbtn" data-zoom="90">近90天</button>
        <button class="btn sm zbtn" data-zoom="30">近30天</button>
        <button class="btn sm zbtn" data-zoom="7">近7天</button>
      </div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>
    <div class="mod-body"><div class="chart" id="c-trend" style="height:300px"></div></div>
  </section>

  <section class="mod" data-pid="source" style="grid-column: span 4">
    <div class="mod-head">
      <h2>阅读来源</h2>
      <div class="spacer"></div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>
    <div class="mod-body"><div class="chart" id="c-source" style="height:250px"></div></div>
  </section>

  <section class="mod" data-pid="fans" style="grid-column: span 6">
    <div class="mod-head">
      <h2>粉丝增长</h2>
      <div class="spacer"></div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>
    <div class="mod-body"><div class="chart" id="c-fans" style="height:250px"></div></div>
  </section>

  <section class="mod" data-pid="rank" style="grid-column: span 6">
    <div class="mod-head">
      <h2>文章阅读 Top 10</h2>
      <div class="spacer"></div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>
    <div class="mod-body"><div class="chart" id="c-rank" style="height:300px"></div></div>
  </section>

  <section class="mod" data-pid="table" style="grid-column: span 12">
    <div class="mod-head">
      <h2>文章明细</h2>
      <div class="spacer"></div>
      <div class="editbar">
        <button class="btn sm" data-move="up">上移</button>
        <button class="btn sm" data-move="down">下移</button>
      </div>
    </div>

    <div class="filters">
      <label class="field">起
        <input type="date" id="f-start" />
      </label>
      <label class="field">止
        <input type="date" id="f-end" />
      </label>
      <label class="field">
        <input type="text" id="f-q" placeholder="搜索标题（支持拼音首字母）" />
      </label>
      <button class="btn" id="f-reset">重置</button>
      <div class="spacer"></div>
      <span class="count" id="f-count">—</span>
    </div>

    <div class="selbar" id="selbar" style="display:none">
      已选 <b id="sel-count">0</b> 篇
      <div class="spacer"></div>
      <button class="btn sm danger" id="sel-delete">删除所选</button>
      <button class="btn sm" id="sel-clear">取消选择</button>
    </div>

    <div class="tablewrap">
      <table>
        <thead>
          <tr>
            <th class="pick" style="width:34px"><input type="checkbox" class="cb" id="cb-all" title="全选本页" /></th>
            <th class="sortable num" data-key="pubdate">发布日期<span class="ar" data-ar="pubdate"></span></th>
            <th class="sortable" data-key="title">标题<span class="ar" data-ar="title"></span></th>
            <th class="sortable num" data-key="read">阅读<span class="ar" data-ar="read"></span></th>
            <th class="sortable num" data-key="share">分享<span class="ar" data-ar="share"></span></th>
            <th class="sortable num" data-key="fav">收藏<span class="ar" data-ar="fav"></span></th>
            <th class="sortable num" data-key="like">点赞<span class="ar" data-ar="like"></span></th>
            <th class="sortable num" data-key="zaikan">在看<span class="ar" data-ar="zaikan"></span></th>
            <th class="sortable num" data-key="comment">留言<span class="ar" data-ar="comment"></span></th>
            <th class="act" style="text-align:right">操作</th>
          </tr>
        </thead>
        <tbody id="tbody"></tbody>
      </table>
    </div>
    <div id="table-empty" class="empty" style="display:none">
      <div class="t">暂无数据</div>
      <div id="empty-desc">点击右上角「更新数据」上传最新的数据文件</div>
    </div>

    <div class="pager">
      <button class="btn sm" id="pg-prev">上一页</button>
      <button class="btn sm" id="pg-next">下一页</button>
      <div class="spacer"></div>
      <span class="info" id="pg-info">—</span>
    </div>
  </section>

</main>

<!-- 密码验证 -->
<div class="overlay" id="auth-overlay" role="dialog" aria-modal="true" aria-labelledby="auth-title">
  <div class="modal" style="width:min(420px,100%)">
    <header>
      <h3 id="auth-title">需要验证</h3>
      <div class="spacer"></div>
      <button class="xbtn" id="auth-close" aria-label="关闭">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
      </button>
    </header>
    <div class="mbody">
      <p class="hint" id="auth-hint">为保障数据安全，该操作需要验证管理密码。</p>
      <div class="inrow">
        <input type="password" id="auth-pwd" placeholder="请输入操作密码" autocomplete="current-password" />
        <button class="btn primary" id="auth-ok">确定</button>
      </div>
      <p class="err" id="auth-err"></p>
      <label class="chk" style="display:none"><input type="checkbox" id="auth-remember" /> 本次会话内不再提示（关闭页面后失效）</label>
    </div>
  </div>
</div>

<!-- 上传数据 -->
<div class="overlay" id="upload-overlay" role="dialog" aria-modal="true" aria-labelledby="up-title">
  <div class="modal">
    <header>
      <h3 id="up-title">更新数据</h3>
      <div class="spacer"></div>
      <button class="xbtn" id="up-close" aria-label="关闭">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
      </button>
    </header>
    <div class="mbody">
      <p class="hint" id="up-hint">选择最新的数据文件（data.js），上传后页面将自动刷新并使用新数据。</p>
      <p class="note-s" id="up-note" style="margin-top:6px">上传的数据保存在当前浏览器，刷新后依然生效；换设备打开仍显示最初部署的数据。</p>
      <div class="mfoot" style="padding:14px 0 0;border:none;justify-content:flex-start">
        <button class="btn primary" id="up-pick">选择数据文件</button>
        <button class="btn" id="up-latest">获取最新版本</button>
        <button class="btn" id="up-sync" style="display:none">从微信服务器采集</button>
      </div>
      <p style="margin-top:10px"><button class="linkbtn" id="up-reset" style="display:none">恢复为最初部署的数据</button></p>
    </div>
  </div>
</div>
<input type="file" id="upload-data-file" accept=".js,.json,.txt,application/json,text/plain" style="display:none" />

<!-- 重命名 -->
<div class="overlay" id="rename-overlay" role="dialog" aria-modal="true">
  <div class="modal" style="width:min(420px,100%)">
    <header>
      <h3>重命名文章</h3>
      <div class="spacer"></div>
      <button class="xbtn" id="rn-close" aria-label="关闭">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
      </button>
    </header>
    <div class="mbody">
      <p class="note-s" id="rn-origin" style="margin-bottom:8px"></p>
      <div class="inrow"><input type="text" id="rn-input" placeholder="输入新的文章标题" /></div>
      <p class="note-s" style="margin-top:8px">重命名仅保存在本机浏览器，用于看板显示，不影响公众号原文。</p>
    </div>
    <div class="mfoot">
      <button class="btn" id="rn-cancel">取消</button>
      <button class="btn primary" id="rn-save">保存</button>
    </div>
  </div>
</div>

<!-- 导出数据（时间范围选择） -->
<div class="overlay" id="export-overlay" role="dialog" aria-modal="true">
  <div class="modal" style="width:min(420px,100%)">
    <header>
      <h3>导出数据</h3>
      <div class="spacer"></div>
      <button class="xbtn" id="exp-close" aria-label="关闭">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
      </button>
    </header>
    <div class="mbody">
      <div class="exp-opts">
        <label><input type="radio" name="exp-range" value="all" checked /> 全部数据</label>
        <label><input type="radio" name="exp-range" value="d1" /> 最近一天</label>
        <label><input type="radio" name="exp-range" value="d7" /> 最近七天</label>
        <label><input type="radio" name="exp-range" value="m1" /> 最近一个月</label>
        <label><input type="radio" name="exp-range" value="custom" /> 自定义时间</label>
      </div>
      <div class="exp-dates" id="exp-dates" style="display:none">
        <input type="date" id="exp-start" />
        <span style="color:var(--ink-3)">至</span>
        <input type="date" id="exp-end" />
      </div>
      <p class="note-s" style="margin-top:10px">按文章发布日期筛选导出；「最近一天 / 七天 / 一个月」以数据中的最新发布日期为基准往前推。</p>
    </div>
    <div class="mfoot">
      <button class="btn" id="exp-cancel">取消</button>
      <button class="btn primary" id="exp-ok">导出</button>
    </div>
  </div>
</div>

<!-- 回收站 -->
<div class="overlay" id="recycle-overlay" role="dialog" aria-modal="true">
  <div class="modal" style="width:min(620px,100%)">
    <header>
      <h3>回收站</h3>
      <span class="note-s" id="rb-sub"></span>
      <div class="spacer"></div>
      <button class="xbtn" id="rb-close" aria-label="关闭">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M18 6L6 18M6 6l12 12"/></svg>
      </button>
    </header>
    <div class="mbody">
      <div class="blist" id="rb-list"></div>
      <p class="note-s" style="margin-top:8px">还原与彻底删除均需验证管理密码。</p>
    </div>
  </div>
</div>

<!-- 通用确认 -->
<div class="overlay" id="confirm-overlay" role="dialog" aria-modal="true">
  <div class="modal" style="width:min(400px,100%)">
    <header><h3 id="cf-title">确认操作</h3></header>
    <div class="mbody"><p class="hint" id="cf-msg"></p></div>
    <div class="mfoot">
      <button class="btn" id="cf-no">取消</button>
      <button class="btn danger" id="cf-yes">确定</button>
    </div>
  </div>
</div>

<div class="toasts" id="toasts" aria-live="polite"></div>

<script>
(function () {
'use strict';

/* ================= 常量 ================= */
/* 管理密码不再保存在页面里：解锁走服务端 POST /api/auth，成功后获得临时 token */
var DATA_OVERRIDE_KEY = 'wechat_data_override_v1';  /* 本机上传的数据覆盖 */
var TITLES_KEY = 'wechat_titles_v1';                /* 本机重命名 */
var DEL_KEY = 'wechat_deleted_v1';                  /* 本机删除 */
var LAYOUT_KEY = 'gzh_layout_v1';                   /* 板块布局（永久） */
var LAYOUT_TMP_KEY = 'gzh_layout_tmp_v1';           /* 板块布局（仅本次查看） */
var AUTH_KEY = 'gzh_auth_session';                  /* 会话内解锁状态 */
var AUTH_TOKEN_KEY = 'gzh_auth_token_v1';           /* 服务端下发的管理令牌 */
var usingUploaded = false;

/* ================= 数据校验（上传文件用） ================= */
function validateData(d) {
  if (!d || typeof d !== 'object') return '数据不是有效的对象';
  if (!d.articles || typeof d.articles !== 'object') return '缺少 articles（文章总览）字段';
  if (!d.article_daily || typeof d.article_daily !== 'object' || !Object.keys(d.article_daily).length) return '缺少 article_daily（文章日报）字段或数据为空';
  if (!d.biz_daily || typeof d.biz_daily !== 'object') return '缺少 biz_daily（公众号日报）字段';
  if (!d.user_daily || typeof d.user_daily !== 'object') return '缺少 user_daily（粉丝日报）字段';
  if (!d.generated_at) return '缺少 generated_at（数据生成时间）字段';
  return null;
}
/* 启动时：若本机上传过数据文件，用它覆盖内置数据 */
(function applyUploadedData() {
  try {
    var raw = localStorage.getItem(DATA_OVERRIDE_KEY);
    if (!raw) return;
    var o = JSON.parse(raw);
    if (!o || !o.data || validateData(o.data)) { localStorage.removeItem(DATA_OVERRIDE_KEY); return; }
    window.DATA = o.data;
    if (o.generated_at) window.DATA_GENERATED_AT = o.generated_at;
    usingUploaded = true;
  } catch (e) {}
})();

/* ================= 数据 ================= */
var D = window.DATA || {};
var articles = D.articles || {};
var articleTotal = D.article_total || {};
var articleDaily = D.article_daily || {};
var bizDaily = D.biz_daily || {};
var userDaily = D.user_daily || {};
var serverManual = D.manual_titles || {};
var serverDeleted = D.deleted_articles || [];

/* ================= 本地存储 ================= */
function LS(k, d) { try { var v = localStorage.getItem(k); return v ? JSON.parse(v) : d; } catch (e) { return d; } }
function LSS(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) {} }
function SS(k, d) { try { var v = sessionStorage.getItem(k); return v ? JSON.parse(v) : d; } catch (e) { return d; } }
function SSS(k, v) { try { sessionStorage.setItem(k, JSON.stringify(v)); } catch (e) {} }
function SSDEL(k) { try { sessionStorage.removeItem(k); } catch (e) {} }
function LSDEL(k) { try { localStorage.removeItem(k); } catch (e) {} }
var localTitles = LS(TITLES_KEY, {});
var localDeleted = LS(DEL_KEY, {});
var autoBinChanged = false;   /* 启动时有文章因标题命中回收站被自动移入（待云端探测后推送） */
function manualTitle(m) { return serverManual[m] || localTitles[m] || ''; }

/* ================= 工具 ================= */
function $(id) { return document.getElementById(id); }
function int(n) { return n == null ? 0 : Math.round(n); }
function fmt(n) {
  n = n || 0;
  if (Math.abs(n) >= 100000000) return (n / 100000000).toFixed(2) + '亿';
  if (Math.abs(n) >= 10000) return (n / 10000).toFixed(1) + '万';
  return String(n);
}
function esc(s) {
  return String(s == null ? '' : s).replace(/[&<>"']/g, function (c) {
    return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c];
  });
}
function shortDate(d) { return d.slice(5); }
var ICON_OK = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round" stroke-linejoin="round"><path d="M20 6L9 17l-5-5"/></svg>';
var ICON_ERR = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><circle cx="12" cy="12" r="10"/><path d="M12 8v5M12 17h.01"/></svg>';
var ICON_INFO = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.4" stroke-linecap="round"><circle cx="12" cy="12" r="10"/><path d="M12 16v-5M12 8h.01"/></svg>';
function toast(msg, type) {
  type = type || 'info';
  var icon = type === 'ok' ? ICON_OK : type === 'err' ? ICON_ERR : ICON_INFO;
  var el = document.createElement('div');
  el.className = 'toast ' + type;
  el.innerHTML = icon + '<span>' + esc(msg) + '</span>';
  $('toasts').appendChild(el);
  setTimeout(function () { el.style.opacity = '0'; el.style.transition = 'opacity .2s'; }, 2600);
  setTimeout(function () { if (el.parentNode) el.parentNode.removeChild(el); }, 2900);
}

/* ================= 弹窗 ================= */
function openOverlay(id) { $(id).classList.add('open'); }
function closeOverlay(id) { $(id).classList.remove('open'); }
Array.prototype.forEach.call(document.querySelectorAll('.overlay'), function (ov) {
  ov.addEventListener('click', function (e) { if (e.target === this) closeOverlay(this.id); });
});
var confirmCb = null;
function askConfirm(title, msg, cb) {
  $('cf-title').textContent = title;
  $('cf-msg').textContent = msg;
  confirmCb = cb;
  openOverlay('confirm-overlay');
}
$('cf-no').addEventListener('click', function () { closeOverlay('confirm-overlay'); confirmCb = null; });
$('cf-yes').addEventListener('click', function () {
  closeOverlay('confirm-overlay');
  var cb = confirmCb; confirmCb = null;
  if (cb) cb();
});

/* ================= 密码门禁（服务端验证） ================= */
function isUnlocked() { try { return sessionStorage.getItem(AUTH_KEY) === '1'; } catch (e) { return false; } }
function authToken() { try { return sessionStorage.getItem(AUTH_TOKEN_KEY) || ''; } catch (e) { return ''; } }
function setAuthSession(token) {
  try {
    sessionStorage.setItem(AUTH_KEY, '1');
    if (token) sessionStorage.setItem(AUTH_TOKEN_KEY, token);
    else sessionStorage.removeItem(AUTH_TOKEN_KEY);
  } catch (e) {}
}
function clearAuthSession() {
  try { sessionStorage.removeItem(AUTH_KEY); sessionStorage.removeItem(AUTH_TOKEN_KEY); } catch (e) {}
}
var authCb = null;
function requireAuth(title, cb) {
  if (isUnlocked()) { cb(); return; }
  authCb = cb;
  var rem = $('auth-remember');
  if (rem && rem.parentNode) rem.parentNode.style.display = '';   /* 解锁场景会隐藏该选项，这里恢复 */
  if (rem) rem.checked = false;
  $('auth-title').textContent = title || '需要验证';
  $('auth-hint').textContent = '为保障数据安全，该操作需要在服务端验证管理密码。';
  $('auth-pwd').value = '';
  $('auth-err').style.display = 'none';
  $('auth-err').textContent = '';
  openOverlay('auth-overlay');
  setTimeout(function () { $('auth-pwd').focus(); }, 60);
}
var authBusy = false;
function submitAuth() {
  var v = $('auth-pwd').value;
  var err = $('auth-err');
  if (!v) { err.textContent = '请输入操作密码'; err.style.display = ''; $('auth-pwd').focus(); return; }
  if (authBusy) return;
  authBusy = true;
  /* 密码只发给服务端比对，页面本身不保存密码 */
  fetch('/api/auth', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ pwd: v })
  }).then(function (r) {
    return r.json().catch(function () { return { ok: false, message: '服务响应异常' }; })
      .then(function (j) { j._status = r.status; return j; });
  }).then(function (j) {
    authBusy = false;
    if (j && j.ok && j.token) {
      setAuthSession(j.token);
      $('auth-pwd').value = '';
      closeOverlay('auth-overlay');
      updateLockBtn();
      var cb = authCb; authCb = null;
      if (cb) cb();
    } else if (j && j._status === 404) {
      err.textContent = '当前为静态访问，管理功能需通过服务端打开（线上链接或本地 run.sh）';
      err.style.display = '';
    } else {
      err.textContent = (j && j.message) || '密码不正确，请重新输入';
      err.style.display = '';
      $('auth-pwd').value = '';
      $('auth-pwd').focus();
    }
  }).catch(function () {
    authBusy = false;
    err.textContent = '无法连接服务端：管理功能需通过服务端访问';
    err.style.display = '';
  });
}
$('auth-ok').addEventListener('click', submitAuth);
$('auth-pwd').addEventListener('keydown', function (e) {
  if (e.key === 'Enter') { e.preventDefault(); submitAuth(); }
  else if (e.key === 'Escape') { closeOverlay('auth-overlay'); }
});
$('auth-close').addEventListener('click', function () { closeOverlay('auth-overlay'); authCb = null; });

var LOCK_SVG = '<rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/>';
var UNLOCK_SVG = '<rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0 0 9.9-.8"/>';
/* 未解锁 = 只读模式：回收站、删除、重命名、批量操作、保存布局等管理入口全部隐藏 */
function applyAdminVisibility() {
  document.body.classList.toggle('admin', isUnlocked());
}
function updateLockBtn() {
  var on = isUnlocked();
  $('lock-text').textContent = on ? '已解锁' : '锁定';
  $('btn-lock').classList.toggle('on', on);
  $('btn-lock').title = on ? '管理功能已开启，点击立即锁定'
                           : '点击输入管理员密码，解锁上传数据、回收站等管理功能';
  $('btn-lock').setAttribute('aria-pressed', on ? 'true' : 'false');
  var ic = $('lock-icon');
  if (ic) ic.innerHTML = on ? UNLOCK_SVG : LOCK_SVG;
  applyAdminVisibility();
  var ub = $('btn-update');
  if (ub) ub.title = on ? '上传新数据，或获取服务端最新版本' : '立即获取服务端最新数据';
}
$('btn-lock').addEventListener('click', function () {
  if (isUnlocked()) {
    clearAuthSession();
    setEditMode(false);
    updateLockBtn();
    toast('已锁定，管理功能已关闭', 'ok');
    return;
  }
  /* 未解锁：管理员密码统一在此输入，由服务端验证 */
  authCb = function () {
    updateLockBtn();
    toast('已解锁，可上传数据并使用回收站等管理功能', 'ok');
  };
  $('auth-title').textContent = '管理员解锁';
  $('auth-hint').textContent = '输入管理员密码后，可上传新数据、使用回收站并管理文章。';
  $('auth-pwd').value = '';
  $('auth-err').style.display = 'none';
  $('auth-err').textContent = '';
  var rem = $('auth-remember');
  if (rem) rem.checked = true;
  if (rem && rem.parentNode) rem.parentNode.style.display = 'none';   /* 解锁即长期保持，无需再选 */
  openOverlay('auth-overlay');
  setTimeout(function () { $('auth-pwd').focus(); }, 60);
});

/* ================= 数据推导 ================= */
var dateSet = {};
Object.keys(articleDaily).forEach(function (d) { dateSet[d] = 1; });
Object.keys(bizDaily).forEach(function (d) { dateSet[d] = 1; });
Object.keys(userDaily).forEach(function (d) { dateSet[d] = 1; });
var dates = Object.keys(dateSet).sort();

function dailyArticleSum(date) {
  var o = { ru: 0, su: 0 };
  var day = articleDaily[date] || {};
  Object.keys(day).forEach(function (m) {
    o.ru += int(day[m].read_user); o.su += int(day[m].share_user);
  });
  return o;
}
var trend = dates.map(function (d) {
  var biz = bizDaily[d] || {}, s = dailyArticleSum(d);
  return { date: d, ru: biz.read_user != null ? int(biz.read_user) : s.ru,
           su: biz.share_user != null ? int(biz.share_user) : s.su };
});
var fans = dates.map(function (d) {
  var u = userDaily[d] || {};
  return { date: d, cum: u.cumulate_user == null ? null : int(u.cumulate_user),
           newer: int(u.new_user), cancel: int(u.cancel_user) };
});
var sourceAgg = {};
dates.forEach(function (d) {
  var rs = (bizDaily[d] || {}).read_source;
  if (rs) Object.keys(rs).forEach(function (k) { sourceAgg[k] = (sourceAgg[k] || 0) + int(rs[k]); });
});
if (!Object.keys(sourceAgg).length) {
  dates.forEach(function (d) {
    var day = articleDaily[d] || {};
    Object.keys(day).forEach(function (m) {
      var rs = day[m].read_source || {};
      Object.keys(rs).forEach(function (k) { sourceAgg[k] = (sourceAgg[k] || 0) + int(rs[k]); });
    });
  });
}
var dailyByArticle = {};
/* 真实首发日期：取该文章在「文章日报」里最早出现的那天。
   采集端早期版本用 setdefault 记录 first_date，一旦写入就不再更新，
   导致部分老文章被错标成最近几天的日期，这里以日报为准做校正。 */
var firstSeen = {};
Object.keys(articleDaily).forEach(function (d) {
  Object.keys(articleDaily[d]).forEach(function (m) {
    var x = articleDaily[d][m];
    var a = dailyByArticle[m] || (dailyByArticle[m] = { read: 0, share: 0, fav: 0 });
    a.read += int(x.read_user); a.share += int(x.share_user); a.fav += int(x.collection_user);
    if (!firstSeen[m] || d < firstSeen[m]) firstSeen[m] = d;
  });
});
/* 统一取发布日期：日报校正值优先，其次采集端 first_date */
function pubdateOf(m) {
  return firstSeen[m] || ((articles[m] || {}).first_date) || '';
}

/* 拼音首字母（模糊搜索用） */
var PY_INIT_BUCKETS = {a:"啊阿埃挨哎唉哀皑癌蔼矮艾碍爱隘鞍氨安俺按暗岸胺案肮昂盎凹敖熬翱袄傲奥懊澳谙坳埯拗捱揞嗷嗄嗳嗌岙犴庵廒遨媪嫒骜瑷桉獒暧砹铵锕锿鹌聱螯霭鏊鳌鏖黯",b:"芭捌扒叭吧笆八疤巴拔跋靶把耙坝霸罢爸白柏百摆佰败拜稗斑班搬扳般颁板版扮拌伴瓣半办绊邦帮梆榜膀绑棒磅蚌镑傍谤苞胞包褒剥薄雹保堡饱宝抱报暴豹鲍爆杯碑悲卑北辈背贝钡倍狈备惫焙被奔苯本笨崩绷甭泵蹦迸逼鼻比鄙笔彼碧蓖蔽毕毙毖币庇痹闭敝弊必壁臂避陛鞭边编贬扁便变卞辨辩辫遍标彪膘表鳖憋别瘪彬斌濒滨宾摈兵冰柄丙秉饼炳病并玻菠播拨钵波博勃搏铂箔伯帛舶脖膊渤驳捕卜哺补埠不布步簿部怖匕孛匾俾傧勹亳禀冫阪陂邴邶弁畚坌坂苄茇荜荸菝萆葆蓓蒡薜捭摒卟吡呗哔啵嘣岜豳狴饽庳忭悖愎汴浜滗濞灞逋弼妣婊婢嬖孢骠缏缤玢璧杓槟檗殡瓿甏晡贲掰擘膑飑飙飚煲煸灬砭碚碥礴畀钚钣钯钸钹铋锛镔镖镳秕鸨鹁鹎瘢瘭癍窆裱裨褙褓褊蝙笾筚箅篦簸舭舨襞粑醭蹩趵跛跸踣龅鐾鲅鳊鳔鞴髀髌魃髟鬓",c:"擦猜裁材才财睬踩采彩菜蔡餐参蚕残惭惨灿苍舱仓沧藏操糙槽曹草厕策侧册测层蹭插叉茬茶查碴搽察岔差诧拆柴豺搀掺蝉馋谗缠铲产阐颤昌猖场尝常偿肠厂敞畅唱倡超抄钞朝嘲潮巢吵炒车扯撤掣彻澈郴臣辰尘晨忱沉陈趁衬撑称城橙成呈乘程惩澄诚承逞骋秤吃痴持池迟弛驰耻齿侈尺赤翅斥炽充冲虫崇宠抽酬畴踌稠愁筹仇绸瞅丑臭初出橱厨躇锄雏滁除楚础储矗搐触处揣川穿椽传船喘串疮窗幢床闯创吹炊捶锤垂春椿醇唇淳纯蠢戳绰疵茨磁雌辞慈瓷词此刺赐次聪葱囱匆从丛凑粗醋簇促蹿篡窜摧崔催脆瘁粹淬翠村存寸磋撮搓措挫错伺畜曾椎亍丞厝伥伧侪俦傺汆冁谄谌谶陲刍鬯坼坻埕墀艹苌苁茌茈茺莼菖萃蒇蔟抻搋撺叱呲哧啐啜啻嗔嗤嘈嘬噌嚓帱岑嵯彳徂徜猝猹舛饬馇廛忖忏忡怅怆怵怊恻惝惆悴憧憷阊汊涔淙漕潺澶宸辶遄孱羼屮姹娼婵媸嫦骖骢骣绌巛琮琛璀璁璨杈杵枨枞柽楮棰槌榇槎榱樗檫殂辍辏昶晟晁觇毳氅氚敕脞腠塍膪焯爨祠禅砗碜礤眵瞠钏钗铖铳锉锸镡镩镲矬鸱鹑鹚痤瘥瘛瘳衩裎褚褫皴耖虿蚩蛏蜍蝽螭螬蟾笞篪舂舡艚艟粲糍酢酲鹾蹙踔踟踹蹉蹰蹴躔龀龊雠鲳魑黜黪",d:"搭达答瘩打大呆歹傣戴带殆代贷袋待逮怠耽担丹单郸掸胆旦氮但惮淡诞弹蛋当挡党荡档刀捣蹈倒岛祷导到稻悼道盗德得的蹬灯登等瞪凳邓堤低滴迪敌笛狄涤翟嫡抵底地蒂第帝弟递缔颠掂滇碘点典靛垫电佃甸店惦奠淀殿碉叼雕凋刁掉吊钓调跌爹碟蝶迭谍叠丁盯叮钉顶鼎锭定订丢东冬董懂动栋侗恫冻洞兜抖斗陡豆逗痘都督毒犊独读堵睹赌杜镀肚度渡妒端短锻段断缎堆兑队对墩吨蹲敦顿囤钝盾遁掇哆多夺垛躲朵跺舵剁惰堕氐刂仃儋籴诋谛谠阽邸凼坫垤垌埭堞芏荻萏菪蔸耷揲甙叨呔咚咄哒哚啖啶喋嗒嘟嗲嘀噔岽岱峒嶝巅忉怛沌渎澹宕迨妲娣骀绐缍玎玷玳柢柁椟棣椴殚戥赕觌氘氡牍牒胨胴腚炖煅焘怼憝沓砀砘砥硐碓碇碡碲礅磴盹眈睇钿铎铛铞铤铥铫锝镝镦镫瓞鸫疔疸瘅癜癫窦裆裰褡耋耵聃蚪蠹笃笪箪篼簖簟簦羝纛酊趸踮蹀踱貂鲷鲽靼鞑骶髑黛黩",e:"蛾峨鹅俄额讹娥恶厄扼遏鄂饿恩而儿耳尔饵洱二贰噩佴诶谔垩苊莪萼蒽摁呃愕阏迩屙婀珥轭腭铒锇锷鸸鹗颚鲕鳄",f:"发罚筏伐乏阀法珐藩帆番翻樊矾钒繁凡烦反返范贩犯饭泛坊芳方肪房防妨仿访纺放菲非啡飞肥匪诽吠肺废沸费芬酚吩氛分纷坟焚汾粉奋份忿愤粪丰封枫蜂峰锋风疯烽逢冯缝讽奉凤佛否夫敷肤孵扶拂辐幅氟符伏俘服浮涪福袱弗甫抚辅俯釜斧腑府腐赴副覆赋复傅付阜父腹负富讣附妇缚咐匚俸偾匐凫阝邡郛酆垡芙芾苻茯莩菔葑蕃蘩拊呋唪幞幡狒怫悱沣淝滏瀵艴妃孚驸绂绋绯枋梵桴棼榧赙腓斐燔扉祓砜砝砩黻黼畈罘钫镄稃馥痱蚨蜉蜚蝠蝮缶篚舫翡麸趺跗蹯霏鲂鲋鲱鲼鳆鼢",g:"噶嘎该改概钙盖溉干甘杆柑竿肝赶感秆敢赣冈刚钢缸肛纲岗港杠篙皋高膏羔糕搞镐稿告哥歌搁戈鸽胳疙割革葛格阁隔铬个各给根跟耕更庚羹埂耿梗工攻功恭龚供躬公宫弓巩汞拱贡共钩勾沟苟狗垢构购够辜菇咕箍估沽孤姑鼓古蛊骨谷股故顾固雇刮瓜剐寡挂褂乖拐怪棺关官冠观管馆罐惯灌贯光广逛瑰规圭硅归龟闺轨鬼诡癸桂柜跪贵刽辊滚棍锅郭国果裹过咯傀炔丐亘鬲丨睾嘏匦卦刿仡佝伽倌馘衮诂诖诟诰陔郜哿圪坩垓埚塥苷茛莞菰藁廾尬尴掴掼擀呙呷呱咣哏哽嗝帼岣崮崞犷猓庋赓汩泔淦涫澉宄遘妫媾尕尜纥绀绠绲缑缟珙枸桄栝桧梏椁槔槁橄轱戤旮旰杲晷赅觏牯牿搿虢肱胍胱膈臌彀毂戆矸硌磙瞽罡罟盥钆钴锆锢镉皈鸪鸹鹄鹳疳痼袼聒虼蚣蛄蜾蝈笱筻篝簋舸艮酐酤觚觥鲑鲠鲧鲴鳏鳜鞲鹘骼",h:"蛤哈骸孩海氦亥害骇酣憨邯韩含涵寒函喊罕翰撼捍旱憾悍焊汗汉夯杭航壕嚎豪毫郝好耗号浩呵喝荷菏核禾和何合盒貉阂河涸赫褐鹤贺嘿黑痕很狠恨哼亨横衡恒轰哄烘虹鸿洪宏弘红喉侯猴吼厚候后呼乎忽瑚壶葫胡蝴狐糊湖弧虎唬护互沪户花哗华猾滑画划化话槐徊怀淮坏欢环桓还缓换患唤痪豢焕涣宦幻荒慌黄磺蝗簧皇凰惶煌晃幌恍谎灰挥辉徽恢蛔回毁悔慧卉惠晦贿秽会烩汇讳诲绘荤昏婚魂浑混豁活伙火获或惑霍货祸劐黉訇冱讧诃诙诨隍邗郇奂劾壑堠茴荟荭萑菡蒿蕙蕻薨薅藿蘅撖擐攉咴哕唿喙嗬嗥嗨嚆嚯囫圜岵後徨猢獾夥馄怙惚隳闳阍阖阚沆泓洹洄浍浒浣湟溷潢滹漶濠瀚灏寰逅逭遑彗骅绗缋缳珩珲琥璜桦桁槲轷昊曷晖晗肓胲觳烀焓煳戽扈祜恚砉盍钬铧铪锪锾镬皓瓠鹕鹱瘊癀耠顸颃颌颔颢虍虺蚝蚶蚵蟥蟪蠖笏篁篌糇翮醐醢踝斛鲎鲩鳇骺鬟麾鼾",j:"击圾基机畸稽积箕肌饥迹激讥鸡姬绩缉吉极棘辑籍集及急疾汲即嫉级挤几脊己蓟技冀季伎祭剂悸济寄寂计记既忌际妓继纪嘉枷夹佳家加荚颊贾甲钾假稼价架驾嫁歼监坚尖笺间煎兼肩艰奸缄茧检柬碱硷拣捡简俭剪减荐鉴践贱见键箭件健舰剑饯渐溅涧建僵姜将浆江疆蒋桨奖讲匠酱降蕉椒礁焦胶交郊浇骄娇嚼搅铰矫侥脚狡角饺缴绞剿教酵轿较叫窖揭接皆秸街阶截劫节桔杰捷睫竭洁结解姐戒藉芥界借介疥诫届巾筋斤金今津襟紧锦仅谨进靳晋禁近烬浸尽劲荆兢茎睛晶鲸京惊精粳经井警景颈静境敬镜径痉靖竟竞净炯窘揪究纠玖韭久灸九酒厩救旧臼舅咎就疚鞠拘狙疽居驹菊局咀矩举沮聚拒据巨具距踞锯俱句惧炬剧捐鹃娟倦眷卷绢撅攫抉掘倔爵觉决诀绝均菌钧军君峻俊竣浚郡骏茄丌亟乩厥刭剞劂冂佶佼倨偈儆僭僬僦讦讵诘谏谫谲卩卺阱郏鄄矍墼艽芨芰苣苴莒荠茭茳荩菁堇菹菅葭蒺蒹蕨蕺拮挢捃掎掬搛叽咭哜唧喈啾嗟噘噍噤噱囝岌岬峤崛嵴徼狷獍獗馑廑憬阄泾浃洎洚涓湫湔蹇謇迥迦迳遽彐屐屦弪妗姣婧婕孑孓骥绛缙缣缰畿玑珏珈琚瑾枧柩桕桀桊桷楗椐楫榘榉槿橛橘殛戋戛戟戢戬炅赆赍觊觐牮犟犄犋犍毽敫肼胛胫腈腱飓齑旌爝扃恝矶碣礓睑羁蠲钅钜铗锏锔锩镌镓镢嵇稷皎鸠鹣鹪鹫疖痂瘕瘠窭衿袷裥裾皲矜耩颉虮蚧蛱蛟笈笄笕笳袈羯糨暨翦赳趄豇醮醵趼跏跻跤跽踺踽蹶觖靓霁龃隽雎鲒鲚鲛鲣鲫鞯鞫骱髻鬏麂麇",k:"槛喀咖卡开揩楷凯慨刊堪勘坎砍看康慷糠扛抗亢炕考拷烤靠坷苛柯棵磕颗科壳咳可渴克刻客课肯啃垦恳坑吭空恐孔控抠口扣寇枯哭窟苦酷库裤夸垮挎跨胯块筷侩快宽款匡筐狂框矿眶旷况亏盔岿窥葵奎魁馈愧溃坤昆捆困括扩廓阔馗匮刳剀蒯伉佧侉侃倥夔诓诳隗邝郐圹垲堀芤莰蒈蒉蔻夼揆叩咔哐哙喹喟喾嗑岢崆狯忾恺恪悝悃愦闶阃溘逵尻骒纩绔缂珂琨栲轲戡暌贶犒氪胩脍龛眍睽瞌瞰钪钶铐铠铿锎锞锟锴稞疴窠裉聩颏蛞蝰蝌筘箜篑醌跬龈鲲骷髁髋髡",l:"垃拉喇蜡腊辣啦莱来赖蓝婪栏拦篮阑兰澜谰揽览懒缆烂滥琅榔狼廊郎朗浪捞劳牢老佬姥酪烙涝勒乐雷镭蕾磊累儡垒擂肋类泪棱楞冷厘梨犁黎篱狸离漓理李里鲤礼莉荔吏栗丽厉励砾历利傈例俐痢立粒沥隶力璃哩俩联莲连镰廉怜涟帘敛脸链恋炼练粮凉梁粱良两辆量晾亮谅撩聊僚疗燎寥辽潦了撂镣廖料列裂烈劣猎琳林磷霖临邻鳞淋凛赁吝拎玲菱零龄铃伶羚凌灵陵岭领另令溜琉榴硫馏留刘瘤流柳六龙聋咙笼窿隆垄拢陇楼娄搂篓漏陋芦卢颅庐炉掳卤虏鲁麓碌露路赂鹿潞禄录陆戮驴吕铝侣旅履屡缕虑氯律率滤绿峦挛孪滦卵乱掠略抡轮伦仑沦纶论萝螺罗逻锣箩骡裸落洛骆络剌仂俪俚倮偻脔蠃羸冽诔郦酃坜垅垆埒塄墚苈茏苓荦莅莨蒌蓠蒗蔹蔺蓼藜奁尥捋捩摞撸叻呖呤咧唠啉啷唳喱喽嘞嘹噜囵囹岚崂崃嵝嶙徕猁猡獠廪愣懔闾阆泐泷泸泠泺洌浏涞渌溧漤漯潋漉濑澧濂寮逦逯遛遴邋娈娌嫠嫘骊骝绫绺缡缧缭珑珞琏璐枥栊栌柃栎栳栾棂椤椋楝榄榈橹檑檩殓轳轹辂辘辚瓴旯赉氇胧胪脶膂臁膦斓旒熘戾砺砻砬睐瞵詈罱罹钌铑铹铼锂锊锍锒镂镏镙镥镧稆稂鸬鸾鹂鹨鹩鹭疠疬痨瘌瘘瘰癃癞裢裣褛褴耒耢耧聆蛎蛉蜊螂蝼蠊蠡笠篥簏籁舻粝粼翎趔酹醪醴跞踉躏躐雳雒銮鎏鲈鲡鲢鲮鳓鳢髅魉鬣麟黧",m:"妈麻玛码蚂马骂嘛吗埋买麦卖迈脉瞒馒蛮满蔓曼慢漫谩芒茫盲氓忙莽猫茅锚毛矛铆卯茂冒帽貌贸么玫枚梅酶霉煤没眉媒镁每美昧寐妹媚门闷们萌蒙檬盟锰猛梦孟眯醚靡糜迷谜弥米秘觅泌蜜密幂棉眠绵冕免勉娩缅面苗描瞄藐秒渺庙妙蔑灭民抿皿敏悯闽明螟鸣铭名命谬摸摹蘑模膜磨摩魔抹末莫墨默沫漠寞陌谋牟某拇牡亩姆母墓暮幕募慕木目睦牧穆乜芈仫侔袤冖冥谟谧邙劢勐坶墁茉苜茆苠茗荬莓蓦甍瞢蘼扪咩咪哞唛喵嘧幔峁岷嵋犸猕猸馍懵闵沐沔汨泖泯浼渑湎湄漭溟宀宓邈弭嫫嬷缈缗缦缪珉瑁杩杪楣殁昴暝牦耄毪脒腼朦旄焖熳祢愍懋懑淼硭礞眄眇眸瞀瞑钔钼镅镆镘秣鹋鹛瘼袂耱颟虻蛑蜢蝥螨蟒蟆蟊蠓蠛篾艋艨敉糸酩貊貘霾黾鍪鳗鳘鞔魅髦麽縻麋",n:"拿哪呐钠那娜纳氖乃奶耐奈南男难囊挠脑恼闹淖呢馁内嫩能妮霓倪泥尼拟你匿腻逆溺蔫拈年碾撵捻念娘酿鸟尿捏聂孽啮镊镍涅您柠狞凝宁拧泞牛扭钮纽脓浓农弄奴努怒女暖虐疟挪懦糯诺辗廿孬鼐佞伲侬傩讷陧坭垴埝艿茑萘蘖捺搦攮咛呶哝喏喃嗫嗯囔囡狃猊猱馕忸怩甯弩妞胬嬲孥驽瑙柰楠辇昵曩肭脲腩旎恧恁硇睨钕铌铙锘镎黏疒衲耨聍颞蛲蝻臬衄袅赧蹑鲇鲵鲶",o:"哦欧鸥殴藕呕偶沤讴喔噢怄瓯耦",p:"辟泊脯啪趴爬帕怕琶拍排牌徘湃派攀潘盘磐盼畔判叛乓庞旁耪胖抛咆刨炮袍跑泡呸胚培裴赔陪配佩沛喷盆砰抨烹澎彭蓬棚硼篷膨朋鹏捧碰坯砒霹批披劈琵毗啤脾疲皮匹痞僻屁譬篇偏片骗飘漂瓢票撇瞥拼频贫品聘乒坪苹萍平凭瓶评屏坡泼颇婆破魄迫粕剖扑铺仆莆葡菩蒲埔朴圃普浦谱曝瀑丕丿叵剽仳俜俳匍裒谝陴邳郫鄱圮埤堋鼙芘苤葩蒎匏拚掊擗哌辔嘌嘭噗噼帔彷狍庀庖怦爿泮淠湓溥滂濮逄姘娉媲嫔嫖骈纰缥珀璞枇杷枰榀殍甓攴牝犏氆氕攵胼脬旆睥瞟罴钋钷铍锫镤镨皤疱癖袢襻疋颦蚍蜱螃螵蟛蟠笸筢翩醅蹁蹒蹼貔霈鲆",q:"期欺栖戚妻七凄漆柒沏其棋奇歧畦崎脐齐旗祈祁骑起岂乞企启契砌器气迄弃汽泣讫掐恰洽牵扦钎铅千迁签仟谦乾黔钱钳前潜遣浅谴堑嵌欠歉枪呛腔羌墙蔷强抢橇锹敲悄桥瞧乔侨巧鞘撬翘峭俏窍切且怯窃钦侵亲秦琴勤芹擒禽寝沁青轻氢倾卿清擎晴氰情顷请庆琼穷秋丘邱球求囚酋泅趋区蛆曲躯屈驱渠取娶龋趣去圈颧权醛泉全痊拳犬券劝缺瘸却鹊榷确雀裙群亓劁俅俟倩佥诎诠诮谯阡邛郄劬凵巯圻芊芎芑芩芡芪苘茕茜荞荃萁萋葜葺蕖蕲蘧掮揿吣嗪嘁噙圊屺岍岐岖衢犭犰悭悛惬愀慊憔阒阕阙丬戕汔淇溱骞搴褰逑逡遒妾嫱骐绮绻缱缲琪琦璩杞桤椠楸樯槭檠樵檎辁戗耆赇觑挈氍肷朐炝祛祺悫愆憩硗碛磬磲畎钤铨锓锖锲锵镪鸲癯穹襁颀虔虬蚯蛩蛐蛴蜣蜻蜞蜷蝤螓蠼罄筇筌箐箧箝衾裘羟糗綦綮麴跫跄跷蹊謦瞿銎鲭鳅鳍鞒髂鬈麒黢黥鼽",r:"然燃冉染瓤壤攘嚷让饶扰绕惹热壬仁人忍韧任认刃妊纫扔仍日戎茸蓉荣融熔溶容绒冗揉柔肉茹蠕儒孺如辱乳汝入褥软阮蕊瑞锐闰润若弱亻仞偌芮苒荛荏葚蓐蕤薷嚅嵘狨饪洳溽濡娆缛枘桡榕轫肜朊禳睿铷稔穰衽襦颥蚋蚺蝾箬糅蹂鞣髯",s:"匙撒洒萨腮鳃塞赛三叁伞散桑嗓丧搔骚扫嫂瑟色涩森僧莎砂杀刹沙纱傻啥煞筛晒珊苫杉山删煽衫闪陕擅赡膳善汕扇缮墒伤商赏晌上尚裳梢捎稍烧芍勺韶少哨邵绍奢赊蛇舌舍赦摄射慑涉社设砷申呻伸身深娠绅神沈审婶甚肾慎渗声生甥牲升绳省盛剩胜圣师失狮施湿诗尸虱十石拾时什食蚀实识史矢使屎驶始式示士世柿事拭誓逝势是嗜噬适仕侍释饰氏市恃室视试收手首守寿授售受瘦兽蔬枢梳殊抒输叔舒淑疏书赎孰熟薯暑曙署蜀黍鼠属术述树束戍竖墅庶数漱恕刷耍摔衰甩帅栓拴霜双爽谁水睡税吮瞬顺舜说硕朔烁斯撕嘶思私司丝死肆寺嗣四似饲巳松耸怂颂送宋讼诵搜艘擞嗽苏酥俗素速粟僳塑溯宿诉肃酸蒜算虽隋随绥髓碎岁穗遂隧祟孙损笋蓑梭唆缩琐索锁所厦卅啬厍厮剡仨倏佘夙兕凇讪诜谂谇谡谥鄯劭叟厶塾埏垧埘埽芟苕荪莳荽莘菘菽蓍蒴蔌薮扌摅搠搡弑咝哂唢唼唰嗖嗉嗦嗍嗾噻崧嵊嵩彡狩狲狻猞飧饣馊馓忪悚愫闩氵汜沭泗涑淞渖涮溲滠澍澌潸潲濉邃妁姒姗姝娑嬗孀驷骟纟纾绱绶缌缫桫椹榫槊殇轼贳挲毵毹胂脎腧臊膻歃歙飒飕殳炻熵燧礻祀磉眚眭睃睢瞍畲钐铄铈铩铯锶锼矧秫穑鸶疝痧瘙竦耜颡蛳蜃蛸螋螫蟀蟮舐笙笥筮筲簌舢艄艏裟羧糁豉酾豕跚觞觫霎隼稣鲥鲨鲺鳋鳝麝",t:"塌他它她塔獭挞蹋踏胎苔抬台泰酞太态汰坍摊贪瘫滩坛檀痰潭谭谈坦毯袒碳探叹炭汤塘搪堂棠膛唐糖倘躺淌趟烫掏涛滔绦萄桃逃淘陶讨套特藤腾疼誊梯剔踢锑提题蹄啼体替嚏惕涕剃屉天添填田甜恬舔腆挑条迢眺跳贴铁帖厅听烃汀廷停亭庭挺艇通桐酮瞳同铜彤童桶捅筒统痛偷投头透凸秃突图徒途涂屠土吐兔湍团推颓腿蜕褪退吞屯臀拖托脱鸵陀驮驼椭妥拓唾乇鼗佟佗佻倜傥僮仝氽亠邰郯坨堍荑茼莛荼萜菟葶薹抟掭忒啕嗵帑饧饨庹恸悌忝闼阗沲沱洮溻溏潼逖遢彖婷绨缇瑭韬柝梃榻樘橐殄昙暾肽豚滕炱煺祧忑忐慝砼砣町畋疃钍钛钭钽铊铴铽锬镗鹈窕裼耥覃蜓蜩螗螳笤箨羰粜酡酴醍醣趿跎跆霆龆鼍鲐鲦鳎骰餮饕髫",w:"挖哇蛙洼娃瓦袜歪外豌弯湾玩顽丸烷完碗挽晚皖惋宛婉万腕汪王亡枉网往旺望忘妄威巍微危韦违桅围唯惟为潍维苇萎委伟伪尾纬未蔚味畏胃喂魏位渭谓尉慰卫瘟温蚊文闻纹吻稳紊问嗡翁瓮挝蜗涡窝我斡卧握沃巫呜钨乌污诬屋无芜梧吾吴毋武五捂午舞伍侮坞戊雾晤物勿务悟误兀刎剜罔佤仵倭偎诿阢隈邬圩圬芄芴莴菀葳蓊薇蕹呒唔囗帏帷幄崴嵬猥猬庑怃忤惘闱阌汶沩洧浯涠渥寤迕逶妩娲娓婺骛纨绾玮玟琬璺韪杌辋軎牾肟脘腽炜焐煨硪畹鹉鹜痦痿蜈蜿艉雯龌鋈鲔魍鼯",x:"昔熙析西硒矽晰嘻吸锡牺稀息希悉膝夕惜熄烯溪汐犀檄袭席习媳喜铣洗系隙戏细瞎虾匣霞辖暇峡侠狭下夏吓掀锨先仙鲜纤咸贤衔舷闲涎弦嫌显险现献县腺馅羡宪陷限线相厢镶香箱襄湘乡翔祥详想响享项巷橡像向象萧硝霄削哮嚣销消宵淆晓小孝校肖啸笑效楔些歇蝎鞋协挟携邪斜胁谐写械卸蟹懈泄泻谢屑薪芯锌欣辛新忻心信衅星腥猩惺兴刑型形邢行醒幸杏性姓兄凶胸匈汹雄熊休修羞朽嗅锈秀袖绣墟戌需虚嘘须徐许蓄酗叙旭序恤絮婿绪续轩喧宣悬旋玄选癣眩绚靴薛学穴雪血勋熏循旬询寻驯巡殉汛训讯逊迅吁囟偕僖儇兮巽亵冼诩谑谖陉隰郗勖勰燮埙馨芗苋荇荀荥荨莶菥葙葸萱蓰蓿蕈薤薰藓奚揎撷擤哓咻唏岘岫峋崤徇徙狎猃獬獯饩饷馐庥庠廨忄恂悻阋泫洫洵浔浠淅渫溆渲溴潇漩瀣逍遐暹邂屣娴嬉骁骧绁绡缃缬顼玺瑕璇枭柙枵栩楦榭榍樨昕暄曛曦觋氙欷歆炫煊燹煦熹祆禊禧泶硎硖碹盱铉镟皙鸺鹇痃痫穸窨胥蚬蜥螅蟋蟓罅筅筱箫舄舾羲籼粞糈翕酰醑醯醺踅跣跹躞貅霰鑫鲞鲟鳕魈飨髹黠鼷",y:"压押鸦鸭呀丫芽牙蚜崖衙涯雅哑亚讶焉咽阉烟淹盐严研蜒岩延言颜阎炎沿奄掩眼衍演艳堰燕厌砚雁唁彦焰宴谚验殃央鸯秧杨扬佯疡羊洋阳氧仰痒养样漾邀腰妖瑶摇尧遥窑谣姚咬舀药要耀椰噎耶爷野冶也页掖业叶曳腋夜液一壹医揖铱依伊衣颐夷遗移仪胰疑沂宜姨彝椅蚁倚已乙矣以艺抑易邑屹亿役臆逸肄疫亦裔意毅忆义益溢诣议谊译异翼翌绎茵荫因殷音阴姻吟银淫寅饮尹引隐印英樱婴鹰应缨莹萤营荧蝇迎赢盈影颖硬映哟拥佣臃痈庸雍踊蛹咏泳涌永恿勇用幽优悠忧尤由邮铀犹油游酉有友右佑釉诱又幼迂淤于盂榆虞愚舆余俞逾鱼愉渝渔隅予娱雨与屿禹宇语羽玉域芋郁遇喻峪御愈欲狱育誉浴寓裕预豫驭鸳渊冤元垣袁原援辕园员圆猿源缘远苑愿怨院曰约越跃钥岳粤月悦阅耘云郧匀陨允运蕴酝晕韵孕轧禺夭爻胤毓厣靥赝卣刈劓伛伢攸佚侑佾俨俣俑偃龠兖嬴讠诒谀谒谕谳邺郓郢郾鄢鄞廴壅圯垭垠垸埸堙塬墉懿芫芸苡茚茔莠莜莸莺萸菸萦蓥蓣薏弈奕尢挹揶揠揄掾撄弋吖吆呓吲呦咦咿唷喁喑嘤噫囿圄圉岈峄崦崾嵛嶷徉徭狁狳狺猗夤饫饴馀庾膺怏怿怡恹恽悒愠慵闫阈沅泱洇湮滟滢潆漪瀛瀹宥迓迤鬻妍妪妤娅媛嫣驿纡纭缢幺邕琊珧瑛琰瑜瑗璎韫杳柚桠楹樾橼檐猷殒殪轶轺昀昱晔晏曜贻觎氩氤氲牖爰刖肴胭腌腴媵欤欹於旖炀烨烊焱煜熨熠燠恙肀聿砑眢眙睚罨钇钰钺铕铘铟镒镛镱甬鸢鹆鹞鹦鹬疣痖痍瘐瘀瘗瘿瘾癔翊窈窬窳衤颍蚓蚰蚴蛘蜮蜴蝓蝣螈罂竽筵筠箢臾舁舣羿翳繇酏酽雩霪龉鼋鱿鳐鳙鞅魇餍饔黝黟鼬鼹",z:"长匝砸杂栽哉灾宰载再在咱攒暂赞赃脏葬遭糟凿藻枣早澡蚤躁噪造皂灶燥责择则泽贼怎增憎赠扎喳渣札铡闸眨栅榨咋乍炸诈摘斋宅窄债寨瞻毡詹粘沾盏斩崭展蘸栈占战站湛绽樟章彰漳张掌涨杖丈帐账仗胀瘴障招昭找沼赵照罩兆肇召遮折哲蛰辙者锗蔗这浙珍斟真甄砧臻贞针侦枕疹诊震振镇阵蒸挣睁征狰争怔整拯正政帧症郑证芝枝支吱蜘知肢脂汁之织职直植殖执值侄址指止趾只旨纸志挚掷至致置帜峙制智秩稚质炙痔滞治窒中盅忠钟衷终种肿重仲众舟周州洲诌粥轴肘帚咒皱宙昼骤珠株蛛朱猪诸诛逐竹烛煮拄瞩嘱主著柱助蛀贮铸筑住注祝驻抓爪拽专砖转撰赚篆桩庄装妆撞壮状锥追赘坠缀谆准捉拙卓桌琢茁酌啄着灼浊兹咨资姿滋淄孜紫仔籽滓子自渍字鬃棕踪宗综总纵邹走奏揍租足卒族祖诅阻组钻纂嘴醉最罪尊遵昨左佐柞做作坐座卮丶仄赜仉伫侏倬偬俎冢诏诤诹诼谘谪谮谵阼陟陬郅邾鄣鄹圳埴芷苎茱荮蓁蕞奘拶揸搌摭摺撙擢攥吒咂咤哳唣唑啧啭啁帙帻幛峥崽嵫嶂徵獐夂馔忮怍惴浈洙浞渚涿潴濯迮彘咫姊妯嫜孳驵驺骓骘纣绉缁缒缜缯缵甾璋瓒杼栉柘枳栀桎桢梓棹楱楂榛槠橥樽轵轸轾辄辎臧甑昃昝贽赀赈肫胄胙胗胝朕腙膣旃炷祉祚祗祯禚恣斫砟砦磔黹眦畛罾钊钲铢铮锃锱镞镯锺雉秭稹鸩鸷鹧痄疰痣瘃瘵窀褶耔颛蚱蛭蜇螽蟑竺笊笫笮筝箦箸箴簪籀舯舳舴粢粽糌翥絷趑趱赭酎酯跖踬踯踵躅躜豸觜觯訾龇隹錾鲰鲻鳟髭麈齄"};
var PY_INIT = {};
Object.keys(PY_INIT_BUCKETS).forEach(function (k) {
  var s = PY_INIT_BUCKETS[k];
  for (var i = 0; i < s.length; i++) PY_INIT[s.charAt(i)] = k;
});
function toInitials(s) {
  var out = '';
  for (var i = 0; i < s.length; i++) {
    var ch = s.charAt(i);
    var ini = PY_INIT[ch];
    out += ini != null ? ini : ch.toLowerCase();
  }
  return out;
}

var idSet = {};
Object.keys(articles).forEach(function (m) { idSet[m] = 1; });
Object.keys(articleTotal).forEach(function (m) { idSet[m] = 1; });
Object.keys(dailyByArticle).forEach(function (m) { idSet[m] = 1; });

/* 回收站标题命中规则：导入的数据若标题与回收站中相同，直接进入回收站，不在列表显示 */
(function autoBinByTitle() {
  var binTitles = {};
  Object.keys(localDeleted).forEach(function (k) {
    var t = String(localDeleted[k].title || '').trim();
    if (t) binTitles[t] = 1;
  });
  serverDeleted.forEach(function (x) {
    var t = String(x.title || '').trim();
    if (t) binTitles[t] = 1;
  });
  if (!Object.keys(binTitles).length) return;
  var changed = false;
  Object.keys(idSet).forEach(function (m) {
    if (localDeleted[m]) return;
    var meta = articles[m] || {};
    var t = String(meta.title || '').trim();
    if (t && binTitles[t]) {
      localDeleted[m] = { title: t, pubdate: pubdateOf(m), read: 0,
                          deleted_at: new Date().toISOString().slice(0, 19).replace('T', ' '), auto: true };
      changed = true;
    }
  });
  if (changed) { LSS(DEL_KEY, localDeleted); autoBinChanged = true; }
})();

var articleRows = Object.keys(idSet).filter(function (m) { return !localDeleted[m]; }).map(function (m) {
  var meta = articles[m] || {}, tot = articleTotal[m] || {}, dai = dailyByArticle[m] || {}, manual = manualTitle(m);
  return {
    id: m,
    title: manual || meta.title || ('未获取到标题（' + m + '）'),
    manual: !!manual,
    url: meta.content_url || '',
    pubdate: pubdateOf(m),
    read: tot.read_user || dai.read || 0,
    share: tot.share_user || dai.share || 0,
    fav: tot.collection_user || dai.fav || 0,
    like: int(tot.like_user),
    zaikan: int(tot.zaikan_user),
    comment: int(tot.comment_count)
  };
});
articleRows.forEach(function (r) { r.py = toInitials(r.title); });

function binList() {
  var out = serverDeleted.filter(function (x) { return !localDeleted[x.id]; }).map(function (x) {
    return { id: x.id, title: x.title || ('未获取到标题（' + x.id + '）'), pubdate: x.pubdate || '',
             deleted_at: x.deleted_at || '', read: int(x.read_user) };
  });
  Object.keys(localDeleted).forEach(function (m) {
    var x = localDeleted[m];
    out.push({ id: m, title: x.title || ('未获取到标题（' + m + '）'), pubdate: x.pubdate || '',
               deleted_at: x.deleted_at || '', read: int(x.read) });
  });
  out.sort(function (a, b) { return (a.deleted_at || '') < (b.deleted_at || '') ? 1 : -1; });
  return out;
}

/* ================= 状态 ================= */
var state = { start: '', end: '', q: '', sortKey: 'pubdate', sortDir: 'desc', page: 1, per: 50 };
var selected = {};

function matchQuery(row, q) {
  var kws = q.toLowerCase().split(/\s+/).filter(function (s) { return s.length > 0; });
  if (!kws.length) return true;
  var hay = (row.title + ' ' + row.id).toLowerCase(), py = row.py || '';
  for (var i = 0; i < kws.length; i++) {
    var kw = kws[i];
    if (hay.indexOf(kw) >= 0) continue;
    if (/^[a-z]+$/.test(kw) && py.indexOf(kw) >= 0) continue;
    return false;
  }
  return true;
}
function filteredRows() {
  var rows = articleRows.filter(function (r) {
    if (state.q && !matchQuery(r, state.q)) return false;
    if (state.start && (!r.pubdate || r.pubdate < state.start)) return false;
    if (state.end && (!r.pubdate || r.pubdate > state.end)) return false;
    return true;
  });
  var k = state.sortKey, dir = state.sortDir === 'asc' ? 1 : -1;
  rows.sort(function (a, b) {
    if (k === 'title') return dir * a.title.localeCompare(b.title, 'zh-Hans-CN');
    if (k === 'pubdate') {
      var va = a.pubdate || '0', vb = b.pubdate || '0';
      return dir * (va < vb ? -1 : va > vb ? 1 : 0);
    }
    return dir * (int(a[k]) - int(b[k]));
  });
  return rows;
}

/* ================= KPI ================= */
var KPI_ICONS = {
  fans: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/><circle cx="9" cy="7" r="4"/><path d="M23 21v-2a4 4 0 0 0-3-3.87"/></svg>',
  net: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M16 7h6v6"/><path d="M22 7l-8.5 8.5-5-5L2 17"/></svg>',
  read: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M2 12s3.5-7 10-7 10 7 10 7-3.5 7-10 7-10-7-10-7z"/><circle cx="12" cy="12" r="3"/></svg>',
  share: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="18" cy="5" r="3"/><circle cx="6" cy="12" r="3"/><circle cx="18" cy="19" r="3"/><path d="M8.6 13.5l6.8 4M15.4 6.5l-6.8 4"/></svg>',
  doc: '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><path d="M14 2v6h6"/></svg>'
};
/* ---- 周统计：两种口径 ----
   roll7 = 含最新数据日在内的过去 7 天（滚动窗口）
   mon   = 最新数据日所在自然周（周一 ~ 周日，统计到有数据为止） */
var WEEK_KEY = 'gzh_week_mode_v1';
var weekMode = (function () { try { var v = localStorage.getItem(WEEK_KEY); return v === 'mon' ? 'mon' : 'roll7'; } catch (e) { return 'roll7'; } })();
function pad2(n) { return (n < 10 ? '0' : '') + n; }
function isoDate(d) { return d.getFullYear() + '-' + pad2(d.getMonth() + 1) + '-' + pad2(d.getDate()); }
function weekRange() {
  if (!dates.length) return null;
  var base = dates[dates.length - 1];
  var b = new Date(base + 'T00:00:00');
  var s = new Date(base + 'T00:00:00');
  if (weekMode === 'mon') {
    var wd = b.getDay();                       /* 0 = 周日 */
    s.setDate(b.getDate() - (wd === 0 ? 6 : wd - 1));
  } else {
    s.setDate(b.getDate() - 6);
  }
  return { start: isoDate(s), end: base };
}
function markWeekBtn() {
  Array.prototype.forEach.call(document.querySelectorAll('#week-zoom .zbtn'), function (b) {
    b.classList.toggle('on', b.getAttribute('data-week') === weekMode);
  });
}
function renderKpis() {
  var last7 = dates.slice(-7), sum7ru = 0, sum7su = 0;
  last7.forEach(function (d) {
    for (var i = 0; i < trend.length; i++) {
      if (trend[i].date === d) { sum7ru += trend[i].ru; sum7su += trend[i].su; break; }
    }
  });
  /* 周统计：按当前口径取区间，统计发文数与阅读人数 */
  var wr = weekRange(), wPub = 0, wRead = 0, wDays = 0;
  if (wr) {
    articleRows.forEach(function (r) {
      if (r.pubdate && r.pubdate >= wr.start && r.pubdate <= wr.end) wPub++;
    });
    trend.forEach(function (t) {
      if (t.date >= wr.start && t.date <= wr.end) { wRead += t.ru; wDays++; }
    });
  }
  var cumList = fans.filter(function (f) { return f.cum != null; });
  var latestCum = cumList.length ? cumList[cumList.length - 1] : null;
  var latestDay = fans.length ? fans[fans.length - 1] : null;
  var net = latestDay ? latestDay.newer - latestDay.cancel : 0;
  var wkDelta = wr ? (wr.start.slice(5) + ' ~ ' + wr.end.slice(5) + (weekMode === 'mon' ? ' · 周一 起' : '')) : '';
  var kpi = [
    { label: '累计粉丝', icon: KPI_ICONS.fans, value: fmt(latestCum ? latestCum.cum : 0), cls: '', delta: latestDay ? ('截至 ' + latestDay.date) : '' },
    { label: '最新日净增', icon: KPI_ICONS.net, value: (net >= 0 ? '+' : '') + fmt(net), cls: net >= 0 ? 'up' : 'down', delta: latestDay ? ('新增 ' + fmt(latestDay.newer) + ' · 取关 ' + fmt(latestDay.cancel)) : '' },
    { label: '近7天阅读人数', icon: KPI_ICONS.read, value: fmt(sum7ru), cls: '', delta: last7.length ? (last7[0].slice(5) + ' 起') : '' },
    { label: '近7天分享人数', icon: KPI_ICONS.share, value: fmt(sum7su), cls: '', delta: '人' },
    { label: '文章总数', icon: KPI_ICONS.doc, value: fmt(articleRows.length), cls: '', delta: '篇' },
    { label: '周发文数', icon: KPI_ICONS.doc, value: fmt(wPub), cls: '', delta: wkDelta },
    { label: '周阅读数', icon: KPI_ICONS.read, value: fmt(wRead), cls: '', delta: wr ? (wDays + ' 天 · 阅读人数') : '' }
  ];
  $('kpis').innerHTML = kpi.map(function (k) {
    return '<div class="kpi">' +
      '<div class="kpi-l">' + k.icon + esc(k.label) + '</div>' +
      '<div class="kpi-v ' + k.cls + '">' + esc(k.value) + '</div>' +
      '<div class="kpi-d">' + esc(k.delta) + '</div></div>';
  }).join('');
  $('gen-time').textContent = D.generated_at || window.DATA_GENERATED_AT || '未知';
  $('kpi-note').textContent = dates.length ? (dates[0] + ' 至 ' + dates[dates.length - 1]) : '';
  markWeekBtn();
}
(function bindWeekZoom() {
  var bar = $('week-zoom');
  if (!bar) return;
  bar.addEventListener('click', function (e) {
    var b = e.target && e.target.closest ? e.target.closest('[data-week]') : null;
    if (!b) return;
    var v = b.getAttribute('data-week');
    if (v === weekMode) return;
    weekMode = v;
    try { localStorage.setItem(WEEK_KEY, v); } catch (e2) {}
    renderKpis();
  });
})();

/* ================= 图表 ================= */
var chartMap = {};
var C = { axis: '#cbd5e1', label: '#64748b', split: '#eef1f4', accent: '#9c1f1f', accent2: '#1b4b7a', up: '#c0392b', bar: '#d8b4a0' };
function draw(id, option) {
  var dom = $(id);
  if (!window.echarts) {
    dom.innerHTML = '<div class="chart-empty">图表库未加载，请检查网络或重新部署</div>';
    return null;
  }
  var old = echarts.getInstanceByDom(dom);
  if (old) old.dispose();
  dom.innerHTML = '';
  var c = echarts.init(dom, null, { renderer: 'canvas' });
  option.animation = false;   /* 关闭图表动画，避免滚动与切换时的掉帧 */
  c.setOption(option);
  chartMap[id] = c;
  return c;
}
function emptyChart(id, text) { $(id).innerHTML = '<div class="chart-empty">' + esc(text) + '</div>'; }
/* 趋势图时间范围：头部快捷按钮与底部比例尺联动 */
var zoomByBtn = false;
function markZoomBtn(kind) {
  Array.prototype.forEach.call(document.querySelectorAll('#trend-zoom .zbtn'), function (b) {
    b.classList.toggle('on', b.getAttribute('data-zoom') === String(kind));
  });
}
function clearZoomBtn() {
  Array.prototype.forEach.call(document.querySelectorAll('#trend-zoom .zbtn'), function (b) { b.classList.remove('on'); });
}
function setTrendZoom(kind) {
  var c = chartMap['c-trend'];
  if (!c || !trend.length) return;
  var start = 0;
  if (String(kind) !== 'all') {
    var k = parseInt(kind, 10);
    if (k > 0) start = Math.max(0, Math.min(100, (trend.length - k) / trend.length * 100));
  }
  zoomByBtn = true;
  c.dispatchAction({ type: 'dataZoom', start: start, end: 100 });
  markZoomBtn(kind);
  setTimeout(function () { zoomByBtn = false; }, 150);
}
function bindTrendZoom(c) {
  if (!c) return;
  c.off('dataZoom');
  /* 手动拖动比例尺后，取消快捷按钮的高亮（避免状态不一致） */
  c.on('dataZoom', function () { if (!zoomByBtn) clearZoomBtn(); });
  var bar = $('trend-zoom');
  if (bar && !bar._bound) {
    bar._bound = true;
    bar.addEventListener('click', function (e) {
      var b = e.target && e.target.closest ? e.target.closest('[data-zoom]') : null;
      if (!b) return;
      setTrendZoom(b.getAttribute('data-zoom'));
    });
  }
  markZoomBtn('all');
}
function renderCharts() {
  if (trend.length && trend.some(function (t) { return t.ru || t.su; })) {
    bindTrendZoom(draw('c-trend', {
      color: [C.accent, C.accent2],
      tooltip: { trigger: 'axis' },
      legend: { data: ['阅读人数', '分享人数'], top: 0, right: 0, itemWidth: 12, itemHeight: 8, textStyle: { color: C.label, fontSize: 11 } },
      grid: { left: 48, right: 14, top: 30, bottom: 58 },
      /* 时间缩放：底部比例尺拖动选区间；图内支持滚轮 / 双指缩放平移 */
      dataZoom: [
        { type: 'inside', xAxisIndex: 0, filterMode: 'none', throttle: 60,
          zoomOnMouseWheel: true, moveOnMouseMove: true, moveOnMouseWheel: false },
        { type: 'slider', xAxisIndex: 0, height: 20, bottom: 6,
          borderColor: 'transparent', backgroundColor: 'rgba(148,163,184,.10)',
          fillerColor: 'rgba(156,31,31,.10)', handleSize: '140%',
          handleStyle: { color: '#fff', borderColor: C.accent, borderWidth: 1 },
          moveHandleSize: 5,
          dataBackground: { lineStyle: { color: 'rgba(148,163,184,.5)', width: 1 }, areaStyle: { color: 'rgba(148,163,184,.10)' } },
          selectedDataBackground: { lineStyle: { color: 'rgba(156,31,31,.6)', width: 1 }, areaStyle: { color: 'rgba(156,31,31,.12)' } },
          textStyle: { color: C.label, fontSize: 10 },
          labelFormatter: function (v, vs) { return String(vs == null ? '' : vs); } }
      ],
      xAxis: { type: 'category', data: trend.map(function (t) { return shortDate(t.date); }), boundaryGap: false, axisLine: { lineStyle: { color: C.axis } }, axisLabel: { color: C.label, fontSize: 11, hideOverlap: true } },
      yAxis: { type: 'value', axisLabel: { formatter: fmt, color: C.label, fontSize: 11 }, splitLine: { lineStyle: { color: C.split } } },
      series: [
        { name: '阅读人数', type: 'line', smooth: true, showSymbol: false, lineStyle: { width: 2 }, areaStyle: { color: 'rgba(156,31,31,.07)' }, data: trend.map(function (t) { return t.ru; }) },
        { name: '分享人数', type: 'line', smooth: true, showSymbol: false, lineStyle: { width: 1.6 }, data: trend.map(function (t) { return t.su; }) }
      ]
    }));
  } else { emptyChart('c-trend', '暂无趋势数据'); }

  var srcKeys = Object.keys(sourceAgg).sort(function (a, b) { return sourceAgg[b] - sourceAgg[a]; });
  if (srcKeys.length) {
    draw('c-source', {
      color: [C.accent, C.accent2, '#b8860b', '#5b7c5b', '#7b5b8a', '#8a6b4b'],
      tooltip: { trigger: 'item', formatter: '{b}<br/>阅读 {c} 人（{d}%）' },
      legend: { bottom: 0, type: 'scroll', itemWidth: 10, itemHeight: 8, textStyle: { color: C.label, fontSize: 11 } },
      series: [{
        type: 'pie', radius: ['40%', '66%'], center: ['50%', '44%'],
        itemStyle: { borderColor: '#fff', borderWidth: 2 },
        label: { color: C.label, fontSize: 11, formatter: '{b}\n{d}%' },
        data: srcKeys.map(function (k) { return { name: k, value: sourceAgg[k] }; })
      }]
    });
  } else { emptyChart('c-source', '暂无来源数据'); }

  if (fans.some(function (f) { return f.cum != null || f.newer || f.cancel; })) {
    draw('c-fans', {
      color: [C.accent, C.up],
      tooltip: { trigger: 'axis' },
      legend: { data: ['累计粉丝', '净增'], top: 0, right: 0, itemWidth: 12, itemHeight: 8, textStyle: { color: C.label, fontSize: 11 } },
      grid: { left: 48, right: 14, top: 30, bottom: 26 },
      xAxis: { type: 'category', data: fans.map(function (f) { return shortDate(f.date); }), boundaryGap: false, axisLine: { lineStyle: { color: C.axis } }, axisLabel: { color: C.label, fontSize: 11 } },
      yAxis: { type: 'value', axisLabel: { formatter: fmt, color: C.label, fontSize: 11 }, splitLine: { lineStyle: { color: C.split } } },
      series: [
        { name: '累计粉丝', type: 'line', smooth: true, showSymbol: false, lineStyle: { width: 2 }, areaStyle: { color: 'rgba(156,31,31,.07)' }, data: fans.map(function (f) { return f.cum; }) },
        { name: '净增', type: 'bar', barMaxWidth: 8, itemStyle: { color: C.up }, data: fans.map(function (f) { return f.newer - f.cancel; }) }
      ]
    });
  } else { emptyChart('c-fans', '暂无粉丝数据'); }

  var top = articleRows.slice().sort(function (a, b) { return b.read - a.read; }).slice(0, 10).reverse();
  if (top.length && top[top.length - 1].read > 0) {
    draw('c-rank', {
      grid: { left: 8, right: 46, top: 6, bottom: 6, containLabel: true },
      tooltip: { trigger: 'axis', axisPointer: { type: 'shadow' } },
      xAxis: { type: 'value', axisLabel: { formatter: fmt, color: C.label, fontSize: 10 }, splitLine: { lineStyle: { color: C.split } } },
      yAxis: { type: 'category', data: top.map(function (r) { return r.title.length > 14 ? r.title.slice(0, 14) + '…' : r.title; }), axisLabel: { color: C.label, fontSize: 10 }, axisLine: { lineStyle: { color: C.axis } } },
      series: [{ type: 'bar', barMaxWidth: 14, showBackground: true, backgroundStyle: { color: C.split },
        itemStyle: { color: C.accent }, data: top.map(function (r) { return r.read; }) }]
    });
  } else { emptyChart('c-rank', '暂无阅读数据'); }
}
var resizeTimer = null;
window.addEventListener('resize', function () {
  if (resizeTimer) cancelAnimationFrame(resizeTimer);
  resizeTimer = requestAnimationFrame(function () {
    Object.keys(chartMap).forEach(function (k) { if (chartMap[k]) chartMap[k].resize(); });
  });
});

/* ================= 表格 ================= */
function renderTable() {
  var rows = filteredRows();
  var pages = Math.max(1, Math.ceil(rows.length / state.per));
  if (state.page > pages) state.page = pages;
  var start = (state.page - 1) * state.per;
  var pageRows = rows.slice(start, start + state.per);

  $('f-count').textContent = '共 ' + rows.length + ' 篇（总 ' + articleRows.length + ' 篇）';
  $('pg-info').textContent = '第 ' + state.page + ' / ' + pages + ' 页';
  $('pg-prev').disabled = state.page <= 1;
  $('pg-next').disabled = state.page >= pages;

  Array.prototype.forEach.call(document.querySelectorAll('[data-ar]'), function (sp) {
    sp.textContent = sp.getAttribute('data-ar') === state.sortKey ? (state.sortDir === 'asc' ? '▲' : '▼') : '';
  });

  $('tbody').innerHTML = pageRows.map(function (r) {
    var titleHtml = r.url
      ? '<a href="' + esc(r.url) + '" target="_blank" rel="noopener">' + esc(r.title) + '</a>'
      : esc(r.title);
    return '<tr class="' + (selected[r.id] ? 'sel' : '') + '" data-id="' + esc(r.id) + '">' +
      '<td class="pick"><input type="checkbox" class="cb" data-sel="' + esc(r.id) + '"' + (selected[r.id] ? ' checked' : '') + ' /></td>' +
      '<td class="num">' + esc(r.pubdate || '—') + '</td>' +
      '<td class="title"><span class="t">' + titleHtml + (r.manual ? '<span class="manual">已重命名</span>' : '') + '</span></td>' +
      '<td class="num">' + fmt(r.read) + '</td>' +
      '<td class="num">' + fmt(r.share) + '</td>' +
      '<td class="num">' + fmt(r.fav) + '</td>' +
      '<td class="num">' + fmt(r.like) + '</td>' +
      '<td class="num">' + fmt(r.zaikan) + '</td>' +
      '<td class="num">' + fmt(r.comment) + '</td>' +
      '<td class="act"><button class="btn sm" data-act="rename">重命名</button>' +
      '<button class="btn sm danger" data-act="del">删除</button></td>' +
      '</tr>';
  }).join('');

  var empty = $('table-empty');
  empty.style.display = pageRows.length ? 'none' : 'block';
  $('empty-desc').textContent = articleRows.length
    ? '试着调整日期范围或清空搜索词'
    : '点击右上角「更新数据」上传最新的数据文件';
  var all = pageRows.length > 0 && pageRows.every(function (r) { return selected[r.id]; });
  $('cb-all').checked = all;
  renderSelBar();
}
function renderSelBar() {
  var ids = Object.keys(selected);
  $('selbar').style.display = ids.length ? 'flex' : 'none';
  $('sel-count').textContent = ids.length;
}
function renderRecycleBadge() {
  var n = binList().length;
  $('recycle-badge').style.display = n ? 'inline-block' : 'none';
}

/* 排序 */
Array.prototype.forEach.call(document.querySelectorAll('th.sortable'), function (th) {
  th.addEventListener('click', function () {
    var k = th.getAttribute('data-key');
    if (state.sortKey === k) state.sortDir = state.sortDir === 'asc' ? 'desc' : 'asc';
    else { state.sortKey = k; state.sortDir = 'desc'; }
    state.page = 1;
    renderTable();
  });
});
/* 筛选 */
var qTimer = null;
$('f-q').addEventListener('input', function () {
  var v = this.value;
  if (qTimer) clearTimeout(qTimer);
  qTimer = setTimeout(function () { state.q = v; state.page = 1; renderTable(); }, 160);
});
$('f-start').addEventListener('change', function () { state.start = this.value; state.page = 1; renderTable(); });
$('f-end').addEventListener('change', function () { state.end = this.value; state.page = 1; renderTable(); });
$('f-reset').addEventListener('click', function () {
  state.start = state.end = state.q = ''; state.page = 1;
  $('f-start').value = ''; $('f-end').value = ''; $('f-q').value = '';
  renderTable();
});
$('pg-prev').addEventListener('click', function () { if (state.page > 1) { state.page--; renderTable(); } });
$('pg-next').addEventListener('click', function () { state.page++; renderTable(); });
$('cb-all').addEventListener('change', function () {
  var on = this.checked;
  var rows = filteredRows().slice((state.page - 1) * state.per, state.page * state.per);
  rows.forEach(function (r) { if (on) selected[r.id] = 1; else delete selected[r.id]; });
  renderTable();
});
$('tbody').addEventListener('change', function (e) {
  var cb = e.target;
  if (!cb.getAttribute) return;
  var id = cb.getAttribute('data-sel');
  if (!id) return;
  if (cb.checked) selected[id] = 1; else delete selected[id];
  renderSelBar();
  var tr = cb.closest ? cb.closest('tr') : null;
  if (tr) tr.className = cb.checked ? 'sel' : '';
});
$('tbody').addEventListener('click', function (e) {
  var btn = e.target.closest ? e.target.closest('[data-act]') : null;
  if (!btn) return;
  var tr = btn.closest('tr');
  var id = tr ? tr.getAttribute('data-id') : null;
  if (!id) return;
  if (btn.getAttribute('data-act') === 'rename') openRename(id);
  else if (btn.getAttribute('data-act') === 'del') doDelete([id]);
});
$('sel-clear').addEventListener('click', function () { selected = {}; renderTable(); });
$('sel-delete').addEventListener('click', function () { doDelete(Object.keys(selected)); });

/* ================= 删除 / 还原 ================= */
function doDelete(ids) {
  if (!ids.length) return;
  requireAuth('删除文章', function () {
    askConfirm('删除文章', '确定删除所选 ' + ids.length + ' 篇文章吗？删除后可在回收站还原。', function () {
      if (serverMode) {
        api('/api/delete', { ids: ids }).then(function (j) {
          if (j && j.ok) { toast('已删除，可在回收站还原', 'ok'); setTimeout(function () { location.reload(); }, 400); }
          else toast((j && j.message) || '删除失败', 'err');
        }).catch(function () { toast('无法连接本机服务', 'err'); });
        return;
      }
      ids.forEach(function (id) {
        var row = null;
        for (var i = 0; i < articleRows.length; i++) { if (articleRows[i].id === id) { row = articleRows[i]; break; } }
        localDeleted[id] = { title: row ? row.title : id, pubdate: row ? row.pubdate : '',
                             read: row ? row.read : 0, deleted_at: new Date().toISOString().slice(0, 19).replace('T', ' ') };
        delete selected[id];
      });
      LSS(DEL_KEY, localDeleted);
      pushCloud();
      articleRows = articleRows.filter(function (r) { return !localDeleted[r.id]; });
      renderTable(); renderRecycleBadge(); renderCharts(); renderKpis();
      toast('已删除 ' + ids.length + ' 篇，可在回收站还原', 'ok');
    });
  });
}
function openRename(id) {
  requireAuth('重命名文章', function () {
    var row = null;
    for (var i = 0; i < articleRows.length; i++) { if (articleRows[i].id === id) { row = articleRows[i]; break; } }
    if (!row) return;
    renameId = id;
    $('rn-origin').textContent = '原标题：' + row.title;
    $('rn-input').value = row.title;
    openOverlay('rename-overlay');
    setTimeout(function () { $('rn-input').focus(); $('rn-input').select(); }, 60);
  });
}
var renameId = null;
$('rn-save').addEventListener('click', function () {
  var v = $('rn-input').value.trim();
  if (!renameId) { closeOverlay('rename-overlay'); return; }
  if (!v) { toast('标题不能为空', 'err'); return; }
  if (serverMode) {
    var rid = renameId;
    api('/api/rename', { id: rid, title: v }).then(function (j) {
      if (j && j.ok) {
        closeOverlay('rename-overlay'); renameId = null;
        toast('已重命名（全设备生效）', 'ok');
        setTimeout(function () { location.reload(); }, 400);
      } else toast((j && j.message) || '重命名失败', 'err');
    }).catch(function () { toast('无法连接本机服务', 'err'); });
    return;
  }
  localTitles[renameId] = v;
  LSS(TITLES_KEY, localTitles);
  pushCloud();
  for (var i = 0; i < articleRows.length; i++) {
    if (articleRows[i].id === renameId) {
      articleRows[i].title = v; articleRows[i].manual = true; articleRows[i].py = toInitials(v); break;
    }
  }
  closeOverlay('rename-overlay');
  renameId = null;
  renderTable(); renderCharts();
  toast('已重命名', 'ok');
});
$('rn-cancel').addEventListener('click', function () { closeOverlay('rename-overlay'); renameId = null; });
$('rn-close').addEventListener('click', function () { closeOverlay('rename-overlay'); renameId = null; });
$('rn-input').addEventListener('keydown', function (e) { if (e.key === 'Enter') { e.preventDefault(); $('rn-save').click(); } });

/* ================= 回收站 ================= */
$('btn-recycle').addEventListener('click', function () {
  if (!isUnlocked()) { toast('请先在顶栏「锁定」处输入管理员密码解锁', 'info'); return; }
  renderRecycle(); openOverlay('recycle-overlay');
});
$('rb-close').addEventListener('click', function () { closeOverlay('recycle-overlay'); });
function renderRecycle() {
  var list = binList();
  $('rb-sub').textContent = list.length ? ('共 ' + list.length + ' 篇') : '暂无删除记录';
  $('rb-list').innerHTML = list.length ? list.map(function (b) {
    return '<div class="bitem" data-id="' + esc(b.id) + '">' +
      '<div class="bt">' + esc(b.title) + '</div>' +
      '<div class="bd">' + esc(b.pubdate || '—') + (b.deleted_at ? ' · ' + esc(b.deleted_at.slice(0, 10)) : '') + '</div>' +
      '<button class="btn sm" data-ract="restore" data-id="' + esc(b.id) + '">还原</button>' +
      '<button class="btn sm danger" data-ract="purge" data-id="' + esc(b.id) + '">彻底删除</button>' +
      '</div>';
  }).join('') : '<div class="empty"><div class="t">回收站为空</div></div>';
}
$('rb-list').addEventListener('click', function (e) {
  var btn = e.target.closest ? e.target.closest('[data-ract]') : null;
  if (!btn) return;
  var id = btn.getAttribute('data-id');
  var act = btn.getAttribute('data-ract');
  if (act === 'restore') {
    requireAuth('还原文章', function () {
      if (serverMode) {
        api('/api/restore', { ids: [id] }).then(function (j) {
          if (j && j.ok) { toast('已还原', 'ok'); setTimeout(function () { location.reload(); }, 400); }
          else toast((j && j.message) || '还原失败', 'err');
        }).catch(function () { toast('无法连接本机服务', 'err'); });
        return;
      }
      delete localDeleted[id];
      LSS(DEL_KEY, localDeleted);
      pushCloud();
      var meta = articles[id] || {}, tot = articleTotal[id] || {}, dai = dailyByArticle[id] || {};
      if (!articleRows.some(function (r) { return r.id === id; })) {
        articleRows.push({
          id: id, title: manualTitle(id) || meta.title || ('未获取到标题（' + id + '）'),
          manual: !!manualTitle(id), url: meta.content_url || '', pubdate: pubdateOf(id),
          read: tot.read_user || dai.read || 0, share: tot.share_user || dai.share || 0,
          fav: tot.collection_user || dai.fav || 0, like: int(tot.like_user),
          zaikan: int(tot.zaikan_user), comment: int(tot.comment_count), py: ''
        });
        articleRows[articleRows.length - 1].py = toInitials(articleRows[articleRows.length - 1].title);
      }
      renderRecycle(); renderRecycleBadge(); renderTable(); renderKpis(); renderCharts();
      toast('已还原', 'ok');
    });
  } else {
    requireAuth('彻底删除', function () {
      askConfirm('彻底删除', '彻底删除后将无法恢复，确定继续吗？', function () {
        if (serverMode) {
          api('/api/purge', { ids: [id] }).then(function (j) {
            if (j && j.ok) { toast('已彻底删除', 'ok'); setTimeout(function () { location.reload(); }, 400); }
            else toast((j && j.message) || '操作失败', 'err');
          }).catch(function () { toast('无法连接本机服务', 'err'); });
          return;
        }
        localDeleted[id] = { title: localDeleted[id] ? localDeleted[id].title : id, purged: true,
                             deleted_at: (localDeleted[id] && localDeleted[id].deleted_at) || new Date().toISOString().slice(0, 19).replace('T', ' ') };
        LSS(DEL_KEY, localDeleted);
        pushCloud();
        renderRecycle(); renderRecycleBadge();
        toast('已彻底删除', 'ok');
      });
    });
  }
});

/* ================= 本机服务模式探测 ================= */
/* 在修复包（serve.py）里运行时，更新/删除/重命名直接走本机服务，全设备生效 */
var serverMode = false;
/* ---- 线上云端模式：删除/重命名/上传写入服务端，全设备同步 ---- */
var cloudMode = false;
var cloudRev = LS('wechat_sync_rev_v1', 0);
var cloudLayout = null;   /* 云端布局：管理员用密码保存后，所有设备同步生效 */
function adoptCloud(j) {
  /* 远端数据更新：整体覆盖本机的删除与重命名记录 */
  localDeleted = (j.deleted && typeof j.deleted === 'object') ? j.deleted : {};
  localTitles = (j.titles && typeof j.titles === 'object') ? j.titles : {};
  cloudRev = j.rev || 0;
  LSS(DEL_KEY, localDeleted);
  LSS(TITLES_KEY, localTitles);
  LSS('wechat_sync_rev_v1', cloudRev);
  /* 布局以云端为准：云端有保存则覆盖本机，云端为空（已恢复默认）则清除本机保存 */
  if ('layout' in j) {
    if (j.layout && j.layout.order && j.layout.order.length) {
      cloudLayout = j.layout;
      LSS(LAYOUT_KEY, j.layout);
    } else {
      cloudLayout = null;
      LSDEL(LAYOUT_KEY);
    }
  }
}
function pushCloud(extra) {
  if (!cloudMode) return;
  cloudRev = Date.now();
  LSS('wechat_sync_rev_v1', cloudRev);
  var payload = { token: authToken(), rev: cloudRev, deleted: localDeleted, titles: localTitles, layout: cloudLayout };
  if (extra) { Object.keys(extra).forEach(function (k) { payload[k] = extra[k]; }); }
  fetch('/api/state', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  }).then(function (r) {
    return r.json().catch(function () { return null; }).then(function (j) { if (j) j._status = r.status; return j; });
  }).then(function (j) {
    if (j && j._status === 403) {
      clearAuthSession();
      updateLockBtn();
      toast('登录已过期，请重新解锁后再操作', 'info');
      return;
    }
    if (!j || !j.ok) toast('同步到其他设备失败：' + ((j && j.message) || '请检查网络'), 'err');
  }).catch(function () { toast('同步到其他设备失败：请检查网络', 'err'); });
}
function probeBackends() {
  var ctl = new AbortController();
  var timer = setTimeout(function () { ctl.abort(); }, 2500);
  fetch('/api/state', { signal: ctl.signal })
    .then(function (r) { return r.ok ? r.json() : null; })
    .then(function (j) {
      clearTimeout(timer);
      if (j && typeof j.rev === 'number') {
        cloudMode = true;
        if ('layout' in j) cloudLayout = (j.layout && j.layout.order && j.layout.order.length) ? j.layout : null;
        setTimeout(checkDataUpdate, 1500);
        setInterval(checkDataUpdate, 180000);   /* 每 3 分钟检查一次服务端数据是否更新 */
        var hadOverride = false;
        try { hadOverride = !!localStorage.getItem(DATA_OVERRIDE_KEY); localStorage.removeItem(DATA_OVERRIDE_KEY); } catch (e) {}
        if (j.rev > cloudRev) { adoptCloud(j); location.reload(); return; }
        if (hadOverride) { location.reload(); return; }   /* 清掉旧的本机覆盖，改用服务端数据 */
        if (autoBinChanged || j.rev < cloudRev) pushCloud();
        return;
      }
      probeServer();
    })
    .catch(function () { clearTimeout(timer); probeServer(); });
}
function probeServer() {
  var ctl = new AbortController();
  var timer = setTimeout(function () { ctl.abort(); }, 2500);
  fetch('/api/status', { signal: ctl.signal })
    .then(function (r) { return r.ok ? r.json() : null; })
    .then(function (j) {
      if (j && j.job) {
        serverMode = true;
        refreshUpstream();
        if (j.job.running) { toast('数据更新进行中，完成后将自动刷新', 'info'); pollSyncStatus(); }
      }
    })
    .catch(function () { serverMode = false; })
    .then(function () { clearTimeout(timer); });
}

/* ---- 本机 → 线上 同步状态 ---- */
function renderUpstream(j) {
  var tag = $('upstream-tag'), st = $('upstream-state');
  if (!tag || !st) return;
  if (!j || !j.configured) { tag.style.display = 'none'; return; }
  tag.style.display = '';
  if (j.last_push_ok === true) { st.textContent = '已同步'; st.style.color = '#1a7f37'; }
  else if (j.last_push_ok === false) { st.textContent = '未同步'; st.style.color = '#c0392b'; }
  else { st.textContent = '待同步'; st.style.color = '#8a94a6'; }
  tag.title = (j.url || '') + ' · 上次推送：' + (j.last_push_at || '尚未推送')
    + (j.last_push_message ? ' · ' + j.last_push_message : '') + '（点击立即同步到线上）';
}
function refreshUpstream() {
  if (!serverMode) return;
  fetch('/api/upstream').then(function (r) { return r.ok ? r.json() : null; })
    .then(function (j) { if (j) renderUpstream(j); }).catch(function () {});
}
if ($('upstream-tag')) {
  $('upstream-tag').addEventListener('click', function () {
    requireAuth('同步到线上', function () {
      toast('正在同步到线上，请稍候…', 'info');
      api('/api/push', {}).then(function (j) {
        if (j && j.ok) toast('已同步到线上（线上所有页面生效）', 'ok');
        else toast('同步失败：' + ((j && j.message) || '请检查网络'), 'err');
        refreshUpstream();
      }).catch(function () { toast('无法连接本机服务', 'err'); });
    });
  });
}
/* ---- 线上：检测服务端数据是否已被更新（提示后由用户决定是否刷新） ---- */
var DATA_REV_KEY = 'wechat_data_rev_v1';
var STATE_REV_KEY = 'wechat_state_rev_v1';
function showUpdateBar(gen, label) {
  if ($('databar')) return;
  var bar = document.createElement('div');
  bar.id = 'databar';
  bar.style.cssText = 'position:fixed;left:50%;transform:translateX(-50%);bottom:18px;z-index:120;'
    + 'background:#fff;border:1px solid #d7dce3;border-radius:10px;box-shadow:0 6px 24px rgba(0,0,0,.14);'
    + 'padding:10px 14px;display:flex;align-items:center;gap:10px;font-size:13px;color:#1b2330;max-width:92vw';
  bar.innerHTML = '<span>' + (label || '数据已更新') + '（' + String(gen).slice(0, 16) + '）</span>';
  var b1 = document.createElement('button');
  b1.className = 'btn primary'; b1.style.cssText = 'padding:5px 12px;font-size:13px'; b1.textContent = '立即刷新';
  b1.onclick = function () { location.reload(); };
  var b2 = document.createElement('button');
  b2.className = 'btn'; b2.style.cssText = 'padding:5px 10px;font-size:13px'; b2.textContent = '稍后';
  b2.onclick = function () { bar.remove(); };
  bar.appendChild(b1); bar.appendChild(b2);
  document.body.appendChild(bar);
}
function checkDataUpdate() {
  if (!cloudMode) return;
  fetch('/api/data').then(function (r) { return r.ok ? r.json() : null; }).then(function (j) {
    if (!j || !j.generated_at) return;
    var cur = (window.DATA && window.DATA.generated_at) || '';
    if (!cur || j.generated_at === cur) return;
    var last = '';
    try { last = localStorage.getItem(DATA_REV_KEY) || ''; } catch (e) {}
    if (last === j.generated_at) return;    /* 已提示过，不重复打扰 */
    try { localStorage.setItem(DATA_REV_KEY, j.generated_at); } catch (e) {}
    showUpdateBar(j.generated_at);
  }).catch(function () {});
  /* 同时检查云端布局 / 回收站状态是否被其他设备更新 */
  fetch('/api/state').then(function (r) { return r.ok ? r.json() : null; }).then(function (j) {
    if (!j || typeof j.rev !== 'number') return;
    if (j.rev <= cloudRev) return;
    var last = '';
    try { last = localStorage.getItem(STATE_REV_KEY) || ''; } catch (e) {}
    if (last === String(j.rev)) return;
    try { localStorage.setItem(STATE_REV_KEY, String(j.rev)); } catch (e) {}
    showUpdateBar(j.rev, '布局或回收站已更新');
  }).catch(function () {});
}
/* 切回页面 / 窗口重新聚焦时立即检测一次，便于及时看到其他端上传的新数据 */
window.__checkDataUpdate = checkDataUpdate;
document.addEventListener('visibilitychange', function () { if (!document.hidden) checkDataUpdate(); });
window.addEventListener('focus', function () { checkDataUpdate(); });
probeBackends();
function api(path, body) {
  body = body || {};
  if (body.token === undefined) body.token = authToken();
  return fetch(path, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  }).then(function (r) {
    return r.json().catch(function () { return { ok: false, message: '响应解析失败' }; })
      .then(function (j) { j._status = r.status; return j; });
  });
}
var syncBusy = false;
function pollSyncStatus() {
  var tick = function () {
    fetch('/api/status').then(function (r) { return r.json(); }).then(function (j) {
      if (!j || !j.job) { syncBusy = false; return; }
      if (j.job.running) { setTimeout(tick, 1500); return; }
      syncBusy = false;
      if (j.job.returncode === 0) {
        var up = j.upstream || {}, m = '数据已更新', warn = false;
        if (up.configured) {
          if (up.last_push_ok === true) m += '，并已同步到线上';
          else if (up.last_push_ok === false) { m += '，同步到线上失败：' + (up.last_push_message || '未知原因'); warn = true; }
        }
        toast(m + '，正在刷新页面', warn ? 'info' : 'ok');
        setTimeout(function () { location.reload(); }, 900);
      } else {
        toast('更新未正常完成，请查看服务日志', 'err');
      }
    }).catch(function () { syncBusy = false; toast('连接中断，请稍后手动刷新查看', 'err'); });
  };
  setTimeout(tick, 600);
}
function startServerSync() {
  if (syncBusy) { toast('数据更新正在进行中，请稍候', 'info'); return; }
  askConfirm('更新数据', '将由本机服务从微信服务器采集最新数据，期间请勿关闭服务。确定开始吗？', function () {
    syncBusy = true;
    toast('正在更新数据，请稍候…', 'info');
    api('/api/sync', {}).then(function (j) {
      if (j && j.ok) { pollSyncStatus(); return; }
      if (j && j.running) { pollSyncStatus(); return; }   /* 已在进行中：直接跟进度 */
      syncBusy = false;
      toast((j && j.message) || '启动更新失败', 'err');
    }).catch(function () { syncBusy = false; toast('无法连接本机服务', 'err'); });
  });
}

/* ================= 上传数据 ================= */
function parseDataFile(text) {
  var s = String(text == null ? '' : text).replace(/^\uFEFF/, '').trim();
  if (!s) throw new Error('文件内容为空');
  var data = null, gen = null;
  if (s.charAt(0) === '{') { try { data = JSON.parse(s); } catch (e) { data = null; } }
  if (!data) {
    var sandbox = {};
    new Function('window', s)(sandbox);
    data = sandbox.DATA;
    gen = sandbox.DATA_GENERATED_AT;
  }
  if (!data || typeof data !== 'object' || Array.isArray(data)) throw new Error('未找到数据对象（应为 data.js 或等价的 JSON）');
  return { data: data, generated_at: data.generated_at || gen || '' };
}
/* 手动获取服务端最新版本：先比对生成时间，确有更新才刷新页面 */
function fetchLatestData() {
  var cur = (window.DATA || {}).generated_at || '';
  var finish = function (remote) {
    if (!remote) { location.reload(); return; }          /* 服务端不支持版本查询时直接刷新 */
    if (remote === cur) { toast('已是最新数据（' + remote + '）', 'ok'); return; }
    toast('发现新数据，正在更新…', 'ok');
    setTimeout(function () { location.reload(); }, 600);
  };
  toast('正在检查最新版本…', 'info');
  if (cloudMode) {
    fetch('/api/data', { cache: 'no-store' }).then(function (r) { return r.ok ? r.json() : null; })
      .then(function (j) { finish(j && j.generated_at); })
      .catch(function () { location.reload(); });
    return;
  }
  if (serverMode) {
    fetch('/data.js?t=' + Date.now(), { cache: 'no-store' }).then(function (r) { return r.text(); })
      .then(function (t) {
        try { finish(parseDataFile(t).data.generated_at || ''); }
        catch (e) { location.reload(); }
      }).catch(function () { location.reload(); });
    return;
  }
  location.reload();
}
/* 已解锁：打开更新面板（可上传新数据，也可取最新版本 / 采集） */
function openUpdatePanel() {
  /* 云端模式下没有“本机覆盖”概念，不显示恢复内置数据入口 */
  $('up-reset').style.display = (!cloudMode && !serverMode && localStorage.getItem(DATA_OVERRIDE_KEY)) ? '' : 'none';
  $('up-sync').style.display = serverMode ? '' : 'none';
  if (serverMode) {
    $('up-title').textContent = '更新数据';
    $('up-hint').textContent = '可上传数据文件、从微信服务器采集，或获取服务端最新版本。保存后会自动同步到线上，所有打开线上页面的设备都会更新。';
    $('up-note').textContent = '数据写入本机并同步到线上站点，同网络的其他设备与线上访问者都会看到新数据。';
  } else if (cloudMode) {
    $('up-title').textContent = '更新数据';
    $('up-hint').textContent = '选择最新的数据文件（data.js）上传，或获取当前服务端的最新版本。上传后所有打开本页面的设备都会立即生效。';
    $('up-note').textContent = '数据保存在服务端，不依赖本机浏览器。';
  } else {
    $('up-title').textContent = '更新数据';
    $('up-hint').textContent = '选择最新的数据文件（data.js）上传，或获取最新版本。上传后页面将自动刷新并使用新数据。';
    $('up-note').textContent = '上传的数据保存在当前浏览器，刷新后依然生效；换设备打开仍显示最初部署的数据。';
  }
  openOverlay('upload-overlay');
}
$('btn-update').addEventListener('click', function () {
  /* 默认（未解锁）：点「更新数据」即直接抓取服务端最新数据，无需密码 */
  if (!isUnlocked()) { fetchLatestData(); return; }
  openUpdatePanel();
});
$('up-sync').addEventListener('click', function () {
  closeOverlay('upload-overlay');
  startServerSync();
});
$('up-latest').addEventListener('click', function () {
  closeOverlay('upload-overlay');
  fetchLatestData();
});
$('up-pick').addEventListener('click', function () { $('upload-data-file').click(); });
$('up-close').addEventListener('click', function () { closeOverlay('upload-overlay'); });
$('up-reset').addEventListener('click', function () {
  askConfirm('恢复内置数据', '将清除本机上传的数据，恢复为最初部署的内置数据（不影响其他浏览器）？', function () {
    try { localStorage.removeItem(DATA_OVERRIDE_KEY); } catch (e) {}
    toast('已恢复内置数据，页面即将刷新', 'ok');
    setTimeout(function () { location.reload(); }, 700);
  });
});
$('upload-data-file').addEventListener('change', function () {
  var f = this.files && this.files[0];
  this.value = '';
  if (!f) return;
  if (f.size > 8 * 1024 * 1024) { toast('文件过大（超过 8MB），请确认是否选对了文件', 'err'); return; }
  var rd = new FileReader();
  rd.onload = function () {
    var parsed;
    try { parsed = parseDataFile(rd.result); }
    catch (e) { toast('文件解析失败：' + e.message, 'err'); return; }
    var invalid = validateData(parsed.data);
    if (invalid) { toast('数据校验未通过：' + invalid, 'err'); return; }
    /* 本机服务模式：写入本机 data.js，并由本机服务同步到线上 */
    if (serverMode) {
      api('/api/data', { data: parsed.data, generated_at: parsed.generated_at }).then(function (j) {
        if (j && j.ok) {
          closeOverlay('upload-overlay');
          var msg = '数据已保存到本机';
          if (j.pushed === true) msg += '，并已同步到线上（所有设备生效）';
          else if (j.pushed === false) msg += '，但同步到线上失败：' + (j.push_message || '未知原因');
          toast(msg + '，页面即将刷新', (j.pushed === false ? 'info' : 'ok'));
          setTimeout(function () { location.reload(); }, 900);
        } else {
          toast('保存失败：' + ((j && j.message) || '未知错误'), 'err');
        }
      }).catch(function () { toast('无法连接本机服务', 'err'); });
      return;
    }
    /* 云端模式：数据写入服务端，所有设备同步生效 */
    if (cloudMode) {
      fetch('/api/data', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ token: authToken(), data: parsed.data, generated_at: parsed.generated_at })
      }).then(function (r) { return r.ok ? r.json() : { ok: false, message: '服务响应异常' }; })
        .then(function (j) {
          if (j && j.ok) {
            closeOverlay('upload-overlay');
            toast('数据已更新（所有设备同步生效），页面即将刷新', 'ok');
            setTimeout(function () { location.reload(); }, 800);
          } else {
            toast('保存失败：' + ((j && j.message) || '未知错误'), 'err');
          }
        }).catch(function () { toast('无法连接服务，请检查网络', 'err'); });
      return;
    }
    try {
      localStorage.setItem(DATA_OVERRIDE_KEY, JSON.stringify({
        data: parsed.data, generated_at: parsed.generated_at,
        saved_at: new Date().toISOString(), file_name: f.name
      }));
    } catch (e) { toast('保存失败：浏览器存储空间不足', 'err'); return; }
    closeOverlay('upload-overlay');
    toast('数据校验通过（' + Object.keys(parsed.data.articles).length + ' 篇文章），页面即将刷新', 'ok');
    setTimeout(function () { location.reload(); }, 800);
  };
  rd.onerror = function () { toast('文件读取失败，请重试', 'err'); };
  rd.readAsText(f);
});

/* ================= Excel 导出 ================= */
function rowsForExcel(start, end) {
  var rows = filteredRows().filter(function (r) {
    if (start && (!r.pubdate || r.pubdate < start)) return false;
    if (end && (!r.pubdate || r.pubdate > end)) return false;
    return true;
  });
  return rows.map(function (r) {
    return { '发布日期': r.pubdate, '标题': r.title, '阅读': r.read, '分享': r.share,
             '收藏': r.fav, '点赞': r.like, '在看': r.zaikan, '留言': r.comment, '链接': r.url };
  });
}
function shiftDate(dStr, days) {
  var d = new Date(dStr + 'T00:00:00');
  d.setDate(d.getDate() - days);
  var m = String(d.getMonth() + 1), dd = String(d.getDate());
  return d.getFullYear() + '-' + (m.length < 2 ? '0' + m : m) + '-' + (dd.length < 2 ? '0' + dd : dd);
}
/* 当前选择的导出范围；返回 null 表示自定义日期未填完整 */
function exportRange() {
  var checked = document.querySelector('input[name="exp-range"]:checked');
  var v = checked ? checked.value : 'all';
  if (v === 'all') return { start: '', end: '', label: '全部数据' };
  if (v === 'custom') {
    var s = $('exp-start').value, e = $('exp-end').value;
    if (!s || !e) return null;
    if (s > e) { var t = s; s = e; e = t; }
    return { start: s, end: e, label: s + ' 至 ' + e };
  }
  /* 快捷选项以数据中最新发布日期为基准 */
  var maxD = '';
  articleRows.forEach(function (r) { if (r.pubdate && r.pubdate > maxD) maxD = r.pubdate; });
  if (!maxD) maxD = new Date().toISOString().slice(0, 10);
  if (v === 'd1') return { start: maxD, end: maxD, label: '最近一天' };
  if (v === 'd7') return { start: shiftDate(maxD, 6), end: maxD, label: '最近七天' };
  return { start: shiftDate(maxD, 29), end: maxD, label: '最近一个月' };
}
$('btn-export').addEventListener('click', function () {
  requireAuth('导出数据', function () {
    if (!window.XLSX) { toast('导出库未加载，无法导出', 'err'); return; }
    openOverlay('export-overlay');
  });
});
Array.prototype.forEach.call(document.querySelectorAll('input[name="exp-range"]'), function (r) {
  r.addEventListener('change', function () {
    $('exp-dates').style.display = this.value === 'custom' ? 'flex' : 'none';
  });
});
$('exp-cancel').addEventListener('click', function () { closeOverlay('export-overlay'); });
$('exp-close').addEventListener('click', function () { closeOverlay('export-overlay'); });
$('exp-ok').addEventListener('click', function () {
  var rg = exportRange();
  if (!rg) { toast('请选择完整的起止日期', 'err'); return; }
  var rows = rowsForExcel(rg.start, rg.end);
  if (!rows.length) { toast('所选范围内没有可导出的数据', 'err'); return; }
  var ws = XLSX.utils.json_to_sheet(rows);
  ws['!cols'] = [{ wch: 12 }, { wch: 46 }, { wch: 10 }, { wch: 10 }, { wch: 10 }, { wch: 10 }, { wch: 10 }, { wch: 10 }, { wch: 40 }];
  var wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, '文章数据');
  var d = (D.generated_at || '').replace(/[-: ]/g, '');
  XLSX.writeFile(wb, '公众号数据_' + rg.label.replace(/[ :]/g, '') + '_' + (d || '导出') + '.xlsx');
  closeOverlay('export-overlay');
  toast('已导出 ' + rows.length + ' 条记录（' + rg.label + '）', 'ok');
});

/* ================= 布局调整 ================= */
var DEFAULT_LAYOUT = { order: ['kpis', 'trend', 'source', 'fans', 'rank', 'table'], span: { kpis: 12, trend: 8, source: 4, fans: 6, rank: 6, table: 12 }, heights: {} };
var layout = null;
function normalizeLayout(saved) {
  if (!saved || !saved.order || !saved.order.length) return JSON.parse(JSON.stringify(DEFAULT_LAYOUT));
  saved.order = saved.order.filter(function (p) { return DEFAULT_LAYOUT.order.indexOf(p) >= 0; });
  DEFAULT_LAYOUT.order.forEach(function (p) { if (saved.order.indexOf(p) < 0) saved.order.push(p); });
  if (!saved.span) saved.span = JSON.parse(JSON.stringify(DEFAULT_LAYOUT.span));
  DEFAULT_LAYOUT.order.forEach(function (p) { if (typeof saved.span[p] !== 'number') saved.span[p] = DEFAULT_LAYOUT.span[p]; });
  if (!saved.heights || typeof saved.heights !== 'object') saved.heights = {};
  return saved;
}
/* 优先级：本次查看的临时布局 > 已保存的默认布局 > 系统默认 */
function loadLayout() {
  var tmp = SS(LAYOUT_TMP_KEY, null);
  if (tmp && tmp.order && tmp.order.length) return normalizeLayout(tmp);
  return normalizeLayout(LS(LAYOUT_KEY, null));
}
function applyLayout() {
  var board = $('board');
  layout.order.forEach(function (pid) {
    var mod = board.querySelector('[data-pid="' + pid + '"]');
    if (!mod) return;
    mod.style.gridColumn = 'span ' + (layout.span[pid] || 12);
    var chart = mod.querySelector('.chart');
    if (chart && layout.heights && layout.heights[pid]) chart.style.height = layout.heights[pid] + 'px';
    board.appendChild(mod);
  });
}
var editMode = false;
var layoutSnapshot = null;   /* 进入调整前的布局，用于「放弃调整」 */
function setEditMode(on) {
  editMode = on;
  document.body.classList.toggle('editing', on);
  $('btn-layout').classList.toggle('on', on);
  $('btn-layout').lastChild.textContent = on ? ' 完成' : ' 布局';
}
/* 进入布局调整：无需密码；退出时只写入本次查看的临时布局 */
$('btn-layout').addEventListener('click', function () {
  if (editMode) {
    setEditMode(false);
    SSS(LAYOUT_TMP_KEY, layout);
    toast('布局已应用，仅本次查看有效', 'ok');
    return;
  }
  layoutSnapshot = JSON.parse(JSON.stringify(layout));
  setEditMode(true);
});
$('btn-layout-cancel').addEventListener('click', function () {
  if (!layoutSnapshot) { setEditMode(false); return; }
  layout = JSON.parse(JSON.stringify(layoutSnapshot));
  applyLayout();
  SSS(LAYOUT_TMP_KEY, layout);
  setEditMode(false);
  Object.keys(chartMap).forEach(function (k) { if (chartMap[k]) chartMap[k].resize(); });
  toast('已放弃本次调整', 'info');
});
/* 保存为默认布局：需要密码，永久生效；线上模式下同步到所有设备 */
$('btn-layout-save').addEventListener('click', function () {
  requireAuth('保存布局', function () {
    var snap = JSON.parse(JSON.stringify(layout));
    snap.saved_at = new Date().toISOString();
    LSS(LAYOUT_KEY, snap);
    SSS(LAYOUT_TMP_KEY, snap);
    setEditMode(false);
    if (cloudMode) {
      cloudLayout = snap;
      pushCloud();
      toast('布局已保存，其他设备刷新后同步生效', 'ok');
    } else {
      toast('布局已保存为默认，下次打开依然生效', 'ok');
    }
  });
});
/* 恢复默认布局：需要密码，清除临时布局与已保存的默认布局；线上模式下同步清除 */
$('btn-layout-reset').addEventListener('click', function () {
  requireAuth('恢复默认布局', function () {
    LSDEL(LAYOUT_KEY);
    SSDEL(LAYOUT_TMP_KEY);
    layout = JSON.parse(JSON.stringify(DEFAULT_LAYOUT));
    applyLayout();
    setEditMode(false);
    Object.keys(chartMap).forEach(function (k) { if (chartMap[k]) chartMap[k].resize(); });
    if (cloudMode) {
      cloudLayout = null;
      pushCloud();
      toast('已恢复系统默认布局，其他设备刷新后同步生效', 'ok');
    } else {
      toast('已恢复系统默认布局', 'ok');
    }
  });
});
$('board').addEventListener('click', function (e) {
  var btn = e.target.closest ? e.target.closest('[data-move]') : null;
  if (!btn || !editMode) return;
  var mod = btn.closest('.mod');
  if (!mod) return;
  var pid = mod.getAttribute('data-pid');
  var i = layout.order.indexOf(pid);
  if (btn.getAttribute('data-move') === 'up' && i > 0) {
    layout.order[i] = layout.order[i - 1]; layout.order[i - 1] = pid;
  } else if (btn.getAttribute('data-move') === 'down' && i < layout.order.length - 1) {
    layout.order[i] = layout.order[i + 1]; layout.order[i + 1] = pid;
  }
  applyLayout();
  Object.keys(chartMap).forEach(function (k) { if (chartMap[k]) chartMap[k].resize(); });
});

/* ================= 自由拖拽编辑（位置 + 大小） ================= */
function resizeAllCharts() { Object.keys(chartMap).forEach(function (k) { if (chartMap[k]) chartMap[k].resize(); }); }
/* 给每个板块注入右下角「调整大小」手柄（仅编辑模式可见） */
(function injectHandles() {
  Array.prototype.forEach.call(document.querySelectorAll('#board .mod'), function (mod) {
    if (mod.querySelector('.rs-handle')) return;
    var h = document.createElement('div');
    h.className = 'rs-handle';
    h.title = '拖动调整大小';
    mod.appendChild(h);
  });
})();
var rsBadge = null;
function showRsBadge(x, y, text) {
  if (!rsBadge) { rsBadge = document.createElement('div'); rsBadge.className = 'rs-badge'; document.body.appendChild(rsBadge); }
  rsBadge.textContent = text;
  rsBadge.style.left = x + 'px'; rsBadge.style.top = y + 'px';
  rsBadge.style.display = '';
}
function hideRsBadge() { if (rsBadge) rsBadge.style.display = 'none'; }

/* ---- 拖动板块标题栏换位 ---- */
(function setupDragSort() {
  var board = $('board');
  var drag = null;
  function moveGhost(x, y) {
    drag.ghost.style.left = (x - drag.dx) + 'px';
    drag.ghost.style.top = (y - drag.dy) + 'px';
  }
  board.addEventListener('pointerdown', function (e) {
    if (!editMode) return;
    var head = e.target.closest ? e.target.closest('.mod-head') : null;
    if (!head || e.target.closest('.editbar')) return;
    var mod = head.closest('.mod');
    if (!mod) return;
    e.preventDefault();
    var r = mod.getBoundingClientRect();
    var ghost = document.createElement('div');
    ghost.className = 'dnd-ghost';
    ghost.style.width = Math.min(r.width, 320) + 'px';
    ghost.textContent = mod.querySelector('h2') ? mod.querySelector('h2').textContent : '板块';
    document.body.appendChild(ghost);
    drag = { mod: mod, ghost: ghost, dx: e.clientX - r.left, dy: e.clientY - r.top };
    mod.classList.add('drag-src');
    document.body.classList.add('dnd-active');
    moveGhost(e.clientX, e.clientY);
    try { board.setPointerCapture(e.pointerId); } catch (err) {}
  });
  board.addEventListener('pointermove', function (e) {
    if (!drag) return;
    e.preventDefault();
    moveGhost(e.clientX, e.clientY);
    var mods = Array.prototype.filter.call(board.querySelectorAll('.mod'), function (m) { return m !== drag.mod; });
    var target = null;
    for (var i = 0; i < mods.length; i++) {
      var r = mods[i].getBoundingClientRect();
      if (e.clientY < r.top + r.height / 2) { target = mods[i]; break; }
      if (e.clientY < r.bottom && e.clientX < r.left + r.width / 2) { target = mods[i]; break; }
    }
    if (target) { if (target.previousElementSibling !== drag.mod) board.insertBefore(drag.mod, target); }
    else if (board.lastElementChild !== drag.mod) board.appendChild(drag.mod);
  });
  function endDrag() {
    if (!drag) return;
    drag.mod.classList.remove('drag-src');
    drag.ghost.parentNode && drag.ghost.parentNode.removeChild(drag.ghost);
    document.body.classList.remove('dnd-active');
    /* 按当前 DOM 顺序回写布局 */
    layout.order = Array.prototype.map.call(board.querySelectorAll('.mod'), function (m) { return m.getAttribute('data-pid'); });
    drag = null;
    resizeAllCharts();
  }
  board.addEventListener('pointerup', endDrag);
  board.addEventListener('pointercancel', endDrag);
})();

/* ---- 拖右下角手柄调大小（宽 = 栅格跨度，高 = 图表高度） ---- */
(function setupResize() {
  var board = $('board');
  var rs = null;
  board.addEventListener('pointerdown', function (e) {
    if (!editMode) return;
    var h = e.target.closest ? e.target.closest('.rs-handle') : null;
    if (!h) return;
    var mod = h.closest('.mod');
    var pid = mod.getAttribute('data-pid');
    e.preventDefault();
    e.stopPropagation();
    var chart = mod.querySelector('.chart');
    rs = {
      mod: mod, pid: pid, chart: chart,
      x: e.clientX, y: e.clientY,
      span: layout.span[pid] || 12,
      h: chart ? chart.offsetHeight : 0,
      colW: board.getBoundingClientRect().width / 12
    };
    document.body.classList.add('dnd-active');
    try { h.setPointerCapture(e.pointerId); } catch (err) {}
  });
  document.addEventListener('pointermove', function (e) {
    if (!rs) return;
    e.preventDefault();
    var span = Math.max(3, Math.min(12, Math.round(rs.span + (e.clientX - rs.x) / rs.colW)));
    if (span !== (layout.span[rs.pid] || 12)) {
      layout.span[rs.pid] = span;
      rs.mod.style.gridColumn = 'span ' + span;
    }
    var txt = '宽 ' + span + '/12';
    if (rs.chart) {
      var h = Math.max(160, Math.min(560, Math.round(rs.h + (e.clientY - rs.y))));
      if (h !== rs.chart.offsetHeight) {
        rs.chart.style.height = h + 'px';
        layout.heights[rs.pid] = h;
      }
      txt += ' · 高 ' + (layout.heights[rs.pid] || rs.chart.offsetHeight) + 'px';
    }
    showRsBadge(e.clientX, e.clientY, txt);
    var c = rs.chart && chartMap[rs.chart.id];
    if (c) c.resize();
  });
  function endRs() {
    if (!rs) return;
    rs = null;
    document.body.classList.remove('dnd-active');
    hideRsBadge();
    resizeAllCharts();
  }
  document.addEventListener('pointerup', endRs);
  document.addEventListener('pointercancel', endRs);
})();

/* ================= 快捷键 ================= */
document.addEventListener('keydown', function (e) {
  var tag = (e.target.tagName || '').toLowerCase();
  if (tag === 'input' || tag === 'textarea') {
    if (e.key === 'Escape') e.target.blur();
    return;
  }
  if (e.key === 'Escape') {
    var open = document.querySelector('.overlay.open');
    if (open) { closeOverlay(open.id); if (open.id === 'auth-overlay') authCb = null; }
    else if (editMode) { setEditMode(false); SSS(LAYOUT_TMP_KEY, layout); }
  } else if (e.key === '/') { e.preventDefault(); $('f-q').focus(); }
});

/* ================= 启动 ================= */
layout = loadLayout();
applyLayout();
renderKpis();
renderTable();
renderRecycleBadge();
updateLockBtn();
window.addEventListener('load', function () {
  requestAnimationFrame(function () {
    renderCharts();
    if (!window.echarts) {
      var s = document.createElement('script');
      s.src = 'vendor/echarts.min.js';
      s.onload = function () { renderCharts(); };
      document.head.appendChild(s);
    }
  });
});
})();
</script>
</body>
</html>
```

> ⚠️ 别忘了 `vendor/` 目录：页面需要 `vendor/echarts.min.js`（图表）和 `vendor/xlsx.full.min.js`（导出 Excel）。
> 这两个是第三方库，体积较大（约 1.9 MB），**请从仓库直接下载**：
> https://github.com/Yellowsoy/gzh-data-dashboard/tree/main/local-deployment/vendor

### 第一次采集

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

### 打开看板

**推荐方式**（功能最全）：启动本地服务

```bash
python3 serve.py           # Windows 用 python serve.py
```

终端会打印 `http://127.0.0.1:8000`，浏览器打开它。Mac/Linux 也可以直接 `./run.sh`。

**备用方式**：直接双击 `index.html`。能看，但"更新数据""删除"这类需要服务端的功能会提示不可用。

> **验收点（KR3）**：页面顶部 KPI 有数字、趋势图有曲线、文章明细表有行。**如果是空的，说明上一步没采到数据，回去看采集日志。**

### 认识顶栏的「锁定」按钮

页面默认**锁定（只读）**，任何人打开都只能看。点顶栏最右的「锁定」，输入管理密码才能解锁管理功能。

| 状态 | 能做什么 |
| --- | --- |
| **锁定**（默认） | 看图表和明细；点「更新数据」拉取服务端最新版本；临时调布局（仅本次有效） |
| **已解锁**（输密码后） | 上传数据、回收站、删除/还原、重命名、保存为默认布局、同步到线上 |

- 密码**只存在服务端**，页面源码里没有；解锁后下发 8 小时令牌，关掉页面即失效
- 再点一次「锁定」立即锁回去
- **务必改掉默认密码**：改 `serve.py` 里的 `ACCESS_PWD`

### 本地部署完整验收清单

- [ ] 6 个文件都放进了同一个目录（KR1 前置）
- [ ] `config.json` 填好，且能拿到 token（KR1）
- [ ] `sync.py` 跑完无报错，`store.json` / `data.js` 已生成（KR2）
- [ ] 页面 KPI、趋势图、明细表都有数据（KR3）
- [ ] 点「锁定」输密码能解锁，刷新后回到锁定状态

## 七、线上部署（完整源码）：把数据送到线上端

做到第六节，看板只有你自己电脑能看。这一节把它变成"手机也能开"。

### 文件 7：`server.py`（线上服务）

> 线上版**只需要 3 样东西**：`server.py`、`index.html`（和本地版**同一个文件**，第六节那个）、`vendor/`。
> 它不调微信接口，所以**不需要 AppSecret、不需要配 IP 白名单**。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
微信公众号数据看板 —— 线上服务（仅标准库）
- 静态提供 index.html / data.js / vendor/（其余文件不对外）
- GET  /api/state   读取共享状态：{rev, deleted, titles, layout}（回收站/重命名/布局，全设备同步）
- POST /api/auth    管理密码验证（密码只存服务端），成功下发临时 token
- POST /api/state   写入共享状态（需 token）
- POST /api/data    上传数据文件内容（需 token），写入 data.js，全设备生效
监听 PORT 环境变量，绑定 0.0.0.0。管理员密码可用环境变量 ADMIN_PWD 覆盖。
"""

import json
import os
import re
import secrets
import threading
import time
import tempfile
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse

BASE = os.path.dirname(os.path.abspath(__file__))
PORT = int(os.environ.get("PORT", "3000"))
# 管理密码只保存在服务端。默认值仅为占位符，部署前必须通过环境变量 ADMIN_PWD 设置。
PWD = os.environ.get("ADMIN_PWD", "change-me")
STATE_PATH = os.path.join(BASE, "state.json")
DATA_PATH = os.path.join(BASE, "data.js")
MAX_BODY = 12 * 1024 * 1024  # 12MB
TOKEN_TTL = 8 * 60 * 60      # 解锁令牌有效期：8 小时

_TOKENS = {}                 # token -> 过期时间戳
_TOKEN_LOCK = threading.Lock()


def issue_token():
    tok = secrets.token_hex(24)
    with _TOKEN_LOCK:
        now = time.time()
        # 顺带清理过期 token
        for k in [k for k, exp in _TOKENS.items() if exp < now]:
            _TOKENS.pop(k, None)
        _TOKENS[tok] = now + TOKEN_TTL
    return tok


def check_token(tok):
    if not isinstance(tok, str) or not tok:
        return False
    with _TOKEN_LOCK:
        exp = _TOKENS.get(tok)
    return bool(exp) and exp > time.time()


def revoke_token(tok):
    with _TOKEN_LOCK:
        _TOKENS.pop(tok, None)

STATIC_ALLOWED = {
    "/": "index.html",
    "/index.html": "index.html",
    "/data.js": "data.js",
    "/vendor/echarts.min.js": "vendor/echarts.min.js",
    "/vendor/xlsx.full.min.js": "vendor/xlsx.full.min.js",
}
CONTENT_TYPE = {".html": "text/html; charset=utf-8", ".js": "application/javascript; charset=utf-8"}


def read_state():
    try:
        with open(STATE_PATH, encoding="utf-8") as f:
            j = json.load(f)
        if isinstance(j, dict) and isinstance(j.get("rev"), (int, float)):
            if not isinstance(j.get("deleted"), dict):
                j["deleted"] = {}
            if not isinstance(j.get("titles"), dict):
                j["titles"] = {}
            if "layout" not in j:
                j["layout"] = None
            return j
    except Exception:
        pass
    return {"rev": 0, "deleted": {}, "titles": {}, "layout": None}


def valid_layout(v):
    """布局：None 表示恢复系统默认；对象需含合法的 order / span。"""
    if v is None:
        return True
    if not isinstance(v, dict):
        return False
    order = v.get("order")
    if not isinstance(order, list) or not order:
        return False
    if any(not isinstance(x, str) for x in order):
        return False
    span = v.get("span")
    if span is not None and not isinstance(span, dict):
        return False
    heights = v.get("heights")
    if heights is not None and not isinstance(heights, dict):
        return False
    return True


_GEN_RE = re.compile(r'window\.DATA_GENERATED_AT\s*=\s*"([^"]*)"')


def data_meta():
    """读取 data.js 的轻量元信息（生成时间/大小/更新时间），供各端检测是否有新数据。"""
    try:
        st = os.stat(DATA_PATH)
        with open(DATA_PATH, "rb") as f:
            f.seek(0, os.SEEK_END)
            size = f.tell()
            f.seek(max(0, size - 8192))
            tail = f.read().decode("utf-8", "replace")
        m = _GEN_RE.search(tail)
        return {"ok": True, "generated_at": m.group(1) if m else "",
                "size": st.st_size, "updated_at": int(st.st_mtime * 1000)}
    except Exception:
        return {"ok": True, "generated_at": "", "size": 0, "updated_at": 0}


class _Win(object):
    pass


def load_data_dict():
    """解析 data.js，返回完整数据字典（供需要回拉数据的端使用）。"""
    w = _Win()
    try:
        with open(DATA_PATH, encoding="utf-8") as f:
            src = f.read()
        # data.js 为 JSON 直出，可能含 false/true/null，需映射为 Python 字面量
        exec(compile(src, "data.js", "exec"),
             {"window": w, "false": False, "true": True, "null": None})
    except Exception:
        return None
    return getattr(w, "DATA", None)


def write_json_atomic(path, obj):
    fd, tmp = tempfile.mkstemp(dir=BASE, suffix=".tmp")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(obj, f, ensure_ascii=False)
        os.replace(tmp, path)
    except Exception:
        try:
            os.unlink(tmp)
        except OSError:
            pass
        raise


def valid_data(d):
    return (
        isinstance(d, dict)
        and isinstance(d.get("articles"), dict)
        and isinstance(d.get("article_daily"), dict)
        and isinstance(d.get("biz_daily"), dict)
        and isinstance(d.get("user_daily"), dict)
        and bool(d.get("generated_at"))
    )


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def log_message(self, *args):  # 静默访问日志
        pass

    def _json(self, obj, code=200):
        body = json.dumps(obj, ensure_ascii=False).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(body)

    def _static(self, rel):
        fp = os.path.join(BASE, rel)
        try:
            with open(fp, "rb") as f:
                body = f.read()
        except OSError:
            self._json({"ok": False, "message": "not found"}, code=404)
            return
        ext = os.path.splitext(fp)[1].lower()
        self.send_response(200)
        self.send_header("Content-Type", CONTENT_TYPE.get(ext, "application/octet-stream"))
        self.send_header("Content-Length", str(len(body)))
        # 数据与页面不缓存，保证各端拿到最新；库文件可缓存
        if rel.startswith("vendor/"):
            self.send_header("Cache-Control", "public, max-age=86400")
        else:
            self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self):
        path = urlparse(self.path).path
        if path == "/api/state":
            self._json(read_state())
            return
        if path == "/api/data":
            self._json(data_meta())
            return
        if path == "/api/data/full":
            d = load_data_dict()
            if not isinstance(d, dict):
                self._json({"ok": False, "message": "数据文件不可用"}, code=500)
            else:
                self._json({"ok": True, "generated_at": d.get("generated_at", ""), "data": d})
            return
        if path == "/favicon.ico":
            self.send_response(204)
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        if path in STATIC_ALLOWED:
            self._static(STATIC_ALLOWED[path])
            return
        self._json({"ok": False, "message": "not found"}, code=404)

    def do_POST(self):
        path = urlparse(self.path).path
        try:
            length = int(self.headers.get("Content-Length") or 0)
        except ValueError:
            length = 0
        if length <= 0 or length > MAX_BODY:
            self._json({"ok": False, "message": "请求体无效或过大"}, code=400)
            return
        try:
            body = json.loads(self.rfile.read(length).decode("utf-8"))
        except Exception:
            self._json({"ok": False, "message": "请求不是有效的 JSON"}, code=400)
            return
        if not isinstance(body, dict):
            self._json({"ok": False, "message": "请求格式错误"}, code=400)
            return

        # 管理密码验证：密码只在这里比对，成功下发临时 token
        if path == "/api/auth":
            if body.get("pwd") != PWD:
                time.sleep(0.4)   # 轻微延迟，增加暴力尝试成本
                self._json({"ok": False, "message": "密码不正确"}, code=403)
                return
            tok = issue_token()
            self._json({"ok": True, "token": tok, "expires_in": TOKEN_TTL})
            return
        if path == "/api/logout":
            revoke_token(body.get("token"))
            self._json({"ok": True})
            return

        if path in ("/api/state", "/api/data"):
            if not check_token(body.get("token")):
                self._json({"ok": False, "message": "未解锁或登录已过期，请重新验证"}, code=403)
                return
            if path == "/api/state":
                rev = body.get("rev")
                deleted = body.get("deleted")
                titles = body.get("titles")
                if not isinstance(rev, (int, float)) or not isinstance(deleted, dict) or not isinstance(titles, dict):
                    self._json({"ok": False, "message": "缺少 rev/deleted/titles 或格式错误"}, code=400)
                    return
                cur = read_state()
                # 布局：本次未传则保持原值；传 null 表示恢复系统默认布局
                layout = cur.get("layout")
                if "layout" in body:
                    lv = body.get("layout")
                    if not valid_layout(lv):
                        self._json({"ok": False, "message": "layout 格式错误"}, code=400)
                        return
                    layout = lv if isinstance(lv, dict) else None
                now_ms = int(time.time() * 1000)
                # 防止异常大的 rev 把状态“锁死”在将来，导致后续同步全部被判为旧数据
                if rev > now_ms + 60 * 60 * 1000:
                    rev = now_ms
                if rev < cur["rev"]:
                    # 旧数据不覆盖新数据，直接返回当前状态
                    self._json({"ok": True, "rev": cur["rev"], "stale": True})
                    return
                write_json_atomic(STATE_PATH, {"rev": rev, "deleted": deleted,
                                               "titles": titles, "layout": layout})
                self._json({"ok": True, "rev": rev, "has_layout": layout is not None})
                return
            data = body.get("data")
            gen = body.get("generated_at") or (data.get("generated_at") if isinstance(data, dict) else "")
            if not valid_data(data):
                self._json({"ok": False, "message": "数据校验未通过（缺少必要字段）"}, code=400)
                return
            if gen:
                data["generated_at"] = gen
            text = "window.DATA = " + json.dumps(data, ensure_ascii=False) + ";\n" \
                   + 'window.DATA_GENERATED_AT = "' + str(gen) + '";\n'
            fd, tmp = tempfile.mkstemp(dir=BASE, suffix=".tmp")
            try:
                with os.fdopen(fd, "w", encoding="utf-8") as f:
                    f.write(text)
                os.replace(tmp, DATA_PATH)
            except Exception:
                try:
                    os.unlink(tmp)
                except OSError:
                    pass
                self._json({"ok": False, "message": "写入数据文件失败"}, code=500)
                return
            self._json({"ok": True, "articles": len(data["articles"])})
            return

        self._json({"ok": False, "message": "not found"}, code=404)


def main():
    srv = ThreadingHTTPServer(("0.0.0.0", PORT), Handler)
    print("看板服务已启动，端口 %d" % PORT, flush=True)
    try:
        srv.serve_forever()
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    main()
```

### 部署步骤

1. 把 `server.py`、`index.html`、`vendor/` 传到任意能跑 Python 的服务器/云平台（同一个目录）
2. **设置管理密码后启动**：

```bash
export ADMIN_PWD=你的密码        # Windows: set ADMIN_PWD=你的密码
python3 server.py
```

3. 服务监听 `PORT` 环境变量（缺省 3000），绑定 `0.0.0.0`
4. 浏览器打开线上地址，能看到页面（此时还没数据，等本地推过来）

> **不要用默认占位密码**，部署前必须通过 `ADMIN_PWD` 设置一个自己的密码。

### 让本地采完自动推上去（推荐）

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

### 另外两种上传方式

**方式二：在线上页面直接上传**

1. 打开线上地址 → 点「锁定」输密码解锁
2. 点「更新数据 → 选择数据文件」
3. 选中本地生成的 `data.js` → 上传

**方式三：直接把 `data.js` 拷到线上目录**（有服务器访问权限时最快），覆盖即可。

### 怎么确认同步成功了

1. 看本地顶栏：**「线上：已同步」**
2. 打开线上页面，看数据生成时间是否变成刚才的
3. 本地删一篇文章 → 线上刷新 → 这篇文章也消失了（说明回收站状态同步了）

显示「未同步」时，鼠标悬停在标记上会看到失败原因。最常见的是**本机暂时访问不了外网**（单位内网限制）——数据已经存在本机，恢复网络后点标记重试即可。每次同步结果都记在 `sync.log` 里。

> **验收点（KR4）**：线上地址能看到和本地一样的数据；本地删一篇，手机上也消失。

### 线上端的小提醒

- 别人正在看线上页面时，检测到数据更新会在底部弹「数据已更新 → 立即刷新 / 稍后」，由他自己决定何时刷新，不会打断操作
- **微信里直接打开链接可能被拦**（部分平台网关按 UA 拦截微信内置浏览器），用系统浏览器打开
- `upstream_auto_push` 设成 `false` 可关掉自动同步，需要时再手动点

## 八、设置每天自动同步

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

## 九、排障大全（几乎所有你可能遇到的问题）

### 环境与安装

**Q：`python3 --version` 提示"不是内部或外部命令"？**
Windows 装 Python 时没勾 "Add Python to PATH"。重装一次并勾上，或手动把 Python 目录加进环境变量。

**Q：Windows 上脚本报编码错误（UnicodeDecodeError / gbk）？**
文件路径或目录名带中文/空格。把整个文件夹挪到纯英文路径，如 `D:\wechat-dashboard`。

**Q：端口被占用？**
不用管，服务会自动顺延到 8001–8020 并打印实际地址。想固定就 `./run.sh --port 8080`，或在 `config.json` 里加 `"serve_port": 8080`。

**Q：双击 `index.html` 页面是空的？**
还没采集数据，先跑一次 `python3 sync.py`。另外文件直开模式下部分功能（删除、上传）不可用，请用 `python3 serve.py` 启动服务访问。

**Q：图表不显示 / 页面白屏？**
`vendor/` 文件夹没一起拷过去。确认目录里有 `vendor/echarts.min.js` 和 `vendor/xlsx.full.min.js`。

### 凭据与权限

**Q：报 `40001` / `40013`？**
AppSecret 填错或 token 失效。核对 `config.json`（注意别多复制空格），脚本会自动重取 token。

**Q：重置 AppSecret 后老系统也挂了？**
重置会让旧值立即失效。如果还有别的系统在用，需要同步更新它们。

**Q：报 `48001`？**
接口未授权。需要公众号**已认证**且具备对应权限。数据接口不受影响的，只是标题可能取不到——可以在明细表标题旁点「命名」手动补。

**Q：我不是管理员，也绑不了运营者？**
只能请管理员代劳：重置 AppSecret 发给你 + 帮忙填 IP 白名单。

**Q：提示"未解锁或登录已过期"？**
令牌 8 小时过期，点顶栏「锁定」重新输密码。

### 网络与白名单

**Q：报 `40164 / not in whitelist`？**
IP 不在白名单。脚本会打印你当前的公网 IP，把它加进白名单，等 1–2 分钟生效后重跑。详见 5.4。

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
老接口已下线，用本版脚本即可，别改回旧接口名。

**Q：有些文章标题显示"未获取到标题（xxx）"？**
脚本会通过「已发布消息列表」自动补全，包括 2025-11-01 之前的老文章。如果仍显示：
1. 发布列表接口无权限（48001）→ 只能点「命名」手动补
2. 该文章确实不在发布列表（未发布 / 已删 / 转载）→ 同上手动补
3. 当天被限流 → **下次运行自动补扫**，不用管。多次不行就 `python3 sync.py --full` 强制全量重扫

**Q：删掉的文章，重新采集后又冒出来了？**
不会。删除记录写在 `store.json` 的台账里，生成数据时始终过滤。想找回就去回收站恢复。

**Q：发布日期和实际不符？**
已按「文章日报最早日期」自动校正。若仍不对，跑一次 `python3 sync.py` 触发重算（日志会打印修正了多少篇）。

### 线上与同步

**Q：本机上传的数据，线上看不到？**
看顶栏「线上：」标记是否为「已同步」。显示「未同步」时悬停看原因，点标记可手动重试（需密码）。本机访问不了外网时，数据仍完整保存在本机，联网后重试即可。

**Q：微信里打不开线上链接？**
网关按 UA 拦截了微信内置浏览器，用系统浏览器打开。

**Q：线上页面一直显示旧数据？**
线上页面需要刷新。检测到更新时底部会弹提示，点「立即刷新」即可。

## 十、备份、迁移与恢复

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

---

## 附录 A：仓库其余 6 个文件的完整内容 + 全部文件对照表

前面十章给出了运行必需的 7 个文件。仓库里还有 6 个文档/配置文件，完整内容如下，与仓库 `main` 分支逐字一致。

### 仓库全部文件 ↔ 本文位置 对照表

| 仓库文件 | 大小 | 本文位置 |
| --- | --- | --- |
| `local-deployment/config.example.json` | 0.4 KB | 第六章 · 文件 1 |
| `local-deployment/run.sh` | 0.9 KB | 第六章 · 文件 2 |
| `local-deployment/stop.sh` | 2.1 KB | 第六章 · 文件 3 |
| `local-deployment/sync.py` | 39.8 KB | 第六章 · 文件 4 |
| `local-deployment/serve.py` | 31.1 KB | 第六章 · 文件 5 |
| `local-deployment/index.html` | 125.4 KB | 第六章 · 文件 6 |
| `workbuddy-deployment/server.py` | 11.9 KB | 第七章 · 文件 7 |
| `workbuddy-deployment/index.html` | 125.4 KB | 与「文件 6」**完全相同**，两版共用同一份 |
| `.gitignore` | 0.3 KB | 附录 A · 文件 8 |
| `LICENSE` | 1.1 KB | 附录 A · 文件 9 |
| `CHANGELOG.md` | 2.0 KB | 附录 A · 文件 10 |
| `CONTRIBUTING.md` | 1.4 KB | 附录 A · 文件 11 |
| `README.md` | 16.3 KB | 附录 A · 文件 12 |
| `local-deployment/README.md` | 21.9 KB | 附录 A · 文件 13 |
| `local-deployment/vendor/`（2 个库） | 1.87 MB | 附录 B · 获取命令 + 校验值 |
| `workbuddy-deployment/vendor/`（2 个库） | 1.87 MB | 同附录 B（内容完全相同） |

> 合计 18 个文件，除 4 个第三方库文件外，**其余 14 个文件的完整内容全部已在本文中给出**。

### 文件 8：`.gitignore`

````text
# 敏感信息与本地运行产物，一律不入库
config.json
token.json
store.json
sync.log
serve.pid
__pycache__/
*.pyc

# 数据文件（由采集脚本生成或页面上传，含业务数据）
data.js
data.js.tmp
state.json
store.json.tmp

# 系统
.DS_Store
Thumbs.db
````

### 文件 9：`LICENSE`（MIT 协议）

````text
MIT License

Copyright (c) 2026 GZH Data Dashboard Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
````

### 文件 10：`CHANGELOG.md`

````markdown
# 变更日志

本项目遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.0.1] - 2026-10-19

### 安全

- 移除仓库内全部单位相关标识：页面标题、文档、目录结构示例中的单位名称及拼音简写均已清除，仓库名与项目描述同步复核
- 重写全部 Git 提交历史（`git filter-branch`）并重建仓库，确保历史提交对象中不残留任何单位标识
- 默认管理密码由硬编码值改为占位符 `change-me`，两版本统一支持环境变量 `ADMIN_PWD` 覆盖；README 标注「部署前必须修改」

## [1.0.0] - 2026-10-19

首个公开版本。

### 新增

- 关键指标：累计粉丝、最新日净增、近 7 天阅读/分享人数、文章总数，以及**周发文数 / 周阅读数**（支持「近7天滚动」与「本周周一至周日」两种口径，选择结果本地记忆）
- 阅读与分享趋势图：底部比例尺拖拽缩放 + 滚轮/双指缩放，配套「全部 / 近90天 / 近30天 / 近7天」快捷范围按钮
- 阅读来源饼图、粉丝趋势、文章排行
- 文章明细：搜索（含拼音首字母）、日期筛选、排序、分页、按时间范围导出 Excel
- 回收站：删除 / 还原 / 彻底删除，状态云端同步；未解锁时入口完全隐藏
- 布局自定义：模块拖拽位置与大小；临时布局（仅本次查看）与默认布局（云端保存）两级
- 服务端密码验证：`POST /api/auth` 校验后下发 8 小时有效令牌，页面源码不含密码
- 本地版：微信官方接口按日采集、本地 → 线上自动同步、发布日期按日报最早日期校正
- 双版本目录：`local-deployment`（本地电脑）与 `workbuddy-deployment`（WorkBuddy 云端）

### 安全

- 所有写接口仅接受服务端令牌，明文密码调用一律 403
- 公众号凭据与运行数据通过 `.gitignore` 排除，不入库

### 已知问题

- 微信内直接打开托管链接会被平台网关按 UA 拦截，需在外部浏览器打开
````

### 文件 11：`CONTRIBUTING.md`

````markdown
# 贡献指南

感谢关注本项目。这是一个面向微信公众号运营数据的小型可视化看板，欢迎提交 Issue 或 Pull Request。

## 提交 Issue

请说明：

- 运行环境（操作系统、Python 版本、浏览器）
- 部署方式（本地部署 / WorkBuddy 部署）
- 复现步骤与预期结果
- 相关日志（本地版可提供 `sync.log`，注意先去除敏感信息）

## 提交 Pull Request

1. Fork 本仓库并新建分支：`git checkout -b feature/your-feature`
2. 保持改动聚焦，一个 PR 只解决一个问题
3. 前端改动请在 Chrome / Edge / 移动端分别自测，确认无控制台报错、无横向溢出
4. 后端改动请用 Python 3.8+ 验证（项目仅使用标准库，**不要引入第三方依赖**）
5. 提交前确认未包含任何真实凭据：`appid`、`secret`、`access_token`、`data.js`、`store.json`

提交信息建议使用如下前缀：`feat:` 新功能、`fix:` 修复、`docs:` 文档、`style:` 样式、`refactor:` 重构。

## 代码约定

- 前端为单文件 `index.html`，不引入构建工具；新增逻辑请沿用现有的 IIFE + 原生 DOM 写法
- Python 代码遵循 PEP 8，注释使用中文
- 所有新增的写操作接口必须校验服务端令牌（`token`），不得以明文密码作为鉴权手段

## 安全问题

如发现安全漏洞（如鉴权绕过、越权写入），请勿直接提交公开 Issue，改为通过邮件私下告知维护者。
````

### 文件 12：`README.md`（仓库主技术文档）

````markdown
# 微信公众号数据看板

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![Dependencies](https://img.shields.io/badge/backend%20deps-zero-brightgreen)](https://docs.python.org/3/library/)
[![ECharts](https://img.shields.io/badge/ECharts-5-red)](https://echarts.apache.org/)
[![Version](https://img.shields.io/badge/version-1.0.0-informational)](CHANGELOG.md)

面向党政机关、企事业单位的微信公众号运营数据可视化看板：从微信官方接口采集文章阅读、分享、粉丝等数据，在浏览器中以图表呈现，支持多端同步、回收站管理、布局自定义与服务端鉴权。

纯标准库实现，**零第三方后端依赖**；前端仅依赖 ECharts 与 SheetJS 两个本地化 JS 库（已内置，无需 CDN，可在内网环境运行）。

> **两个部署版本**
> - [`local-deployment/`](local-deployment/) —— 在单位内网电脑上运行，自动采集公众号数据，并同步到线上
> - [`workbuddy-deployment/`](workbuddy-deployment/) —— 部署到 [WorkBuddy](https://www.workbuddy.cn) 云端，获得固定公网链接，各端直接访问

## 功能特性

- **关键指标**：累计粉丝、最新日净增、近 7 天阅读/分享人数、文章总数，以及可切换口径的**周发文数 / 周阅读数**
  - `近7天`：含最新数据日在内的过去 7 天（滚动窗口）
  - `本周`：最新数据日所在自然周（周一至周日）
- **阅读与分享趋势**：折线图，底部比例尺 + 滚轮/双指缩放，附「全部 / 近90天 / 近30天 / 近7天」快捷按钮
- **阅读来源、粉丝趋势、文章排行**：饼图、面积图、条形图
- **文章明细**：搜索、日期筛选、排序、分页、导出 Excel（支持按最近一天/七天/一个月/自定义时间导出）
- **回收站**：删除 / 还原 / 彻底删除，**云端全设备同步**
- **布局自定义**：模块可拖拽调整位置与大小，临时布局（免密，仅本次查看）与默认布局（需密码，云端同步）两级保存
- **数据自动采集**（本地版）：调用微信公众平台官方接口，按日采集文章与粉丝数据
- **本地 → 线上自动同步**（本地版）：采集完成后自动把数据与回收站状态推送到线上站点
- **服务端密码验证**：管理密码只保存在服务端，页面源码中不含密码；验证通过后下发 8 小时有效的临时令牌，所有写操作凭令牌放行
- **发布日期自动校正**：以文章日报中的最早出现日期为权威口径，防止历史补扫导致的日期错标
- **响应式**：桌面 / 平板 / 手机自适应，无横向溢出

## 目录

- [技术栈](#技术栈)
- [目录结构](#目录结构)
- [快速开始](#快速开始)
- [获取公众号凭据（AppID / AppSecret）](#获取公众号凭据appid--appsecret)
- [数据流](#数据流)
- [数据结构](#数据结构datajs)
- [API 接口](#api-接口)
- [安全机制](#安全机制)
- [已知限制](#已知限制)
- [常见问题](#常见问题)
- [参与贡献](#参与贡献)
- [开源协议](#开源协议)

## 技术栈

| 层 | 技术 | 说明 |
|---|---|---|
| 前端 | 原生 HTML / CSS / JavaScript | 单文件 `index.html`（约 130KB），无构建步骤 |
| 图表 | ECharts 5（本地化） | Canvas 渲染，动画关闭以保证流畅 |
| 导出 | SheetJS（本地化） | 前端直接生成 .xlsx |
| 本地服务 | Python 3 标准库 | `serve.py`（HTTP 服务）+ `sync.py`（数据采集） |
| 线上服务 | Python 3 标准库 | `server.py`（静态资源 + 同步 API） |

## 目录结构

```
gzh-data-dashboard/
├── README.md                     # 本文档
├── LICENSE                       # MIT 开源协议
├── local-deployment/             # 版本一：本地电脑部署
│   ├── index.html                # 看板页面
│   ├── serve.py                  # 本地 HTTP 服务（含上行同步）
│   ├── sync.py                   # 微信官方接口采集脚本
│   ├── config.example.json       # 配置模板（复制为 config.json 后填写）
│   ├── run.sh / stop.sh          # 一键启动 / 停止
│   ├── README.md                 # 本地部署详细说明
│   └── vendor/                   # 本地化的前端库
│       ├── echarts.min.js
│       └── xlsx.full.min.js
└── workbuddy-deployment/         # 版本二：WorkBuddy 云端部署
    ├── index.html                # 看板页面（与本地版同一份）
    ├── server.py                 # 云端服务（静态资源 + 同步 API + 令牌鉴权）
    └── vendor/
```

## 快速开始

### 版本一：本地电脑部署

适用于：在单位内网电脑上运行，通过浏览器访问，数据保存在本机，可自动同步到线上站点。

> 开始前需要先取得公众号的 AppID 与 AppSecret，并配置 IP 白名单（约 5 分钟，需管理员扫码）。完整步骤见 [获取公众号凭据](#获取公众号凭据appid--appsecret)。

```bash
cd local-deployment
cp config.example.json config.json
# 编辑 config.json，填入公众号的 appid 与 secret
./run.sh            # 启动，默认 http://127.0.0.1:8000
```

启动后：

1. 打开看板页面，点击顶栏「锁定」按钮，输入管理密码解锁（默认占位密码 `change-me`，**部署前务必修改**，见下方「安全机制」）
2. 点击「数据更新」触发采集（子进程运行 `sync.py`，单航班防重入）
3. 采集完成后数据自动写入本机 `data.js`；若配置了 `upstream`，会自动推送到线上站点

### 版本二：WorkBuddy 云端部署

适用于：通过 WorkBuddy（或其他支持 Python 服务的平台）发布为固定链接，所有设备直接访问，数据与状态云端共享。

```bash
cd workbuddy-deployment
python3 server.py          # 本地预览（默认端口 3000，可用 PORT 环境变量覆盖）
```

在 WorkBuddy 中发布时，将 `workbuddy-deployment` 目录作为站点目录，启动命令设为 `python3 server.py`。发布后链接固定，重复部署（更新代码）链接不变。

首次部署后需要导入一份 `data.js`（由本地版采集生成，或在页面上传），数据格式见下文「数据结构」。

## 获取公众号凭据（AppID / AppSecret）

本地版的自动采集需要调用微信公众平台官方接口，因此必须准备两个凭据：**AppID**（账号标识）与 **AppSecret**（接口密钥）。二者的关系可以理解成「账号 + 密码」——AppID 是公开的，AppSecret 等同于密码，**一旦泄露，他人就能以你的公众号身份调用接口**。

只有本地部署版需要这两个凭据；WorkBuddy 部署版只做展示，不需要任何微信凭据。

### 前置条件

| 条件 | 要求 |
|---|---|
| 公众号类型 | **已认证的订阅号或服务号** |
| 操作人员 | 需要**管理员**（或具有开发权限的运营者）扫码确认 |
| 接口权限 | 需具备「数据统计」类接口权限（认证后默认开通） |

> 未认证的个人号无法调用数据接口，会返回 `48001`（无接口权限）。如果你不确定，可在公众平台「设置与开发 → 接口权限」中查看「数据统计」一栏是否为已获得。

### 第一步：获取 AppID 与 AppSecret

1. 用管理员账号登录 [微信公众平台](https://mp.weixin.qq.com)
2. 进入 **设置与开发 → 基本配置**（旧版菜单为「开发 → 基本配置」）
3. 在页面右侧找到 **公众号开发信息** 区块：
   - **开发者ID（AppID）**：直接可见，形如 `wx` 开头、后跟 16 位小写字母与数字（共 18 位）
   - **开发者密码（AppSecret）**：点击 **重置** 或 **查看**
4. 此时会弹出二维码，**必须用管理员微信扫码**
5. 扫码后页面显示完整的 AppSecret（32 位字符串），**请立即复制保存**

> **重要**：AppSecret 只在这一个页面显示一次。关闭弹窗后无法再次查看，只能重新重置。如果忘了，重复上述步骤重置即可，但重置后**旧的 AppSecret 会立即失效**，正在运行的采集服务需要同步更新配置。

### 第二步：配置 IP 白名单（最容易踩的坑）

微信出于安全考虑，只允许白名单内的服务器调用接口。**这是新手最常遇到的失败原因**——凭据填对了，但 IP 没加，接口会返回：

```json
{"errcode": 40164, "errmsg": "invalid ip 1.2.3.4, not in whitelist"}
```

配置方法：

1. 仍在 **设置与开发 → 基本配置** 页面，找到 **IP白名单** 区块，点击 **查看 / 修改**
2. 用管理员微信扫码
3. 填入运行采集脚本的那台电脑的**公网出口 IP**

查询本机公网 IP 的方式：

- 浏览器打开 [https://ip.cn](https://ip.cn) 或 [https://myip.ipip.net](https://myip.ipip.net)
- 命令行执行：`curl https://api.ipify.org`
- 或者不必手动查——直接运行采集，脚本报错时会把当前出口 IP 打印出来，照提示填即可

填写格式：

| 写法 | 是否支持 | 示例 |
|---|---|---|
| 单个 IP | ✅ | `1.2.3.4` |
| CIDR 网段 | ✅ | `1.2.3.4/24` |
| 通配符 | ❌ | `1.2.3.*` 不支持 |

> 家庭宽带的公网 IP 会不定期变化。IP 变了之后采集会失败并提示新 IP，把它补进白名单即可。单位固定专线 IP 一般不变。

### 第三步：填入配置文件

```bash
cd local-deployment
cp config.example.json config.json
```

编辑 `config.json`：

```json
{
  "appid": "wx你的18位AppID",
  "secret": "你的32位AppSecret",
  "lookback_days": 7,
  "article_detail_days": 30,
  "upstream": "",
  "upstream_pwd": "你的线上管理密码",
  "upstream_auto_push": true
}
```

保存后启动服务，页面点「数据更新」即可开始采集。

### 凭据保管的安全要求

- `config.json` 已被 `.gitignore` 排除，**永远不会进入版本库**，切勿手动提交或截图外发
- 运行目录下的 `token.json` 是接口访问令牌的缓存，同样不要外发
- 如果需要交接给同事，建议由对方用自己的管理员账号重新生成 AppSecret，而不是传递同一个密钥
- 怀疑泄露时，第一时间去公众平台**重置 AppSecret**

## 数据流

```
微信公众平台 API
      │  sync.py 按日采集（文章明细 / 来源 / 粉丝）
      ▼
本地 store.json ──生成──> data.js（页面数据源）
      │                      │
      │                serve.py 上行同步（自动）
      ▼                      ▼
  本机浏览器  <─────>  线上 server.py  <─────>  各设备浏览器
                          │
                    state.json（回收站 / 重命名 / 布局，全设备同步）
```

## 数据结构（data.js）

```js
window.DATA = {
  generated_at: "2026-10-19 15:42:24",   // 生成时间
  articles:      { "文章ID": { title, content_url, first_date } },
  article_total: { "文章ID": { read_user, share_user, ... } },   // 累计指标
  article_daily: { "日期": { "文章ID": { read_user, read_source, share_user, ... } } },
  biz_daily:     { "日期": { int_page_read_user, ... } },        // 公众号日报
  user_daily:    { "日期": { cancel, cumulate_user, new_user } },// 粉丝日报
  manual_titles: { "文章ID": "手动命名" },
  deleted_articles: [ ... ]                                       // 回收站快照
};
```

## API 接口

### 公共（读）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/state` | 共享状态：`{rev, deleted, titles, layout}` |
| GET | `/api/data` | 数据元信息（生成时间 / 大小 / 更新时间），供各端检测新数据 |
| GET | `/api/data/full` | 完整数据回拉 |

### 鉴权（写）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth` | `{pwd}` → `{ok, token, expires_in}`；密码错误返回 403 |
| POST | `/api/state` | `{token, rev, deleted, titles, layout}` 写共享状态 |
| POST | `/api/data` | `{token, data, generated_at}` 上传新数据 |

所有写接口只认 `token`；直接传明文密码一律 403。令牌有效期 8 小时，过期后页面自动提示重新解锁。

### 本地版额外接口

`/api/sync`（触发采集）、`/api/delete`、`/api/restore`、`/api/purge`、`/api/rename`、`/api/push`（手动同步线上）、`/api/upstream`（同步配置）、`/api/status`（采集任务状态）——均需令牌。

## 安全机制

1. **密码只在服务端**：页面源码不含密码；解锁时密码通过 `POST /api/auth` 发到服务端比对（比对失败有 0.4s 延迟，增加暴力尝试成本），成功后仅在 `sessionStorage` 保存临时令牌，关闭页面即失效
2. **令牌鉴权**：所有写操作凭令牌放行，令牌 8 小时过期；服务端密码通过环境变量 `ADMIN_PWD` 设置（两版通用）。仓库内默认值 `change-me` 仅为占位符，**部署前必须修改**，否则任何人都可解锁
3. **密钥隔离**：公众号 AppID / AppSecret 只保存在本机 `config.json`（已列入 `.gitignore`），服务不对外提供该文件；`token.json`（access_token 缓存）同样不对外
4. **上传校验**：数据上传时服务端对数据结构做完整性校验，非法数据拒绝写入
5. **门禁可见性**：未解锁时回收站、删除、重命名、布局保存等管理入口在 DOM 层面隐藏（`body:not(.admin)` 规则），而非仅视觉隐藏

## 已知限制

- 微信内打开 WorkBuddy 托管链接会被平台网关按 UA 拦截（返回 403 引导页），与本项目代码无关；请复制链接到浏览器打开，或使用版本一本地部署
- `article_daily` 等接口受微信官方回溯窗口限制（如 `datacube` 类接口一般只能取最近 7 天），采集脚本按日增量入库以累积历史
- 布局的「仅本次查看」保存于 `sessionStorage`，「保存为默认布局」写入云端 `state.json`

## 常见问题

**Q：如何更换管理密码？**
线上版：设置环境变量 `ADMIN_PWD` 后重启服务。本地版：修改 `serve.py` 中 `ACCESS_PWD`；同时修改 `config.json` 的 `upstream_pwd`（用于本地推送线上时的鉴权）。

**Q：采集时报 `40164 invalid ip not in whitelist` 怎么办？**
说明运行采集的电脑公网 IP 没加进白名单。按提示中的 IP 去「设置与开发 → 基本配置 → IP白名单」添加即可，详见[第二步](#第二步配置-ip-白名单最容易踩的坑)。

**Q：报 `48001`（无接口权限）怎么办？**
通常是公众号未认证，或没有「数据统计」接口权限。确认已做微信认证，并在「设置与开发 → 接口权限」中确认已获得数据统计类接口。

**Q：AppSecret 忘了 / 泄露了怎么办？**
在「设置与开发 → 基本配置」点击 AppSecret 的「重置」，管理员扫码后生成新的。注意重置后旧密钥立即失效，需同步更新 `config.json`。

**Q：更换了 AppSecret 后采集失败？**
检查 `config.json` 里的 `secret` 是否已更新为新值；同时确认 `token.json` 中缓存的旧访问令牌已失效——直接删除 `token.json` 后重新采集即可。

**Q：其他电脑怎么访问本地版？**
同一局域网内打开 `http://<本机IP>:8000`（`run.sh` 启动时会打印局域网地址）。

**Q：线上数据如何更新？**
两种方式：① 本地版采集后自动推送（推荐）；② 任意设备解锁后在页面「数据更新」上传 data.js 文件。

**Q：文章发布日期的统计口径？**
以文章在日报数据中**最早出现的日期**为准（页面加载时自动校正，采集端入库时按最小日期回填），可避免历史补扫造成的错标。

## 参与贡献

欢迎提交 Issue 与 Pull Request。请先阅读 [贡献指南](CONTRIBUTING.md) 与 [变更日志](CHANGELOG.md)。

提交前请确认代码中不含任何真实凭据（`appid`、`secret`、`access_token`）与业务数据。

## 开源协议

[MIT](LICENSE)。可自由使用、修改、分发与商用，需保留版权声明。

本项目内置的第三方库版权归原作者所有：

| 库 | 协议 |
|---|---|
| [Apache ECharts](https://echarts.apache.org/) | Apache-2.0 |
| [SheetJS Community Edition](https://sheetjs.com/) | Apache-2.0 |

## 致谢

本项目在开发过程中使用了 WorkBuddy（https://www.workbuddy.cn ）作为云端部署与测试平台。
````

### 文件 13：`local-deployment/README.md`（本地部署详细说明）

````markdown
# 微信公众号数据看板（本地版 · 方案A · v7）

纯本地运行，**不需要数据库、不需要云服务器、不花钱**。
一个 Python 脚本负责采集，结果存成 JSON 文件；一个本地服务 + 一个 HTML 页面负责显示与操作。
适用于 **统信 UOS（Debian 系 Linux）**，仅用 Python 标准库，无需 pip 安装任何依赖。

> 本版已切换到微信「**发表内容新接口**」。老的群发数据接口（getarticlesummary 等）已下线（错误码 47009）。
> 新口径（**2025-11-01 起**）只统计「**阅读人数**」，**不再提供「阅读次数 PV」**；分享同样只保留「分享人数」。

---

## 一、你需要准备的信息

### 1.1 获取 AppID 与 AppSecret

登录 [微信公众平台](https://mp.weixin.qq.com) → **设置与开发 → 基本配置** → 右侧「公众号开发信息」：

| 信息 | 获取方式 | 说明 |
|---|---|---|
| **AppID**（开发者ID） | 页面直接可见 | 以 `wx` 开头、后跟 16 位字母数字（共 18 位） |
| **AppSecret**（开发者密码） | 点「**重置**」或「查看」，**需管理员微信扫码** | 32 位字符串，等同于接口密码 |

**两个关键点：**

1. **AppSecret 只显示一次**。弹出的二维码扫码后才会显示完整密钥，关闭弹窗后无法再次查看，请当场复制保存。
2. **重置会使旧密钥立即失效**。忘了只能重置，但重置后正在运行的服务需同步更新 `config.json`，否则采集会失败。

> AppSecret 等同于账号密码：**不要发给任何人、不要贴在聊天里、不要提交到代码仓库**。怀疑泄露时立即去平台重置。

### 1.2 其他需要确认的信息

| 信息 | 获取方式 | 说明 |
|---|---|---|
| **本机出口公网 IP** | 命令行 `curl https://api.ipify.org`，或浏览器打开 `https://ip.cn` | 必须加进 IP 白名单，否则报 `40164` |
| 公众号状态 | —— | 必须是 **已认证的订阅号 / 服务号**，未认证或个人号调不了数据接口（`48001`） |
| 接口权限 | 设置与开发 → 接口权限 | 需已获得「**数据统计**」类接口权限（认证后默认开通） |

> 详细的图文步骤（含 IP 白名单配置与常见报错处理）见主文档的
> [获取公众号凭据](https://github.com/Yellowsoy/gzh-data-dashboard#获取公众号凭据appid--appsecret) 一章。

> 建议把整个 `wechat-dashboard` 文件夹放在固定位置，例如 `~/wechat-dashboard`，以后不要移动。

---

## 二、安装与配置（统信 UOS）

### 1. 确认 Python3

UOS 一般自带。在文件夹空白处 **右键 → 在终端打开**，执行：

```bash
python3 --version
```

能看到 `Python 3.7` 及以上即可（本版已在 Python 3.7 上验证）。若提示没有：

```bash
sudo apt update
sudo apt install python3
```

### 2. 填写 AppID / AppSecret

用文本编辑器打开 `config.json`，改成你自己的：

```json
{
  "appid": "wx你的真实AppID",
  "secret": "你的真实AppSecret",
  "lookback_days": 7,
  "article_detail_days": 30
}
```

- `lookback_days`：按「统计日」每日接口往回补几天，日常保持 `7`。
- `article_detail_days`：按「发表日」取多少天内发表的文章（含标题/链接、累计轨迹），默认 `30`。

### 3. 配置 IP 白名单（关键）

登录 [微信公众平台](https://mp.weixin.qq.com) → **设置与开发 → 基本配置 → IP 白名单**，
把本机出口公网 IP 加进去：

- 支持单个 IP：`1.2.3.4`
- 支持 IP 段：`1.2.3.4/24`（**不支持** `1.2.3.*` 写法）
- 家庭宽带 IP 偶尔会变，变了脚本会再次报错并提示新 IP，照提示更新即可。

---

## 三、启动看板（推荐方式）

在文件夹里打开终端，执行：

```bash
chmod +x run.sh
./run.sh
```

看到 `看板已启动：http://127.0.0.1:8000/` 后，浏览器会**自动打开**看板页面。

- 指定端口：`./run.sh --port 8080`
- 不自动打开浏览器：`./run.sh --no-browser`
- 看板只监听本机（127.0.0.1），**不会暴露给局域网**，AppSecret 等文件也不对外提供。
- 停止服务：在看板终端按 `Ctrl+C`，或执行 `./stop.sh`。
- 端口 8000 被占用时会自动顺延到 8001–8020（终端会打印实际地址）。

### 页面上的按钮

| 按钮 | 作用 |
|---|---|
| **更新数据** | 手动触发一次采集（快捷键 `U`）。任务在服务端子进程执行，弹窗显示实时日志，**完成后页面自动刷新**。更新中再次点击会提示“已在进行中”（单航班，不会并发重入） |
| **回收站** | 查看/恢复被删除的文章，或彻底删除（快捷键 `B`）；第二个页签是「命名管理」（手动命名的备份/导入/修改） |
| **布局编辑** | 进入/退出板块布局编辑（快捷键 `L`）：拖动 ⠿ 手柄换位，拖右下角调宽高；编辑时旁边出现「重置布局」 |
| **主题切换** | 明暗主题切换（快捷键 `T`），记住你的选择 |

> 也可以只跑一次采集不启动服务：`./run.sh --sync-only --days 30`（等价于旧版行为）。

### 文件直开模式（兼容旧习惯）

仍然可以**双击 `index.html`** 直接打开看板（数据来自 `data.js`）。此模式下：

- 「更新数据」按钮不可用（会提示先运行 `run.sh`）；
- 删除 / 命名保存在**当前浏览器**（localStorage），换浏览器或清理数据前请先在「回收站 → 命名管理」里备份；
- 推荐用上面的服务模式，删除和命名存到 `store.json`，全设备生效、不会因清理浏览器丢失。

---

## 四、首次采集

启动看板后，点页面右上角 **更新数据**（或在终端执行 `./run.sh --sync-only --days 30`）：

- 首次建议 `--days 30`，拉取近 30 天的新格式数据（阅读、分享、来源、点赞、在看、留言、收藏）。
- 首次运行还会自动补扫全部历史发表日期，并通过「已发布消息列表」补全文章标题——**包括 2025-11-01 之前发表的老文章**，所以明细表里不会再有文章 ID 或"未获取到标题"。
- 首次补扫约需几百次调用，可能触及微信的**日调用量上限（45009）**：脚本会自动暂停、明天继续，标题当天就能补齐，无需处理；想当天跑完可在后台「接口权限模板」清零后重跑。
- 若报 IP 白名单错误，按屏幕提示把 IP 加好后再跑一次。

---

## 五、设置每天自动同步（crontab）

### 1. 确认定时服务已开启

```bash
sudo systemctl enable --now cron
```

### 2. 编辑定时任务

```bash
crontab -e
```

在末尾加一行（把路径改成你的实际路径，可用 `pwd` 查看）：

```cron
30 8 * * * cd /home/你的用户名/wechat-dashboard && python3 sync.py >> sync.log 2>&1
```

含义：**每天早上 8:30** 自动采集前一天数据（微信要求 8 点后数据才统计完）。
保存后每天到点自动更新，看板页面刷新即可（服务模式下手动点「更新数据」也可以随时补一次）。

> 定时任务要求电脑在 8:30 处于开机联网状态。
> 当天没开机也不用担心，下次运行会自动补最近约 7 天。

---

## 六、统信 UOS 桌面集成（可选）

安装桌面启动器后，可以像普通软件一样从开始菜单启动，不必每次开终端。

### 1. 安装启动器

```bash
chmod +x install.sh stop.sh
./install.sh
```

安装后：

- 开始菜单 / 桌面搜索「**公众号数据看板**」即可启动（绿色图表图标）；
- 在启动器图标上 **右键 → 「停止看板服务」**（等价于 `./stop.sh`）；
- 服务只监听 `127.0.0.1`，**不需要配置防火墙**，也不会被局域网访问。

### 2. 设置开机自启（可选）

```bash
./install.sh --autostart
```

登录桌面后自动后台启动看板，浏览器不会自动弹出（需要时从开始菜单打开）。

### 3. 定时采集（一条命令代替手改 crontab）

```bash
./install.sh --cron
```

自动添加「每天 08:30 采集一次」的 crontab 任务（等价于第五节的第 2 步，二选一即可，不要重复添加）。
若系统没有 crontab，会提示安装：`sudo apt install cron`。

### 4. 卸载集成

```bash
./install.sh --uninstall
```

移除启动器、开机自启和定时任务；**不会删除**看板文件与数据。

### 5. 本地部署增强（自动生效，无需配置）

| 能力 | 说明 |
|---|---|
| **端口自适应** | 默认 8000 被占用时自动改用 8001–8020，终端会打印实际地址；也可在 `config.json` 加 `"serve_port": 8080` 固定端口 |
| **防重复启动** | 已有实例运行时，再次双击启动器只会打开浏览器，不会起第二个服务 |
| **优雅停止** | `./stop.sh` 先发 SIGTERM（保存状态、清理 PID 文件），3 秒不退再 SIGKILL；运行时目录下生成 `serve.pid`，退出自动删除 |
| **日志滚动** | `sync.log` 超过 2MB 自动截断到最近 3000 行，长期挂机不占满磁盘 |
| **旧版浏览器兼容** | 页面为不支持 `color-mix()` 的旧版 Chromium（UOS 自带浏览器等）提供了等效回退样式 |
| **断连静默** | 页面刷新/关闭导致的连接中断不再往终端刷错误栈 |

---

## 七、看板功能

### 1. 文章删除与回收站

- 每行右侧有删除图标；也可勾选行首复选框**批量选择**，底部弹出操作条「删除」；
- 删除前有确认框；删除后文章进入**回收站**（顶栏按钮，带数量徽标）；
- 回收站中可 **恢复**（瞬间还原，包括历史数据）或 **彻底删除**（物理移除，不可恢复）；
- 删除是“软删除”：只影响看板展示，**重新采集不会让已删除的文章复活**；
- 服务端模式下删除记录存 `store.json`；文件直开模式下存当前浏览器。

### 2. 手动命名（老文章补标题）

2025-11-01 之前发表的老文章，微信接口不提供标题。正常情况下脚本会通过发布列表自动补全；
万一补不上（接口无权限/文章不在发布列表），在明细表标题旁点 **命名** 手动补上：

- 服务端模式：命名存入 `store.json`，刷新/重启不丢失；
- 文件直开模式：命名存入浏览器 localStorage；
- 「回收站 → 命名管理」页签可查看、修改、删除全部命名，并 **备份/导入 JSON**。

### 3. 图表与明细

- **KPI**：累计粉丝、最新日净增、近7天阅读人数、近7天分享人数、文章总数；
- **图表**：每日阅读与分享趋势、阅读来源渠道（环形）、粉丝增长（累计/净增）、文章阅读 Top 10（默认全宽）；
- **明细表**：日期范围筛选、**模糊搜索**（快捷键 `/`）、点击表头排序（标题/发表日期/阅读/分享/收藏/点赞/在看/留言）、分页（每页 50 篇）；
- **导出 Excel**：可选「全部」或「近30天」，列包含发表日期、标题、各指标、文章ID、链接；
- **响应式**：窄屏（手机）下 KPI 变两列、图表与表格自动堆叠，表格转为卡片式布局。

### 4. 模糊搜索

搜索框支持三种写法，可组合：

| 写法 | 示例 | 效果 |
|---|---|---|
| 普通关键词 | `食堂` | 标题包含「食堂」的文章 |
| 多关键词（空格分隔，且关系） | `单位食堂 改造` | 标题同时包含「单位食堂」和「改造」 |
| 拼音首字母 | `dwst` | 匹配「**单**位**食**堂**改**造」类标题；`xc` 匹配「**西**充政协…」 |

- 大小写不敏感（`DWST` 与 `dwst` 等效）；
- 拼音库内置约 6700 个常用汉字的声母，纯本地计算，无网络请求；
- 搜索结果实时过滤表格与分页，不影响导出（导出仍按当前筛选结果）。

### 5. 布局自由编辑

看板由 6 个板块组成（关键指标 / 每日趋势 / 阅读来源 / 粉丝增长 / 文章阅读 Top10 / 文章明细），
位置和大小都可以自己调：

- 点顶栏的 **布局按钮**（或按快捷键 `L`）进入编辑模式；
- **拖动板块左上角的 ⠿ 手柄** 换位置：拖到目标板块左半边插到它前面，右半边插到它后面；
- **拖动板块右下角的手柄** 同时调整宽度（2~6 列）和高度；
- 「关键指标」与「文章明细」两个板块锁定全宽，其余四个可自由摆放；
- 调整自动保存在本浏览器（localStorage），刷新、重启不丢失；
- 点 **「重置布局」** 恢复默认排列；按 `Esc` 退出编辑模式；
- 窄屏（手机）下不提供布局编辑，请在电脑上调整。

---

### 6. 数据同步到线上（本地 → 线上，2026-10-19 新增）

本机采集或上传的数据，会自动同步到线上站点，所有打开线上链接的设备都会看到新数据，不必再单独到线上上传一次。

**什么时候自动同步**

- 点「更新数据 → 从微信服务器采集」，采集成功后自动同步一次；
- 点「更新数据 → 选择数据文件」，上传成功后自动同步一次；
- 删除 / 恢复 / 彻底删除 / 重命名文章后，自动同步回收站与命名状态（后台进行，不影响操作）；
- 顶栏「数据更新时间」后面有「线上：已同步 / 未同步」标记，点击可**立即手动同步一次**（需密码）。

**同步的内容**

1. 数据本体（阅读、分享、粉丝等，写入线上 data.js）；
2. 回收站与手动命名状态（写入线上共享状态）。

**相关配置（config.json）**

```json
"upstream": "https://your-dashboard.example.com",
"upstream_pwd": "你的线上管理密码",
"upstream_auto_push": true
```

- `upstream`：线上站点地址，换地址时改这里；
- `upstream_pwd`：线上写接口密码（与页面操作密码一致）；
- `upstream_auto_push`：设为 `false` 可关闭自动同步（数据只在本机生效，需要时再手动点顶栏同步）。

**同步失败怎么办**

- 顶栏显示「线上：未同步」，鼠标悬停可看到失败原因；
- 常见原因是本机暂时无法访问外网（内网限制）。数据已保存在本机，恢复网络后点顶栏标记即可重试；
- 每次同步结果都写入 `sync.log`，可用「上行同步」关键字检索。

**线上那一侧的表现**

- 其他人正在浏览线上页面时，检测到数据更新会在底部弹出「数据已更新 → 立即刷新 / 稍后」，由查看者自行决定何时刷新，不会打断正在进行的查看与操作。

### 7. 锁定与解锁（管理员模式，2026-10-19 新增）

顶栏最右侧的「锁定」按钮是**唯一的管理员入口**。页面默认是**锁定（只读）**状态，任何人打开链接看到的都是这个状态。

| 状态 | 按钮显示 | 可用功能 |
|---|---|---|
| **锁定**（默认） | 锁定 | 查看图表与明细；**点「更新数据」立即获取服务端最新版本**；临时调整布局（仅本次查看有效）；导出（仍需输密码） |
| **已解锁**（点「锁定」输入管理员密码后） | 已解锁（高亮） | 上传新数据、回收站、删除 / 还原 / 彻底删除、重命名、保存为默认布局、恢复默认布局、同步到线上 |

要点：

- **「更新数据」默认是取最新**：不输密码直接点，页面会去服务端比对数据时间——有更新就提示「发现新数据，正在更新…」并自动刷新；没有就提示「已是最新数据（时间）」，全程不弹密码框；
- **解锁后「更新数据」才是上传**：面板里提供「选择数据文件」（上传新数据）与「获取最新版本」；本机服务模式下还会多出「从微信服务器采集」；
- **未解锁看不到管理入口**：回收站按钮、表格的勾选列与「操作」列（重命名 / 删除）、「删除所选」、「保存 / 恢复默认布局」全部隐藏，界面保持干净；
- **再点一次即锁定**：按钮显示「已解锁」时点击立即锁定，管理入口重新隐藏；
- 解锁状态只存在当前浏览器标签页（关闭页面即失效），不会留在别人电脑上；
- 密码错误会给出提示且不会解锁，可反复重试。
- **密码只保存在服务端**：解锁时密码发到服务端比对，成功后下发临时令牌（8 小时有效）；页面源码中不含密码，令牌过期后写操作会被拒绝并提示重新解锁。本地服务与线上服务均已按此方式验证。

## 八、文件说明

| 文件 | 作用 |
|---|---|
| `sync.py` | 采集脚本（核心，不用手动改） |
| `serve.py` | 本地看板服务（静态页面 + 更新/删除/恢复/命名 API） |
| `config.json` | 配置：AppID / AppSecret / 回溯天数 / 可选 serve_port |
| `run.sh` | 一键启动入口（默认启动看板服务） |
| `stop.sh` | 停止看板服务（PID 精确停止 + 兜底扫描） |
| `install.sh` | 桌面集成安装/卸载（启动器 / 开机自启 / 定时任务） |
| `icon.png` | 启动器图标（256×256） |
| `index.html` | 数据看板页面 |
| `token.json` | 自动生成，缓存 access_token（2 小时有效） |
| `store.json` | 自动生成，累积的全部历史数据 + 删除台账 + 手动命名（即"数据库"） |
| `data.js` | 自动生成，看板读取的数据（已过滤删除、已应用命名） |
| `sync.log` | 运行日志（含服务端手动更新的完整日志，超 2MB 自动滚动） |
| `serve.pid` | 运行时生成，记录服务进程 PID，退出自动删除 |

> 备份时只需保留 `config.json`、`store.json`；其余文件都能重新生成。

---

## 九、常见问题

**Q：双击 index.html 看板是空的？**
A：还没采集数据。先运行 `./run.sh --sync-only`（或启动服务后点「更新数据」），完成后刷新页面。

**Q：点「更新数据」没反应 / 提示文件直开模式？**
A：该按钮需要服务模式。请运行 `./run.sh` 启动看板服务，通过 `http://127.0.0.1:8000/` 访问页面。

**Q：端口被占用？**
A：一般不用管——服务会自动顺延到 8001–8020 并在终端打印实际地址。想固定端口就换一个：`./run.sh --port 8080`，或在 `config.json` 里加 `"serve_port": 8080`。

**Q：报错 `47009 / this api is offline`？**
A：老接口已下线。请使用本版脚本（已切换到新接口），不要改回旧接口名。

**Q：报错 `40164 / not in whitelist`？**
A：IP 白名单问题。脚本会显示本机当前 IP，把它加进公众平台白名单，等 1–2 分钟生效后重跑。

**Q：报错 `40001`？**
A：AppSecret 填错或 token 失效。核对 `config.json`，脚本会自动重取 token。

**Q：为什么没有阅读次数（PV）？**
A：为统一口径，微信自 2025-11-01 起下线了「阅读次数」，只保留「阅读人数」。2025-11-01 之前的历史次数可在公众平台后台「数据统计 → 历史数据入口」查看。

**Q：为什么没有更早的历史数据？**
A：微信不提供批量导出历史，新接口数据从 2025-11-01 起。从现在起每天采集，数据会越攒越多。

**Q：换网络 / 重启后不能用了？**
A：多半是公网 IP 变了。运行一次脚本，按提示把新 IP 加进白名单即可。

**Q：删除的文章会因为重新采集又出现吗？**
A：不会。删除记录在 `store.json` 的 deleted 台账里，生成数据时始终过滤；想找回就去回收站恢复。

**Q：部分文章标题显示成“未获取到标题（2247508546_1）”？**
A：正常情况下不会——脚本会通过「获取已发布的消息列表」（freepublish/batchget）自动补全标题，包括 2025-11-01 之前发表的老文章（该接口是内容接口，不受数据起点限制；其返回的文章链接中含 mid/idx，与 msgid 一一对应）。如果仍看到这个提示，原因和处理：
1. **发布列表接口无权限（48001）**：日志会打印醒目提示。老文章标题无法自动补全，只能在明细表标题旁点「命名」手动补上（服务模式存 store.json，文件直开模式存浏览器，可在「命名管理」里备份/导入）；
2. **该文章确实不在发布列表里**（未发布过/已删除/转载等）：微信不提供其标题，同样用「命名」手动补；
3. **采集当天被限流/网络抖动**：脚本会记入台账，**下次运行自动补扫**，无需手动干预；结束时会打印标题情况汇总。如多次补扫仍不行，可执行一次 `python3 sync.py --full` 强制全量重扫。

**Q：日志提示 getarticletotaldetail 或 freepublish/batchget 返回 48001？**
A：账号没有对应接口权限（需已认证且具备数据接口权限）。阅读/分享/粉丝数据不受影响；标题方面，getarticletotaldetail 无权限时新文章标题也取不到，只能用「命名」手动补。

**Q：日志提示 45009 reach max api daily quota limit?**
A：日调用量上限（按接口分别计算、每天 0 点重置）。脚本会立即暂停该接口的剩余扫描，**不会反复重试**；未完成的日期下次运行自动继续，标题以发布列表补全为准。如需当天跑完：公众平台后台「接口权限模板」可查看实时调用量，认证账号每月有 10 次清零机会。

**Q：板块位置/大小能自己调吗？**
A：可以。点顶栏「布局编辑」按钮（或按 `L`），拖动板块左上角 ⠿ 手柄换位，拖右下角调宽高；调整自动保存在本浏览器，刷新不丢失，「重置布局」恢复默认。窄屏（手机）下不提供编辑。

**Q：搜索能用拼音首字母吗？**
A：可以。搜索框直接输入拼音首字母，如 `dwst` 匹配「单位食堂改造」类标题；多关键词用空格分隔（且关系）。拼音库内置常用汉字声母，纯本地计算。搜索结果同时作用于表格与导出。

**Q：本机上传的数据，线上页面看不到？**
A：先看顶栏「线上：」标记是否为「已同步」。显示「未同步」时把鼠标悬停在标记上可看到失败原因，也可点该标记手动同步一次（需密码）。若本机无法访问外网，数据仍完整保存在本机，联网后重试即可。线上页面需刷新（或点底部提示的「立即刷新」）后才会显示新数据。
````

## 附录 B：两个离线库文件（vendor/）的获取方式

`vendor/echarts.min.js`（图表库）和 `vendor/xlsx.full.min.js`（Excel 导出库）是**第三方压缩代码**，本地版与线上版各存一份（内容完全相同）。

这两个文件合计约 3.7 MB，且是压缩成一行的一体式代码——全文贴入文章既没有可读性、也会让页面体积失控。因此这里给出**可复现的获取方式与校验值**，保证你拿到的是和仓库完全一致的那一份。

**方式一：从仓库直接下载（推荐）**

```bash
mkdir -p vendor
curl -L -o vendor/echarts.min.js   https://github.com/Yellowsoy/gzh-data-dashboard/raw/main/local-deployment/vendor/echarts.min.js
curl -L -o vendor/xlsx.full.min.js https://github.com/Yellowsoy/gzh-data-dashboard/raw/main/local-deployment/vendor/xlsx.full.min.js
```

**Windows PowerShell**

```powershell
New-Item -ItemType Directory -Force vendor | Out-Null
Invoke-WebRequest -OutFile vendor\echarts.min.js   "https://github.com/Yellowsoy/gzh-data-dashboard/raw/main/local-deployment/vendor/echarts.min.js"
Invoke-WebRequest -OutFile vendor\xlsx.full.min.js "https://github.com/Yellowsoy/gzh-data-dashboard/raw/main/local-deployment/vendor/xlsx.full.min.js"
```

**方式二：整包下载**

打开 https://github.com/Yellowsoy/gzh-data-dashboard → **Code → Download ZIP**，解压后 `vendor/` 目录直接可用。

**校验值**（下载后核对，确认与仓库一致）

| 文件 | MD5 | 字节数 |
| --- | --- | --- |
| `echarts.min.js` | `334d8b37c4eed7279d054cf21637bc24` | 1029203 |
| `xlsx.full.min.js` | `31e9848e80e1ddf43c5aa31009cd2b7b` | 881727 |

- Linux / macOS：`md5sum vendor/echarts.min.js`
- Windows PowerShell：`Get-FileHash vendor\echarts.min.js -Algorithm MD5`

**离线环境的替代方案**：若不想带 vendor 目录，也可改用 CDN 引入——把 `index.html` 中引用 `vendor/echarts.min.js` 与 `vendor/xlsx.full.min.js` 的两处替换为：

- ECharts：`https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js`
- SheetJS：`https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js`

（代价是看板必须联网才能打开。）


## 写在最后

这套方案谈不上多高级，但它很"干净"：**不依赖任何云服务器、不花钱、数据全部留在自己机器上**，同时把"自动更新、趋势、榜单、明细、导出、多端同步"这几件事都做到了。

回头看，最值钱的经验不是某段代码，而是这三条：

1. **动手前先把平台边界搞清楚**——哪些是接口能力问题，哪些是网络环境问题。分清楚这个，就不会在"为什么不行"上反复打转。
2. **凡是和身份有关的判断，一律放服务端**。前端做权限校验本质上等于没做。
3. **给每一步定一个可验证的验收点**。本文那张清单不是形式主义：它让你在任何一步卡住时，都能立刻知道是哪一步没过，而不是对着一个不工作的系统发呆。

源码在 [GitHub](https://github.com/Yellowsoy/gzh-data-dashboard)（MIT），有问题欢迎提 Issue。希望这篇能帮你少走几段弯路。
