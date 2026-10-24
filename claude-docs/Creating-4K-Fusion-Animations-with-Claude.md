# Creating 4K Fusion Animations with Claude Code

You have a 3D model in Autodesk Fusion and you want a clean animation of it: parts appearing group by group, the camera swinging round, the model exploding apart. Fusion's own animation tools aren't built around driving it from a plain list of names. This guide shows how to get there by asking Claude Code for three things: a small show/hide add-in, a naming scheme, and a frame-capture step. A video tool such as HyperFrames then turns the frames into a finished 4K video with captions, voice-over and music.

> **Scope and caveats.** Written for Autodesk Fusion on macOS, driven through Fusion's Python API. **Tested on macOS only**; the Fusion API is the same on Windows, so the approach should carry over with different file paths, but that is untested. The examples come from a large building model (about 2,400 bodies), but nothing here is specific to buildings. Timings and memory figures are from one Mac, so treat every number as an example. Independent, unofficial guide; not affiliated with or endorsed by Autodesk or HeyGen.

## What you'll need

- **Autodesk Fusion** with a model whose bodies you can rename, and a copy to work on.
- **Claude Code**, and a way for it to run Python scripts against the open design (Step 0).
- **A video tool** for the last step. This guide uses [HyperFrames](https://developers.heygen.com/hyperframes-overview); any tool that can assemble an image sequence will do.

## The core idea

Fusion can show or hide a body by name, take a screenshot of the viewport, and move the camera, all from a script. The trick is to make those three things controllable from one line of text:

```
U;-,+Plinth,+Walls;SHOT:walls;P;+Windows;SHOT:windows;P;V2;SHOT:view_two;P
```

Read it as a script for a film:

- `;` ends a step. After each step the viewport redraws, so nothing flickers between steps.
- `,` separates the terms inside one step. All of them happen at once, in the background.
- `+Name` shows every body whose name contains `Name`. `-Name` hides it. A lone `-` hides everything and `U` unhides it all.
- `SHOT:label` saves a frame and writes a row to a manifest file.
- `P` pauses for a second. `V2` restores the saved view called View2.

Because matching is by name, **how you name your bodies is your storyboard**. A model whose bodies are called `Body1`, `Body2`, … can't be animated this way; one whose bodies are called `Wall_North_Window_03` can.

> **⚠️ Safety first.** Work on a copy of your design, and save a version before each big step. Scripts that change a design should write a log file, not just print. In our runs Fusion rolled back any script that ended with an error, which is handy for read-only checks and a trap for scripts that are meant to change something.

## Step 0 — Install Claude Code and open a folder

Install Claude Code from <https://claude.com/claude-code>, open a new folder in your editor and start a session there. Keep your scripts, logs and captured frames in that folder so Claude can read them. Fusion needs an add-in or script that Claude can call. A good first message:

> I want to drive Autodesk Fusion from Python scripts that you write. Set up a small helper that runs a script file against the open design and returns the result or any error, and tell me what I need to enable in Fusion.

- **Claude will** propose the helper (Fusion's built-in scripts folder is the easiest route) and tell you what to switch on.
- **Verify:** ask it to run a read-only script that reports the design's name and body count.

## Step 1 — Ask for the add-in

> Create an Autodesk Fusion add-in called ShowHideAnimate. It takes one line of text. `;` separates steps, `,` separates terms. `+name` and `-name` show or hide any body whose component, body or appearance name contains `name`, ignoring case. `U` unhides everything and `C` fits the view. Redraw only at the end of each step.

Claude writes it, and you test it on a handful of bodies before going further (**verify:** type `-,+Wall` and only the walls remain visible; `U` brings everything back). Expect a few rounds. Ask it to add what you need as you go: a pause (`P`), saved views (`V1`… or `V:name`), and settings in user parameters, such as `{scale=200}`.

## Step 2 — Fix the names

> Look at the body and component names in this design. Suggest renames so that a short name fragment picks out each group I want to reveal, such as walls, windows, roof. Show me the list first. Don't rename anything yet.

Claude lists the groups and finds collisions, for example `Side` matching both `Porch_Side` and `Side_Wall`. Agree the list, then ask it to apply the renames in a script and to log every change. Names that appear in many places need care: if two groups share a fragment, one step will show both.

## Step 3 — Write the animation as text

> Add text user parameters `_anim1` to `_anim4`. Each is one animation string. `_anim1` builds the model up group by group in this order: foundation, walls, roof, windows, details. Hold one second after each group.

Parameters that start with `_` act as shortcuts. Type `_anim1` into the add-in and the whole film runs. Keep each film short so you can test it on its own.

## Step 4 — Capture the frames

> Add a `SHOT` tag. `SHOT:label` saves a PNG of the viewport at that point, with a transparent background, at the size in the user parameters `shotWidth` and `shotHeight`. Write a `manifest.csv` with one row per frame: the label, the step text, the hold time, the camera and the visible body count.

Capturing is cheap. In our runs a 4K frame took well under a second, so don't be afraid of high resolution. **Verify:** run one short film and check that the folder holds one PNG per `SHOT` and that `manifest.csv` has a matching row for each. The manifest is what lets the video tool add captions and voice-over later, because every frame carries the label of the step that made it.

For smooth camera moves, add an `ORBIT` tag that steps the camera between two saved views and captures a frame at each step:

> Add `ORBIT:label@V1>V2:8` which moves the camera from View1 to View2 in eight steps around the model's centre, capturing a frame at each step. `>` is the short way round, `>>` forces anticlockwise and `<<` clockwise, and the same view on both ends is a full turn. The last step must land exactly on the end view.

## Step 5 — Move parts, not just show them

Show and hide can't move anything. Moving real bodies in a design is slow, because every move changes the model. A faster way is to copy the visible bodies once as lightweight meshes in Fusion's custom graphics, hide the originals, and move the copies. The copies can't be selected, so nothing highlights in your screenshots, and in our runs each frame took about a tenth of a second.

> Write a script that groups the bodies by part, copies each group as a mesh into its own custom graphics group, then generates frames: parts slide apart along their facing direction, the camera turns once round the exploded model, then each part rotates flat and slides into a layout on one plane. Write every frame to a folder with a manifest. End the script with an error so the document rolls back, and delete the graphics when done.

Build the meshes once per run. Creating them as B-rep copies instead was roughly fifty times slower per frame in our tests.

## Step 6 — Turn frames into a video

> Take the frames and manifest in this folder and build a video with HyperFrames: a caption for each stage, hard cuts between frames of a camera move and a short dissolve between stages, voice-over and music. Render 1080p first, then 4K.

HyperFrames assembles the frames into one timeline, so the same frames can be re-cut with new captions or timing without going back to Fusion.

## Useful beyond animation

The same add-in earns its place long before you make a film:

- **While you model.** Show only the group you are working on, then bring everything back with one line. Large designs stay quick to work in.
- **Variations in one design.** Keep several versions of a model, such as different finishes or print scales, in the same file. A named line shows one variation and hides the rest, with no copies of the design.
- **Control what is exported.** Show exactly the bodies you want, then export what is visible. A line for each print part gives you the same parts every time.

## What to expect

- The first run of a new animation string rarely looks right. Run it live and watch before capturing.
- On our Mac, a 400-frame capture of a 2,400-body model took about two minutes, and turning that into a 4K video took about a minute. Yours will differ.
- Fusion's memory grew as we switched views. Save, close and reopen the design when it gets very high (we did at about 16–20 GB); saving alone didn't release it.

## Troubleshooting

### A step shows more than you expected
Matching is a substring match, so `+Side` also picks up `Porch_Side_…`. Use a longer fragment, or rename.

### A shortcut inside a longer name does nothing
Shortcuts only expand when they stand alone between separators. A name that merely contains `_anim1` won't expand.

### A script that should change the design changes nothing
It probably ended with a deliberate error, which Fusion treats as a reason to roll back. Use that only for read-only checks.

### Every frame is the finished model
`SHOT` captures at its own position in the line. Put it after the step it should record, and make sure the steps before it are separate steps, not one long term list.

## Why this works well with Claude Code

Each piece is small and checkable: a text format, a naming list, a capture step. Claude can test each against your real design and read the logs and manifests it writes. When something goes wrong the evidence is in a file, not in a screenshot you have to describe.

## Resources

- **Claude Code:** <https://claude.com/claude-code>
- **Fusion API reference:** [`Viewport.saveAsImageFileWithOptions`](https://help.autodesk.com/cloudhelp/ENU/Fusion-360-API/files/Viewport_saveAsImageFileWithOptions.htm) (and the same reference for `CustomGraphicsGroup`)
- **HyperFrames:** <https://developers.heygen.com/hyperframes-overview>

Happy animating!
