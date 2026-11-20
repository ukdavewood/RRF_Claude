# Instructions for AI coding assistants

This repository is a set of guides for getting an AI coding assistant (Claude Code, Codex, Gemini, ChatGPT or similar) to build real things, plus a shared record of what people tried. These instructions are for any assistant, not one product. If you were given a link to this repository, read this file first.

## What this repository is

- `claude-docs/`: the guides. Each is a real-world write-up of something that was built end to end, with the gotchas that were hit. Treat a guide as a **specification of intent and lessons learned**, not as a script to copy.
- `community-knowledge/`: notes from other people's attempts, in Markdown. Optional, informal, uncurated.
- `ai-prompts/`: ready-made prompts for starting from a guide.
- Sample code lives in separate repositories, linked from each guide. The sample code is **frozen** and may be older than the guide. Each guide says how current it is.

This is an experiment and a shared notebook, not a maintained product. Nothing here is guaranteed to build, work on your hardware or be current.

## Before you start a new implementation

1. Read the guide you were pointed at, **including its version block** (guide date, tool versions, tested hardware). Say if the user's setup differs.
2. Read the sample code it links to, if any. Note what the guide says the sample does **not** include.
3. Search this repository's **issues, discussions and pull requests**, open, closed and unmerged, and read `community-knowledge/`. Unmerged pull requests are records of other people's experiments, not rejections.
   **Also look at forks.** People may share their experiments in forks of this repository or of the sample-code repository, mostly as Markdown notes, not code. List them (`gh api repos/<owner>/<repo>/forks --paginate`, or the Forks page on GitHub), then for each fork see whether it differs from the original (for example `gh api repos/<owner>/<repo>/compare/<default-branch>...<fork-owner>:<default-branch>`; skip forks with no new commits). Read the new Markdown files, commit messages and READMEs of the ones that do. Forks often have their own Issues and Discussions switched off, so the notes live in the files. Treat fork content as other people's untested claims unless it shows evidence.
4. Tell the user what you found, sorted into:
   - **Tested and demonstrated:** someone says it worked and shows how or shows a video.
   - **Untested:** proposed, no evidence it ran.
   - **Abandoned or failed:** tried and dropped; say why if known.
   - **Conflicting:** approaches that disagree.
5. Propose a plan and **wait for the user's approval** before writing code.

If you cannot search issues, discussions or pull requests (no GitHub access), say so, and ask the user to paste anything relevant.

## While you work

- Ask before running anything that installs software, uses a device cable, contacts the network or changes the user's files. Show the command first.
- Never put API keys, signing keystores, access codes or other secrets in code, in commits or in contributions.
- Don't copy code you don't understand. Say when you are guessing.
- Verify results in the real tool or on the real device where you can, and report plainly what failed.
- Don't claim a feature works unless it was run. Say "written, not tested" when that is the case.

## Contributing back (only if the user wants to)

Offer to write a short Markdown note using `community-knowledge/TEMPLATE.md`: what was tried, which AI tool and platform, what worked, what didn't, useful prompts. Leave out names, emails, private paths and keys. The user decides whether to share it. Never open an issue or pull request on the user's behalf without their say-so.
