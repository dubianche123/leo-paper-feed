<div align="center">

<img src="docs/readme/cover.svg" alt="Leo Paper Feed project overview" width="100%" />

# Leo Paper Feed

**论文精读内容的数据入口：日期、频道和 Markdown 正文。**

**中文** · [English](README_EN.md)

[读取](#直接读取) · [结构](#json-结构) · [本地查看](#本地查看)

</div>

Leo 论文精读 App 的公开数据源。当前仓库提供 [feed.json](feed.json)；生成程序与阅读 App 的代码不在这里。内容按日期和频道组织，正文使用 Markdown，适合由阅读端读取并展示。

## 数据如何流动

![工作原理概览](docs/readme/flow-cn.svg)

## 直接读取

[原始 JSON](https://raw.githubusercontent.com/dubianche123/leo-paper-feed/main/feed.json) · [查看当前数据](feed.json)

```javascript
const response = await fetch("https://raw.githubusercontent.com/dubianche123/leo-paper-feed/main/feed.json");
if (!response.ok) throw new Error(`Feed HTTP ${response.status}`);
const feed = await response.json();

for (const day of feed.days ?? []) {
  for (const channel of day.channels ?? []) {
    console.log(day.date, channel.id, channel.title);
    // channel.content 是 Markdown 正文，由阅读端渲染。
  }
}
```

不需要 GitHub Token。这个地址是版本库中的静态文件地址，不是带分页和查询参数的后端接口。

## JSON 结构

以下是结构示例，正文与标题均为占位内容：

```json
{
  "days": [
    {
      "date": "YYYY-MM-DD",
      "model": "source-label",
      "channels": [
        {
          "id": "01",
          "title": "本期阅读主题",
          "content": "# 示例标题\n\nMarkdown 正文",
          "items": [{"title": "示例条目", "summary": "摘要", "pickId": "example-id"}]
        }
      ]
    }
  ]
}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `days` | 数组 | 按日期组织的内容集合 |
| `date` | 字符串 | 当前内容块中的日期 |
| `model` | 字符串 | 数据携带的 model 标识，阅读端保留原值 |
| `channels` | 数组 | 该日期下的频道集合 |
| `id`、`title` | 字符串 | 频道标识与显示标题 |
| `content` | 字符串 | Markdown 正文 |
| `items` | 可选数组 | 部分频道提供的条目摘要，当前条目字段包括 `title`、`summary`、`pickId` |

不同日期或频道不一定都有 `items`；读取时按可选字段处理，不要把某一期的条目数量当作固定协议。

## 本地查看

克隆后，可以只用 Python 标准库检查当前结构：

```bash
git clone https://github.com/dubianche123/leo-paper-feed.git
cd leo-paper-feed
python3 -c 'import json; d=json.load(open("feed.json")); print([(r["date"], len(r.get("channels", []))) for r in d.get("days", [])])'
```

## 范围与更新

- 本仓库用于公开、只读消费的数据分发；读取示例不需要写入权限。
- 内容由外部流程生成；仓库不包含采集器、定时任务或 App 构建入口。
- 内容和日期会随更新变化。集成时保留缺失字段处理，并由客户端负责刷新、Markdown 展示和链接处理。
