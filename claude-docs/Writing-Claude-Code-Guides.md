# Writing Claude Code Guides — a House Style & Template

A practical guide to writing **guides about using Claude Code**, in the style of *Building (and Enhancing) RepRapFirmware with Claude Code*. Its job is to give every project a shared shape and voice, so a first draft produced in one project reads like it belongs alongside all the others.

This is itself written in the style it describes, so it doubles as a worked example. If you are a project being asked to draft a guide, read this first, then follow the **skeleton** and **checklist** at the end.

## The core idea

Every guide in this collection teaches the same paradigm: **you describe what you want in plain English, and Claude works out the specifics.** The reader's job is to state intent clearly, approve steps, and verify the result — not to memorise commands, versions, or file layouts.

So the guides are written as *things to ask Claude*, not as command sequences to type. Wherever you would be tempted to write "run this command", prefer "ask Claude to…" and show the prompt. Commands still appear — but as something Claude runs and explains, or as the one or two things only a human can do (graphical installers, editing a PATH, flashing hardware).

---

## Who these guides are for

- **Non-specialists doing a real task.** The reader is capable and motivated but need not be a programmer. Assume no developer tools installed at the start.
- **People who want a reproducible result**, not a tutorial to skim. They will follow along on their own machine and hardware.
- **Both macOS and Windows** where the task allows it — call out platform differences explicitly rather than assuming one.

Write for that reader throughout: second person ("you"), plain English, and honest about what is slow, risky, or fiddly.

---

## House style (voice & principles)

- **Describe intent; let Claude handle specifics.** Reinforce this framing in every section. The value proposition is that the reader doesn't need to know the versions, branches, file locations, or error fixes.
- **Show prompts as `>` quoted blocks.** The exact words the reader gives Claude go in a blockquote, often as a multi-part numbered list. Follow a prompt with a short "**Claude will…**" bullet list saying what happens.
- **Bold the load-bearing words** — key terms on first use, warnings, and the lead-in label of a bulleted tip (e.g. "**State the interface precisely** — …").
- **Be a real-world write-up.** Say it was actually done end-to-end. Include the genuine gotchas you hit; they are the most valuable part.
- **Link to authoritative docs instead of reproducing detailed procedures.** For anything the vendor documents and maintains (installers, recovery tools, API references), link to the official page rather than copying steps that will go stale. Reproduce steps only where they are Claude-specific or where no good source exists.
- **Add real-world hardware/platform caveats** from your own experience — the "you need a Windows machine for this board, a Mac is fine for that one" details that only come from doing it.
- **Trim redundancy.** After replacing a procedure with a link, remove the now-orphaned links and caveats that only supported it.
- **Prefer verification over trust.** Show the reader how to confirm each result (a size comparison, a version string, an expected output), so success is observable.

---

## Formatting conventions

- **Prompts:** blockquote (`>`). Nested prompts inside a callout use `>>`.
- **Callouts / gotchas:** a blockquote led by a bold phrase, e.g. `> **The one ordering rule that matters:** …`.
- **Code:** fenced blocks with a language tag — ` ```bash `, ` ```powershell `, ` ```json `, ` ```python `, and domain languages like ` ```gcode `.
- **Folder layouts:** an ASCII tree in a fenced block, with `# inline comments` explaining each entry.
- **Steps:** `## Step 0 — …`, `## Step 1 — …`. Step 0 is always "install Claude Code and open your project".
- **Troubleshooting entries:** `### "verbatim error text"` or `### Short symptom`, then a one-paragraph fix that usually ends by telling the reader Claude detects/fixes it automatically.
- **Emphasis on approval prompts:** include a ready-to-paste `.claude/settings.json` that pre-approves the safe, repetitive commands, and note that risky actions still prompt.

---

## The skeleton (copy this)

Use these sections in this order. Drop any that genuinely don't apply, but keep the spine — title, core idea, safety, prerequisites, Step 0, the work, verification, troubleshooting, why-it-works, resources.

```markdown
# <Building / Creating / Automating> <the thing> with Claude Code

<One or two sentences: what this is, which platforms, and the starting point
(e.g. "starting from a computer with no developer tools installed"). Bold the
key terms. Add: "This is a real-world write-up… You don't need to be a
programmer to follow it.">

## The core idea
<Restate the paradigm for this domain: what the reader describes vs. what Claude
works out. End with: "most instructions below are phrased as things to ask
Claude, not commands you type.">

---

## ⚠️ Read this first: safety and rollback
<Only if the task can damage hardware, lose data, or cost money. Bullet the real
risks, and give a concrete rollback / known-good plan before they start.>

---

## What you'll need
- <hardware / accounts / a Claude Code plan recommendation>
- <internet, disk space, realistic time for first run vs later runs>
Everything else is installed in Step 0/1 below.

> **Heads-up on permissions:** Claude asks approval before running commands…
> the "Reduce approval prompts" section near the end pre-approves the safe ones.

---

## Step 0 — Install Claude Code and open your project
1. Install from https://claude.com/claude-code (CLI, desktop, web, IDE).
2. Sign in. 3. Make an empty project folder and open it in Claude Code.
4. A good first message:
> I've saved "<this-guide>.md" in this folder. Please read it, check what I
> already have installed, and tell me the next step.

## Step 1 — <set up the toolchain / environment>
<Let Claude drive; flag the human-only bits. Split "### macOS" / "### Windows"
where they differ. Use a bold callout for the one thing people get wrong.>

## Step 2…N — <do the actual work>
<Each step: the goal, then the prompt as a `>` block, then a "Claude will…"
bullet list, then how to verify the result.>

## <Worked example> — <a concrete end-to-end task>
<A real feature/output, the actual prompt used, and the expected result the
reader can check (a command's output, a file, a screenshot).>

### Tips for good <domain> prompts
- **<Bold lead-in>** — <what and why>.

---

## Reduce approval prompts (optional settings file)
<Explain, then show a `.claude/settings.json` with an `allow` list tailored to
this task's safe, repetitive commands. Note risky actions still prompt.>

## Organising your files
<An ASCII folder tree with inline comments; naming suggestions.>

## Troubleshooting
### "<verbatim error>"
<One-paragraph fix; note Claude usually handles it.>

## <Emergency recovery / fallback>
<Link to the vendor's official recovery docs; add your platform caveats.>

---

## Why this works well with Claude Code
- **<Bold lead-in>** — <one line>.  (3–6 bullets restating the paradigm payoff.)

## Resources
- **<Name>:** <official link>
<Sign off with a short line, e.g. "Happy building!">
```

---

## Adapting the skeleton per domain

The RRF guide is a *firmware build* guide; other guides are different shapes but keep the same spine:

- **A "build a tool" guide** (e.g. a Chrome extension): Step 1 is scaffolding the extension, the "worked example" is one concrete feature (archive a thread, export to Markdown), verification is loading it unpacked and running it, and "Organising your files" describes the extension's folder layout.
- **A "create post-processors" guide** (e.g. GCode): the "worked example" is one processor end-to-end, verification is running it against a sample file and inspecting the output, and Troubleshooting covers the domain's common failures.

In every case, keep: intent-first framing, prompts as blockquotes, a verifiable worked example, a settings.json section, and honest real-world caveats.

---

## Checklist for a first-draft guide

Before handing a draft to the docs project for review, confirm:

- [ ] Title is action-first and names **Claude Code**.
- [ ] Opening states what it is, the platform(s), and the starting point.
- [ ] "The core idea" frames the task as intent-first.
- [ ] Safety/rollback section present if the task can do harm.
- [ ] "What you'll need" lists prerequisites with realistic time/space, plus a permissions heads-up.
- [ ] **Step 0 installs Claude Code and opens the project.**
- [ ] Instructions are phrased as prompts (`>` blocks), each with a "Claude will…" and a way to verify.
- [ ] At least one full **worked example** with the real prompt and expected output.
- [ ] Platform differences are split out (macOS / Windows) where relevant.
- [ ] Official docs are **linked**, not reproduced, for vendor procedures.
- [ ] A `.claude/settings.json` approval-reduction section is included.
- [ ] Troubleshooting uses per-symptom `###` headings.
- [ ] Real-world caveats from actually doing it are included.
- [ ] Ends with "Why this works well with Claude Code" + Resources + sign-off.

The docs project will review against this, apply the house editorial pass, and publish.
