# 技术架构

> 给未来接手维护这个项目的工程师 / HR 看的。从这份文档加上 README + CHANGELOG 应该能完整接手。

---

## 整体形态

**单文件 HTML, 无构建、无后端、无外部依赖。**

整个 `index.html` 大约 2000 行 / 100 KB，包含：

- HTML 结构 (`<head>` 元数据 + `<body>` 容器)
- 全套 CSS 样式 (深紫蓝渐变背景、毛玻璃卡片、日 / 夜模式)
- Logo PNG (base64 内嵌)
- 数据层 (DEPARTMENTS / MODULES / CHECKLIST_GROUPS 三个常量数组)
- 渲染层 (renderWelcome / renderHub / renderModule / renderCert)
- 持久化层 (localStorage `partykeys_offboarding_v1`)
- 背景动画 (Canvas 浮动音符粒子)

---

## 数据流

```
用户访问 lastday.partykeys.org
  → 浏览器加载 index.html (约 100 KB)
  → 读取 localStorage 的 partykeys_offboarding_v1
       ├─ 有 name + dept → 直接进 hub
       └─ 没有 → 进 welcome
  → 用户点击模块卡片
  → renderModule(mid) 渲染该模块
       ├─ 同时把该模块标记为 passed (写入 localStorage)
       └─ 检查是否所有必修模块都完成 → 解锁完成单
  → 用户完成所有模块 → 点击校友卡 → renderCert() 渲染交接完成单
```

**所有数据只在浏览器本地。** 没有任何后端。换浏览器 / 清缓存会重置。

---

## 数据结构 · 改内容时只改这三处

### 1. `DEPARTMENTS` 数组

部门列表。每个对象：

```js
{ id: 'product', label: '产品研发部', emoji: '🔧', color: 'var(--c-product)' }
```

加新部门 → 在这里加一行 + 加对应的 `d_xxx` 模块。

### 2. `MODULES` 数组

所有模块。每个对象的 `kind` 是三种之一：

- `general` — 通用模块 (g1, g2, g3)，按顺序解锁
- `department` — 部门模块 (d_product 等)，只显示用户自己部门的那一个
- `tool` — 工具 (t_handover, t_help)，永远解锁

每个模块的 `blocks` 数组就是页面内容。`block.type` 支持：

- `block` (默认) — 普通内容块，标题 + body (HTML)
- `checklist` — 渲染 `t_handover` 用的可勾选清单 (从 CHECKLIST_GROUPS 读)

模块没有 `quiz` 字段了 — 删除测验机制后，模块"看完就算"。

### 3. `CHECKLIST_GROUPS` 数组

`t_handover` 模块用的清单。每项必须有唯一 `id` (用于 localStorage 存勾选状态)。

加新条目 → 在对应分组的 `items` 数组里加一项 + 给唯一 id。

---

## 渲染层 · 4 个 render 函数

```js
navigate(view, payload)
  ├─ 'welcome'      → renderWelcome()        // 输姓名 + 部门
  ├─ 'admin-login'  → renderAdminLogin()    // 管理者密码 (AISuperman)
  ├─ 'hub'          → renderHub()           // 大厅卡片墙
  ├─ 'module'       → renderModule(mid)     // 模块详情页 + 自动标 passed
  └─ 'cert'         → renderCert()          // 交接完成单
```

**模块完成机制 (v3 简化版)**：

```js
// renderModule 内部
if (!admin && !isTool && !isPassed(mod.id)) {
  state.moduleStatus[mod.id] = { passed: true, score: 100, attempts: 1 };
  saveState();
}
```

打开模块 = 自动标记完成。没有测验、没有打分、没有"提交"按钮。

---

## CSS 关键变量

```css
:root {
  --c-general:  #38bdf8;  /* 通用模块色 */
  --c-product:  #f472b6;  /* 产研 */
  --c-supply:   #34d399;  /* 交付 */
  --c-cn:       #fbbf24;  /* 国内电商 */
  --c-overseas: #a78bfa;  /* 海外电商 */
  --c-done:     #4ade80;  /* 完成绿 */
}
```

改色 → 改这里就行。

日 / 夜模式：`body.light { ... }` 一整套覆盖样式，跟主题切换按钮联动。

---

## Logo

**实现**：base64 PNG 嵌在 `LOGO_SVG` 常量里 (虽然变量名叫 SVG 是历史遗留，实际是 `<img>` 标签)。

**使用位置 5 处**：

1. welcome 页大 logo (140×140)
2. 管理者登录页 logo (140×140)
3. hub 头部 brand (26×26)
4. 模块页头部 brand (26×26)
5. 交接完成单页头 brand (26×26)

**改 logo**：

1. 把新 logo 放到 `~/Downloads/PartyKeys Logo.png`
2. 跑 `outputs/build_pdf.py` 同目录的转码脚本 (或手动 base64 -i logo.png)
3. 替换 `index.html` 里 `const LOGO_SVG = ...` 那一行

5 处全部跟着更新，因为都引用同一个常量。

---

## 部署链路

```
本地编辑 index.html
  → git push
  → GitHub (PartyBohan/lastday)
  → Vercel webhook 触发自动部署
  → lastday.partykeys.org 1-2 分钟后更新
```

**Vercel 配置** (Dashboard 里设置)：

- Framework Preset: Other
- Root Directory: 空 (根目录)
- Build Command: 空 (无构建)
- Output Directory: 空 (静态)

---

## 维护备忘 (前任工程师留言)

- **Logo 文件别动**：`PartyKeys Logo.png` 是公司官方 logo, 改 logo 要走品牌部门审批
- **localStorage key 别改**：`partykeys_offboarding_v1` 改了 → 已经走过流程的同事进度全清空
- **管理者密码 `AISuperman`** 在 `index.html` 里硬编码 — 想换密码改这一行就行 (但记得告诉 HR)
- **Vercel 自动部署常常忘记缓存**：浏览器 Cmd+Shift+R 硬刷可以避免看到旧版
- **改完 index.html 后跑一遍 Node 语法检查**：

```bash
node -e "
const fs=require('fs'); const m=fs.readFileSync('index.html','utf-8').match(/<script>([\s\S]+?)<\/script>/);
try { new Function(m[1]); console.log('OK'); } catch(e){ console.log('SYNTAX ERROR:', e.message); }
"
```

---

## 已知限制 / 未来 TODO

- 进度只在浏览器本地，无法跨设备同步
- 没有跟 HR 系统打通 (理想方案：HR 审批通过离职后，自动生成本人专属链接)
- 没有英文版 (海外团队用)
- 完成单只能打印 / 保存 PDF，没有"一键归档到飞书云文档"
- 部门交接清单太粗 — 现在每部门 1 个块，应该按子板块细化

发起改动建议：飞书 @ Bohan 或 GitHub Issues。
