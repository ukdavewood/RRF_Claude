Building (and Enhancing) RepRapFirmware with Claude Code
A practical guide to using Claude Code (Anthropic's AI coding assistant) to compile RepRapFirmware (RRF) for Duet3D boards on Windows or macOS — and then to add your own features to the firmware — starting from a computer with no developer tools installed.

This is a real-world write-up: everything here was done end-to-end with Claude Code over a few days. You don't need to be a programmer to follow it.

The core idea
You don't have to know the exact toolchain version, dependency branches, or source-code line numbers. You describe what you want in plain English, and Claude works out the specifics for you — which repositories to clone, which versions match the RRF release you picked, and where to change the code. Your job is to state intent clearly, approve the steps, and flash/test the result on your board.

That means most instructions below are phrased as things to ask Claude, not commands you have to type yourself.

⚠️ Read this first: safety and rollback
RepRapFirmware controls a machine with heaters and motors. Flashing custom or beta firmware can stop a printer working, and in rare cases behave unexpectedly. Do not flash to a machine mid-print, and stay near it on first boot after an update.
Beta releases (e.g. 3.7.0-beta.1) are not stable releases. Only use one if you specifically want to.
Keep a known-good firmware file and know how to roll back before you start. Download the official release for your board from Duet3D and keep it safe. If a build won't boot, you re-flash the known-good file (see Emergency Recovery (BOSSA) near the end).
Build and test on hardware you can afford to have offline for a while.
What you'll need
Regardless of platform:

A Duet3D board you want to build for (e.g. Duet 3 MB6HC, Duet 3 Mini 5+, or a TOOL1RR expansion board).
Claude Code (installation covered in Step 0). A Pro or higher plan is recommended — a first full build is a long session and can exceed free-tier usage limits.
Internet connection (Claude clones several repositories and downloads a toolchain).
~3–5 GB free disk space for the repositories, toolchain, and build outputs.
Time: the first build takes roughly 45–90 minutes including dependency compilation; later builds take 5–10 minutes.
Everything else — Git, the compiler, build tools, Python — is installed in Step 0 and Step 1 below. You do not need any of it beforehand.

Heads-up on permissions: Claude Code asks for your approval before it runs commands (cloning, building, copying files). During a long build you'll see many of these. The "Reduce approval prompts" section near the end shows a ready-made settings file that pre-approves the safe, repetitive commands so the build isn't constantly interrupted.

Step 0 — Install Claude Code and open your project
Install Claude Code by following the official instructions at https://claude.com/claude-code (available as a CLI, a desktop app for Mac/Windows, a web app at claude.ai/code, and IDE extensions for VS Code and JetBrains).
Sign in with your Anthropic account.
Create an empty folder for this project (e.g. RRF in your Documents) and open that folder in Claude Code. This folder is where your notes, patches, and firmware outputs will live.
Start a chat. You interact with Claude by typing requests in plain English, just like the prompts quoted throughout this guide (shown as > quoted blocks).
A good first message is simply:

I've saved "Building-RRF-with-Claude-Code.md" in this folder. Please read it, check what build tools I already have, and tell me the next step to get started building RRF on [macOS / Windows].

Claude will read the guide, inspect your machine, and tell you exactly what's missing.

Step 1 — Install the build toolchain
You can let Claude drive most of this. The one thing that usually needs a human is running a graphical installer or editing your system PATH — Claude will tell you precisely what to do and can verify each step afterwards.

Toolchain version matters. Each RRF release is designed to compile with a specific ARM GCC version. Don't guess it — ask Claude:

Which exact ARM GCC (arm-none-eabi) version does the RRF release I'm building expect? Give me the download link for my operating system and CPU.

Then install the version Claude names. Using the wrong compiler version is a common cause of build failures.

macOS
macOS is the simplest starting point (Apple Silicon Macs — M1 through M5 — are all fine).

Xcode Command Line Tools — this provides git and make in one step:

xcode-select --install
Rosetta 2 (only needed on Apple Silicon, for the x86 CRC helper that ships with RRF; harmless to install):

softwareupdate --install-rosetta --agree-to-license
(If you'd rather not install Rosetta, Claude can append the CRC with a small Python script instead — see Troubleshooting.)

Python 3 — usually already present. Check with python3 --version; if missing, install Homebrew then brew install python3. Python is used for .uf2 conversion (Mini 5+).

ARM GCC — ask Claude for the correct version/link (see the box above). It's a .pkg installer that installs without admin rights. After installing, add its bin folder to your PATH. Ask Claude:

I've installed ARM GCC to [path the installer reported]. Please add it to my PATH in ~/.zshrc and verify arm-none-eabi-gcc --version works.

Windows
A clean Windows PC needs three things: Git, a POSIX build environment (MSYS2, which provides make), and ARM GCC.

Git for Windows — download and install from https://git-scm.com/download/win (accept the defaults).
MSYS2 — download from https://www.msys2.org/, run the installer, accept the default location (C:\msys64). Then install make from an MSYS2 terminal or PowerShell:
C:\msys64\usr\bin\pacman -S make
Add C:\msys64\usr\bin to your Windows PATH (Settings → Edit environment variables).
Python 3 — install from https://www.python.org/downloads/ (tick "Add Python to PATH"). Needed for .uf2 conversion.
ARM GCC — ask Claude for the correct version/link (see the box above). The portable .zip needs no admin rights: extract it somewhere like C:\dev\RRF\toolchain\ and add its bin\ folder to your Windows PATH. (An .msi installer also exists but needs admin.)
After any PATH change on Windows, close and reopen your terminal (and Claude Code) so the new PATH is picked up.

Where to put your files
Pick a working folder outside any cloud-sync directory (OneDrive, iCloud Drive, Dropbox). Sync locks during clone/build cause hangs and corrupt builds. Good choices:

macOS: ~/dev/RRF
Windows: C:\dev\RRF (use whichever drive you have — D: is not required)
Keep the source repositories and your firmware outputs under the same working folder so everything is easy to find, e.g. ~/dev/RRF/repos for source and ~/dev/RRF/firmware for outputs.

Step 2 — Pick a version and let Claude resolve the dependencies
You don't need to know the dependency version numbers. Choose the RRF release you want to build, and ask Claude to work out the matching versions of everything else. For example:

I want to build RepRapFirmware 3.7.0-beta.1. Please:

Find the matching versions/branches of all the dependency repositories for that release (CoreN2G, FreeRTOS, RRFLibraries, CANlib, LibTinyusb, LibMbedTls, WiFiSocketServerRTOS, and Duet3Expansion if I'm building an expansion board).
Clone them all into ~/dev/RRF/repos/ (they're on GitHub under the Duet3D organisation).
Initialise any git submodules (LibTinyusb needs this).
List what you cloned with the exact branch/commit for each, so I have a record.
Claude will batch the clones in parallel and report a table of repos, branches, and commits. Keep that list — it's your reproducible record of exactly what went into the build.

Why let Claude find the versions? The correct dependency branches shift between RRF releases (and some, like the WiFi socket server, need a specific feature branch). Asking Claude to resolve them for your chosen release is more reliable than copying fixed version numbers out of a guide that may be out of date.

Step 3 — Do a base verification build
Always build the unmodified firmware first and confirm it's healthy before you change anything. That gives you a known-good baseline to compare against.

Build the standard, unmodified 6HC firmware from ~/dev/RRF/repos/RepRapFirmware.

Use the ARM GCC toolchain I installed.
After linking, append the CRC32 checksum to the .bin (the board won't boot without it).
Copy the result to ~/dev/RRF/firmware/RRF-standard/.
Report the final file size and CRC value, and compare the size against the official Duet3D release for this board as a sanity check.
Claude will compile the dependencies (slow the first time), link the firmware, append the CRC, organise the output, and report sizes. A standard build should come out within ~1–2% of the official release size — a larger difference is a red flag worth investigating before you flash.

Building for other boards / multiple boards at once
The build targets differ per board. You can ask for several in one go:

Build all three boards from the repos:

6HC (main board) → produces a .bin
Mini 5+ → produces a .uf2
TOOL1RR (expansion, from Duet3Expansion) → produces a .bin
For each: append the CRC checksum. For the Mini 5+, append the CRC to the .bin first, then regenerate the .uf2 from the CRC-corrected .bin (the CRC must be inside the UF2). Put everything in ~/dev/RRF/firmware/RRF-standard/ and report all sizes and CRCs.

The one ordering rule that matters: CRC is appended to the .bin before UF2 conversion. Claude handles this automatically, but it's the most common cause of a "won't flash" Mini 5+.

Step 4 — Flash and verify
Flashing (DuetWebControl — all boards)
For any board, the normal method is to upload the firmware file to the board's system folder and let DuetWebControl (DWC) install it:

Open DWC in your browser, connected to your board.
Drag-and-drop the firmware file onto the DWC window (or upload it to the 0:/sys folder):
6HC: Duet3Firmware_MB6HC.bin
Mini 5+: Duet3Firmware_Mini5plus.uf2
TOOL1RR: Duet3Firmware_TOOL1RR.bin (installed over CAN from the main board — you still upload it to the main board)
DWC recognises the firmware filename and prompts to install it automatically. Confirm.
The board flashes and reboots (usually 30–60 s). DWC reconnects after ~10 s.
Verify the boot
Once reconnected, run in the DWC console:

M115
You should see the firmware name, version, electronics, and build date, e.g.:

FIRMWARE_NAME: RepRapFirmware for Duet 3 MB6HC
FIRMWARE_VERSION: 3.7.0-beta.1
ELECTRONICS: Duet 3 MB6HC v1.02c or later
Success looks like: board reboots cleanly, status LED flashes normally, DWC reconnects, and M115 reports the version you built. If instead the LEDs stay solid on, the CRC is likely missing/corrupt — re-flash, and if that fails use BOSSA recovery below.

Enhancing the firmware — just describe the feature you want
This is where letting Claude do the work really pays off. You don't need to know C++, the function-table layout, or which files to edit. Describe the feature in plain English, point Claude at any relevant background, and let it implement, compile, fix its own compiler errors, and produce a testable binary.

Worked example: add a format() function for number formatting
Suppose you want RRF to format numbers into fixed-width strings (like the classic ZZZ9.99 "picture" formatting — leading spaces, a fixed number of decimals). You'd simply ask:

Please add a new built-in function to RRF to format numbers into strings.

The function name should be format.
It should return a formatted string.
The parameters should be: the number, the minimum number of digits/spaces to include to the left of the decimal point, and the number of digits after the decimal point.
For background, see this thread on the Duet3D community: https://forum.duet3d.com/topic/39277/rounding-numbers-on-echo-output-to-zz9-99-format

Once implemented, add a short suffix to the firmware version string so I can tell my build apart from stock (e.g. +dw1), build the 6HC firmware, append the CRC, and copy it to ~/dev/RRF/firmware/RRF-format/. Report the size difference versus my standard build.

Claude will:

Read the background thread and work out where built-in functions live in the source.
Add the function (declaration, implementation, and hook into the expression parser's function table).
Bump the version string so the change is identifiable via M115.
Compile, and if there are errors (e.g. implicit float/double conversions), diagnose and fix them itself, then rebuild.
Compare the new firmware size against your standard baseline so unexpected bloat is caught.
Hand you a .bin ready to flash.
Then flash it (Step 4) and test in the DWC console, e.g.:

echo format(3.14159, 4, 2)
Tips for good enhancement prompts
State the interface precisely — name, return type, and each parameter's meaning. That's the part only you can decide.
Link to background — a forum thread, a wiki page, or an example of the desired output helps Claude match your intent exactly.
Ask for a version suffix so M115 clearly distinguishes your build from stock.
Always build from a clean, verified baseline (Step 3) so any size/behaviour difference is attributable to your change.
Ask Claude to save the change as a patch/diff file in your project folder, so you can re-apply it to future RRF releases or propose it upstream.
Moving your change to a newer RRF release later
When a newer RRF version ships, you don't re-do the work by hand:

RRF [new version] is out. Please find the matching dependency versions, clone them fresh, re-apply my saved format() patch, resolve any conflicts, build the 6HC, and report any issues.

Getting your enhancement into official RepRapFirmware
The builds you make here are for your own testing — a way to try an idea out on real hardware before anyone else is involved. If the idea proves useful and you'd like it in the official firmware, the reliable route is to get the Duet3D community and maintainers on board first, and let coding follow agreement — not the other way round.

In parallel with testing your enhancement locally, raise the requirement in two places:

A thread on the Duet3D community forum — https://forum.duet3d.com — describing the feature, why it's useful, and the behaviour/interface you're proposing. This is where the idea gets discussed and shaped.
An enhancement request (issue) on the RepRapFirmware GitHub — https://github.com/Duet3D/RepRapFirmware/issues — so it's formally on the maintainers' radar.
Then, while you're waiting to see whether it's accepted, you can go ahead and actually make the change locally (exactly as in the worked example above), fork the repository, and prepare a pull request to add it to the official firmware.

Important: get the change agreed on the Duet3D forum before opening a pull request. Unsolicited or unexpected pull requests — however well made — are likely to be rejected if the maintainers haven't already agreed the feature and its design. Discussion and agreement first; PR second.

Claude can help with each stage: drafting the forum post and GitHub issue, implementing and testing the change, and — once there's agreement — creating the fork, branch, and pull request with a clear description and your patch/diff.

Reduce approval prompts (optional settings file)
A full build runs the same safe commands many times (clone, make, list files, copy outputs), and Claude asks permission each time by default. You can pre-approve the low/medium-risk ones by creating .claude/settings.json in your project folder. Ask Claude:

Create a .claude/settings.json that pre-approves the safe, repetitive commands this RRF build needs, and log anything it still has to ask me about so I can review it.

A reasonable starting point looks like this (Claude will tailor it to your setup):

{
  "permissions": {
    "allow": [
      "Bash(git clone:*)",
      "Bash(git fetch:*)",
      "Bash(git submodule:*)",
      "Bash(git status:*)",
      "Bash(git log:*)",
      "Bash(git diff:*)",
      "Bash(git checkout:*)",
      "Bash(make:*)",
      "Bash(arm-none-eabi-gcc:*)",
      "Bash(arm-none-eabi-g++:*)",
      "Bash(arm-none-eabi-objcopy:*)",
      "Bash(arm-none-eabi-size:*)",
      "Bash(python3:*)",
      "Bash(ls:*)",
      "Bash(find:*)",
      "Bash(mkdir:*)",
      "Bash(cp:*)"
    ]
  }
}
Everything genuinely risky (deleting files, changing system settings, anything outside the project) will still prompt you.

Organising your firmware files
Keep a tidy, self-documenting folder so you can tell builds apart at a glance:

firmware/
├── RRF-standard/          # Pristine build, unmodified — your baseline (all boards)
│   ├── Duet3Firmware_MB6HC.bin
│   ├── Duet3Firmware_Mini5plus.uf2
│   └── Duet3Firmware_TOOL1RR.bin
│
├── RRF-format/            # With your custom format() function (6HC)
│   └── Duet3Firmware_MB6HC.bin
│
└── official-reference/    # Official Duet3D release, kept for comparison + rollback
    └── Duet3Firmware_MB6HC.bin
Naming suggestions:

Prefix with the platform if you build on both (mac_, win_).
Add a short version suffix for custom builds (+dw1, +dw2, …) that also appears in M115.
Keep the official release alongside your builds — it's both your size-comparison reference and your rollback file.
Troubleshooting
Most of these Claude detects and fixes automatically — this list is so you understand what's happening and can intervene if needed.

"arm-none-eabi-gcc: command not found"
ARM GCC isn't on your PATH. Verify it's installed, add its bin folder to PATH, then open a fresh terminal (Windows) or source ~/.zshrc (macOS). Ask Claude to re-verify with arm-none-eabi-gcc --version.

"CrcAppender not found" (or no CRC helper for your CPU)
The CRC helper that ships with RRF is an x86 binary. On Apple Silicon, either install Rosetta 2 (softwareupdate --install-rosetta) so it runs transparently, or let Claude append the CRC with a tiny platform-independent Python script instead:

import zlib, struct
data = open("firmware.bin", "rb").read()
crc = zlib.crc32(data) & 0xffffffff
open("firmware.bin", "ab").write(struct.pack('<I', crc))
Windows: "Cannot create temporary file in C:\WINDOWS"
An MSYS2 + native-compiler quirk. Claude fixes it by pointing the temp directory at your project folder for the build (env TMP='<path>' TEMP='<path>' MSYS2_ARG_CONV_EXCL='*' make ...) and retrying.

Board won't boot (all LEDs solid on, not flashing)
Almost always a missing/corrupt CRC. Claude always appends the CRC and verifies the file grew by exactly 4 bytes. If it still happens: re-flash via DWC; if DWC won't take it, use BOSSA recovery below.

Mini 5+ UF2 flash fails with a CRC error
The .uf2 was made from a .bin that didn't have its CRC yet. Ask Claude to regenerate the UF2 from the CRC-corrected .bin and confirm the CRC is present, then re-flash.

Firmware much larger than expected (>~2% over the official release)
Possible debug symbols or an unintended change. Ask Claude to diff your source tree against the official release and identify what was added.

Emergency Recovery (BOSSA — 6HC)
Only needed if normal DWC flashing fails and the board won't boot.

Download BOSSA: https://github.com/shumatech/BOSSA/releases
Connect the 6HC via USB (not Ethernet).
Enter bootloader mode: hold the BOOT button, power on the board, release BOOT after 2–3 s.
In BOSSA, select the board's port ("Duet3 Bootloader" or similar).
Select your firmware .bin (your known-good/official file if recovering), click Upload, wait for success.
Power-cycle or reset — the board should boot normally.
BOSSA is recovery only; normal updates always go through DWC.

Why this works well with Claude Code
You describe intent; Claude handles specifics — versions, branches, file locations, and compiler errors.
Toolchain and dependency setup is automated — Claude downloads, verifies, and configures, and clones repos in parallel.
Platform quirks are handled in-flight — CRC ordering, UF2 conversion, MSYS2 temp-file bug, Rosetta/Python fallbacks.
Enhancements are a conversation, not a coding project — describe the feature, link the background, get a testable binary.
Always verified on real hardware — size comparisons and M115 confirm you built what you intended.
Resources
Duet3D — Building RepRapFirmware: https://github.com/Duet3D/RepRapFirmware/wiki/Building-RepRapFirmware
RepRapFirmware source (and dependencies): https://github.com/Duet3D
ARM GNU toolchain downloads: https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
DuetWebControl: https://github.com/Duet3D/DuetWebControl
BOSSA (6HC recovery): https://github.com/shumatech/BOSSA
Claude Code: https://claude.com/claude-code
Happy building!
