# Building SolidCraft for iPad with Claude Code

You have an open-source desktop CAD program written in Rust and you want it running natively on your iPad, with an AI assistant that can drive it by voice or text. This guide shows how to get there by asking Claude Code for each piece: an iPad host app, touch and keyboard fixes, file access, and then an optional AI assistant with three tiers (Apple's on-device model, Claude online, and a small model that works offline).

This is a real-world write-up. It was done end to end on a Mac and an iPad Pro over about three days (25 to 27 October 2026), starting from the open-source [SolidCraft](https://github.com/tangency-labs/solidcraft) v0.18.0 source. You don't need to be a programmer, but you do need to be comfortable approving steps and testing on a real device.

> **Scope and caveats.** Written for **macOS with Xcode**, targeting an **iPad** (tested on an iPad Pro 12.9-inch, M2, iPadOS 26). **Not tested on iPhone, Windows or Linux** (building iOS apps needs a Mac). It is an independent, unofficial guide: not affiliated with or endorsed by the SolidCraft project (`tangency-labs`), Apple or Anthropic. The result is a personal, work-in-progress port, not a supported product. Timings and costs are from one setup, so treat them as examples.

## The core idea

You describe what you want in plain English, and Claude works out the specifics: which Rust crates to use, how to wrap them in an Xcode project, how to sign and install the app, and how to fix what breaks on a touch screen. Your job is to state intent clearly, approve steps, **test on the real iPad**, and report what you see ("the keyboard covers the box", "it looped"). Claude reads the logs and fixes the code.

Most instructions below are phrased as things to ask Claude, not commands you type.

---

## What you'll need

- **A Mac** with a recent **Xcode** (26 or later) and about 30 GB of free disk space (Rust, Xcode, build products).
- **An iPad** on iPadOS 26 or later, with a USB cable and Developer Mode switched on (Apple explains how: [Enabling Developer Mode on a device](https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device)).
- **An Apple ID.** A free account works but its app signing expires every **7 days**, so you reinstall weekly. The paid **Apple Developer Program** (about £79 / $99 a year) makes it a year.
- **Claude Code** on a plan you're happy with. The port itself was built with Claude Sonnet on a $20/month Pro plan.
- For the optional AI part: an **Anthropic API key** for the online tier (our testing, including many demo runs, cost about $0.14 with Claude Haiku), and about 2.3 GB of iPad storage for the offline model.
- The SolidCraft source, as a git repository on your Mac. Everything else is installed in Step 0 and Step 1.

> **Heads-up on permissions:** Claude asks before running commands, and iOS builds run a lot of them. The "Reduce approval prompts" section near the end pre-approves the safe, repetitive ones. Risky actions still prompt.

---

## Step 0 — Install Claude Code and open the project

1. Install Claude Code from <https://claude.com/claude-code> (command line, desktop, web or editor extension) and sign in.
2. Clone the SolidCraft source into a folder, and open that folder in Claude Code.
3. A good first message:

> I want to run this CAD program natively on my iPad. Please read the repository's README and AGENTS.md, check what I already have installed (Xcode, Rust, xcodegen), and tell me the next step. Don't change anything yet.

- **Claude will** read the project rules (this repo has strict ones: no `unwrap`, no `unsafe` outside listed modules, changes by pull request), check your tools and list what's missing.
- **Verify:** ask it to run `cargo --version` and `xcodebuild -version` and show you the output.

## Step 1 — Set up the toolchain

> Install anything missing for building an iOS app from Rust: the Rust target for a real iPad, xcodegen, and tell me which parts I have to do by hand.

- **Human-only bits:** installing Xcode from the App Store, signing in to your Apple ID in Xcode (Settings → Accounts), and switching on Developer Mode on the iPad.
- **Claude will** add the target (`rustup target add aarch64-apple-ios`, plus the simulator target if you want one) and install xcodegen with Homebrew.
- **Verify:** `rustup target list --installed` shows `aarch64-apple-ios`.

## Step 2 — Ask for a minimal iPad host

The cleanest route for an app that already uses eframe/egui is a small new crate that hosts the existing UI on iOS, and an Xcode project that wraps the Rust binary.

> Add a new crate `solidcraft-ios` that runs the existing SolidCraft UI on iPadOS using eframe on winit's iOS backend with wgpu on Metal. Keep it minimal. Then create an Xcode project (generated with xcodegen from a `project.yml`) that builds the Rust binary in a pre-build script and signs it as an app. Show me the plan first.

- **Claude will** propose the crate, an `Info.plist`, a `project.yml`, and a build script that compiles Rust for the device and places the executable inside the app bundle so Xcode signs it afterwards.
- **Verify:** it builds without errors for the iOS simulator first (ask for a simulator build). It will probably complain about graphics limits; see Troubleshooting.

> **The one ordering rule that matters:** let Xcode sign the bundle *after* the Rust binary is copied in. If the build reuses an old bundle, you get an unsigned or stale app. Claude's build script deletes the old bundle first.

## Step 3 — Build, sign and install on your iPad

> Write a script `build_device.sh` that generates the project, builds a Release build for my iPad, installs it with `xcrun devicectl` and launches it. Use my Apple ID team. Tell me my iPad's identifier first.

- **Human-only bits:** keep the iPad unlocked and trust the computer; the first install may ask you to trust your developer profile (Settings → General → VPN & Device Management).
- **Claude will** list devices, fill in your team ID, and run the build. A first build takes several minutes (Rust compiles everything); later ones are much faster.
- **Verify:** the app icon appears and SolidCraft opens with its sample part. Orbit with a finger.

## Step 4 — Make it feel like an iPad app

Run the app, use it, and report what's wrong in plain words. Each of these was a real round of feedback:

> The toolbar is under the iPad's status bar and rounded corners. Keep the whole UI inside the safe area.

> Tapping a number field brings up a letter keyboard. Show a number keyboard for number fields, and lay the window out above a docked keyboard so it isn't hidden.

> A finger on empty space should orbit the model, and my settings should survive restarts.

> The software keyboard's Return key doesn't act as Enter. Fix it.

Claude changes the Rust (and a small amount of Objective-C-style glue) and rebuilds. Expect many short rounds.

## Step 5 — Open and save files

> Add Open and Save through the system document picker, and open files in place so Save writes back to the original. Also let me browse the app's Documents folder, which should be visible in the Files app.

- **Claude will** use `UIDocumentPickerViewController` and set the right `Info.plist` keys (file sharing, opening documents in place).
- **Verify:** save a design, open the Files app on the iPad, find it, and open it again from there.

---

## Worked example — a first design on the iPad

Test everything so far with one realistic job. In SolidCraft on the iPad, ask Claude (or do it by hand) for a mounting plate: 80 × 50 × 8 mm, rounded corners, two holes, a round boss on top. The same plate becomes the test case for the AI assistant later, so keep it. **Expected result:** a body of about 35,924 mm³ that you can orbit, save, close and reopen.

### Tips for good iPad port prompts

- **Say what you see, not what you think is wrong** — "the keyboard hides the text box" gets a fix; "the layout is broken" gets a guess.
- **Test on the real device early** — the simulator hides touch, keyboard, safe-area and graphics-limit problems.
- **Ask for a plan before a big change** — "show me the plan first" avoids rework.
- **Keep Claude inside the project's rules** — if the repository has contribution rules, tell Claude to follow them and to run the project's own checks before you commit.
- **Commit after each working step** — a small commit per fix makes it easy to roll back.

---

## Part 2 (optional) — An AI assistant that drives the CAD app

Everything above works without any AI. This part adds an assistant in a window inside the app: you type or speak a request ("move the plate 30 mm to the left", "fillet the top edges 2 mm"), and the app turns it into real CAD commands.

## How it fits together

Three tiers, tried in order, with a fall-through:

| Tier | Runs | Used for |
|---|---|---|
| **Remembered requests** | Locally, instantly | Anything you've rated Good before, replayed with no model call |
| **Apple on-device model** | On the iPad (Apple's FoundationModels) | Simple edits: shapes, move, fillet, show/hide, view changes |
| **Claude Haiku** (online) or **Qwen3 4B** (offline) | The Anthropic API, or a small model on the iPad | Harder requests, and the hand-over when the first tier can't do it |

If the Apple model fails or loops, the app **undoes** what it did and hands the request to the next tier. Which one you get depends on whether you're online. The assistant works through a small set of **typed tools** (for example "add a box here", "fillet these edges"), not by writing free-form code, which is what makes a small model reliable.

## Step A — The assistant window with Apple's on-device model

> Add an AI window to the iPad app. The user types a request. Use Apple's on-device model (FoundationModels) with typed tools for the common actions: find a command by name, run a command, add a cube, box, cylinder or sphere, move a body, fillet or chamfer edges, show or hide bodies, change the view, and inspect the design. Show the tool calls in the window so I can see what it did.

- **Claude will** write a small Swift framework (loaded by the Rust app) and the tool definitions, and add the window.
- **Verify:** "Create a 30 mm cube" makes a cube; "Fillet all edges of the cube 5 mm" rounds it.

> **Real gotcha:** Apple's on-device model has a **small context window** (about 4,000 tokens). Don't give it the whole command list; give it short examples and typed tools. Claude learned this by watching it fail, and the fix was to send it examples only.

## Step B — A way to learn what works

> Log every request with its tool calls and result, and add Good, Partly and Poor rating buttons. Remember requests rated Good and replay them without calling a model. Keep hints in a JSON file on the iPad that I can edit without rebuilding.

After a test session, tell Claude Code to read the iPad's log (it pulls it with `xcrun devicectl device copy from`), see what went wrong, and improve the hints or code. This loop did more for reliability than any single feature.

## Step C — Guards against bad behaviour

Small models loop and invent things. Describe each failure you see and ask for a guard:

> The model repeated the same tool call and never finished, and it added a shape I didn't ask for. Stop after a repeated failed call or 30 calls, only allow a new shape or a new design when the request clearly asks for one, and when handing over to the next tier undo the first tier's changes first.

## Step D — Speech

> Add a Speak button using on-device speech recognition, with a clear recording state. Send the request after a short pause and keep listening until I type, paste or say "stop".

Needs the microphone and speech-recognition permission strings in `Info.plist`. Speech recognition mishears CAD words ("create new design" became "great to do design"), so keep a rewrites list in your hints file.

## Step E — The Claude tier (online)

> Add a Claude tier using the Anthropic Messages API with tool use. Store my API key in the iOS Keychain, with a paste button. Give Claude the full command instructions and the same typed tools, plus the ability to run any engine command, and tell it about the current selection.

- **Claude will** add a Keychain entry, a request loop and a hand-over from the Apple tier.
- **You must:** create your own API key at <https://console.anthropic.com>. Never put a key in the source or in a commit.
- **Cost:** Haiku is cheap; all our testing and many demo runs cost about **$0.14**.
- **Set a spending limit first.** In the Anthropic console, set a low monthly spending limit on the API key's workspace (for example $5) before you paste the key into the app. A looping request or a forgotten demo script can then only cost you the limit. Check the console's usage page now and then, and delete the key if you stop using the app.

## Step F — The offline tier (a small local model)

Before choosing a model, **test several on your Mac first**, using the real app's context:

> Download these small models and run the same 23 test requests through each, scoring the result against the correct answer. Tell me which is best on accuracy, size and speed. Use LM Studio for GGUF models and mlx_lm for MLX ones.

We tested nine. **Qwen3-4B-Instruct-2507 (4-bit MLX)** scored 22/23, equal best with two others, and was small enough for an 8 GB iPad. Others ranged from 18/23 down to 0/23. The result depends on your prompts, so run your own test.

> Add Qwen3 4B Instruct as an offline fallback using MLX Swift. Download it once from Hugging Face on request (about 2.3 GB), show progress, and give it typed tools, the current design with each body's extents, and the same instructions. Use it when I'm offline.

- **Real gotcha:** the app crashed at the end of the download with "Failed to load the default metallib". MLX looks for its GPU shaders in a bundle inside the app. The fix is a post-build script that copies `mlx-swift_Cmlx.bundle` into the app.
- **Real limit:** small models do arithmetic badly. Qwen put one cube at x = −90 when asked for −30. Give it the body extents and tell it to place new shapes with explicit x, y, z, and expect the odd slip.

---

## Reduce approval prompts (optional settings file)

iOS builds run the same safe commands over and over. Put a `.claude/settings.json` in your project to pre-approve them. Adjust the paths to your own project; risky commands (deleting files, pushing to git, anything outside the project) still prompt.

```json
{
  "permissions": {
    "allow": [
      "Bash(./build_device.sh:*)",
      "Bash(xcodegen generate:*)",
      "Bash(xcodebuild -project SolidCraft.xcodeproj:*)",
      "Bash(cargo check:*)",
      "Bash(cargo build:*)",
      "Bash(xcrun devicectl list devices:*)",
      "Bash(xcrun devicectl device info files:*)",
      "Bash(xcrun devicectl device copy:*)",
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ]
  }
}
```

> **Note:** a build that adds Swift packages with macros needs `-skipMacroValidation -skipPackagePluginValidation`. Claude Code's safety checks may stop it and ask. That is working as intended: read the command and approve it yourself, or add the specific build script to the allow list. Note that allowing `./build_device.sh` lets Claude run whatever that script contains, including installing to your iPad, so read the script once first.

## Organising your files

```
solidcraft/                       # the cloned repository
  apps/solidcraft-ios/
    src/                          # Rust: main.rs, ai.rs, ai_data.rs, files.rs, keyboard.rs, picker.rs
    ios-device/
      project.yml                 # xcodegen spec: the app, the AI framework, Swift packages
      build_device.sh             # build, install and launch on a device
      build_rust.sh               # compiles the Rust binary into the app bundle
      Info.plist                  # permissions, orientations, file sharing
      AI/                         # SolidCraftAI framework: AIBridge.swift, LocalLLM.swift
```

On the iPad (visible in the Files app under the app's folder): `Documents/ai/` holds `ai-log.jsonl`, `ai-hints.json` and `ai-learned.json`.

## What to expect

- Installing a new build restarts the app, so save your design first.
- The AI will not be perfect. Rate results honestly; the log is how it improves.

## Troubleshooting

### "wgpu: required limits not supported" in the simulator
The simulator reports lower graphics limits than a real iPad. Ask Claude to request only the limits the device or simulator supports, or test on the device.

### The app installs but won't launch, or says "untrusted developer"
On the iPad, open Settings → General → VPN & Device Management and trust your developer profile. Keep the iPad unlocked when installing.

### The app stops working after about a week
A free Apple account's signing expires after **7 days**. Rebuild and reinstall with `./build_device.sh`. With the paid developer program the profile lasts a year. If you upgrade and Xcode keeps issuing short profiles, it may be using a cached one: ask Claude to move the old files out of `~/Library/Developer/Xcode/UserData/Provisioning Profiles/` and rebuild.

### The AI loops, or adds stray shapes
Ask Claude for a guard (Step C) and give it the log. It usually points to one rule that was missing in the hints or tool descriptions.

### "Failed to load the default metallib" when starting the offline model
MLX can't find its shader bundle. Ask Claude to copy `mlx-swift_Cmlx.bundle` into the app in a post-build script (Step F).

### The online tier does nothing
Check the API key in the Keychain box and that the iPad is online. The assistant says which tier answered.

## Sharing your work

If you publish your version, ask Claude to check the repository for anything personal before you push:

> List every file that would be committed. Find and replace my Apple team ID, bundle ID, device identifier, user name and any key with placeholders or environment variables. Show me the list before committing. Then give me the exact commands to push to my fork. Don't push anything yourself.

Respect the upstream project's licence and contribution rules, and call your version an unofficial fork.

## Why this works well with Claude Code

- **Small, checkable steps.** Each feature is something you can test on the iPad in a minute.
- **The log is the evidence.** Claude reads the iPad's request log and fixes things from facts, not from your description of a screenshot.
- **It takes care of toolchain detail.** Targets, signing, bundle layout, shader bundles and permissions strings are the parts a newcomer would stall on.
- **You stay in control.** Builds, installs and anything that touches your account or plan are your decision.

## Resources

- **Claude Code:** <https://claude.com/claude-code>
- **SolidCraft (upstream):** <https://github.com/tangency-labs/solidcraft>
- **This work (unofficial fork):** <https://github.com/ukdavewood/solidcraft> (branch `ipad/fork-publish`)
- **Apple Developer Program:** <https://developer.apple.com/programs/>
- **Enabling Developer Mode on a device:** <https://developer.apple.com/documentation/xcode/enabling-developer-mode-on-a-device>
- **MLX Swift LM:** <https://github.com/ml-explore/mlx-swift-lm>
- **xcodegen:** <https://github.com/yonaskolb/XcodeGen>
- **Demo video:** <https://youtu.be/DE2SXOfs0-4>

Happy building!
