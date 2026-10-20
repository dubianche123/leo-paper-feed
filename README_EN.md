<div align="center">

<img src="docs/readme/cover.svg" alt="Leo Paper Feed project overview" width="100%" />

# Leo Paper Feed · Paper-reading notes data

**A versioned publication surface for paper notes, ready for a reader to consume.**

[![Format](https://img.shields.io/badge/Format-JSON%20%2B%20Markdown-567A62)](feed.json)

[简体中文](README.md) · **English**

[Fetch](#read-the-feed) · [Contract](#data-contract) · [Reader adapter](#adapt-it-to-a-reader) · [Inspect locally](#inspect-locally)

</div>

Running a dedicated database and backend server just to serve daily paper-reading notes is unnecessary overhead. This repository takes a lighter approach: daily curated reading notes are compiled into a single **[feed.json](feed.json)** file and published via Git, so any reader app or script can fetch the static JSON directly.

That keeps the setup **zero-backend**, makes **every daily update auditable in `git log`**, and lets any reader client integrate in ~20 lines of code. This repository serves strictly as the public data publication layer; the note generator, scheduler, and original reader app live in separate repositories.

## Read the feed

[Open the raw JSON](https://raw.githubusercontent.com/dubianche123/paper-reading-notes-feed/main/feed.json) · [Inspect the current file](feed.json) · [Publication history](https://github.com/dubianche123/paper-reading-notes-feed/commits/main/feed.json)

No GitHub token is required. The `main` branch URL tracks the latest daily release; if you want a pinned snapshot for testing or archiving, grab a commit permalink from the file history.

## Data contract

![Date blocks, channels, Markdown and optional items](docs/readme/flow.svg)

`feed.json` groups notes by publication date (`days`) and reading channel (`channels`):

```json
{
  "days": [
    {
      "date": "YYYY-MM-DD",
      "model": "source-label",
      "channels": [
        {
          "id": "01",
          "title": "Reading theme",
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

| Path | Type | How a reader uses it |
| --- | --- | --- |
| `days` | Array | Iterate date blocks; do not hard-code a day count. |
| `days[].date` | String | Display the date supplied by the publisher. |
| `days[].model` | String | Preserve the supplied model/source label; it is metadata, not a provider API configuration. |
| `days[].channels` | Array | Build the channel list for that date. |
| `channels[].id`, `.title` | Strings | Identify and label a channel; retain the date alongside its ID. |
| `channels[].content` | String | Render the complete Markdown reading body. |
| `channels[].items` | Optional array | Supplementary structured summaries; an absent array is valid in the current data. |
| `items[].title`, `.summary`, `.pickId` | Strings | Display item metadata and preserve `pickId` as an opaque identifier. |

`content` and `items` serve different roles. A reader can display a complete channel from `content` even when there is no `items` list. Do not rebuild the Markdown body by concatenating item summaries or infer a paper URL from `pickId`.

## Adapt it to a reader

This small adapter fetches the file and returns normalized channel records. It does not assume every date has the same channels or every channel has `items`.

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

Connect the returned `title`, `date` and `markdown` to your reader's components. Keep Markdown rendering/link handling in the reader; a channel body may contain headings, lists and external links. Use item cards only when `items` is present and useful for that view.

The endpoint is a **static JSON file**: no server-side pagination, date filter or query API is implemented here. Refreshing, loading/error states, search and local caching belong to the consuming application. Preserve the last successfully loaded records if you want reading to continue through a transient fetch error.

## Inspect locally

Python's standard library is enough to inspect the actual date/channel structure:

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

| You want to… | Start here |
| --- | --- |
| Read the latest published notes | [feed.json](feed.json) |
| See exactly what changed | [File history](https://github.com/dubianche123/paper-reading-notes-feed/commits/main/feed.json) |
| Build a consumer | The data-contract table and adapter above |
| Run a producer or build the original App | Those components are outside this repository |

## Scope & compatibility

Channel themes and note layouts may evolve over time as daily reading topics shift. As long as your consumer relies on the core fields in the table above (and falls back to `[]` when optional `items` are omitted), new dates and channels will render cleanly without breaking existing readers.
