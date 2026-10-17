# Agentic Coding Playbook

## Context rules

- ~100k tokens is the smart zone (a rule of thumb, not a hard limit). Stay in here at all times.
- `/clear` clears the current context.
- `/handoff` writes a handoff doc to give to a new session, if we are too deep into the dumb zone. Prefer this over `/compact`, since compacting is lossy.
- `/context` shows how full the context currently is.
- Keep `CLAUDE.md` or `AGENTS.md` as small as possible, since it gets loaded into every session.

## Phase 0: Setup

1. Install the skills in Claude Code:
   ```bash
   claude plugin install mattpocock-skills@claude-plugins-official
   ```
   The plugin will update itself. The alternative, skills.sh (`npx skills@latest add mattpocock/skills`), copies editable files into the project that we update by hand. Pick one per agent, because installing both gives you every skill twice.
2. Run `/setup-matt-pocock-skills` once per repo. It asks which issue tracker we want to use (GitHub, GitLab, local files, or anything else you describe), what labels we apply to tickets when triaging, and where to save any docs it creates. If we want an `issues/` folder, pick local files and tell it to use `issues/`. The `prompt.md` in step 3 reads from `@issues/`, so the two must match.
3. Add the Ralph files to the repo (see below). These are not part of the plugin, so we add them to each repo ourselves.

### File layout

Everything sits at the repo root, next to `package.json` / `.git`:

```
your-repo/
├── prompt.md        # repo root: the prompt Ralph runs each iteration
├── progress.txt     # repo root: Ralph's log, one entry per ticket
├── issues/          # repo root: tickets, written by /to-tickets in Phase 4
└── ralph/           # repo root: the loop scripts
    ├── once.sh
    └── afk.sh
```

Always run the scripts from the repo root (`./ralph/once.sh`), since `prompt.md`, `@issues/` and `@progress.txt` are all resolved from there.

### `prompt.md` (repo root)

```markdown
Read the tickets in @issues/ and @progress.txt.

1. Pick the highest-priority unblocked AFK ticket that isn't done.
2. Implement it using the /tdd skill (red-green-refactor).
3. Run tests and type checks. Fix until green.
4. Mark the ticket done and append what you did to progress.txt.
5. Commit with a conventional-commit message.
   ONLY WORK ON A SINGLE TICKET.
   If no AFK tickets remain, output <promise>COMPLETE</promise>.
```

### `ralph/once.sh`

Tackles one ticket at a time. We watch, check the commit, and run it again.

```bash
#!/bin/bash
claude --permission-mode acceptEdits "$(cat prompt.md)"
```

### `ralph/afk.sh`

Loops over the tickets unattended. Requires Docker Desktop 4.50+; the sandbox isolates the agent from our machine.

```bash
#!/bin/bash
set -e
if [ -z "$1" ]; then echo "Usage: $0 <iterations>"; exit 1; fi

for ((i=1; i<=$1; i++)); do
  result=$(docker sandbox run claude --permission-mode acceptEdits -p "$(cat prompt.md)")
  echo "$result"
  if [[ "$result" == *"<promise>COMPLETE</promise>"* ]]; then
    echo "Done after $i iterations."
    exit 0
  fi
done
```

### Create the files

From the repo root:

```bash
mkdir -p ralph issues
# paste the contents above into prompt.md, ralph/once.sh and ralph/afk.sh
touch progress.txt
chmod +x ralph/*.sh
```

Commit `prompt.md` and the `ralph/` scripts. `progress.txt` is per sprint (see Phase 5).

Alternative without scripts: the official plugin, `/plugin install ralph-wiggum@claude-plugins-official`, then `/ralph-loop "task" --max-iterations 20`.

## Phase 1: Research and prototyping

We gather all relevant info of the feature we are supposed to do and turn it into an `.md` file that we keep in the repo for this specific piece of work.

- `/research` investigates and captures the findings as a cited `.md` file in the repo, if needed.
- `/prototype` builds a throwaway prototype to answer a design question.

## Phase 2: The grill session

- **In a codebase:** use `/grill-with-docs @spec.md`. It's a grilling session that also builds a project's domain model, sharpening terminology and updating `GLOSSARY.md` and ADRs inline. Matt calls the shared glossary possibly the coolest technique in the repo, so this is the one to use for real projects.
- **No codebase:** use `/grill-me`.

> **IMPORTANT:** Phases 2, 3 and 4 must happen in the same session. Do NOT `/clear` or compact until the tickets are written, so the grilling, spec and tickets all build on the same thinking.

## Phase 3: Writing the spec (formerly PRD)

We use the `/to-spec` skill (formerly `write-a-prd` / `to-prd`) to turn the grilling session into a spec. There is no interview here, it just summarises what we already discussed. We do NOT need to read it closely, since we already aligned during the grill session.

## Phase 4: Slicing work into tickets

We now use the `/to-tickets` skill (formerly `prd-to-issues` / `to-issues`) to slice the spec into tickets. They are saved wherever we configured in Phase 0, so `issues/` if we set that up.

The tickets must be verified. What is important here is that tickets are not only horizontal work, eg: only DB, only service layer, only UI. We need each ticket to be as vertical as possible. Matt calls these "tracer bullets": a thin end-to-end slice (schema, API, UI, tests).

Each ticket is tagged **HITL** (needs us present) or **AFK** (safe to run unattended), and declares which tickets block it.

Review the tickets, and tell the LLM to fix anything worth fixing. Push back on horizontal tickets, and on tickets marked AFK that really need our judgment.

## Phase 5: Implement with AI

We should start a new session now, since we are done with the planning phase. Each Ralph iteration also starts with fresh context, which keeps us in the smart zone.

We can now turn the tickets into code, either by Ralph looping over them, or by using `/implement` on one ticket at a time (one fresh session per ticket). `/implement` runs `/tdd` and finishes with `/code-review` before committing.

Ralph is just a bash loop that runs the same prompt over and over, using the `prompt.md` and `ralph/` scripts we added in Phase 0. The agent picks the next ticket, implements it, commits, and logs what it did in `progress.txt`.

`/tdd` comes from the same mattpocock/skills repo. It makes the agent write a failing test first, then make it pass (red-green-refactor). It should ALWAYS be used by the agents.

### Running Ralph

From the repo root:

1. Make sure `progress.txt` exists (`touch progress.txt` if this is a new sprint).
2. Start with `./ralph/once.sh` until we trust the prompt. Watch it, check the commit, run it again.
3. Switch to `./ralph/afk.sh <iterations>` to loop unattended. Always cap the iterations, eg: `./ralph/afk.sh 20`, as a guardrail, so it runs at most 20 times. Pick roughly the number of AFK tickets plus a small buffer for retries.
4. If `afk.sh` hits the cap before finishing, check `progress.txt` and the commits to see why before running it again.

Delete `progress.txt` when the sprint is done.

## Phase 6: Human-in-the-loop review

- We do the HITL tickets ourselves, together with the agent.
- Review the commit from each Ralph iteration. The git history plus `progress.txt` is our audit trail.
- Run `/code-review`. It checks whether the code follows our coding standards, and whether it actually implements the ticket.
- Run `/retro` after a session to get suggestions for improving the agent's setup (checks, coding standards, `CLAUDE.md`, tooling). Fix those things and the next loop goes better.

## Phase 7: Deployment and monitoring

This phase is not really part of Matt's workflow, so we define it ourselves.

- Use the `/pr` skill to write the PR description. It includes before/after evidence and how risky the merge is.
- Use the `/wizard` skill to generate a step-by-step script for things only we can do, eg: infrastructure, credentials, CI secrets.
- Our CI/CD is the real gate. The same tests and type checks the loop runs should block the merge.

---

Not sure which skill fits a situation? Use `/ask-matt`.
