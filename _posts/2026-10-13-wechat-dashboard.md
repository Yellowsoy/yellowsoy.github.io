---
layout:     post
title:      "公众号数据看板 v2：从单机本地，到多端同步"
subtitle:   " \"依旧零成本、不装数据库，但这次补上了服务端鉴权、云端同步和周统计口径\""
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

> **更新说明**：本文最初发布于 2026-10-13，讲的是「本地 Python + JSON + 单文件 HTML」的零成本方案。
> 实际用了两周之后，这套东西暴露出不少问题——**密码写在前端、回收站不同步、发布日期错标、趋势图拉长了看不清**。
> 于是有了 v2。本文已按 v2 **整体重写**，完整源码开源在 GitHub（见第六节）。

最近在运营公众号，想要一个能**自动更新**的数据看板：顶部一排 KPI、阅读分享趋势、粉丝增长、来源构成、文章榜单和明细表。而我的要求一直很朴素：**不花钱、不买服务器、不装数据库、不依赖云服务**。

v1 做到了这些，但用起来有七处明显的别扭。这篇文章记录 v2 的来龙去脉、**关键改造的代码**、零基础部署教程、踩坑与维护建议。

> 面向 **Windows / macOS / Linux（含统信 UOS 等国产系统）**，代码以 Linux 路径为主，Windows/Mac 的差异在对应位置单独说明。

* * *

## 目录

01. 从 v1 到 v2：到底改了什么
02. 先搞懂几个名词（小白必看）
03. 三条"能白嫖"的路，为什么依然走不通
04. v2 架构：本地采集 + 双版本部署
05. 你需要准备什么，以及如何获取 AppID / AppSecret
06. 五个关键改造（含新版代码）
07. 完整源码获取与目录结构
08. 手把手：两种部署方式
09. 设置每天自动同步
10. 看板功能说明
11. 常见问题与错误码速查
12. 后续维护与备份

* * *

## 一、从 v1 到 v2：到底改了什么

先把 v1 的问题摆出来，这决定了 v2 改什么：

| 问题 | v1 的做法 | v2 的做法 |
| --- | --- | --- |
| **密码裸奔**：密码写在前端 JS 里，F12 一眼看到 | `var PWD = '...'` | 挪到服务端 `POST /api/auth`，校验后下发 8 小时令牌 |
| **回收站不同步**：在这台电脑删的，另一台还在 | 只存本机 `localStorage` | 云端 `/api/state` 全设备同步 |
| **只能单机看**：换台电脑、手机上就打不开 | 手动拷文件夹 | 增加「线上部署版」，本地采集完自动推送 |
| **周统计只有一种口径** | 固定"近 7 天" | 双口径：近 7 天滚动 / 本周（周一至周日） |
| **发布日期错标**：老文章被标成最近的日期 | `setdefault` 写死首次日期 | 按「文章日报」取最早日期重算 |
| **趋势图拉长看不清** | 无缩放 | 底部比例尺拖拽 + 滚轮/双指缩放 |
| **布局固定** | 无 | 模块可拖拽调位置和大小，临时/默认两级 |

一句话总结：**v2 没有放弃"零成本"，只是把"谁来验证身份、数据存在哪"这两件事重新安排了一下。**

## 二、先搞懂几个名词（小白必看）

名词 | 通俗解释
--- | ---
**AppID（开发者ID）** | 公众号的"身份证号"，以 `wx` 开头，公开可查，不保密
**AppSecret（开发者密码）** | 公众号的"密码"，**绝不能泄露**，用来换 access_token
**access_token（访问令牌）** | 拿 AppID + AppSecret 换来的"临时通行证"，**有效期 7200 秒（2 小时）**，所有数据请求都要带它
**IP 白名单** | 微信只允许白名单里的 IP 取数据，必须把自己电脑的**公网 IP** 填进后台
**公网 IP** | 你家宽带在互联网上的"门牌号"，可在 https://ip.cn 查到，脚本首次运行也会提示
**datacube** | 微信官方的一组数据统计接口，取阅读、分享、粉丝等数据
**UV / PV** | UV = 阅读**人数**（同一人看多次算 1）；PV = 阅读**次数**。**微信自 2025-11-01 起只保留 UV，取消了 PV**

还有一个**最重要的前提**：

> 数据统计接口**只对"已认证的订阅号 / 服务号"开放**。未认证的订阅号、个人号调不了这些接口，请先确认你的公众号已通过微信认证。

## 三、三条"能白嫖"的路，为什么依然走不通

v2 依然坚持本地采集，因为这三条路一条都没变通畅：

**1. 纯静态 HTML 直接调微信接口？不行**
- **CORS**：浏览器跨域被拦，微信接口不允许网页直接请求；
- **AppSecret 泄露**：前端写死密码等于公之于众；
- **IP 白名单**：页面跑在每个访客自己的网络里，IP 不固定，没法加白名单。

**2. 用 Cloudflare Workers 免费版？不行**
免费版出口 IP 是动态共享地址，每天在变，无法加白名单；想要固定出口 IP 就得付费。

**3. 用 GitHub Actions 代跑？也不行**
出口 IP 同样是一大段动态地址段，无法固定。

### 那 v2 的"线上版"是怎么回事？

这是 v2 最关键的设计：**线上版不采集，它只负责"展示 + 状态同步"。**

采集仍然只能在你本机跑（因为 IP 白名单和 AppSecret 都在你手上）。本地采完生成 `data.js`，再推到线上覆盖——线上只是一个"能给别人看的、能多端同步回收站和布局的展示层"。

这样既守住了"零成本 + 凭据不外出"，又解决了"换台设备就看不了"的问题。

## 四、v2 架构：本地采集 + 双版本部署

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
                 │ （双击打开）      │            │ （手机/其他电脑也能看）│
                 └─────────────────┘            └──────────┬─────────┘
                                                           │ /api/state
                                                 ┌─────────▼─────────┐
                                                 │ state.json（云端）  │
                                                 │ 回收站/重命名/布局   │
                                                 └───────────────────┘
```

三个数据文件各司其职：

| 文件 | 作用 | 谁能改 |
| --- | --- | --- |
| `store.json` | 本地累积库：文章字典、日报、重命名记录 | 只有 `sync.py` 写 |
| `data.js` | 看板渲染数据（KPI、趋势、文章明细） | 本地生成 / 上传覆盖 |
| `state.json` | 云端共享状态：回收站、标题重命名、布局 | 页面写接口推到线上 |

## 五、你需要准备什么，以及如何获取 AppID / AppSecret

### 1. 需要准备的 4 样东西

信息 | 在哪里获取 | 说明
--- | --- | ---
**AppID** | 微信公众平台 → 设置与开发 → 基本配置 | 以 `wx` 开头
**AppSecret** | 同上 → 开发者密码（点「重置」可见，需管理员扫码） | 只填到本机，**不要发给任何人**
**本机公网 IP** | https://ip.cn 查看，或首次运行脚本时的提示 | 要加进白名单
**公众号状态** | —— | 必须是**已认证**的订阅号 / 服务号

### 2. 手把手获取 AppID / AppSecret（跟着点）

1. 电脑浏览器打开微信公众平台：**https://mp.weixin.qq.com**
2. 用**绑定该公众号的管理员微信**扫码登录
3. 左侧菜单栏拉到最下方，**「设置与开发」→「基本配置」**
4. 页面中部「开发者ID」区域：
   - **AppID**：直接显示，`wx` 开头，点「复制」
   - **AppSecret**：默认隐藏，点右边「重置」（有的显示「生成」）→ 管理员微信扫码确认 → **立刻复制保存**（只完整显示这一次）
5. 同页面往下找「IP白名单」，点「修改」：
   - 填你的公网 IP；支持单个 `1.2.3.4`，也支持 `1.2.3.4/24` 这种网段
   - **不支持** `1.2.3.*` 写法
   - 家庭宽带 IP 偶尔会变，变了脚本会提示，照提示更新即可

> 安全提醒：AppSecret 只填在你自己电脑的 `config.json` 里，**不要发到群里、不要写进任何网页、不要提交到 Git**。

### 3. 确认电脑装了 Python

- **Linux / 统信 UOS**：一般自带，`python3 --version` 看到 3.7+ 即可
- **Windows**：https://www.python.org/downloads/ 下载，**安装时勾选 "Add Python to PATH"**
- **macOS**：终端执行 `python3 --version`，没有就按提示装

## 六、五个关键改造（含新版代码）

下面这五处是 v2 相对 v1 的实质改动，代码都取自开源仓库。

### 改造 1：密码挪到服务端，前端不留任何密码

v1 最严重的问题。v2 的做法是：`POST /api/auth` 由服务端比对密码，通过则下发一个**随机令牌**，之后所有写操作只认令牌。

```python
# server.py —— 线上服务
import secrets, threading, time

# 管理密码只保存在服务端；部署前必须通过环境变量 ADMIN_PWD 修改
PWD = os.environ.get("ADMIN_PWD", "change-me")
TOKEN_TTL = 8 * 60 * 60      # 解锁令牌有效期：8 小时
_TOKENS = {}                 # token -> 过期时间戳
_TOKEN_LOCK = threading.Lock()

def issue_token():
    with _TOKEN_LOCK:
        now = time.time()
        for k in [k for k, exp in _TOKENS.items() if exp < now]:
            _TOKENS.pop(k, None)          # 顺手清理过期令牌
        tok = secrets.token_hex(24)       # 48 位十六进制随机数
        _TOKENS[tok] = now + TOKEN_TTL
        return tok

def check_token(tok):
    if not tok:
        return False
    with _TOKEN_LOCK:
        exp = _TOKENS.get(tok)
        return bool(exp and exp > time.time())
```

路由层面，密码错误有 0.4 秒延迟，写接口一律只认令牌：

```python
if path == "/api/auth":
    if body.get("pwd") != PWD:
        time.sleep(0.4)                  # 轻微延迟，增加暴力尝试成本
        self._json({"ok": False, "message": "密码不正确"}, code=403)
        return
    tok = issue_token()
    self._json({"ok": True, "token": tok, "expires_in": TOKEN_TTL})
    return

if path in ("/api/state", "/api/data"):
    if not check_token(body.get("token")):
        self._json({"ok": False, "message": "未解锁或登录已过期，请重新验证"},
                   code=403)
        return
```

前端相应改成拿令牌、存 `sessionStorage`（关掉页面即失效）：

```javascript
function submitAuth() {
  var v = ($('pwd-input').value || '').trim();
  fetch('/api/auth', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ pwd: v })
  }).then(function (r) {
    if (r.status === 404) { toast('当前为静态访问，服务端接口不可用'); return null; }
    return r.json();
  }).then(function (j) {
    if (!j) return;
    if (j.ok) { setAuthSession(j.token); /* 解锁 UI */ }
    else { toast(j.message || '密码不正确'); }
  });
}
```

**这一步带来的额外好处**：本地版推送数据到线上时，也要先拿令牌——所以本地 `config.json` 里虽然有一份线上密码，但每次请求走的都是临时令牌，密码不会在网络上来回传。

### 改造 2：本地上行同步改用令牌，失效自动重试

```python
# serve.py —— 本地服务推送线上
_UP_TOKEN_CACHE = {"key": "", "token": "", "exp": 0}

def _upstream_token(conf):
    key = conf["url"] + "|" + conf["pwd"]
    now = time.time()
    if _UP_TOKEN_CACHE["key"] == key and _UP_TOKEN_CACHE["exp"] > now + 60:
        return _UP_TOKEN_CACHE["token"]                # 缓存未过期直接复用
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
    payload.pop("pwd", None)          # 绝不明文带密码
    payload["token"] = _upstream_token(conf)
    j = _http_post_json(conf["url"] + api_path, payload, timeout=timeout)
    if not j.get("ok"):
        msg = str(j.get("message") or "")
        if "过期" in msg or "未解锁" in msg:
            _UP_TOKEN_CACHE["exp"] = 0                # 强制刷新
            payload["token"] = _upstream_token(conf)
            j = _http_post_json(conf["url"] + api_path, payload, timeout=timeout)
    return j
```

### 改造 3：周发文数 / 周阅读数，两种口径

「周」有两种理解，看板上做成可切换的按钮，选择结果记在本机：

```javascript
function weekRange() {
  if (!dates.length) return null;
  var base = dates[dates.length - 1];              // 最新数据日
  var b = new Date(base + 'T00:00:00');
  var s = new Date(base + 'T00:00:00');
  if (weekMode === 'mon') {
    var wd = b.getDay();                           // 0 = 周日
    s.setDate(b.getDate() - (wd === 0 ? 6 : wd - 1));  // 回退到本周一
  } else {
    s.setDate(b.getDate() - 6);                    // 含今天往前 7 天
  }
  return { start: isoDate(s), end: base };
}
```

- **近 7 天**：从最新数据日往前数 6 天，滚动窗口，永远满 7 天
- **本周**：最新数据日所在自然周的周一至周日，周一时只有 1 天

### 改造 4：发布日期校正（修掉错标的老 bug）

v1 用 `setdefault` 写入首发日期——**只写一次，之后再也不更新**。补扫历史数据后，几十篇老文章会被错标成"最近几天发的"。

权威口径应该是：**文章在某统计日有数据 → 取最早的那天为发布日期**。

```python
# sync.py —— 保存前统一回填校正
def fix_first_dates(store):
    """按「文章日报」重算所有文章的首发日期。

    老版本（setdefault）把首次写入的日期固定下来，补扫历史数据后
    也不会更新，导致多篇老文章被错标为最近几天的日期。
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
```

前端也同步算一遍，保证不刷新服务端也能看到正确日期：

```javascript
var firstSeen = {};
Object.keys(articleDaily).forEach(function (d) {
  Object.keys(articleDaily[d]).forEach(function (m) {
    var x = articleDaily[d][m];
    var a = dailyByArticle[m] || (dailyByArticle[m] = { read: 0, share: 0, fav: 0 });
    a.read += int(x.read_user);
    a.share += int(x.share_user);
    a.fav += int(x.collection_user);
    if (!firstSeen[m] || d < firstSeen[m]) firstSeen[m] = d;   // 取最早
  });
});

function pubdateOf(m) {
  return firstSeen[m] || ((articles[m] || {}).first_date) || '';
}
```

### 改造 5：趋势图加比例尺

数据一多，趋势图挤成一团。ECharts 的 `dataZoom` 用「内部 + 底部滑块」双通道，鼠标滚轮、触屏双指、拖滑块都能缩放：

```javascript
grid: { left: 48, right: 14, top: 30, bottom: 58 },   // 底部留 58px 给滑块
dataZoom: [
  { type: 'inside', xAxisIndex: 0, zoomOnMouseWheel: true,
    moveOnMouseMove: true, moveOnMouseWheel: false },
  { type: 'slider', xAxisIndex: 0, height: 20, bottom: 6,
    borderColor: 'transparent', backgroundColor: 'rgba(0,0,0,.04)',
    fillerColor: 'rgba(37,99,235,.12)',
    handleStyle: { color: '#fff', borderColor: '#2563eb' },
    labelFormatter: function (v) {
      return String(v).slice(5);        // 只显示 MM-DD，避免拥挤
    }
  }
],
```

再配一排快捷按钮（全部 / 近90天 / 近30天 / 近7天），手动拖滑块时自动取消按钮高亮，避免状态不一致：

```javascript
function bindTrendZoom(c) {
  if (!c) return;
  c.off('dataZoom');
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
```

## 七、完整源码获取与目录结构

v2 的 `index.html` 已经一千多行、一百多 KB，全贴进文章既没意义也没法复制。完整源码放在 GitHub：

> **https://github.com/Yellowsoy/gzh-data-dashboard**（MIT 协议）

```
gzh-data-dashboard/
├── README.md                     # 完整技术文档
├── LICENSE                       # MIT
├── local-deployment/             # 版本一：本地电脑部署
│   ├── index.html                # 看板页面（单文件）
│   ├── serve.py                  # 本地服务：采集调度 + 写接口 + 上行同步
│   ├── sync.py                   # 微信官方接口采集脚本
│   ├── config.example.json       # 配置模板
│   ├── run.sh / stop.sh          # 一键启动 / 停止
│   └── vendor/                   # echarts / xlsx 离线库
└── workbuddy-deployment/         # 版本二：线上部署
    ├── index.html
    ├── server.py                 # 静态资源 + 同步 API + 令牌鉴权
    └── vendor/
```

拿到源码后，**唯一需要你手写的文件**是 `config.json`：

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
`lookback_days` | 按"统计日"往回补几天，日常保持 `7`
`article_detail_days` | 按"发表日"取多少天内发表的文章，默认 `30`
`upstream` | 线上站点地址，**留空就是纯本地，不同步**
`upstream_pwd` | 线上写接口密码（与管理密码一致）
`upstream_auto_push` | 采集完成后是否自动推送线上

## 八、手把手：两种部署方式

### 方式 A：本地电脑部署（推荐先跑通这个）

```bash
cd local-deployment
cp config.example.json config.json
# 编辑 config.json，填入 appid / secret
./run.sh            # 启动，默认 http://127.0.0.1:8000
./stop.sh           # 停止
```

启动后：

1. 打开页面，点顶栏「锁定」按钮，**输入管理密码解锁**
2. 点「数据更新」触发采集（服务端子进程跑 `sync.py`，有防重入）
3. 采完自动写入本机 `data.js`；若配了 `upstream`，会自动推到线上

`run.sh` 的几个参数：

```bash
./run.sh                 # 启动服务并尝试打开浏览器
./run.sh --port 8080     # 指定端口
./run.sh --no-browser    # 不自动开浏览器
./run.sh --sync-only     # 只跑一次采集，不启动服务（等价 v1 行为）
```

> **部署前务必改掉默认密码**。仓库里的默认值是占位符 `change-me`，本地版改 `serve.py` 里 `ACCESS_PWD`，线上版设置环境变量 `ADMIN_PWD` 后重启。

### 方式 B：线上部署（给别人看 / 手机上也能开）

1. 取 `workbuddy-deployment/` 全部文件
2. 传到任意能跑 Python 的环境，启动：`python3 server.py`
3. 服务监听 `PORT` 环境变量（缺省 3000），绑定 `0.0.0.0`
4. 数据不用手动传——本地采完会自动推；也可以在线上页面用「上传数据文件」手动覆盖

## 九、设置每天自动同步

```cron
# 每天 8:30 自动采集（Linux / macOS / 统信 UOS）
30 8 * * * cd /path/to/local-deployment && /usr/bin/python3 sync.py >> sync.log 2>&1
```

编辑方式：终端执行 `crontab -e`，粘贴上面一行，路径换成你自己的。Windows 用「任务计划程序」，触发器选"每天"，操作填 `python` 和 `sync.py` 的完整路径。

> 漏跑不用慌，下次运行会自动补最近约 7 天。

## 十、看板功能说明

**关键指标**：累计粉丝、最新日净增、近 7 天阅读/分享人数、文章总数、**周发文数 / 周阅读数**（两种口径可切换）

**图表**：阅读与分享趋势（带比例尺缩放）、阅读来源构成、粉丝趋势、文章排行

**文章明细**：搜索（支持拼音首字母）、日期筛选、排序、分页、按时间范围导出 Excel

**回收站**：删除 / 还原 / 彻底删除，状态云端同步；**未解锁时入口完全隐藏**（不只在视觉上隐藏，是 DOM 层面就没有）

**布局自定义**：模块可拖拽调位置和大小，分两级——临时布局只影响本次查看；默认布局输入密码保存到云端，所有设备生效

## 十一、常见问题与错误码速查

**Q：提示 40164 / "invalid ip"？**
出口 IP 不在白名单。脚本会打印你当前的公网 IP，去「基本配置 → IP白名单」加上，等 1–2 分钟生效。

**Q：提示 48001？**
接口未授权。确认公众号已通过微信认证，且具备对应接口权限。

**Q：提示"未解锁或登录已过期"？**
令牌 8 小时过期，点顶栏「锁定」重新输密码。

**Q：微信里打不开线上链接？**
部分托管平台的网关会按 UA 拦截微信内置浏览器，用系统浏览器打开即可。

**Q：发布日期和实际不符？**
v2 已按「文章日报最早日期」校正。若仍不对，跑一次 `./run.sh --sync-only` 触发 `fix_first_dates()` 重算。

**Q：换网络 / 重启后不能用了？**
多半是公网 IP 变了，跑一次脚本，按提示把新 IP 加进白名单。

## 十二、后续维护与备份

- **每天**：定时任务到点自动采集，刷新页面即可
- **漏跑**：下次运行自动补最近约 7 天
- **备份**：只需保留 **`config.json`** 和 **`store.json`**，其余都能重新生成
- **换机迁移**：整个文件夹连同这两个文件一起拷走，在新机器配好 IP 白名单即可
- **安全**：`config.json` 含 AppSecret，**已被 `.gitignore` 排除**，不要提交到任何仓库

---

## 写在最后

v2 相比 v1，代码量翻了一倍多，但**"零成本"这条底线没有破**：依然不买服务器、不装数据库、不依赖云服务，Python 端依然只用标准库。

回头看，最值钱的经验不是某段代码，而是这两条：

1. **动手前先把平台边界搞清楚**——哪些是接口能力问题，哪些是网络环境问题。分清楚这个，就不会在"为什么不行"上反复打转。
2. **凡是和用户身份有关的判断，一律放在服务端**。前端做权限校验，本质上等于没做——v1 的密码裸奔就是这么来的。

如果你也在做类似的事，希望这篇能帮你少走几段弯路。源码在 [GitHub](https://github.com/Yellowsoy/gzh-data-dashboard)，有问题欢迎提 Issue。
