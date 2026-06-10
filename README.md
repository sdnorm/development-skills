# development-skills

A central handbook of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for our team — the conventions, patterns, and tool-usage rules we follow when building apps. Existing and new projects point at this repo so Claude picks up our house rules automatically instead of every project having to redefine them in its own `CLAUDE.md`.

Update a skill here once and every project that links it gets the new guidance on the next pull.

These are **experimental**. Expect the descriptions and trigger language to change as we see what fires when.

## Skills

| Skill | Category | Status | When it triggers |
|-------|----------|--------|------------------|
| [rails](skills/rails/SKILL.md) | Conventions | 🧪 experimental | Anytime Claude is modifying or scaffolding Rails code — concerns over service objects, Hotwire/Turbo, Stimulus, Minitest with fixtures, local CI via `gh-signoff`. |
| [anycable](skills/anycable/SKILL.md) | Infrastructure | 🧪 experimental | Setting up AnyCable as a Redis-free replacement for Action Cable (HTTP broadcasting + embedded NATS), including Railway deployment. |
| [rails-query](skills/rails-query/SKILL.md) | Tooling | 🧪 experimental | Answering data questions about a Rails 8.2+ app via the `rails query` command — locally or remote via Kamal. Adapted from [lewispb/rails-query-skill](https://github.com/lewispb/rails-query-skill). |
| [hatchbox-sqlite](skills/hatchbox-sqlite/SKILL.md) | Deployment | 🧪 experimental | Deploying a Rails app to Hatchbox with SQLite — put the DB file under `shared/` so it survives deploys. Single-server only. |

## Using these skills

Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`) plus the body. Pick whichever consumption method fits how you work:

### Per-project (recommended)

Clone or submodule this repo somewhere stable, then symlink the skills you want into a project's `.claude/skills/` directory:

```bash
git clone git@github.com:<your-org>/development-skills.git ~/src/development-skills

cd your-rails-project
mkdir -p .claude/skills
ln -s ~/src/development-skills/skills/rails       .claude/skills/rails
ln -s ~/src/development-skills/skills/rails-query .claude/skills/rails-query
```

Claude Code auto-discovers anything under `.claude/skills/` and fires it based on its `description`.

### User-wide

Symlink into `~/.claude/skills/` to make a skill available in every project:

```bash
ln -s ~/src/development-skills/skills/rails ~/.claude/skills/rails
```

### Ad-hoc

Drop the link and just `@`-mention the file in a Claude Code conversation when you want it loaded for that session.

## External skill collections

Skill collections maintained outside this repo that complement our house rules:

| Collection | What it covers | How to use |
|------------|----------------|------------|
| [marckohlbrugge/37signals-skills](https://github.com/marckohlbrugge/37signals-skills) | Rails in the 37signals style, extracted from their open-source Campfire and Fizzy codebases and DHH's code reviews — core conventions, Hotwire/Turbo, jobs, migrations, security/multitenancy, testing, webhooks, plus a `/dhh` code-review skill. Also includes a `guide/` directory of 35+ deeper reference docs. | Clone and copy or symlink the skills you want into `.claude/skills/` (per-project) or `~/.claude/skills/` (user-wide), same as the methods above. |

Where an external skill overlaps with one of ours (e.g. its `rails-best-practices-core` vs our [rails](skills/rails/SKILL.md)), our skill wins for team conventions — treat the external one as supplementary reference.

## Adding a new skill

1. Create `skills/<short-name>/SKILL.md`.
2. Add YAML frontmatter:
   ```yaml
   ---
   name: <short-name>
   description: |
     One paragraph. Lead with what the skill does, then list the trigger conditions
     ("Use when ...") so Claude can decide when to load it.
   ---
   ```
3. Add a row to the table above.
4. Default the **Status** column to 🧪 experimental until you've used it on real work without surprises.

If a skill needs supporting files (templates, scripts, examples), put them in the same `skills/<short-name>/` directory.

## License

MIT — see [LICENSE](LICENSE). Adapted skills retain attribution to their upstream sources inside the skill file.
