+++
title = '用 Pi 搭一套会自我归档的 AI 运维知识库'
date = '2026-09-07T00:20:00+08:00'
draft = false
tags = ['Pi', 'AI Agent', '运维', 'DevOps', '提示词工程', '自托管', '教程']
description = '让 AI 运维不再「每次开工忘光上下文、同一个坑踩两次、改完不留痕」：用 Pi 的项目级扩展和技能，搭一套 AGENTS.md 静态注入 + ops-context 动态注入 + 技能 SOP 的三层知识库，附一键脚手架提示词。'
[cover]
  image = 'https://s3.blog.zeroicey.me/covers/pi-ops-knowledge-base.jpg'
+++

前面几篇都是「给机器装东西」，这篇换个角度——**给替你干活的那个 AI 装一套记忆**。

用 AI 做运维有一阵了，我最大的感受是：它能力没问题，但**记性是真的差**。每次新开一个会话，它就把上一次的活儿忘得干干净净——这台机器跑着哪些服务、端口怎么分的、上次那个坑是怎么解开的、哪台服务动过之后要特别注意……全都要我重新交代一遍。更要命的是，**同一个坑它能踩两次**：上次花了半小时定位的诡异报错，下次换个会话又原样中招。

所以我花了点时间，把我这套运维工作流沉淀成了一个「外置记忆」，刻进仓库里。现在每次新会话开工，AI 不用我问，就知道自己该去哪查资料、该守什么规矩、还欠着什么没收拾。这篇文章把这套东西完整拆开给你看。

## 为什么是 Pi，而不是通用做法

这套方案是**完全绑在 Pi 上**的——如果你用的不是 Pi，下面的东西大部分直接失效。Pi 有几个别的 agent 没有的「钩子」，正是这套架构的地基：

1. **`AGENTS.md` 自动加载**：把 `AGENTS.md` 放进项目根目录，Pi 每次启动就把它当作常驻上下文喂进去，相当于给 AI 一段「永久系统提示词」。这跟某些工具要手动 `@file`、或者要把规则塞进全局配置里是两回事——它是**项目级、自动、零操作**的。
2. **`.pi/extensions/*.ts` 项目级扩展**：你可以在仓库里放 TypeScript 扩展，Pi 启动时自动发现并加载。扩展能挂在 `before_agent_start`、`session_start` 这些生命周期钩子上，还能注册自定义斜杠命令。
3. **`.pi/skills/*/SKILL.md` 技能自动发现**：一个带 `name` + `description` 前置元数据的 markdown 文件就是一个「技能」，当任务匹配描述时 Pi 会自动把它的内容读进来。等于你能给 AI 写一段「遇到这种场景，按这个 SOP 来」。

三个钩子叠起来，正好对应「记性差」的三个病因，我据此搭了三层机制：

| 层 | 载体 | 触发时机 | 解决什么 |
| --- | --- | --- | --- |
| 静态注入 | `AGENTS.md` | Pi 启动即加载 | 不知道「去哪查资料、守什么纪律」 |
| 动态注入 | `.pi/extensions/ops-context.ts` | 每会话首次开工 | 不知道「之前干了什么、还没收什么尾」 |
| 流程技能 | `.pi/skills/ops-*` | 按任务描述触发 | 排障乱来、改完不留痕 |

## 文件夹架构（核心）

整套东西就是一个普通的 git 仓库，长这样：

```text
ops/
├── AGENTS.md                          # 常驻上下文：系统概况 + 检索路由 + 维护纪律
├── README.md                          # 仓库说明 + 目录树 + 自动化机制表
├── .gitignore                         # 忽略任务产物 / 缓存
├── specs/                             # 规范（活文档，随时修订保持最新）
│   ├── conventions.md                 #   仓库写作约定（命名/格式/只追加等）
│   ├── deployment.md                  #   服务部署 house standard（docker compose 规范）
│   ├── maintenance.md                 #   监控 / 备份 / 更新 / 对账审计
│   └── decisions.md                   #   决策账本（append-only，关键决定可考）
├── runbooks/                          # 服务快速上手手册（一事一档，实测验证）
│   └── <服务>.md                       #   端口 / compose / 数据 / 常用操作 / 排障
├── pitfalls/                          # 踩坑档案（四段式：现象→原因→解法→预防）
│   ├── _index.md                       #   索引表：日期 | 标题 | 标签 | 状态 | 文档
│   └── YYYY-MM-DD-slug.md
├── logs/
│   └── changelog.md                   # 变更日志（只追加，审计用）
├── inventory/                         # 系统资产快照（以实测为准，不凭记忆）
│   ├── machine.md                      #   硬件 / 系统 / 关键路径
│   ├── services.md                     #   服务清单：端口、compose、数据位置
│   ├── hosts.md                        #   主机台账：设备 + SSH 别名 + 密钥映射
│   └── network.md                      #   三层网络 + 节点清单 + 代理链
├── reports/                           # 历史审计报告（只读存档）
├── scripts/                           # 可复用运维脚本
└── templates/
    └── pitfall.md                     # 坑文档模板
```

每个目录各管一件事，职责不重叠，我逐一说下「为什么这么分」：

- **`inventory/` 是「事实层」**——这台机器现在到底什么状态，一切以此为准，而且要**实测**（`docker ps` / `ss -tlnp` / `systemctl cat`），绝不凭记忆。文档漂移的根源就是「记」而不是「测」。
- **`specs/` 是「规则层」**——怎么写文档、怎么部署服务、怎么备份更新，是活文档，随时修订。
- **`runbooks/` 是「操作层」**——每个服务一份快速上手手册，一事一档。
- **`pitfalls/` 是「教训层」**——踩过的坑写成四段式，下次开工先查 `_index.md`，命中率极高。
- **`logs/changelog.md` 是「时间层」**——只追加、永不改历史，专门回答「之前改过什么」。

## 第一层：AGENTS.md（静态注入）

这是整个体系里性价比最高的一段文字——极小的体量，把「去哪查、守什么纪律、哪些事要先确认」三条主线交代清楚。它在仓库根目录，Pi 一启动就加载：

```markdown
# ops —— AI 工作规则

你在 core 服务器的运维仓库中工作。本文件是你的常驻上下文：任何针对这台机器的系统操作都必须遵循以下规则。

## 系统概况（速览）

- 主机：**core** · Arch Linux · 8C16T · 32GB · NVMe 同盘 /data
- 网络：局域网 `192.168.0.5`（eno1）· **Tailscale 虚拟网 `100.64.12.1`（headscale 自托管）**
- 代理：mihomo 系统级服务（mixed-port `127.0.0.1:7890`，rule 模式）
- 服务形态：Docker Compose 为主（compose 在 `/srv/compose/<service>/`，数据在 `/data/services/<service>/`）
- sudo：已开免密，AI 可直接 `sudo -n`；危险操作仍须先说明影响并确认
- 完整事实快照：inventory/machine.md · services.md · network.md

## 检索路由（先查资料再动手）

| 场景 | 先看什么 |
| --- | --- |
| 开始任何维护/部署任务 | `inventory/services.md` 确认端口与依赖现状 |
| 操作某个具体服务 | 对应 `runbooks/<服务>.md` |
| 排查故障/异常 | `pitfalls/_index.md` → 命中坑文档 → 实测 |
| 需要「之前改过什么」 | `logs/changelog.md` |
| 部署新服务 / 下线服务 | `specs/deployment.md` |
| 监控 / 备份 / 更新 | `specs/maintenance.md` |

## 维护纪律

1. 动手前：对照 inventory 确认端口/依赖不冲突；不确定以实测为准，不凭记忆。
2. 完成后必须归档：changelog 追加、踩坑沉淀 pitfall、拓扑变化同步 inventory、规范过时修订 specs。
3. 诚实记录：changelog 只追加不改历史；坑文档写全「现象→原因→解法→预防」。
4. 危险操作先确认：删数据、下线共享服务、动代理/组网前说明影响并获确认。
```

> ⚠️ **替换说明**：与前几篇一致，文中主机名（`core`）、局域网 IP（`192.168.0.5`）、Tailscale IP（`100.64.12.1`）均为虚构示例，照着做时换成你自己的真实值。服务名我都做了泛化处理。

注意那张「检索路由」表——它是**指针而非内容**。不管以后 `runbooks/` 里的服务手册涨到多少份，这张表永远只有几行，因为 AI 只需要知道「去哪查」，而不是把全部内容灌进上下文。这就是这套架构控制上下文体积的关键：**静态注入的量是恒定且克制的**。

## 第二层：ops-context.ts（每会话的动态简报）

AGENTS.md 是「永远的」，但它管不了「上一次会话刚改了啥」。所以我写了一个扩展，挂在 Pi 的 `before_agent_start` 钩子上：每次会话 agent 首次开工时，自动读仓库里的 `logs/changelog.md` 尾部 12 条 + `pitfalls/_index.md` 里「未解决」的坑，拼成一段简报注入上下文——对用户不可见，但 AI 一开工就知道自己的「上一段进度」到哪了。

```typescript
// .pi/extensions/ops-context.ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { readFile } from "node:fs/promises";
import { join } from "node:path";

const CHANGELOG_TAIL_LINES = 12;
const MAX_BRIEF_CHARS = 2400;

async function readText(path: string): Promise<string | null> {
	try { return await readFile(path, "utf8"); } catch { return null; }
}

function changelogTail(raw: string): string | null {
	const lines = raw.split("\n").map((l) => l.trim()).filter((l) => l.startsWith("- "));
	if (lines.length === 0) return null;
	return lines.slice(-CHANGELOG_TAIL_LINES).join("\n");
}

function openPitfalls(raw: string) {
	const results: { title: string; file: string }[] = [];
	for (const line of raw.split("\n")) {
		if (!line.trim().startsWith("|")) continue;
		const cells = line.split("|").map((c) => c.trim());
		if (cells.length < 6 || cells[1] === "日期") continue;
		if (!(cells[4] ?? "").includes("未解决")) continue;
		results.push({ title: cells[2] ?? "(无标题)", file: cells[5] ?? "" });
	}
	return results;
}

export default function opsContextExtension(pi: ExtensionAPI) {
	let injected = false;

	const buildBrief = async (cwd: string) => {
		const parts: string[] = [];
		const changelog = await readText(join(cwd, "logs", "changelog.md"));
		if (changelog) {
			const tail = changelogTail(changelog);
			if (tail) parts.push(`【最近运维变更】\n${tail}`);
		}
		const index = await readText(join(cwd, "pitfalls", "_index.md"));
		if (index) {
			const open = openPitfalls(index);
			if (open.length > 0) {
				parts.push(`【未解决的坑 ×${open.length}】处理相关任务前先读对应文档：\n`
					+ open.map((p) => `- ${p.title} ${p.file}`).join("\n"));
			}
		}
		if (parts.length === 0) return "";
		let brief = "[ops-context] 运维知识库自动简报。维护纪律见 AGENTS.md；任务收尾必须走 ops-record 归档流程。\n\n";
		brief += parts.join("\n\n");
		if (brief.length > MAX_BRIEF_CHARS) brief = brief.slice(0, MAX_BRIEF_CHARS) + "\n…(已截断，完整内容请读源文件)";
		return brief;
	};

	pi.on("before_agent_start", async (_event, ctx) => {
		if (injected) return undefined;
		const brief = await buildBrief(ctx.cwd);
		if (!brief) return undefined;
		injected = true;
		return { message: { customType: "ops-context", content: brief, display: false } };
	});

	pi.on("session_start", () => { injected = false; });

	pi.registerCommand("ops-brief", {
		description: "显示 ops 知识库简报并刷新下次注入",
		handler: async (_args, ctx) => {
			injected = false;
			ctx.ui.notify((await buildBrief(ctx.cwd)) || "ops 简报为空", "info");
		},
	});
}
```

三个关键设计，说一下：

1. **`MAX_BRIEF_CHARS = 2400` 硬截断**——不管 changelog 涨到多大，注入的简报永远不超过这个量。真正读懂「全量历史」靠的是按需去读源文件，而不是全灌进来。
2. **「未解决的坑」单独拎出来**——已解决的坑不打扰你，只有还没收尾的才天天在耳边提醒。等哪天全解决了，这段就自动消失（空转，不报错）。
3. **`/ops-brief` 手动刷新**——偶尔想在会话中途看简报、或强制下一轮重新注入，一条斜杠命令搞定。

## 第三层：两个技能（排障 SOP + 归档纪律）

技能是「遇到什么场景就触发什么」。这里有两个，一个管「出问题怎么查」，一个管「改完怎么收尾」。

**① ops-troubleshoot（排障 SOP）** —— 描述里写明触发词，Pi 检测到「服务不可达 / 容器异常 / 端口不通」等场景就自动加载：

```markdown
---
name: ops-troubleshoot
description: 系统排障 SOP。当出现服务不可达、容器异常退出、端口不通、报错堆栈、磁盘/内存告警、网络断连等需要排查定位的场景时使用。
---

# 排障 SOP

严格按以下顺序，不要跳步。

## 0. 先查坑（命中率最高）
读 pitfalls/_index.md，按症状匹配标签，命中就先读对应坑文档。

## 1. 定位对象
查 inventory/services.md 找到涉事服务的端口、compose 目录、数据目录、依赖关系。

## 2. 实测现状（以命令输出为准，不信记忆）
docker ps -a --filter name=<name> / docker logs --tail 100 <name>
systemctl status <unit> / journalctl -u <unit> -n 100
ss -tlnp | grep :<port> / df -h / && free -h

## 3. 网络类问题：分层排查
容器内 → 宿主绑定(127.0.0.1) → 组网层(Tailscale IP) → 局域网(LAN IP)

## 4. 回溯变更
翻 logs/changelog.md 最近条目：「之前改了什么」往往就是根因。

## 5. 解决后必须归档
新坑 → pitfall 四段式；一切处置 → changelog；拓扑变化 → inventory。

## 红线
需要重启的共享服务先说明影响；代理/组网出问题会断网，动手前告知。
```

「先查坑」这一步是我自己也没料到的高命中率一步——跑久了你会发现，**大部分「新问题」其实都是「老坑」的变体**，查一下索引比从头查日志快得多。

**② ops-record（归档纪律）** —— 管收尾。没有它，前面所有「知识库」都攒不起来：

```markdown
---
name: ops-record
description: 运维动作收尾归档纪律。完成任何系统变更、故障修复、服务部署/下线、配置修改之后必须使用。
---

# 运维归档流程（做完事必须走，缺一不可）

1. 记 changelog：向 logs/changelog.md 末尾追加
   `- YYYY-MM-DD [类别] 一句话结果 (@操作者)`
   类别：deploy / fix / maint / pitfall / cron。只追加，不改历史。

2. 踩了新坑 → 建 pitfall 文档：templates/pitfall.md 复制为
   pitfalls/YYYY-MM-DD-slug.md，写全四段（现象/原因/解法/预防），
   在 _index.md 加一行，解决了标「已解决」。

3. 拓扑变化 → 同步 inventory/services.md（必要时 network.md），刷新快照日期。

4. 规范过时 → 修订 specs/。

5. git add -A && git commit -m "<类别>: <一句话>"

自检：changelog 有新条目？新坑进了索引？inventory 与现实一致？git 已提交？
```

## 规范层：specs/ 的三份活文档

除了上面的「骨架 + 提示词」，还要三份规范把「怎么做」定死。篇幅关系只列要点：

- **`conventions.md`**：仓库写作约定——pitfall 文件名 `YYYY-MM-DD-slug.md`、changelog 只追加、inventory 以实测为准、decisions.md append-only。**格式统一是整套体系能自动被解析的前提**（比如扩展就是靠解析 `_index.md` 表格来找「未解决」坑的，格式乱了注入就废）。
- **`deployment.md`**：服务部署 house standard——compose 固定放 `/srv/compose/<service>/`、数据放 `/data/services/<service>/`、密钥走 `.env` 绝不内联、端口**双绑**（`127.0.0.1` + 组网 IP）、必须有 healthcheck（healthy 才是部署成功的验收标准，别用外部 curl 探活）。
- **`maintenance.md`**：监控基线怎么改、备份原则（先备份再动手、落地即归档 `/data/backups/`）、更新流程、以及一条我后来补的**「文档对账审计」**——定期拿 inventory 的权威事实去 grep 全仓活文档，揪出「已经下线的技术名还残留在文档里教 AI 怎么做」的漂移。

## 一键脚手架：别手抄，让 Pi 自己搭

上面这一整套，其实**不用你手动一个个文件去建**。我把它整理成了一条 bootstrap 提示词——在新机器上装好 Pi、建一个空 `ops/` 目录、开好 sudo 免密之后，把这条提示词整个丢给它，它就会自己把 `AGENTS.md`、扩展、两个技能、所有 specs 和模板全部生成好，再用命令实测这台机器的现状，把 `inventory/` 和 `runbooks/` 填实，最后 `git init` 提交。

提示词我放在 [zeroicey/vibe](https://github.com/zeroicey/vibe) 仓库的 `prompts/init-pi-ops.md`，点开复制即用：

> 📄 [init-pi-ops.md —— Pi 运维知识库一键脚手架提示词](https://github.com/zeroicey/vibe/blob/main/prompts/init-pi-ops.md)

它的流程是「**先照模板搭骨架 → 再实测填事实 → 最后强制自检**」，确保生成出来的不是一个空壳。你唯一要做的，就是开头那句「装好 Pi + 建好 ops 目录 + sudo 免密」。

## 关于保密

这套东西本质是把「你服务器的一切事实」写进文本，所以**必须和代码一样做隔离**：

- 仓库本身放私有 git（或本地），**别 push 到公开仓库**——里面全是真实端口、IP、服务名。
- 像我这篇文章这样要公开分享时，一律**脱敏**：主机名换虚构（`core`）、内网 IP 换虚构段（`100.64.12.x`）、自研服务名换泛称（「一个 API 网关」而不是真名）。
- 密钥绝不在知识库里留明文——只记「密钥存在哪个 env 文件」，不记值。

## 收尾

这套东西跑起来之后，运维就变成了一种很舒服的节奏：新会话开工，AI 自动知道自己上一段干到哪了；出问题，先查坑索引再动手，老坑不再二犯；每改一次，changelog 和 pitfall 都如实归档。**你负责做决定，它负责记、查、守规矩**——这才是「AI 运维」该有的分工。

跟前面的服务系列串起来，这套流程管的是**「变了什么、怎么复现」**，跟 Tailscale 组网、Caddy 域名、Komari 监控这些「基础设施」正好互补：一个是机器的骨架，一个是 AI 的记忆。

相关链接：

- [zeroicey/vibe](https://github.com/zeroicey/vibe) —— 我的 vibe coding 资源集，含本文的脚手架提示词
- [Pi coding agent](https://github.com/earendil-works/pi-coding-agent) —— 项目级扩展 / 技能 / AGENTS.md 的宿主