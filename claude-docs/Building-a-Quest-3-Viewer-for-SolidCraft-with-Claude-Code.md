# Building a Quest 3 Viewer and AI Remote for SolidCraft with Claude Code

You have a CAD design on your Mac and a Meta Quest 3. You want to stand in your room, see the design at table size over the real furniture, pick parts with the controllers, and change the design by talking to it ("make the selected sails red", "fillet this edge", "scale the whole ship by half"). This guide shows how to get there by asking Claude Code for each piece: a small OpenXR app for the headset, a server on your Mac that drives SolidCraft, and an AI that does the editing.

This is a real-world write-up of an **experiment**. It was built end to end in about three days (15 to 17 November 2026) on a Mac and a Quest 3, starting from the open-source [SolidCraft](https://github.com/tangency-labs/solidcraft) command-line program (v0.41). You don't need to be a programmer, but you do need to be comfortable approving steps, wearing the headset to test, and describing what you see.

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
- **Keeping it there:** we saved the placement relative to the headset's stage space. Meta's documentation advises spatial anchors for this, and says the stage origin can move for apps that run without a boundary, so treat it as unreliable until you have tested it across restarts.

### A keyboard and mouse

A Bluetooth keyboard and mouse can be paired with the Quest and used in the app. Mouse movement needs Android **pointer capture**; a mouse the app has not captured sends nothing. We first asked for capture only after a mouse event arrived, which never happens, so the mouse was silent until we asked for capture at start-up. Shortcut keys (Ctrl+Z undo, F1 help) and an on-screen keyboard are in the app.

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

### Photos and recordings
The Quest's own camera can take photos and record video. Recording and photos can't be started from your own app (the camera service is protected), but `adb` can take a photo over a cable, which a server button can use. The Quest records only one of casting or recording at a time, so stop casting before you record.

---

## Sharing your work

- **Keep secrets out of the repository:** the signing keystore, your API key and the access code file.
- **Don't publish the OpenXR loader** library; list where to download it instead.
- Check any recording before you share it; the panel can show an estimated API balance.
- Say clearly that it is unofficial and experimental, and credit SolidCraft.

## Sample code

A cleaned-up copy of the viewer and server from this project is published as a standalone repository: <https://github.com/ukdavewood/solidcraft-quest-viewer> (licensed MIT or Apache-2.0, like SolidCraft's code). It is intended as something to read and start from, not a finished product. Meta's OpenXR loader library is not included; the README says where to get it.

**A tip for Claude Code:** you can ask Claude to read the repository first, so it learns the structure and the problems already solved, then build your own version. For example:

> Read <https://github.com/ukdavewood/solidcraft-quest-viewer> (the README and `src/` and `tools/serve.py`). It is a working Quest 3 viewer and AI server for SolidCraft. Summarise how it is organised, then help me build my own version for my own models and setup, borrowing what is useful. Don't copy anything you don't understand.

## Resources

- SolidCraft: <https://github.com/tangency-labs/solidcraft>
- OpenXR specification and the `XR_META_environment_raycast` definition: <https://registry.khronos.org/OpenXR/>
- Meta Horizon OS developer documentation: <https://developers.meta.com/horizon/>
- Claude Code: <https://claude.com/claude-code>
- Anthropic API documentation: <https://platform.claude.com/docs/>
- Sample code for this guide: <https://github.com/ukdavewood/solidcraft-quest-viewer>
- Companion guide: [Building SolidCraft for iPad with Claude Code](Building-SolidCraft-for-iPad-with-Claude-Code.md)
