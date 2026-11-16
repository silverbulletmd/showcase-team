The team wiki’s home page: the current sprint, open tasks per person and where our decisions stand, plus the open tasks shown at the top of every person page. The space opens in dark mode, with the [[Library/Team Theme]] theme.

```space-lua
team = {}

-- Widget links open pages through editor.navigate, so they work wherever the space is served
local function go(ref)
  return function()
    editor.navigate(ref)
  end
end

local function avatar(handle)
  return dom.span {
    class = "team-avatar team-" .. handle,
    handle:sub(1, 1):upper(),
  }
end

-- Whether a task is addressed to an identity, written as "@hana"
local function addressedTo(task, target)
  return table.find(task.recipients, function(r)
    return r == target
  end) ~= nil
end

local MONTHS = { "January", "February", "March", "April", "May", "June", "July",
  "August", "September", "October", "November", "December" }

-- "2026-10-09" as "9 October"
local function longDate(iso)
  local _, m, d = tostring(iso):match "^(%d+)-(%d+)-(%d+)$"
  if not m then
    return tostring(iso)
  end
  return tonumber(d) .. " " .. MONTHS[tonumber(m)]
end

local function plural(n, word)
  return n .. " " .. word .. (n == 1 and "" or "s")
end

-- A terminal-style status line: open tasks, the last weekly sync and decisions still waiting
function team.header()
  local meetings = query[[
    from p = index.contentPages "meeting"
    order by p.date desc
    limit 1
    select p.name
  ]]
  local meeting = meetings[1]
  local open = query[[
    from t = index.tasks()
    where not t.done and t.recipients
    select t
  ]]
  local waiting = query[[
    from p = index.contentPages "adr"
    where p.status == "proposed"
    select p
  ]]
  return widget.htmlBlock(dom.div {
    class = "team team-header",
    dom.div {
      class = "team-prompt",
      dom.span { class = "team-host", "driftwood" },
      dom.span { ":~$ status" },
      dom.span { class = "team-cursor", "_" },
    },
    dom.div {
      class = "team-pills",
      dom.span {
        class = "team-pill",
        plural(#open, "open task"),
      },
      dom.a {
        class = "team-pill",
        onclick = go(meeting),
        "last sync " .. meeting:match "%d+%-%d+%-%d+",
      },
      dom.a {
        class = "team-pill team-pill-warn",
        onclick = go "ADR",
        plural(#waiting, "decision") .. " waiting",
      },
    },
  })
end

function team.sprint(page)
  local tasks = query[[
    from t = index.tasks()
    where t.page == page
    select t
  ]]
  local done = 0
  local rows = {}
  for _, t in ipairs(tasks) do
    if t.done then
      done = done + 1
    end
    table.insert(rows, dom.a {
      class = t.done and "team-task team-done" or "team-task",
      onclick = go(t.ref),
      avatar(t.text:match "@(%w+)"),
      dom.span { (t.text:gsub("@%w+ ", "")) },
    })
  end
  local progress = #tasks > 0 and math.max(4, math.floor(100 * done / #tasks)) or 0
  local ends = query[[
    from p = index.pages()
    where p.name == page
    select p.ends
  ]]
  ends = ends[1]
  return widget.htmlBlock(dom.div {
    class = "team team-sprint",
    dom.div {
      class = "team-sprint-head",
      dom.a {
        onclick = go(page),
        page:match "[^/]+$",
      },
      dom.span { done .. " of " .. #tasks .. " done" .. (ends and (" · ends " .. longDate(ends)) or "") },
    },
    dom.div {
      class = "team-bar",
      dom.div { style = "width:" .. progress .. "%" },
    },
    dom.div {
      class = "team-tasks",
      table.unpack(rows),
    },
  })
end

-- Open tasks addressed to an identity, written as "@hana"
function team.openTasks(id)
  local target = "@" .. id
  return query[[
    from t = index.tasks()
    where not t.done and addressedTo(t, target)
    select templates.taskItem(t)
  ]]
end

-- Above every page with an `identity`: that person's open tasks, wherever they were assigned
view.define {
  name = "team.openTasks",
  title = "Open tasks",
  dock = "page-top",
  frame = "minimal",
  supportedDocks = { "page-top", "page-bottom", "rhs" },
  refreshOn = { "navigate", "index" },
  content = function()
    local id = (editor.getCurrentPageMeta() or {}).identity
    if not id then
      return nil
    end
    local tasks = team.openTasks(id)
    if #tasks == 0 then
      return nil
    end
    return "**Open tasks for @" .. id .. "**\n" .. table.concat(tasks)
  end,
}

-- Open tasks per team member: pages tagged #team, matched to tasks by their `identity` (e.g. hana, addressed as @hana)
function team.people()
  local open = query[[
    from t = index.tasks()
    where not t.done and t.recipients
    select t
  ]]
  local members = query[[
    from p = index.contentPages "team"
    order by p.name
    select p
  ]]
  local cards = {}
  for _, p in ipairs(members) do
    local n = 0
    for _, t in ipairs(open) do
      if addressedTo(t, "@" .. p.identity) then
        n = n + 1
      end
    end
    table.insert(cards, dom.a {
      class = "team-person",
      onclick = go(p.name),
      avatar(p.identity),
      dom.span { p.name:match "[^/]+$" },
      dom.span { class = "team-load", string.rep("■", n) },
      dom.strong { tostring(n) },
    })
  end
  return widget.htmlBlock(dom.div {
    class = "team team-people",
    table.unpack(cards),
  })
end

function team.decisions()
  local adrs = query[[
    from p = index.contentPages "adr"
    order by p.date desc
    select p
  ]]
  local counts, waiting = {}, nil
  for _, a in ipairs(adrs) do
    counts[a.status] = (counts[a.status] or 0) + 1
    if a.status == "proposed" then
      waiting = a.name
    end
  end
  local chips = {}
  for _, status in ipairs { "accepted", "proposed", "superseded", "deprecated" } do
    table.insert(chips, dom.span {
      class = "team-chip team-chip-" .. status,
      (counts[status] or 0) .. " " .. status,
    })
  end
  return widget.htmlBlock(dom.div {
    class = "team team-decisions",
    dom.div {
      class = "team-chips",
      table.unpack(chips),
    },
    waiting and dom.p {
      "Waiting on a decision: ",
      dom.a {
        onclick = go(waiting),
        (waiting:gsub("^ADR/", "")),
      },
    } or nil,
  })
end
```

```space-style
/* Let the home widgets sit on the page without the usual widget frame */
.sb-lua-directive-block:has(.team) {
  border: none !important;
  background: none !important;
}

/* Cards: hairline frame, soft accent header band */
.team-header,
.team-sprint {
  border: 1px solid var(--ui-surface-border-color);
  border-radius: 12px;
  overflow: hidden;
}

/* The status line */
.team-header {
  padding: 14px 16px;
  background: var(--ui-accent-surface-color);
}
.team-prompt {
  font-family: var(--editor-monospace-font);
  font-size: 1.15em;
  font-weight: 600;
}
.team-host {
  color: var(--ui-accent-text-color);
}
.team-cursor {
  margin-left: 4px;
  color: var(--ui-accent-color);
  animation: team-blink 1s steps(1) infinite;
}
@keyframes team-blink {
  50% { opacity: 0; }
}

/* Pills: accent-soft, warning tone for what needs attention */
.team-pills,
.team-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.team-pills {
  margin-top: 10px;
}
.team-pill,
.team-chip {
  padding: 2px 10px;
  border-radius: 99px;
  background: var(--tone-accent-soft);
  color: var(--ui-accent-text-color) !important;
  font-size: 0.8em;
  font-weight: 500;
  text-decoration: none !important;
}
a.team-pill:hover {
  background: color-mix(in srgb, var(--ui-accent-color) 30%, transparent);
}
.team .team-pill-warn,
.team-chip-proposed {
  background: var(--tone-warning-soft);
  color: var(--tone-warning) !important;
}
a.team-pill-warn:hover {
  background: color-mix(in srgb, var(--tone-warning) 30%, transparent);
}
.team-chip-superseded {
  background: var(--tone-neutral-soft);
  color: var(--tone-neutral) !important;
}
.team-chip-deprecated {
  background: var(--tone-danger-soft);
  color: var(--tone-danger) !important;
}

/* The current sprint: header band, progress bar and its tasks */
.team-sprint-head {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  padding: 10px 16px;
  background: var(--ui-accent-surface-color);
  border-bottom: 1px solid var(--tone-accent-soft);
}
.team-sprint-head a {
  font-size: 14px;
  font-weight: 650;
  color: var(--ui-accent-text-color) !important;
  text-decoration: none !important;
}
.team-sprint-head a:hover {
  text-decoration: underline !important;
}
.team-sprint-head > span {
  font-size: 13px;
  color: var(--subtle-color);
}
.team-bar {
  height: 6px;
  margin: 14px 16px 8px;
  border-radius: 99px;
  background: var(--tone-accent-soft);
  overflow: hidden;
}
.team-bar div {
  height: 100%;
  border-radius: 99px;
  background: var(--ui-accent-color);
}
.team-tasks {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding: 0 8px 10px;
}
.team-task {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 5px 8px;
  border-radius: 8px;
  font-size: 0.9em;
  color: var(--root-color) !important;
  text-decoration: none !important;
}
.team-task:hover {
  background: var(--ui-accent-surface-color);
}
.team-task::before {
  content: "";
  flex: none;
  width: 0.9em;
  height: 0.9em;
  border: 1.5px solid color-mix(in srgb, var(--ui-accent-color) 55%, var(--root-background-color));
  border-radius: 50%;
}
.team-done::before {
  background: var(--ui-accent-color);
  border-color: var(--ui-accent-color);
}
.team-done span:last-child {
  color: var(--subtle-color);
  text-decoration: line-through;
}

/* Avatars: one color per person */
.team-avatar {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  color: #0b1514;
  font-size: 0.72em;
  font-weight: 700;
}
.team-sam {
  background: #2dd4bf;
}
.team-pete {
  background: #60a5fa;
}
.team-jo {
  background: #f472b6;
}
.team-jane {
  background: #a78bfa;
}
.team-mateo {
  background: #fb923c;
}
.team-hana {
  background: #a3e635;
}

/* Open tasks per person, as a load bar */
.team-people {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 6px;
}
.team-person {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 12px;
  border: 1px solid var(--ui-surface-border-color);
  border-radius: 10px;
  color: var(--root-color) !important;
  text-decoration: none !important;
  font-size: 0.9em;
}
.team-person:hover {
  background: var(--ui-accent-surface-color);
  border-color: var(--tone-accent-soft);
}
.team-person > span {
  white-space: nowrap;
}
.team-load {
  margin-left: auto;
  color: color-mix(in srgb, var(--ui-accent-color) 70%, transparent);
  letter-spacing: 1px;
}
.team-person strong {
  width: 1.5em;
  text-align: right;
  color: var(--ui-accent-text-color);
  font-variant-numeric: tabular-nums;
}

/* Where decisions stand, and the one still waiting */
.team-decisions p {
  margin: 10px 0 0;
  font-size: 0.9em;
}

/* Widget links navigate on click, so give them a pointer */
.team a {
  cursor: pointer;
}
```
