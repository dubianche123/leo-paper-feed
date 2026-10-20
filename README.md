<div align="center">

<img src="docs/readme/cover.svg" alt="Leo Paper Feed project overview" width="100%" />

# Leo Paper Feed · 论文精读笔记数据源

**一个可追溯版本的论文笔记发布面，交给阅读端直接消费。**

[![Format](https://img.shields.io/badge/Format-JSON%20%2B%20Markdown-567A62)](feed.json)

**简体中文** · [English](README_EN.md)

[读取](#直接读取) · [数据契约](#数据契约) · [阅读端对接](#接到自己的阅读端) · [本地查看](#本地查看)

</div>

为日常论文精读 App 专门维护一台后端服务器和数据库其实很划不来。这个仓库采用了更轻的做法：把每天生成的论文精读笔记直接编译成单文件 **[feed.json](feed.json)**，通过 Git 版本化发布，让客户端或任何第三方阅读器直接拉取静态 JSON 消费。

这样做有三个很实在的好处：**零后端运维**、**每次更新在 Git Commit 里一清二楚**，而且任何阅读端只需要几十行代码就能接入。本仓库专门作为公开数据发布层，笔记生成脚本、定时调度器与原阅读 App 的源码不在本仓库内。

## 直接读取

[打开原始 JSON](https://raw.githubusercontent.com/dubianche123/paper-reading-notes-feed/main/feed.json) · [查看当前文件](feed.json) · [发布历史](https://github.com/dubianche123/paper-reading-notes-feed/commits/main/feed.json)

直接拉取公开文件不需要 GitHub Token。`main` 分支地址会随每日发布更新；如果你想在测试或阅读器里固定某天的快照，可以直接从提交历史里复制对应 Commit 的 permalink。

## 数据契约

![日期块、频道、Markdown 与可选条目](docs/readme/flow-cn.svg)

`feed.json` 顶层按日期（`days`）和阅读频道（`channels`）组织：

```json
{
  "days": [
    {
      "date": "YYYY-MM-DD",
      "model": "source-label",
      "channels": [
        {
          "id": "01",
          "title": "Reading theme / 阅读主题",
          "content": "# Example title\n\nMarkdown body with [a link](https://example.org).",
          "items": [
            {"title": "Example item", "summary": "Short summary", "pickId": "opaque-id"}
          ]
        }
      ]
    }
  ]
}
```

| 路径 | 类型 | 阅读端怎样使用 |
| --- | --- | --- |
| `days` | 数组 | 遍历日期块，不写死天数。 |
| `days[].date` | 字符串 | 展示发布数据携带的日期。 |
| `days[].model` | 字符串 | 保留原始模型/来源标识；这是元数据，不是供应商 API 配置。 |
| `days[].channels` | 数组 | 生成这个日期下的频道列表。 |
| `channels[].id`、`.title` | 字符串 | 频道标识与名称；将日期和 ID 一起保留。 |
| `channels[].content` | 字符串 | 渲染完整的 Markdown 阅读正文。 |
| `channels[].items` | 可选数组 | 补充结构化摘要；当前数据允许没有这个数组。 |
| `items[].title`、`.summary`、`.pickId` | 字符串 | 展示条目信息，保留 `pickId` 作为不透明标识。 |

`content` 与 `items` 的职责不同。即使没有 `items`，阅读端仍可通过 `content` 展示完整频道。不要把条目摘要拼接成正文，也不要从 `pickId` 猜测论文 URL。

## 接到自己的阅读端

下面的适配函数获取文件并返回规整后的频道记录，不假定每天都有相同频道，也不假定每个频道都有 `items`。

```javascript
const FEED_URL = "https://raw.githubusercontent.com/dubianche123/paper-reading-notes-feed/main/feed.json";

async function loadReadingChannels(url = FEED_URL) {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`Feed HTTP ${response.status}`);
  const feed = await response.json();
  if (!Array.isArray(feed.days)) throw new Error("Expected feed.days array");

  return feed.days.flatMap(day =>
    (Array.isArray(day.channels) ? day.channels : []).map(channel => ({
      date: day.date,
      model: day.model,
      id: channel.id,
      title: channel.title,
      markdown: typeof channel.content === "string" ? channel.content : "",
      items: Array.isArray(channel.items) ? channel.items : [],
    }))
  );
}

const channels = await loadReadingChannels();
console.log(channels.map(({ date, id, title }) => ({ date, id, title })));
```

将返回的 `title`、`date` 和 `markdown` 接到阅读组件。Markdown 渲染与链接处理留在阅读端；正文可能包含标题、列表和外部链接。确实存在且适合展示时，再使用 `items` 渲染条目卡片。

这个入口是**静态 JSON 文件**，没有服务端分页、日期筛选或查询 API。刷新、加载/错误状态、搜索和本地缓存由消费端实现；如果希望短暂读取失败时还能继续阅读，可以保留上一次成功载入的数据。

## 本地查看

只用 Python 标准库即可检查实际日期与频道结构：

```bash
git clone https://github.com/dubianche123/paper-reading-notes-feed.git
cd paper-reading-notes-feed
python3 - <<'PYTHON'
import json
from pathlib import Path
feed = json.loads(Path("feed.json").read_text(encoding="utf-8"))
for day in feed.get("days", []):
    print(day.get("date"), day.get("model"))
    for channel in day.get("channels", []):
        print(" ", channel.get("id"), channel.get("title"),
              "items:", len(channel.get("items", [])))
PYTHON
```

| 目标 | 入口 |
| --- | --- |
| 读取最新发布笔记 | [feed.json](feed.json) |
| 检查某次发布改了什么 | [文件历史](https://github.com/dubianche123/paper-reading-notes-feed/commits/main/feed.json) |
| 开发消费端 | 上面的字段契约与适配函数 |
| 运行生成程序或构建原 App | 这两个组件位于仓库之外 |

## 边界说明

频道划分与笔记排布可能会随日常阅读主题微调。消费端解析时只需依赖上表中的基础字段（并为可选的 `items` 做好空数组回退），这样即使某天新增或调整了频道，阅读器也能平稳兼容。
