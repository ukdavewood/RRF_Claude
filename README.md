# Claude Code Guides

Practical, real-world guides to getting things done with **[Claude Code](https://claude.com/claude-code)** — written for capable non-specialists, starting from a computer with no developer tools installed. Each guide follows the same approach: **you describe what you want in plain English, and Claude works out the specifics.**

## Guides

- **[Building (and Enhancing) RepRapFirmware with Claude Code](claude-docs/Building-RRF-with-Claude-Code.md)** — compile RepRapFirmware for Duet3D boards on Windows or macOS, then add your own features to the firmware, without needing to be a programmer.
- **[Building a Forum Post Capture Chrome Extension with Claude Code](claude-docs/Building-a-Forum-Post-Capture-Extension.md)** — build a Chrome extension that captures a forum thread (posts and images) to a local database and exports it as Markdown, a ZIP, or HTML — for offline keeping and handing to Claude for analysis.
- **[Building a 2-Reader NFC Filament Sensor](claude-docs/two-board-nfc-reader-build-guide.md)** — wire two PN5180 NFC readers to an ESP32 and get an AI coding assistant to write the firmware, reading both Bambu Lab and OpenPrintTag filament spool tags.
- **[Automating BambuStudio Plate Labels with Claude Code](claude-docs/Automating-BambuStudio-Plate-Labels-with-Claude-Code.md)** — get Claude Code to build a macOS script that drives the real BambuStudio app to add a name label to every plate of a multi-plate project, optionally from a Stream Deck button.
- **[Creating 4K Fusion Animations with Claude Code](claude-docs/Creating-4K-Fusion-Animations-with-Claude.md)** — get Claude Code to build a small Autodesk Fusion add-in and naming scheme that turns a 3D model into 4K animation frames (parts appearing group by group, camera orbits, exploded views), ready to assemble into a video.
- **[Building SolidCraft for iPad with Claude Code](claude-docs/Building-SolidCraft-for-iPad-with-Claude-Code.md)** — get Claude Code to port an open-source Rust CAD program to run natively on an iPad, then optionally add an AI assistant that drives it by text or voice (Apple's on-device model, Claude online, or a small offline model), run it on an iPhone and the iPhone Duo simulator (UIScene lifecycle for the iOS 27 SDK, simulator build, Duo simulator quirks), and use an iPhone or a browser on an old iPad as its remote keyboard (SolidCraft Remote).

- **[Building a Quest 3 Viewer and AI Remote for SolidCraft with Claude Code](claude-docs/Building-a-Quest-3-Viewer-for-SolidCraft-with-Claude-Code.md)** — an experiment: get Claude Code to build a Meta Quest 3 viewer that shows a CAD model over your room, plus a Mac server that lets you select parts with the controllers and edit the design by voice through SolidCraft, including parametric sketches and dimensions you can select and change by voice, and (Part 5) a live link to the normal SolidCraft app, a window onto the Mac screen, an editable timeline and a first Autodesk Fusion version. Companion to the iPad guide, with [sample code](https://github.com/ukdavewood/solidcraft-quest-viewer) (frozen at 17 November 2026) and a [demo video playlist](https://www.youtube.com/playlist?list=PLS769WnSAR1A).

*More guides are in preparation (GCode post-processors for simulation).*

## Community-driven AI development (an experiment)

These guides show how something was built with an AI coding assistant. Instead of shipping a finished, maintained product, the idea is that you give your assistant a guide (and any sample code) and have it build **your own version** for your own setup, then share what you learned so the next person, and their assistant, starts further along.

- **Contribute knowledge, not necessarily code:** alternative approaches, other platforms, prompts that worked, workarounds, videos, failures. See [CONTRIBUTING.md](CONTRIBUTING.md) and the [community-knowledge](community-knowledge/README.md) folder.
- **Issues, discussions and pull requests are the knowledge base.** They don't have to be merged to count. There is no commitment to review, merge, build or support anything.
- **Written for AI assistants too:** [AGENTS.md](AGENTS.md) tells an assistant to read the guide, then search the community's issues, discussions and pull requests, and sort findings into tested, untested, abandoned and conflicting before it starts. Ready-made [starter prompts](ai-prompts/starter-prompts.md) are provided.
- **No official builds.** Sample code is shared as read-and-start-from references, frozen at the date each guide states.

## About

These guides are reviewed, versioned, and published from a dedicated documentation project. They share a common house style and structure — described in **[Writing Claude Code Guides](claude-docs/Writing-Claude-Code-Guides.md)** — so each one reads the same way.
