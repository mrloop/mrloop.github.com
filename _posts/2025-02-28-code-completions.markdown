---
layout: post
title: "Exploring LLMs: A Blind Trial for Code Completions"
categories: [Neovim, LLM, AI]
excerpt: "This article presents findings from a four-month blind trial of LLM code completions in a professional development environment. By tracking over"
---

> **Abstract:** This article presents findings from a four-month blind trial of LLM code completions in a professional development environment. By tracking over 94,000 code suggestions across three leading AI assistants, I found that GitHub Copilot delivers the highest quality suggestions (3.4% acceptance rate) while Supermaven provides the greatest volume of useful completions. Codeium performed significantly worse (0.5% acceptance rate) and was discontinued after seven days. This data-driven approach reveals meaningful differences in LLM performance for practical coding assistance.

Large language models (LLMs) have transformed the coding experience with capabilities ranging from auto-completion to chat interfaces and agentic file editing. In this post, I'll share results from a systematic evaluation of three popular AI code completion tools in a professional development environment.

Rather than relying on subjective impressions, I conducted a rigorous blind trial to determine which model genuinely performs best in day-to-day coding work. This methodology eliminates the confirmation bias that often affects tool selection ("I think this one feels better") and provides concrete data about which assistant actually delivers the most value.

To conduct a randomized blind trial of large language model code completions, I established four key requirements:
1. Configure multiple LLMs for code completions in my development environment
2. Display completion suggestions without revealing which LLM generated them
3. Randomize the presentation order to eliminate selection bias
4. Collect comprehensive usage data over an extended period

### LLMs Under Evaluation

I selected three popular AI code assistants for this experiment:

- **GitHub Copilot**: Microsoft and GitHub's AI pair programmer, built on OpenAI technology, trained on public code repositories and avialbe as a paid subscription service ($10/month or $100/year).
- **Codeium**: A free AI coding assistant from Exafunction that promises high-quality completions with low latency.
- **Supermaven**: AI code completion tool that claims to offer deeper contextual understanding and multi-line completions. Available as both free and paid tiers. The free tier was used for this trial.

By comparing these different systems, I aimed to determine which would be most effective for my daily development workflow.

## Implementation

In this section, I'll explain the technical approach used to create a fair testing environment for these AI auto completions. If you're primarily interested in the findings rather than the methodology, feel free to skip directly to the [results](#results) section.

I use Neovim daily; it is highly customizable and can be configured to meet my needs. The configuration samples shown use [Lazy.nvim](https://github.com/folke/lazy.nvim).

### LLM Configurations

Codeium, GitHub Copilot, and Supermaven are used for code completions.

```lua
{
  "Exafunction/codeium.nvim",
  enabled = os.getenv("CODEIUM") == "true",
  dependencies = {
    "nvim-lua/plenary.nvim",
    "hrsh7th/nvim-cmp",
  },
  config = true,
},
{
  "zbirenbaum/copilot.lua",
  enabled = os.getenv("COPILOT") == "true",
  cmd = "Copilot",
  event = "InsertEnter",
  opts = {
    suggestion = { enabled = false },
    panel = { enabled = false },
  }
},
{
  "zbirenbaum/copilot-cmp",
  enabled = os.getenv("COPILOT") == "true",
  dependencies = { "copilot.lua", "lspkind.nvim" },
  config = true,
  init = function()
    vim.api.nvim_set_hl(0, "CmpItemKindCopilot", { fg = "#6CC644" })
  end,
},
{
  "supermaven-inc/supermaven-nvim",
  enabled = os.getenv("SUPERMAVEN") == "true",
  opts = {
    disable_inline_completion = true,
  },
  init = function ()
    vim.api.nvim_set_hl(0, "CmpItemKindSupermaven", { fg = "#6CC644" })
  end,
}
```

### UI Configuration

For each LLM source, only a symbol is shown in the UI, and the same symbol is used for all LLM completions.

```lua
cmp.setup({
  -- only showing formatting option for brevity
  formatting = {
    fields = {'menu', 'abbr', 'kind'},
    format = require('lspkind').cmp_format({
      mode = "symbol",
      maxwidth = 50,
      show_labelDetails = true,
      ellipsis_char = '...',
      symbol_map = {
        Supermaven = "",
        Codeium = "",
        Copilot = "",
      },
    })
  },
})
```


### Randomize the Order of Completion Suggestions

To eliminate position bias (the tendency to select the first suggestion), I implemented a system to randomize the order in which completion suggestions appear. Rather than defining a fixed order of completion sources in the setup function, I dynamically shuffle them each time a new buffer is loaded:

```diff
 cmp.setup({
  -- only showing sources option for brevity
-  sources = cmp.config.sources({
-    { name = 'nvim_lsp' },
-    { name = 'luasnip' },
-    { name = 'path' },
-  }, {
-    { name = 'nvim-cmp-ts-tag-close' },
-    { name = "codeium" },
-    { name = 'copilot'},
-    { name = 'supermaven'},
-    { name = 'nvim_lua' },
-    { name = 'spell' },
-  }, {
-    { name = 'buffer' },
-  }),
-
 })
```

This approach ensures that no AI completion source consistently appears first in the suggestion list, which is crucial for unbiased evaluation:

```lua
-- Fisher-Yates shuffle algorithm to randomize suggestion order
-- This is critical for reducing selection bias - if suggestions always appeared
-- in the same order, I might unconsciously favor the first option
local function shuffle(t)
  local n = #t
  for i = n, 2, -1 do
    local j = math.random(1, i)
    t[i], t[j] = t[j], t[i]
  end
  return t;
end

vim.api.nvim_create_autocmd('BufReadPre', {
  callback = function(t)
    local sources = cmp.config.sources({
      { name = 'nvim_lsp' },
      { name = 'luasnip' },
      { name = 'path' },
    }, shuffle({
      { name = 'nvim-cmp-ts-tag-close' },
      { name = "codeium" },
      { name = 'copilot'},
      { name = 'supermaven'},
      { name = 'nvim_lua' },
      { name = 'spell' },
    }), {
      { name = 'buffer' },
    });
    cmp.setup.buffer { sources = sources }
  end
})
```


![Auto completion UI](/images/code-completion/ui.png)
Fig 1: Example of the auto completion UI showing showing suggestions for `this.objectUrl`

### Data Collection System

To objectively measure each LLM's performance, I built a comprehensive data collection system using SQLite. This system silently recorded every completion suggestion and user acceptance decision during my normal development activities:

```lua
-- Setup lsqlite3
-- Run `luarocks install lsqlite3` if needed
local status, sqlite3 = pcall(require, 'lsqlite3')

if status then
  local db_path = vim.fn.stdpath('data') .. '/completion-log.sqlite3'
  local db = sqlite3.open(db_path)

  -- Create a log table if it doesn't exist
  db:exec[[
    CREATE TABLE IF NOT EXISTS cmp_log (
      id INTEGER PRIMARY KEY,
      timestamp TEXT,
      event TEXT,
      multiline INTEGER,
      source_name TEXT,
      filetype TEXT
    )
  ]]

  print("Logging completions to " .. db_path)

  -- Prepare the insert statement
  local stmt = db:prepare("INSERT INTO cmp_log (timestamp, event, multiline, source_name, filetype) VALUES (?, ?, ?, ?, ?)")
  -- Close the database connection when Neovim exits
  vim.api.nvim_create_autocmd("VimLeavePre", {
    callback = function()
      stmt:finalize()
      db:close()
    end,
  })

  -- Table of source names to log
  local log_sources = {
    supermaven = true,
    codeium = true,
    copilot = true,
    cmp_ai = true
  }

  -- Function to log completion events
  local function log_completion(event, item, source_name, filetype)
    local multiline = item.textEdit and item.textEdit.newText and item.textEdit.newText:find("\n") and 1 or 0
    if log_sources[source_name] then
      stmt:reset()
      stmt:bind_values(os.date('%Y-%m-%d %H:%M:%S'), event, multiline, source_name, filetype)
      stmt:step()
    end
  end


  -- Register event listeners
  cmp.event:on('menu_opened', function(evt)
    for _, entry in ipairs(evt.window.entries) do
      local source_name = entry.source.name
      log_completion("completion_suggested", entry:get_completion_item(), source_name, entry.context.filetype)
    end
  end)

  cmp.event:on('confirm_done', function(evt)
    log_completion("completion_used", evt.entry:get_completion_item(), evt.entry.source.name, evt.entry.context.filetype)
  end)
else
  print(sqlite3)
end
```

[SQLite is used](https://www.sqlite.org/whentouse.html) because it's lightweight, requires no server setup, and provides a self-contained database solution ideal for this type of data collection. Additionally, [Datasette](https://datasette.io/) can be used to easily query, visualize, and publish the data for later analysis.

The database schema tracked five key data points for each completion event:

```sql
CREATE TABLE IF NOT EXISTS cmp_log (
  id INTEGER PRIMARY KEY,
  timestamp TEXT,
  event TEXT,
  multiline INTEGER,
  source_name TEXT,
  filetype TEXT
)
```

The `timestamp` is the date and time the event was logged and is useful to see if the effectiveness of the LLMs changes over time. The `event` is the name of the `nvim-cmp` event that was fired. The `multiline` is a boolean value that is true if the completion is multiline. The `source_name` is the name of the source that provided the completion, for example, `Copilot`. The `filetype` is the filetype of the buffer that the completion was provided in.

The system tracked two primary event types:
1. **completion_suggested**: Recorded whenever a completion was offered
2. **completion_used**: Recorded when I accepted a completion

This approach allowed me to calculate the critical acceptance rate metric (accepted completions ÷ suggested completions) that forms the foundation of this analysis.

## Results

After collecting data for approximately four months across real-world development projects, clear patterns emerged in how these AI assistants performed in day-to-day coding scenarios.

The most immediate finding was Codeium's poor performance. It achieved only a 0.5% acceptance rate (meaning I accepted only 1 out of every 200 suggestions). This led me to discontinue it after seven days of evaluation.

### Quantitative Results

**Summary Statistics**

| LLM | Total Suggestions | Accepted Completions | Acceptance Rate | Avg Daily Suggestions | Avg Daily Acceptances |
| ----- |------------------- | ---------------------- | ---------------- | ---------------------- | ---------------------- |
| Supermaven | 60,508 | 1,171 | 1.9% | 293.3 | 5.7 |
| Copilot |27,395 | 921 | 3.4% | 133.5 | 4.5 |
| Codeium |7, 091 | 32 | 0.5% | 745.4 | 3.4 |

[Data here](https://completion-log-281060383488.europe-west1.run.app/completion-log?sql=WITH+daily_stats+AS+%28%0D%0A++SELECT+%0D%0A++++source_name+AS+LLM%2C%0D%0A++++COUNT%28*%29+AS+total_suggestions%2C%0D%0A++++SUM%28CASE+WHEN+event+%3D+%27completion_used%27+THEN+1+ELSE+0+END%29+AS+accepted_completions%2C%0D%0A++++%28JULIANDAY%28MAX%28timestamp%29%29+-+JULIANDAY%28MIN%28timestamp%29%29+%2B+1%29+AS+total_days%0D%0A++FROM+cmp_log%0D%0A++WHERE+event+IN+%28%27completion_suggested%27%2C+%27completion_used%27%29%0D%0A++GROUP+BY+source_name%0D%0A%29%0D%0ASELECT+%0D%0A++LLM%2C%0D%0A++total_suggestions+AS+%22Total+Suggestions%22%2C%0D%0A++accepted_completions+AS+%22Accepted+Completions%22%2C%0D%0A++ROUND%28%28accepted_completions+*+100.0%29+%2F+total_suggestions%2C+1%29+%7C%7C+%27%25%27+AS+%22Acceptance+Rate%22%2C%0D%0A++ROUND%28total_suggestions+%2F+total_days%2C+1%29+AS+%22Avg+Daily+Suggestions%22%2C%0D%0A++ROUND%28accepted_completions+%2F+total_days%2C+1%29+AS+%22Avg+Daily+Acceptances%22%2C%0D%0A++total_days%0D%0AFROM+daily_stats%0D%0AORDER+BY+total_suggestions+DESC%3B%0D%0A){:target="_blank"}

These statistics reveal two distinct AI assistant strategies:
- **Copilot**: Offers fewer but higher quality suggestions (3.4% acceptance)
- **Supermaven**: Provides more aggressive suggestions with lower precision (1.9% acceptance)

**Daily Used Count**
[![daily used count chart](/images/code-completion/daily-used-count.svg)](/images/code-completion/daily-used-count.svg){:target="_blank" title="Click to open SVG in new tab for zooming"}
*Fig 1: Daily count of accepted completions by LLM. Note Supermaven's consistently higher daily usage (average: 5.7 completions) compared to Copilot (average: 4.5 completions).*

**Daily Suggested Count**
[![daily suggested count chart](/images/code-completion/daily-suggested-count.svg)](/images/code-completion/daily-suggested-count.svg){:target="_blank" title="Click to open SVG in new tab for zooming"}
*Fig 2: Daily count of suggested completions by LLM. Supermaven offers significantly more suggestions (an average of 293 per day) than Copilot (134 per day).*

**Daily Acceptance Ratio**
[![daily used over suggested ratio chart](/images/code-completion/daily-used-over-suggested-ratio.svg)](/images/code-completion/daily-used-over-suggested-ratio.svg){:target="_blank" title="Click to open SVG in new tab for zooming"}
*Fig 3: Daily acceptance ratio (accepted/suggested) by LLM. Copilot consistently maintains higher suggested to accepted completion ratios (3.4%), while Supermaven typically ranges between 0.1.5-0.2% of suggestions accepted.*

[View raw data here](https://completion-log-281060383488.europe-west1.run.app/completion-log?sql=WITH+daily_counts+AS+%28%0D%0A++++SELECT+%0D%0A++++++++DATE%28timestamp%29+AS+day%2C+%0D%0A++++++++source_name%2C+%0D%0A++++++++COUNT%28CASE+WHEN+event+%3D+%27completion_used%27+THEN+1+END%29+AS+daily_used_count%2C%0D%0A++++++++COUNT%28CASE+WHEN+event+%3D+%27completion_suggested%27+THEN+1+END%29+AS+daily_suggested_count%0D%0A++++FROM+cmp_log%0D%0A++++WHERE+source_name+%21%3D+%27cmp_ai%27%0D%0A++++GROUP+BY+day%2C+source_name%0D%0A%29%0D%0ASELECT+%0D%0A++++day%2C%0D%0A++++source_name%2C%0D%0A++++daily_used_count%2C%0D%0A++++daily_suggested_count%2C%0D%0A++++ROUND%28%0D%0A++++++++CAST%28daily_used_count+AS+FLOAT%29+%2F+NULLIF%28daily_suggested_count%2C+0%29%2C+%0D%0A++++++++5%0D%0A++++%29+AS+daily_used_suggested_ratio%2C%0D%0A++++SUM%28daily_used_count%29+OVER+%28%0D%0A++++++++PARTITION+BY+source_name+%0D%0A++++++++ORDER+BY+day+%0D%0A++++++++ROWS+BETWEEN+UNBOUNDED+PRECEDING+AND+CURRENT+ROW%0D%0A++++%29+AS+cumulative_used_count%2C%0D%0A++++SUM%28daily_suggested_count%29+OVER+%28%0D%0A++++++++PARTITION+BY+source_name+%0D%0A++++++++ORDER+BY+day+%0D%0A++++++++ROWS+BETWEEN+UNBOUNDED+PRECEDING+AND+CURRENT+ROW%0D%0A++++%29+AS+cumulative_suggested_count%2C%0D%0A++++ROUND%28%0D%0A++++++++CAST%28%0D%0A++++++++++++SUM%28daily_used_count%29+OVER+%28%0D%0A++++++++++++++++PARTITION+BY+source_name+%0D%0A++++++++++++++++ORDER+BY+day+%0D%0A++++++++++++++++ROWS+BETWEEN+UNBOUNDED+PRECEDING+AND+CURRENT+ROW%0D%0A++++++++++++%29+AS+FLOAT%0D%0A++++++++%29+%2F+NULLIF%28%0D%0A++++++++++++SUM%28daily_suggested_count%29+OVER+%28%0D%0A++++++++++++++++PARTITION+BY+source_name+%0D%0A++++++++++++++++ORDER+BY+day+%0D%0A++++++++++++++++ROWS+BETWEEN+UNBOUNDED+PRECEDING+AND+CURRENT+ROW%0D%0A++++++++++++%29%2C+0%0D%0A++++++++%29%2C+%0D%0A++++++++5%0D%0A++++%29+AS+cumulative_used_suggested_ratio%0D%0AFROM+daily_counts%0D%0AORDER+BY+day%2C+source_name%3B%0D%0A#g.mark=bar&g.x_column=day&g.x_type=ordinal&g.y_column=daily_suggested_count&g.y_type=quantitative&g.color_column=source_name){:target="_blank"}

### Performance by Language

Analysis of completions by file type revealed interesting patterns:

| Language | Most Accurate LLM | Acceptance Rate |
| -------- |------------------ | ---------------- |
| http | copilot | 8.3%|
| zsh | copilot | 5.9%|
| typescriptreact | copilot | 5.8%|
| css | copilot | 5.6%|
| handlebars | copilot | 5.0%|
| jsonc | copilot | 5.0%|
| lua | copilot | 4.4%|
| make | copilot | 3.9%|
| javascript.glimmer | copilot | 3.6%|
| JavaScript/TypeScript | copilot | 2.8%|
| conf | copilot | 2.4%|
| yaml | supermaven | 1.4%|
| git* | supermaven | 1.0%|
| markdown | supermaven | 0.9%|

[Data here](https://completion-log-281060383488.europe-west1.run.app/completion-log?sql=WITH+llm_stats+AS+%28%0D%0A++SELECT+%0D%0A++++CASE+%0D%0A++++++WHEN+filetype+IN+%28%27javascript%27%2C+%27typescript%27%2C+%27javascriptreact%27%29+THEN+%27JavaScript%2FTypeScript%27%0D%0A++++++WHEN+filetype+LIKE+%27git%25%27+THEN+%27Git%27%0D%0A++++++ELSE+filetype%0D%0A++++END+AS+Language%2C%0D%0A++++source_name+AS+LLM%2C%0D%0A++++COUNT%28*%29+AS+total_suggestions%2C%0D%0A++++SUM%28CASE+WHEN+event+%3D+%27completion_used%27+THEN+1+ELSE+0+END%29+AS+accepted_completions%0D%0A++FROM+cmp_log%0D%0A++WHERE+event+IN+%28%27completion_suggested%27%2C+%27completion_used%27%29%0D%0A++AND+source_name+%21%3D+%27cmp_ai%27++--+Exclude+cmp_ai%0D%0A++GROUP+BY+Language%2C+source_name%0D%0A%29%2C%0D%0Aranked_llms+AS+%28%0D%0A++SELECT+%0D%0A++++Language%2C%0D%0A++++LLM+AS+%22Most+Accurate+LLM%22%2C%0D%0A++++ROUND%28%28accepted_completions+*+100.0%29+%2F+total_suggestions%2C+1%29+AS+acceptance_rate%2C%0D%0A++++RANK%28%29+OVER+%28PARTITION+BY+Language+ORDER+BY+%28accepted_completions+*+100.0%29+%2F+total_suggestions+DESC%29+AS+rnk%0D%0A++FROM+llm_stats%0D%0A%29%0D%0ASELECT+Language%2C+%22Most+Accurate+LLM%22%2C+acceptance_rate+%7C%7C+%27%25%27+AS+%22Acceptance+Rate%22%0D%0AFROM+ranked_llms%0D%0AWHERE+rnk+%3D+1+AND+acceptance_rate+%3E+0++--+Ignore+0.0%25+acceptance+rates%0D%0AORDER+BY+acceptance_rate+DESC%3B%0D%0A){:target="_blank"}



## Conclusion

Out of Codeium, Copilot, and Supermaven, Codeium is the clear loser the lowest acceptance rate (0.5%).

Supermaven provided the highest volume of useful completions (1,171 accepted suggestions) but at the cost of efficiency - it generated 60508 total suggestions, meaning 98.1% of its completions were rejected.

Copilot demonstrated the best balance of quality and quantity - with 921 accepted completions from just 27395 suggestions, achieving a 3.4% acceptance rate. More than twice the acceptance rate of Supermaven.


### Limitations and Future Work

While this study provides valuable insights, several limitations should be acknowledged:

1. **Single-user perspective** - Results primarily reflect my personal coding style, projects, and preferences
2. **Project-specific patterns** - Performance likely varies across different codebases and domains
3. **Temporal factors** - AI models are continuously improving; results may not reflect current capabilities
4. **Commercial focus** - Only commercial LLMs were tested; promising open-source models weren't evaluated
5. **Data collection gaps** - There were two notable gaps in data collection:
   - Eight days in July with no Copilot data (likely due to authentication expiration)
   - A period between October and December when I transitioned to a new development machine

Based on these findings and limitations, I've identified several opportunities for future work:

1. **Expanded model comparison**: Integrate additional code completion tools, particularly open-source models via plugins like [cmp-ai](https://github.com/tzachar/cmp-ai) or [minuet-ai.nvim](https://github.com/milanglacier/minuet-ai.nvim).

2. **Methodological refinements**: Switch from the `menu_opened` event to `menu_closed` for asynchronous completions to ensure all displayed suggestions are properly captured. This would provide a more accurate picture of what completions were actually seen and evaluated.

3. **Multi-user study**: Expand data collection to include other developers to better understand how these tools perform across different coding styles and domains.

## Summary

This blind trial of LLM code completions revealed significant differences between the three tested models:

- **Codeium**: Discontinued after 7 days due to poor performance (0.5% acceptance rate)
- **GitHub Copilot**: Highest quality suggestions (3.4% acceptance rate)
- **Supermaven**: Highest volume of accepted suggestions but lower efficiency (1.9% acceptance rate)

The findings demonstrate that different LLMs have distinct strengths - Copilot excels at providing precise, focused suggestions that are more likely to be accepted, while Supermaven offers more aggressive suggestions covering a wider range of possibilities.

The quantitative approach revealed patterns that wouldn't be obvious from casual usage, such as the significant difference in acceptance rates between the tools (Copilot's 3.4% vs. Supermaven's 1.9%).

Based on these results, I've adopted a hybrid approach in my development workflow:
1. Continue using Copilot for auto complete.
2. Trial paid for Supermaven to see if this improves the acceptance rate.
3. Test new models to ensure I'm using the most effective tools

This experiment demonstrates the value of data-driven decision making when selecting developer tools, and I look forward to expanding this methodology to evaluate additional LLM-based coding assistants in the future.
