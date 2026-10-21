# Automating BambuStudio Plate Labels with Claude Code

You print several plates from one project and want each plate to carry a small **flat label with the file name and plate name cut out of it**, so you can tell which plate is which once they're printed. This guide shows how to get **Claude Code** to build a script that does it for you inside the real **BambuStudio** app on a Mac, with no programming on your part.

> **Scope and caveats:** **macOS only**, written and tested against **BambuStudio 02.08.03.66**. It works by driving the app's interface, so menus, shortcuts and behaviour can change between versions — treat the specific findings below as what held for that version and expect to re-test after updates. **Tested on:** macOS 26 (Apple Silicon), a multi-plate project using the Bambu Lab X1 Carbon profile, run from Terminal, VS Code and an Elgato Stream Deck XL. **Not tested:** Windows, other versions or printer profiles, plates that print by object (the script skips those). Independent, unofficial project; not affiliated with or endorsed by Bambu Lab.

## The core idea

You describe the **result** you want, and give Claude the **rules** and the few **things already learned the hard way**. Claude reads the slicer's source, tries small experiments against the live app, builds the script, and tests it on a copy of your project.

> ## ⚠️ Safety first
> Work on a **copy** of your project and keep a pristine backup beside it. The script takes over the mouse for short moments (when you're idle) and restores it afterwards; early experiments crashed BambuStudio once and left its menu stuck once. If a run goes wrong, close without saving; once the script has saved the project (it does, after each label), copy the backup back over the test copy.

## Step 0 — Install Claude Code and open a folder

Install **Claude Code** from <https://claude.com/claude-code>, make an empty folder for the automation, and open it in Claude Code. Open your **test copy** of the project in BambuStudio. Grant the app that runs the script (Terminal, VS Code or the Stream Deck app) **Accessibility** and **Screen Recording** permission — Apple explains how: [Accessibility](https://support.apple.com/guide/mac-help/allow-accessibility-apps-to-access-your-mac-mh43185/mac) · [Screen Recording](https://support.apple.com/guide/mac-help/control-access-screen-recording-mchld6aa7d23/mac). Claude checks for and installs Python and the libraries it needs.

## Step 1 — Ask for what you want

> Create me a script that labels every plate of the open BambuStudio project with the file name and plate name/number automatically, so I can see which plate is which when they are printed.

That alone gets you a conversation and some trial and error. To get a safe, reliable script quickly, follow it with this brief:

## Step 2 — Add this guidance

> How to build it:
> - **Drive the real BambuStudio UI** with macOS Accessibility. Don't modify the app or its network plug-in. Never write the project file — BambuStudio must save it (⌘S). You may *read* the saved project (a zip) to plan and to verify.
> - **First read the source for my exact version** and test each unknown with a small experiment; tell me yes/no for each before building.
> - **Label = a thin slab with the text cut through it**, generated as an STL. Import it with `open -g -a` (background, no dialog) after a *real click* on the plate's row makes that plate current.
> - **Position:** back-left of the plate if free, else the next free spot along the back edge, then row by row. Free = clear of every object, the prime tower and the blocked bed corner (read from the saved project). No space → warn me, don't place it.
> - **Settings:** tick *Others → Purge options → Purge into this object*. Single-colour plate → label gets that filament; multi-colour plate → leave the colour alone. Skip plates that print by object and plates that already have a label.
> - **Rules from experience:** the position fields aren't visible to automation, so nudge with the arrow keys in the 3D view (10 mm, 1 mm with Shift) after selecting the row with a real click and focusing the view with a middle-click on empty space. Never press menu items (use shortcuts) and never write into list cells — both broke the app. Refuse any click unless the window under the pointer is the expected one. Re-read control positions (such as object-list rows) after the app comes to the front, because the list can scroll back and a click on an old coordinate lands on the wrong row.
> - **Verify** every label (position, purge setting, filament) from the saved project and compare the whole file with my backup.
> - **User experience:** wait until I'm idle; show a persistent "hands off" banner while it runs and a result panel with an OK button at the end; add a log (`logs/run.log`), exit codes and a `--dry-run`. Optionally make it launchable from a single Stream Deck button.

- **Claude will** confirm the plan, run its experiments, build the script piece by piece, and test it on your copy.
- **Verify:** the dry run's plan matches what you'd decide by eye, and after a real run only the labels differ from your backup.

## What to expect

On a three-plate test project — one single-colour plate, one two-colour plate, one packed plate — the script put a label at the back-left of the first (in that plate's filament), moved the second's label along the back edge because the prime tower blocked the corner, and left the third alone with an amber panel: *"NO SPACE for the label — rearrange this plate and run again"* (exit code 2). Each label took roughly 20 seconds in our runs.

## Troubleshooting

### "refusing click: window at … is Bambu Studio/None"
The launching app lacks **Screen Recording**, so macOS hides window titles. Grant it, restart that app, and run again. (The script also has a fallback that matches windows by size and position.)

### The button moves the cursor, then nothing happens
Read `logs/run.log` — the last lines say which safety check refused. It stopped rather than click somewhere unexpected; paste the line to Claude.

### "NO SPACE for the label"
Not an error: that plate is full. Make a gap near the back edge and run again — plates that already have a label are skipped.

### BambuStudio's menu is stuck, or it ignores the mouse
Something pressed a menu item through automation. Save, quit and reopen, and ask Claude to switch that step to a keyboard shortcut.

## Why this works well with Claude Code

- **Intent first** — you state the result and the rules; Claude finds how.
- **Experiments before commitments** — small probes of the live app replace long builds on wrong assumptions.
- **Verification from the file** — every claim ("it's at the back-left", "purge is on") is checked in the saved project.

## Resources

- **Claude Code:** <https://claude.com/claude-code>
- **BambuStudio source (pick the tag matching your version):** <https://github.com/bambulab/BambuStudio>
- **PyObjC (Python access to macOS Accessibility):** <https://pyobjc.readthedocs.io>
- **Elgato Stream Deck:** <https://www.elgato.com/stream-deck>

Happy automating!
