# Building a Quest 3 Viewer and AI Remote for SolidCraft with Claude Code

You have a CAD design on your Mac and a Meta Quest 3. You want to stand in your room, see the design at table size over the real furniture, pick parts with the controllers, and change the design by talking to it ("make the selected sails red", "fillet this edge", "scale the whole ship by half"). This guide shows how to get there by asking Claude Code for each piece: a small OpenXR app for the headset, a server on your Mac that drives SolidCraft, and an AI that does the editing.

This is a real-world write-up of an **experiment**. It was built end to end in about three days (15 to 17 November 2026) on a Mac and a Quest 3, extended on 18 November with sketches and dimensions (Part 4), and on 19 November with a live link to the normal SolidCraft app, a window onto the Mac screen, an editable timeline and a first Autodesk Fusion version (Part 5). It starts from the open-source [SolidCraft](https://github.com/tangency-labs/solidcraft) command-line program (v0.41). You don't need to be a programmer, but you do need to be comfortable approving steps, wearing the headset to test, and describing what you see.

> **Version of this guide**
>
> - **Guide updated:** 20 November 2026 (covers Parts 1 to 5)
> - **SolidCraft:** 0.41.0 (the desktop app and its command-line program)
> - **Tested on:** one Mac and one Quest 3; Autodesk Fusion for the Part 5 Fusion version
> - **Reference code:** the sample repository, frozen at the 17 November 2026 version (Parts 1 to 3). See [Sample code](#sample-code).
> - **Demo videos:** the [playlist on YouTube](https://www.youtube.com/playlist?list=PLS769WnSAR1A)

> **About the sharing approach.** This guide, the videos and a small sample repository are shared as an experiment, not as a maintained product. The idea is that you give your AI coding assistant the guide (and the sample code, if useful) and have it build your own version. If you try it, please add what you learned: see [Community knowledge](#community-knowledge-and-contributing).

> **Scope and caveats.** Written for a **Mac** and a **Quest 3** (the headset's software was Horizon OS with OpenXR runtime "Oculus 210.117"). **Not tested on Windows or Linux** (the server is plain Python and should port, but the voice and keychain parts were only run on macOS) and not tested on other headsets. It is an independent, unofficial guide: not affiliated with or endorsed by the SolidCraft project (`tangency-labs`), Meta or Anthropic. The result is a personal, experimental tool, not a supported product. Timings, costs and menu names are from one setup and may change with system updates, so treat them as examples.

## The core idea

You describe what you want in plain English, and Claude works out the specifics: which Rust crates, how to talk to OpenXR and Vulkan, how to package an Android app for the Quest, how to drive SolidCraft's command interface, and how to fix what breaks when it meets the real headset. Your job is to state intent clearly, approve steps, **test in the headset**, and report what you see ("the panel is too small", "the mouse does nothing", "it said done but nothing changed"). Claude reads the logs and fixes the code.

Most instructions below are phrased as things to ask Claude, not commands you type.

---

## How it fits together

Four parts, left to right:

```
Quest 3 viewer  ──Wi-Fi──>  Server on the Mac  ──MCP──>  SolidCraft (headless)
 (Rust/OpenXR)               (Python)                        (the real CAD program)
                                  │
                                  └──HTTPS──>  Claude Haiku API
```

> Since Part 5 the server can attach to the normal SolidCraft app instead of a hidden copy (or drive Autodesk Fusion through an adapter). Parts 1 to 4 use the headless copy.

1. **The viewer** runs on the headset. It shows the model over the real room (passthrough), lets you select and move things, and records your voice when you hold a trigger.
2. **The server** is one Python script on the Mac. It finds the headset on the local network, sends it the model, turns your speech into text with whisper, builds the request for Claude, runs Claude's tool calls, and sends back the changed model.
3. **SolidCraft** runs on the Mac with no window (`solidcraft-cli mcp --headless`). It is driven through MCP, the same command interface Claude uses. It opens the STEP file, executes commands such as colour, scale, hole and fillet, and exports a mesh.
4. **Claude (Haiku)** gets your words, what you have selected or are pointing at, and a summary of the design. It decides which SolidCraft commands to run and answers in one sentence.

One voice request, step by step: you speak and select in the headset → the audio goes to the Mac → whisper turns it into text → text plus selection go to Claude → Claude calls SolidCraft commands through the server → the changed model is exported as a mesh → it appears in the headset a few seconds later.

Speech-to-text is local, so no audio leaves your network. The only thing that leaves the Mac is the connection to Anthropic's API. In our testing a typical request cost a fraction of a cent.

---

## What you'll need

- **A Quest 3** (Quest 2 and Pro are listed as supported devices in the app's manifest, but only a Quest 3 was tested), with **Developer Mode** switched on (create a Meta developer account, then switch it on in the Meta Horizon phone app), a USB-C cable, and `adb` on your Mac.
- **A Mac** and about 20 GB of free disk space (Rust, the Android SDK and NDK, build products).
- **The SolidCraft command-line program** (`solidcraft-cli`) from the SolidCraft releases. This guide used v0.41.0. Check that `solidcraft-cli mcp --headless` starts.
- **whisper.cpp** (`whisper-cli`, for example from Homebrew) and an English model file such as `ggml-small.en.bin`.
- **An Anthropic API key** for the Claude tier, stored in the Mac's keychain, with a small prepaid balance or spending limit. Our testing for this whole project cost a few cents with Claude Haiku.
- **Claude Code** on a plan you're happy with.
- A **STEP file** of your own design to look at.

> **Heads-up on permissions:** Claude asks before running commands. Read each one before approving it, especially anything that installs software, uses `adb`, or contacts the network.

---

## Part 1 — The viewer on the headset

### Step 0 — Install Claude Code and describe the project

1. Install Claude Code from <https://claude.com/claude-code> and sign in.
2. Make an empty folder for the project and open it in Claude Code.
3. A good first message:

> I want to build a viewer for my CAD models that runs on a Meta Quest 3 as a native OpenXR app written in Rust, showing the model in my room through passthrough. A working example exists at https://github.com/ukdavewood/solidcraft-quest-viewer if you want to read it for ideas. Check what I have installed (Rust, the Android SDK and NDK, a JDK, adb, cargo-apk) and tell me what is missing and which parts I have to do by hand. Don't change anything yet.

- **Human-only bits:** installing the Android command-line tools, switching on Developer Mode on the headset, and tapping "Allow USB debugging" in the headset the first time you connect it.
- **Claude will** propose `cargo-apk` for building the Android package, the `aarch64-linux-android` Rust target, and Meta's OpenXR loader library (a `.so` you download from Meta's OpenXR Mobile SDK and keep out of your repository).
- **Verify:** `adb devices` lists your headset.

### Step 1 — A first scene in the headset

> Make a minimal native OpenXR app for the Quest 3 in Rust (the openxr crate, Vulkan through ash, android-activity) that shows one triangle in front of me, with the room visible behind it. Build it with cargo-apk, sign it with a test key, install it with adb, and show me the plan first.

- **Claude will** write the OpenXR instance and session set-up, the Vulkan render loop with multiview, and a manifest with the VR and passthrough settings. It builds an APK and installs it.
- **Verify:** the triangle floats in front of you over the real room. It appears in the headset under Library, in the **Unknown Sources** filter.
- Expect problems here. Report them as you see them ("black screen", "app closes when I take the headset off"). Real fixes in our run: the app must submit an empty frame while the head pose is not valid after the headset wakes up, and the OpenXR instance must be created with the Android activity (not the application context), or Horizon OS shows its loading scene instead of your frames.

### Step 2 — Load your model

> Convert the design to a mesh with a body list, load it in the viewer, scale it to a sensible size, and let me grab it with the grip buttons to move, turn and resize it. Draw edges as lines.

- **Claude will** pick a simple mesh file format, write a converter on the Mac (SolidCraft can export OBJ) and show the model with lighting, per-body colours and outline edges.
- **Verify:** you can pick the model up and put it down. Ask for the right controller to show a pointing ray and for the part under the ray to be highlighted.

### Step 3 — A panel and a pointer

> Add a floating panel for buttons that follows my left hand, with tabs, icons and text, a pointing ray from the right controller, and the trigger to select the part under the ray. Draw text without any UI library (a small bitmap font is fine).

- **Claude will** build the panel from coloured quads and a bitmap font it generates itself. Test every button in the headset; this is where most of the feedback rounds happen.
- **Tips we learned:** ask for bigger text and gaps between list rows (small rows are easy to mis-hit), a panel that stays put when you put the controller down, and tabs with both an icon and a word.

### Step 4 — Get the model from the Mac over Wi-Fi

> The headset should find the Mac by itself: the Mac broadcasts a small UDP beacon, and when the model version in the beacon changes, the headset downloads the new model over HTTP. Write the Mac side as a Python script `serve.py` with no extra packages.

- **Claude will** write the beacon (a UDP broadcast on a fixed port every second), an HTTP endpoint for the model file, and the downloading thread in the viewer.
- **Verify:** change the model on the Mac and watch it replace itself in the headset.

---

## Part 2 — The server and the AI

### Step A — Drive SolidCraft by MCP

> Start `solidcraft-cli mcp --headless` as a subprocess from `serve.py` and talk to it with MCP (JSON-RPC over stdin and stdout). Open my STEP file, export an OBJ, convert it and send it to the headset. Show me the tools the CLI offers first.

- **Claude will** list the CLI's tools (for example `open`, `execute`, `inspect_design`, `body_topology`, `measure`, `export`, `undo`, `describe_command`) and write a small client for them.
- **Verify:** run one command by hand, such as making a body red, and watch the mesh update in the headset.

### Step B — Speech

> Hold the left trigger to record my voice in the headset, send the audio to the Mac, turn it into text with `whisper-cli`, and show what it heard in the panel. Give whisper a list of CAD words so it hears "fillet" and "chamfer".

- **Claude will** record with the Android audio API, send a WAV file, and call `whisper-cli` on the Mac with a prompt of vocabulary words.
- **Human-only bit:** the app has to ask for the microphone permission itself; the Quest shows a system prompt.
- **Expect mishearings.** Our recordings produced "please" as "breeze" and "hole" as "whole". The AI often copes, because it sees the design and the selection.

### Step C — The Claude tier

> Send the transcript to Claude Haiku with SolidCraft's tools. Give it the selected parts, the point I'm pointing at and a short summary of the design. Run its tool calls against SolidCraft, loop until it's done, and reply with one sentence.

- **Claude will** write the API client (use the official Anthropic SDK if you prefer; ours used plain HTTP from Python's standard library), the tool-use loop, the system prompt, and prompt caching so the same prompt is cheap to re-send.
- **Key storage:** keep the API key in the Mac's keychain, not in a file in the project.
- **Verify:** say "make this part blue" while pointing at a part.

### Step D — Guards against bad behaviour

Real problems from our testing, and what fixed them. Ask Claude for each in plain words.

- **The AI said "Done" when nothing changed.** It had run out of output tokens halfway through 90 separate commands. We added a stop-reason check and two server-side helper tools: `find_bodies` (look bodies up by name pattern) and `colour_each` (give each body its own colour in one call).
- **"The whole ship" changed only the selected parts.** The server now expands `"*"` and name patterns such as `flat_sail*` into body lists, so the AI never has to write hundreds of names.
- **A big request could burn money.** Anything over a token limit now waits for your OK in the panel (Go on / Stop) and shows an estimated cost before it runs.
- **Undo.** The panel's Undo undoes the whole last request (all its steps), not one step. A spoken "undo" does the same.
- **Follow-ups.** The last few requests are put in the next prompt, so "do the same to each one" works.
- **Repeat without the AI.** The server records the commands the AI used, so a saved request can be repeated with no API call.
- **Cancel.** A request ending in "cancel" or "scratch that" is dropped, which helps when the transcription is wrong.

### Step E — Keep casual access off your server

The server listens on your local network and can change your design and spend your API credit. Add an access code:

> Make `serve.py` create a random access code the first time it runs, keep it in a file only I can read, and refuse every request that doesn't carry it. The headset should ask me to type it once and remember it.

- **Claude will** add a header check, a prompt in the headset (with the on-screen keyboard), and a deploy script that can push the code for you.
- Traffic is **not encrypted**, so the code travels in the clear. It stops other devices and accidents, not a determined person on the same network, so only use this on a network you trust.

---

## Part 3 — Making it feel like CAD

### Selecting faces and edges, push/pull

> Let me switch between picking parts, faces and edges. Highlight only the face or edge under the pointer. For edges, pick the whole CAD edge (a circle, a straight edge), not one mesh segment. When I speak, send the picked faces or edges with the request and match them to SolidCraft's own face and edge numbers.

- **Claude will** group the mesh into face patches and follow smooth joins to find whole edges, then match them on the server using SolidCraft's `body_topology` (start, middle, end and length), so "fillet these edges 1 mm" refers to exactly what you picked.
- Push/pull a face by distance is the same idea: you drag a face and the server calls SolidCraft's push/pull command. Some faces fail inside SolidCraft's geometry kernel (a bug worth reporting upstream); the server shows the reason.

### Setting the model down on a real surface

> Add a Set down button: point at a table or the floor, show where the model will land, and on the trigger set it on that surface, level, at its real size.

- **What happened for us:** the headset's runtime did not list the usual scene and anchor extensions (`XR_FB_scene`, `XR_FB_spatial_entity`), even with the scene permission granted. It did list `XR_META_environment_raycast`, which casts a ray against the real room. That extension was newer than the Rust OpenXR bindings, so we asked Claude to write the structures and function calls by hand from the OpenXR header, and it worked. The raycast only returns hits inside the headset cameras' view, so look towards the surface as well as pointing.
- **Real size:** designs are in millimetres, so a small part is set down at its true size; a large one (a ship, a building) is scaled to a tabletop. A Size button steps 0.5×, 1×, 2×, 4× of the file's size (our models are 1:1000, so ×2 gives 1:500).
- **Near surfaces the scan misses:** the room scan did not find surfaces very close to the headset, such as the front edge of a desk, so the marker fell back to a point in mid-air. We asked Claude to remember the height of the last flat, upward-facing surface the raycast found and, when the ray misses but points downwards, to use the same height. Aim at the far part of the desk first and the front then works.
- **Keeping it there:** we saved the placement relative to the headset's stage space. Meta's documentation advises spatial anchors for this, and says the stage origin can move for apps that run without a boundary, so treat it as unreliable until you have tested it across restarts.

### A keyboard and mouse

A Bluetooth keyboard and mouse can be paired with the Quest and used in the app. Mouse movement needs Android **pointer capture**; a mouse the app has not captured sends nothing. We first asked for capture only after a mouse event arrived, which never happens, so the mouse was silent until we asked for capture at start-up. Shortcut keys (Ctrl+Z undo, F1 help) and an on-screen keyboard are in the app.

With the mouse, left click selects, the right button turns the model, the wheel resizes it, and **Shift with the right button or the wheel moves it** (sideways and up/down, or nearer and further). The app only sees the keys and mouse events it is sent, so Shift has to reach it as its own event.

**Use the keyboard and mouse that are connected to your Mac:** instead of pairing them with the headset, you can ask Claude to relay them. A **Mac keys** button in the app asks the server to capture the Mac's keyboard and mouse and send them to the headset over UDP, so nothing is paired and nothing has to be switched back.

> Add a Mac keys button. When it is on, capture the Mac's keyboard and mouse with a macOS event tap in the server (swallow them on the Mac), send them to the headset as UDP datagrams carrying the access code, and treat them like a paired keyboard and mouse. Give them back when I press the button again, when I press Tab in the headset, when the headset loses focus or is taken off (a heartbeat from the headset every second; let go after four seconds of silence), and on a fixed chord on the Mac (Ctrl+Option+Esc) as a way out.

- **What to know:** macOS asks for Accessibility and Input Monitoring permission for the program that runs the server. A first version captured nothing because the event loop ran in the wrong mode (RunInMode needs the default mode, not the common-modes name). Swallowing mouse-move events doesn't stop the pointer moving on the Mac, so also detach the pointer from the mouse while captured (`CGAssociateMouseAndMouseCursorPosition`). With passthrough on, taking the headset off only loses focus rather than ending the session, so release on loss of focus rather than waiting for the session to go idle.

---

## Part 4 — Parametric sketches

A model built from a sketch and dimensions changes when you change a dimension. This part shows the sketch in the room, with its dimensions and constraints, and lets you change a dimension by selecting it and saying a number.

### How it works, in short

1. A **parametric model** is built from a sketch (an outline, holes and a circle for a boss) whose sizes are fixed by dimensions, plus extrusions made from the sketch. Changing a dimension rebuilds the sketch and everything built from it.
2. The headset asks the Mac for the **design tree** (SolidCraft's timeline: the sketches and features, in order) and shows it as a list. Pick a sketch.
3. The server asks SolidCraft for the sketch's curves and draws them on the sketch's plane over the model. In our setup SolidCraft's `inspect_design` lists the sketches and the timeline, and `sketch.inspect` and `sketch.export_dxf` give the curves, points, constraints and dimensions.
4. **Dimensions** appear as labels with dimension lines and arrowheads (for example `plate_w = 80`), and **constraints** (horizontal, vertical, coincident and so on) as small boxes. Select one or more with the pointer; they turn orange.
5. Hold the trigger and say, for example, "change to 18". The server sends Claude the selected items with the named parameter behind each one. Claude changes the parameter with SolidCraft's `set_parameter` command, SolidCraft rebuilds, and the headset shows the new shape and redraws the sketch a moment later.

### A test model with one sketch

Ask Claude to build a small model for testing:

> Build a test part in SolidCraft through MCP with ONE sketch on the XY plane: an 80 x 50 mm plate outline, two 8 mm holes dimensioned from the corners (the second from the opposite corner, so they stay symmetric), a 24 mm boss circle centred on the plate with plate_w / 2 and plate_h / 2, and a 10 mm bore circle concentric with it. Give every dimension a named parameter (plate_w, plate_h, hole_d, hole_x, hole_y, boss_d, bore_d). The sketch should be fully constrained. Then extrude the outline profile as the plate and join the boss ring profile as the boss, leaving the holes and bore open. Check that changing plate_w rebuilds the model.

- **What happened for us:** the sketch had 17 constraints and no free movement left, and changing one parameter recomputed three features. Remember that SolidCraft numbers the profiles of a sketch (the outline with its holes, each hole, the boss ring, the bore); extrude by profile index.

### Showing the sketch and its dimensions

> Add a Sketches button that lists the design tree from the Mac and lets me pick a sketch to show over the model, one at a time, drawn as thin lines even when they are inside the solid. A pressed button hides it again. Refresh the sketch only when the design changes, not on a timer.

> Draw each dimension as a label with a dimension line and arrowheads, and each constraint as a small box. Make the text and the arrowheads a fixed size on screen, whatever the size of the model, and hide them when there is no room: arrowheads first, then the dimension. Let me point at a label, select it, and send the selected ones with my next request.

- **Refresh:** the Mac already announces each new design version, so the headset fetches the sketch again only when that version changes, not on a timer. We first used a timer, then replaced it.
- **Keep the server stateless about the view:** the headset should say which sketch it is showing with each request. When the server remembered the choice, opening another model made the two disagree and the labels went stale.
- **Draw order:** lines are drawn without a depth test so a sketch that lies inside the solid is still visible.
- **Deselect after a request:** selected items go with the next request, so a dimension you forgot to deselect gets changed along with the new one. We deselect them once the request has been answered. The AI says when two dimensions were selected and it had to guess.

### Keep each CAD program behind an adapter

The headset only needs a few things from the CAD program: a mesh, a list of bodies, a design tree, a sketch's curves, dimensions and constraints, and a way to run an edit. We kept the SolidCraft specifics in one adapter file on the server, with the contract written down, so the same viewer could later be pointed at another program (we plan a Fusion version). Ask Claude to keep program-specific code in one place from the start; it makes guides and sample code easier to share per tool.

---

## Part 5 — The real SolidCraft app, the Mac screen and Fusion

Parts 1 to 4 used SolidCraft with no window. This part adds four things: the server attaches to the normal SolidCraft app so the same design is open in both places, the headset shows a window onto the Mac screen, the design tree becomes an editable timeline, and a first version drives Autodesk Fusion instead. All four were built on 19 November 2026 and tested lightly, on one setup.

### Attach to the running SolidCraft app

SolidCraft 0.41.0 (the desktop app) opens a local control port, and its command-line program can connect to it instead of starting its own hidden copy: `solidcraft-cli mcp --connect 127.0.0.1:<port> --token <token>`. The port and token are files in the app's control folder. (The older 0.28.0 app did not have this.)

> Add a live mode to `serve.py` that attaches to the SolidCraft app that is already running, using the control port and token the app writes in its control folder, instead of starting a headless copy. Use the same design the app has open. Show me what you find before you change anything.

- **Claude will** find the port and token, connect over MCP, and export the open design to the headset as before.
- **Verify:** change the model by voice in the headset and watch the same change appear in the app window on the Mac.
- **Edits made by hand in the app:** two ways to get them into the headset. The server checks the design every couple of seconds and re-sends it when it changed (written, lightly tested); and a **Refresh** button in the headset's Tools tab asks the server to re-read the design now. The panel says "Model loaded." when the new model has arrived. A refresh can also be started from the Mac, so no cable is needed.

### A window onto the Mac screen

> Add a window in the headset that shows the left half of my Mac's screen, hanging behind the model, with the real Mac pointer drawn in it. Buttons for on and off, which part of the screen (whole, left half, right half, zoomed around the pointer) and a zoom.

- **Claude will** grab the screen on the Mac, compress it, and send it to the headset as a picture that the app draws as a flat window in the room.
- **Human-only bit:** macOS asks for **Screen Recording** permission for the program that runs the server. Without it the picture shows only the desktop background.
- **Real gotchas from our run:**
  - **The window covered everything.** A flat window drawn as an OpenXR composition layer goes on top of the 3D scene unless you order it first. Draw it before the main scene layer and the model, panel and pointer stay in front.
  - **The pointer was missing.** The screenshot tool's pointer option did not include it. The server now sends the pointer position separately (about every 30 ms) and the headset draws the arrow into the picture, which also makes it more responsive.
  - **Typing felt slow.** Running a screenshot program for every picture took too long. Asking for the pixels in-process through macOS's graphics library brought a picture down to roughly a tenth of a second.
- **Not done:** the window is view-only. Clicking in it with the headset pointer and typing into the Mac from the headset are not built (the Mac keys relay in Part 3 is separate).

### A timeline you can edit

> Rename the Sketches list to Timeline and show every step of the design in order, with an icon for each kind (sketch, extrude, fillet, chamfer) and the number it uses (a distance, a radius, an angle). Let me select a step and change its number with a minus and a plus button.

- **Claude will** read the design tree and each step's inputs from SolidCraft (`timeline.inspect` lists a step's own inputs, including ones that `inspect_design` does not show, such as a fillet radius), and add a `set_parameter` call for the minus and plus buttons. A step is 1 mm (0.5 mm under 5 mm), 5 degrees for angles, never below 0.1.
- **Verify:** press plus on a fillet and watch its radius grow in both the headset and the app; Undo in the panel undoes it.
- **Limits:** only steps whose number is a plain value or a named parameter can be changed. Steps driven by a formula are shown but not editable yet. Extrude rows needed a second pass before their number showed.
- **Next ideas, not built:** tell the AI which step is selected so "make it 20" works, and mark on the model where a step is.

### A first Fusion version

Because the headset only talks to the server, the server can be pointed at a different CAD program. We wrote a second adapter that views and edits an Autodesk Fusion design.

> Add a Fusion adapter that implements the same contract as the SolidCraft one: export a mesh, list the bodies, the design tree, and run an edit. Drive Fusion through its Python API and keep every Fusion-specific line in that one file.

- **What worked:** viewing a Fusion model in the headset, filleting every edge, changing the body colour, and pulling the result back with Refresh. The AI used the same voice requests.
- **Gotcha (important):** a script that changes a Fusion document must put its work inside a `run(context)` function. The same changes made at the top level of the script were silently rolled back, with no error. Read-only checks can end by raising an error with their result.
- **Fusion memory grows.** After many scripted edits Fusion's memory use climbed into the tens of gigabytes; it only dropped when the document was saved, closed and reopened. If several tools drive the same Fusion, send every script through one runner that takes a lock and refuses to start when memory is high.
- **Not done:** undo through the headset for Fusion edits, and editing a Fusion step's number from the timeline.

---

## What to expect

- **Time:** about three days of back-and-forth, most of it short rounds of "I tried it, here's what I saw". The first scene in the headset took the longest.
- **Cost:** Claude Code time on your plan; for the AI itself, in our testing a fraction of a cent per request with Haiku, and repeated saved requests cost nothing.
- **Quality:** it is an experiment. The AI sometimes picks the wrong interpretation, voice recognition is not perfect, some SolidCraft operations fail on some faces, and the Mac does all the real work, so it has to be on and on the same network.

---

## Troubleshooting

### The app installs but doesn't start from `adb`
The Quest only opens a window for an app when the headset is on and awake. Put the headset on and start the app from Library → Unknown Sources.

### A black screen, or "the app closes when I take the headset off"
Usually the head pose is not valid yet after a wake-up. Ask Claude to submit empty frames until tracking is valid and to log errors from the end of each frame.

### The extension I need is "not present"
Check what the headset actually offers (log the list of extension names at start-up) before you build on one. Ours listed no scene or anchor extensions. A request for a name that isn't listed makes `xrCreateInstance` fail.

### The mouse does nothing
It needs pointer capture. Pair it, then ask Claude to request capture as soon as the app has focus.

### The AI says "Done" but nothing changed
Ask Claude to log each tool call and the stop reason. A cut-off answer or an expensive loop is the usual cause.

### The keyboard stays captured, or Mac keys won't switch on
The Mac only releases the keyboard and mouse when the headset says so, goes quiet for four seconds, or you press Ctrl+Option+Esc on the Mac. Check that macOS has allowed Accessibility and Input Monitoring for the program that runs the server. A pressed Tab in the headset also gives them back.

### A sketch won't clear, or its labels show an old value
Press the Sketches button again (it reads Hide sketch while one is on view). If a label is stale, the headset and server disagree about which sketch is shown or which design version it has; ask Claude to make the headset send the sketch id with every request and to refetch when the design version changes.

### Photos and recordings
The Quest's own camera can take photos and record video. Microphone audio is only recorded if the recording settings allow it, and while the app is using the microphone for speech the recording may come out quiet or silent; plan to add a voice-over. Recording and photos can't be started from your own app (the camera service is protected), but `adb` can take a photo over a cable, which a server button can use. The Quest records only one of casting or recording at a time, so stop casting before you record.

---

## Sharing your work

- **Keep secrets out of the repository:** the signing keystore, your API key and the access code file.
- **Don't publish the OpenXR loader** library; list where to download it instead.
- Check any recording before you share it; the panel can show an estimated API balance.
- Say clearly that it is unofficial and experimental, and credit SolidCraft.

## Community knowledge and contributing

This is an open experiment, not a maintained product. The aim is a shared notebook that people and their AI assistants can learn from:

- **Share what you learned**, not necessarily code: a different approach, a Windows or other-headset attempt, a prompt that worked better, a workaround, a video, or something that failed. Use GitHub **Discussions**, **Issues** or a **pull request** that adds a Markdown note to `community-knowledge/` (a template is there).
- **Contributions don't have to be merged.** Open and closed issues, discussions and even unmerged pull requests are part of the record. There is no commitment to review, merge, build, maintain or support anything.
- **Telling your AI to use it:** the repository's [AGENTS.md](../AGENTS.md) and [starter prompts](../ai-prompts/starter-prompts.md) ask an assistant to read this guide and the sample code, then search the issues, discussions and pull requests (including unmerged ones), and to say which findings are tested, untested, abandoned or in conflict before it starts building.

See [CONTRIBUTING.md](../CONTRIBUTING.md) and the [community-knowledge folder](../community-knowledge/README.md).

## Sample code

A cleaned-up copy of the viewer and server from this project is published as a standalone repository: <https://github.com/ukdavewood/solidcraft-quest-viewer> (licensed MIT or Apache-2.0, like SolidCraft's code). It is intended as something to read and start from, not a finished product. Meta's OpenXR loader library is not included; the README says where to get it.

> **How current the sample code is:** the repository was last updated on **17 November 2026** (commit `b5a5e35`, "Sample code: Quest 3 viewer and AI remote for SolidCraft"). It covers Part 1 to Part 3 as they were on that day: the viewer, the server, voice, faces, edges and push/pull, setting the model down with the room raycast, sizing, photos and the access code. It does **not** include the later work in this guide: the Mac keys relay and Shift-move (Part 3), the near-surface set-down fix, everything in **Part 4** (sketches, dimensions, constraints, the CAD adapter and the one-sketch test model) and everything in **Part 5** (live SolidCraft, the Mac screen window, the editable timeline and the Fusion adapter). **No further code releases are planned**: the sample stays as it is. For the rest, ask Claude to build it from the descriptions in this guide, using the repository for the structure.

**A tip for Claude Code:** you can ask Claude to read the repository first, so it learns the structure and the problems already solved, then build your own version. For example:

> Read <https://github.com/ukdavewood/solidcraft-quest-viewer> (the README and `src/` and `tools/serve.py`). It is a working Quest 3 viewer and AI server for SolidCraft as it was on 17 November 2026; it does not yet have the Mac keys relay, the sketch and dimension view, the CAD adapter, or the live SolidCraft, Mac screen window, timeline and Fusion work described in Parts 3 to 5 of the guide. Summarise how it is organised. Then search the repository's issues, discussions and pull requests (including unmerged ones) and tell me what other people have tried, marking each as tested, untested, abandoned or conflicting. Then help me build my own version for my own models and setup, borrowing what is useful and adding those features from the guide. Don't copy anything you don't understand.

## Resources

- SolidCraft: <https://github.com/tangency-labs/solidcraft>
- OpenXR specification and the `XR_META_environment_raycast` definition: <https://registry.khronos.org/OpenXR/>
- Meta Horizon OS developer documentation: <https://developers.meta.com/horizon/>
- Claude Code: <https://claude.com/claude-code>
- Anthropic API documentation: <https://platform.claude.com/docs/>
- Sample code for this guide: <https://github.com/ukdavewood/solidcraft-quest-viewer>
- Demo videos (playlist): <https://www.youtube.com/playlist?list=PLS769WnSAR1A>
- Companion guide: [Building SolidCraft for iPad with Claude Code](Building-SolidCraft-for-iPad-with-Claude-Code.md)
