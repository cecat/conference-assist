# conference-assist

Skills that let an AI agent help run a conference: watch a proposals form,
screen abstracts, curate per-track spreadsheets, track proposal status,
email track leaders, and chart registration against prior years.

**Status: specified, not built.** `SPEC.md` says what the agent must do and
`GRILLING.md` records how those requirements were arrived at. The `skills/`
directory is empty. Nothing here has run.

## What you need first

An AI agent that can reach Google — Gmail, Drive, Sheets, Docs and Forms —
and that can run on a schedule.

This repo does not set that up, and deliberately does not explain how. Two
routes that work:

- **[OpenClaw](https://openclaw.ai)** with
  [`gog`](https://github.com/openclaw/gogcli) registered as an MCP server.
  This is what the skills are written against.
- **A sandboxed pair of agents**, if you want the whole thing isolated in a
  VM: [`cecat/bootstrap`](https://github.com/cecat/bootstrap) builds that,
  and its `addons/google-openclaw` add-on does the Google half.

If your agent can already read one of your Google Sheets when you ask it
to, you have what you need.

## Applying the skills to an agent

### OpenClaw — supported, and the only one tested

OpenClaw reads skills from a handful of locations, highest precedence
first. The workspace one is what you want:

```
<workspace>/skills/          <- put them here
<workspace>/.agents/skills/
~/.agents/skills/
~/.openclaw/skills/          <- shared across agents
```

Find your agent's workspace — do not assume it:

```sh
openclaw config get agents.entries.<agent-id>.workspace
# falls back to agents.defaults.workspace, usually ~/.openclaw/workspace
```

Then copy the skills in:

```sh
cp -r skills/* "$(openclaw config get agents.defaults.workspace)/skills/"
```

Each skill is a directory containing a `SKILL.md` with frontmatter. The
skill is exposed under the `name` in that frontmatter, not the directory
name, so grouped folders like `skills/conference/triage/SKILL.md` are fine.

Then **confirm the agent can actually see them**, rather than assuming:

```sh
openclaw doctor --lint --all --deep
```

Doctor warns when a skill is allowed but unavailable in the current
runtime — a missing binary, an unset environment variable. That warning is
the useful output here.

Two traps worth knowing before you hit them:

- **Per-agent skill lists replace, they do not merge.** Setting
  `agents.entries.<id>.skills` to a list silently drops every skill not in
  it. Read the current value before you write one.
- **Enabled is not loaded.** A skill present and enabled may still not be
  in the agent's prompt. Ask the agent `/context list` in a chat and check
  what it reports — the only reading that proves what was injected.

### Hermes — a port, not yet done

[Hermes](https://hermes-agent.nousresearch.com) also has skills, and the
duties in `SPEC.md` are not OpenClaw-specific. But the mechanism differs
enough that calling this "supported" before testing it would be a lie:

| | OpenClaw | Hermes |
|---|---|---|
| Skills live in | `<workspace>/skills/` | `$HERMES_HOME/skills/<category>/<name>/` |
| Loaded by | workspace discovery | `hermes skills list`, plus `skills.auto_load` |
| Gotcha | per-agent lists replace defaults | enabled ≠ auto-loaded; an agent may never find the skill |
| Google via | `gog` as an MCP server | its own bundled `google-workspace` skill |

That last row is the real work: the skills in this repo will call Google
through whatever tool names their agent exposes, and those names differ
between the two. Porting means a translation layer or a second set of
skills, not a copy.

If you port it, a PR would be welcome — and please say which parts you
actually exercised.

## Making it yours

`SPEC.md` describes one conference's needs (a multi-track technical
programme, ~135 proposals a year, a light screen for sales pitches and
out-of-scope topics). The shape generalises; the specifics will not match
yours.

Two things to change before using it:

1. **The rubric.** What counts as out of scope is yours to define, and the
   agent applies it literally.
2. **The email boundary.** `SPEC.md` sets out which messages an agent may
   send unsupervised and which it must only draft. Read that section
   before you let anything send. A wrong rejection email cannot be unsent.

Conference-specific values — form ids, folder names, addresses — belong in
a local config file that is gitignored, never in the skills.

## A warning worth reading twice

These skills can give an agent the ability to **email conference
submitters on its own**.

Google publishes no OAuth scope that permits drafting but forbids sending:
creating a draft needs `gmail.modify`, and anything holding `gmail.modify`
can also send. So "the agent drafts, a human approves" **cannot be enforced
by the credential**. It is an instruction in the agent's own
personality file, and an agent that misreads it can mail a stranger.

If that is not acceptable, grant read-only Gmail and have the agent write
its proposed replies somewhere a human copies from.

## Layout

```
SPEC.md       what the agent must do; settled and open items separated
GRILLING.md   the full session that produced SPEC.md, verbatim
skills/       the skills themselves (empty)
```

Read `SPEC.md` before writing any skill. It has an *Open* section of eight
things that were never decided — the rubric's criteria, who moves a
proposal between states, what track leaders see, retention of submitter
data. Ask rather than inventing answers.
