# Building a Forum Post Capture Chrome Extension with Claude Code

You post a lot in a technical forum — build logs, troubleshooting, configs — and
you want your own copy: the **whole thread, including images**, saved locally so
you can read it offline, keep it as the forum changes, and hand it to **Claude**
for help (writing it up, or working out what went wrong). This guide walks you
through getting Claude Code to build exactly that: a Chrome extension that
captures a forum thread to a local database and exports it as Markdown, a ZIP, or
a self-contained HTML page.

## The core idea

You do **not** need to be a developer. You describe what you want — "capture the
whole thread, get the images, make something I can hand to Claude" — and Claude
Code writes the code, explains it, and helps you test it. Most instructions below
are phrased as *things to ask Claude*, not code you write yourself. The most
useful parts of this guide are the **real gotchas** near the end — the things
that only show up once you point the extension at a live forum.

## Before you start — is this OK to do?

- **Capture your own content, for personal use.** Archiving threads you took part
  in, for offline reading and private analysis, is the intent here. Don't
  republish other people's posts, and check the forum's Terms of Service.
- **Be a polite guest.** Scraping means loading pages and images from someone
  else's server. Build in **small random delays** and don't hammer it — a
  capture that takes a couple of minutes is fine; 500 requests a second is not.
- **It's read-only.** This extension only reads pages you can already see in your
  browser. It doesn't post, log in on your behalf, or change anything on the forum.

## What you'll end up with

A **Manifest V3** Chrome extension with a popup that lists the threads you've
captured. Click **Capture** on a thread you're viewing; it scrolls through,
saves every post and image to a local database (**IndexedDB**), and lets you
export the thread three ways — a **ZIP** (Markdown split into parts + a folder of
full-size images), a single **Markdown** file (images embedded, shrunk to upload
to Claude), or a **HTML** page (full-resolution, nicely formatted).

## Prerequisites

- A **Chromium browser** (Chrome, Edge, Brave…) in developer mode.
- **Node.js** (for building the extension) — Claude will tell you if it's missing.
- An account on the **forum** you want to capture, and a thread open in your browser.

## Step 0 — Install Claude Code and open your project

Install **Claude Code** and open a new empty folder for this project. See the
official docs: <https://claude.com/claude-code>.

Everything below is a prompt you type to Claude Code in that folder. When a step
shows a **>** block, that's the language to give Claude — adapt it to your forum.

## Step 1 — Scaffold the extension

> Set up a Manifest V3 Chrome extension in TypeScript, built with webpack. It
> should have a background service worker, a content script that runs on my forum,
> a popup UI, and an IndexedDB storage layer. Give me an `npm run build` that
> produces a loadable `dist/` folder.

- Claude will create the `manifest.json`, build config, and a skeleton of the
  service worker, content script, popup, and storage modules.
- **Verify:** run `npm run build`, then in your browser open `chrome://extensions`,
  turn on **Developer mode**, click **Load unpacked**, and select `dist/`. The
  extension should appear with no errors.

**Tip:** whenever you change code, **bump the `version` in `manifest.json`**. The
version shown on `chrome://extensions` is how you confirm you reloaded the new build.

## Step 2 — Work out what platform your forum runs on

Forums aren't all the same underneath, and the scraper depends on the platform.
The two most common are **Discourse** and **NodeBB**.

> My forum is at <URL>. Look at a thread's page structure and tell me what forum
> software it runs (Discourse, NodeBB, phpBB, XenForo, …), and what HTML element
> and attributes identify a single post, its author, timestamp, and body.

- Claude will inspect the page (you can paste the URL, or the page's HTML) and
  identify the post container and the fields to read from it.
- **Verify:** the selectors Claude names actually match posts when you look at the
  page with your browser's DevTools (right-click a post → Inspect).

## Step 3 — Capture the visible posts of one thread

Start small: grab what's on screen, before worrying about scrolling.

> Write a scraper for <platform> that extracts every post currently in the page:
> id, author, timestamp, the post's HTML body, and the image URLs in it. Wire a
> "Capture visible" button in the popup that runs it and logs what it found.

- Claude will add a scraper plus a debug button that reports how many posts and
  images it saw.
- **Verify:** open a thread, click the button, and check the console — the counts
  and a sample post should look right.

## Step 4 — Capture the *whole* thread by scrolling

Long threads aren't all in the page at once. This is where forums get tricky.

> Make the capture scroll through the entire thread, collecting posts as they
> load, with small random delays between scrolls. Handle forums that only keep a
> few posts in the DOM at a time. De-duplicate posts by id.

- Claude will add a scroll loop that collects posts across the thread.
- **Verify:** capture a long thread and confirm the post count matches the thread.

**The big gotcha here — "cloaking".** Discourse (and similar) *remove* off-screen
posts from the page as you scroll, to save memory. If you scroll in big jumps or
too fast, posts get skipped. Tell Claude:

> Scroll in steps no larger than the viewport, and pause long enough for posts to
> render before reading them, so cloaked/virtualised posts aren't skipped.

## Step 5 — Store posts and images locally, without duplicates

> Save captured threads, posts, and images to IndexedDB. Hash each image with
> SHA-256 and store each unique image once, even if it appears in several posts or
> is re-captured later.

- Claude will build the storage schema and image de-duplication.
- **Verify:** capture, then re-capture the same thread; the image count shouldn't
  balloon — identical images are stored once.

## Step 6 — Download the images reliably (the part that fights back)

Getting text is easy; getting **images** is where most of the real problems live.
Ask Claude to handle these, and keep them in mind when images come out missing:

> When downloading images: send the forum's cookies, only keep responses that are
> actually images, throttle with small random delays, and skip (don't retry
> forever) anything that hard-fails.

- **Cookies / Cloudflare:** many forums sit behind Cloudflare. A cookieless
  request gets an "Access denied" HTML page, which naïve code will happily save
  *as if it were an image*. Fetch with **credentials included** so the browser's
  clearance cookie goes along, and **check the content-type is an image**.
- **Images on a different host:** forums often serve uploads from a **CDN**
  (e.g. `cdn-forum.example.com`). That host must be in the manifest's
  **`host_permissions`**, or every image request is blocked. If images fail and
  the console shows the requests never completing, this is usually why.
- **Dead thumbnails:** some platforms show a resized thumbnail (`…-resized.jpg`)
  that **404s**, while the full-size original the thumbnail links to is fine.
  Capture the **original from the image's link**, not the thumbnail — you also get
  archival-quality images.
- **Match images to posts by one canonical URL.** The URL written in the saved
  post HTML and the URL you downloaded must be the *same string*, or the export
  can't line them up. Have the scraper rewrite each image to one absolute URL at
  capture time.

**Verify:** export a thread and open it — images should actually appear, not show
as broken links.

## Step 7 — Survive long threads (save as you go, and merge)

A 350-post thread can take minutes. If the whole capture is held in memory and
only saved at the end, one closed popup loses everything.

> Capture in blocks: as I scroll through, save each block of new posts straight to
> the database, so closing the popup doesn't lose progress. If I capture the same
> thread again, merge the new run into what's already stored rather than replacing
> it. Number posts by their position in the thread.

- Claude will make capture **incremental** (each block persisted immediately) and
  make re-captures **merge**, so you can finish a huge thread across several runs.
- **Verify:** start a big capture, close the popup halfway, reopen — the posts
  captured so far are still there; capture again and the total grows.

## Step 8 — Export for archiving *and* for Claude

You have two different needs, so offer more than one format.

> Add three exports, built in the popup directly from IndexedDB: (1) a ZIP with
> the thread as Markdown plus a folder of full-size images; (2) a single Markdown
> file with images embedded but downscaled so it's small enough to upload to
> Claude; (3) a single self-contained HTML page with full-resolution images.

- **Build exports in the popup, reading the database directly** — don't pass the
  image data from the background worker to the popup as one giant message; on big
  threads that serialisation blows the stack. The popup and worker share the same
  IndexedDB, so let the popup read it.
- **ZIP is the best format for Claude analysis.** Inside, the Markdown is tiny
  (just text + `./images/…` links) and the images are separate files. Split the
  Markdown into `thread-part-01.md`, `thread-part-02.md`, … for very long threads
  so each part uploads on its own.
- **Downscale only the embedded-Markdown export** (e.g. longest side 1600px);
  keep HTML and the ZIP's images full-resolution for archiving.

**Verify:** export all three; the embedded Markdown should be a fraction of the
size of the HTML, and every image should render in the HTML.

**Gotcha — Claude can't read inside a `.zip`.** In the normal Claude chat you
upload files directly (`.md`, images, PDFs); it won't open entries inside a zip.
So for analysis, **expand the zip** and upload the `thread-part-NN.md` file(s) you
care about. The zip is your tidy archive; the extracted parts are what you feed Claude.

## Worked example — BambuLab and Duet3D

This guide came out of an extension targeting two real forums:

- **BambuLab** (`forum.bambulab.com`) runs **Discourse**. Posts are cloaked while
  scrolling (Step 4), and images come from a **separate CDN** host that had to be
  added to `host_permissions` (Step 6).
- **Duet3D** (`forum.duet3d.com`) runs **NodeBB 4.x**. Its `…-resized` image
  thumbnails **404**, so the scraper captures the full-size original from each
  image's lightbox link (Step 6). Its code blocks render as a line-numbered
  table, which needed a little export CSS to look right.

Point the same design at your forum, identify its platform (Step 2), and the rest
of the steps carry over.

## Approve routine actions with `.claude/settings.json`

To stop Claude asking permission for every safe, repetitive action, add a
`.claude/settings.json` in the project. This pre-approves building and reading
files while still prompting for anything unusual:

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run build)",
      "Bash(npm install)",
      "Bash(npm run dev)",
      "Read",
      "Edit",
      "Write"
    ],
    "ask": [
      "Bash(rm *)",
      "Bash(git push *)"
    ]
  }
}
```

Adjust to taste — the point is to green-light low-risk work and keep a prompt on
anything destructive or outward-facing. Two of these are broader than they look:
`npm install` can run package post-install scripts, and bare `Read`/`Edit`/`Write`
allow every file in the project — tighten them if you want a narrower allowlist.

## How the project is organised

A layout that keeps the platform-specific bits separate from the shared plumbing:

```
manifest.json                 Extension config (Manifest V3)
src/
  background/service-worker.ts Saves captures, downloads images, merges re-captures
  content/content-script.ts    Runs on the forum page, drives the capture
  scrapers/
    base-scraper.ts            Shared scroll/capture/image-normalising logic
    <platform>-scraper.ts      Per-forum selectors and quirks (one per platform)
  storage/indexeddb-manager.ts Local database + image de-duplication
  exporters/thread-exporter.ts Builds the ZIP / Markdown / HTML exports
  ui/popup/                    The popup: capture button + thread list + exports
```

When you add a forum, you add one scraper file and reuse everything else.

## Troubleshooting — the gotchas that actually bite

### Capture stops partway / misses posts
The forum cloaks off-screen posts. Scroll ≤ one viewport per step and slow down
(Step 4). Re-capturing merges the gaps in.

### No images at all, and capture is slow
Image requests are failing. Usual causes: the CDN host isn't in `host_permissions`;
or requests go without cookies and Cloudflare blocks them. Fetch with credentials
and add the CDN host (Step 6).

### Broken-image icons in the export
Either the download saved an error page (validate content-type), or the saved post
URL and the downloaded URL don't match (canonicalise image URLs at capture time),
or the thumbnail 404'd (use the original).

### Export fails with "Maximum call stack size exceeded"
You're passing all the image data through a message, or you have accidental
recursion. Build exports in the popup from IndexedDB (Step 8).

### The file's too big for Claude
Use the downscaled embedded-Markdown export, or the expanded ZIP's split
`thread-part-NN.md` files.

### My change didn't take effect
You didn't reload — **bump the version** and reload the unpacked extension. If you
changed `host_permissions`, you may need to remove and re-add the extension so the
browser grants the new host.

## Why this works

You never had to know Discourse's DOM, Cloudflare's cookie, or IndexedDB's API.
You stated **intent** — "capture the whole thread, get the images, make something
I can hand to Claude" — approved the steps, and **verified each result by looking
at it**. The hard-won knowledge is in the gotchas above; the rest, Claude writes.

## Resources

- Chrome extensions (Manifest V3): <https://developer.chrome.com/docs/extensions>
- Discourse: <https://www.discourse.org> · NodeBB: <https://nodebb.org>
- Claude Code: <https://claude.com/claude-code>
- Claude (uploading files for analysis): <https://claude.ai>

Happy capturing!
