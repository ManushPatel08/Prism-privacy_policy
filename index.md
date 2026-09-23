# PRISM — Privacy Policy

**Applies to:** PRISM — contextual gesture shortcuts (demonstrator), extension version 0.1.0
**Last updated:** 23 September 2026
**Effective date:** 2026-09-23

---

## 0. What this extension is, before anything about data

PRISM is a **research demonstrator** written for a Human–Computer Interaction course project at
Stony Brook University. It exists to let a person try one interaction technique — direction gestures
drawn with the right mouse button, with an on-screen guide that shows what the gesture would do —
on real web pages instead of only inside a lab instrument.

It is not a supported consumer product. It has no company behind it, no support desk, and no
guarantee of continued maintenance. **No study results are claimed here, because the study has not
been run yet.**

This policy describes what the published extension actually does. Every statement below was checked
against the extension's source code rather than written from memory.

---

## 1. The short version

- There is no server, no backend, no API, no analytics, no telemetry, no crash reporting, no remote
  configuration and no remote endpoint of any kind. There is no account and nothing to sign in to.
- The extension makes **one kind of network request of its own, and only when you ask for it**: when
  you draw **Copy image** on a picture, it re-fetches that one picture from the address the page
  already loaded it from, so that the picture itself can be put on your clipboard (§5). No cookies
  are sent with that request, and the bytes go only to your local clipboard.
- **No personal data is collected, transmitted, sold or shared.** None is collected at all.
- The **only** thing it stores is the **custom gesture bindings you create yourself**, in
  `chrome.storage.local` on your own machine.
- Three things leave the page, and only because you asked for them with a gesture: **clipboard
  copies** (§5), **Save image**, which saves the picture you gestured on to your Downloads folder
  (§3), and the **Define / Search this** commands, which open a new tab at a public website with your
  selected text in the URL (§6). All are described in full below rather than left out.

---

## 2. What is stored on your machine

| | |
|---|---|
| Where | `chrome.storage.local` — local to this browser profile on this computer |
| Key | exactly one: `prism.custom-bindings` |
| What is in it | the custom gestures you recorded on the extension's options page |
| Per binding | the context (`page`, `link`, `image` or `selection`), the direction tokens (e.g. `U,L`), the id and label of the command it runs, and the time it was created |
| Written by | the options page, when you save a gesture |
| Read by | the content script, once per page load, so a saved gesture works after a browser restart |

No page content, no URLs, no page titles, no timing data and no identifiers are stored — the options
page has no free-text field to type any into; its only inputs are two drop-down lists and a pad you
draw on.

`chrome.storage.sync` is **not** used, so nothing is uploaded to your Google account. The extension
also uses no `localStorage`, no `sessionStorage`, no IndexedDB and no cookies.

Deleting a binding on the options page removes it. Uninstalling the extension removes the key and
everything in it.

---

## 3. The permissions, and exactly what each is spent on

### `bookmarks`

Spent by **one call in the whole extension**: `chrome.bookmarks.create({ title, url })` in
`background.js`, in the handler for the `bookmark` action. It is reachable only one way — by drawing
the **`R` gesture on a link**, the "Bookmark link" command. The title and URL passed to it are the
title and URL of that link.

The extension **never reads, searches, lists, edits, moves or deletes bookmarks**. `create` is the
only bookmarks API it calls, and without this permission that one command cannot work.

### `storage`

Spent by **two calls**, a `get` and a `set` on `chrome.storage.local`, both in `bindings.js`, both
against the single key described in §2. A custom gesture has to survive a browser restart; a content
script's `localStorage` belongs to the website you are visiting, so it is per-site, cleared with site
data, and invisible to the options page. `chrome.storage.local` is the only store the options page
and the content script can share.

### `downloads`

Spent by **one command**: **Save image**, the **`D` gesture on an image**. `background.js` calls
`chrome.downloads.download({ url, filename, saveAs: false })` with the address of the image you
gestured on (or, for an inline SVG figure, a `data:` address built from its own markup, which
involves no request), so the file is saved straight to your browser's default Downloads folder
without a Save-As dialog. The file name is made from the image's description or address and cleaned
of anything that is not safe in a file name. The browser makes the download request, exactly as it
would for "Save image as…"; the file appears in Chrome's own downloads list, and you can delete it
like any other download.

To report truthfully whether the file actually finished, the worker also looks up **that one
download's** state (`chrome.downloads.search` by its id, and `onChanged` filtered to that id). It
never lists, reads, opens, deletes or otherwise touches any other entry in your download history.

### Host access: `http://*/*` and `https://*/*`

Spent by **one command**: **Copy image**, the **`L` gesture on an image** (§5). A web page is not
allowed to read the pixels of a picture that came from another website, so the content script
cannot turn such a picture into clipboard data itself. The extension's background worker can, but
only for websites it holds host permission for — and the picture can be on any website, which is why
the grant is broad. It is used for exactly one request: a `fetch` of the address of the picture you
gestured on, inside the Copy image handler in `background.js`, described in §5. The host permission
is **not** used to read, change or inject into any page; the content script's page access is
described separately in §4.

### Nothing else is requested

There is no `tabs`, `cookies`, `history`, `identity`, `management`, `webRequest`, `scripting`,
`clipboardRead` or `clipboardWrite` permission.

The extension does open, close and create tabs and windows, but those APIs are ungated and buy no
access to your data: the tab it closes is the one the gesture came from, identified by the message
sender, and when it reads the geometry of your current window in order to size a new one, it does so
without `populate`, so no tab list, URL or title is returned to it.

---

## 4. Why the content script runs on every `http://` and `https://` page

The manifest declares a content script matching `http://*/*` and `https://*/*`. The technique is a
replacement for the right-click context menu, so it has to be available wherever you right-drag; the
extension cannot know in advance which pages those will be, and a narrower match would make it work
on some sites and silently do nothing on others. It does not run on `chrome://` pages, the Chrome
Web Store or the new tab page, and it checks the page's scheme itself before starting.

**What it does on a page you visit:**

- adds a stylesheet link for its own on-screen guide;
- adds pointer listeners (`pointerdown`, `pointermove`, `pointerup`) to the document;
- reads your saved bindings from `chrome.storage.local` once, on load;
- **at the moment a right-drag begins**, looks at the element under the pointer — its tag name, a
  link's `href`, an image's `src` — and the current text selection, purely to decide which of the
  four contexts applies and which command a stroke would run;
- draws its guide overlay while you hold the button down;
- prints one line to the page's developer console, and sends one message to **its own background
  service worker** containing the page URL, which the worker prints to the extension's own console
  log. That message is local to your browser; it is not stored and it is not sent anywhere;
- for the commands that need a privileged API (open link in a background tab, open link in a new
  window, bookmark link, new tab, close tab, save image, copy image), sends the link's URL and title,
  or the image's address and a file name, to **its own service worker** over Chrome's internal
  messaging so the worker can perform the action. The messaging is in-browser, not a network
  request; what the worker then does for Save image and Copy image is described in §3 and §5.

**What it does not do:** it does not read or index page text beyond the selection you have made
yourself; it does not read form fields, passwords, cookies, `localStorage` or site data; it does not
scrape, copy or store page content; it does not record your browsing history; it does not capture
keystrokes; it does not modify the page apart from adding its own overlay and stylesheet; and it
writes **nothing** about any page you visit to disk. Nothing it reads is retained after the gesture
ends. It also installs no `contextmenu` handler and calls `preventDefault()` nowhere, so a short
right-click still opens the website's own menu exactly as that site intended.

---

## 5. The clipboard

Three commands write to your system clipboard, each only when you draw its gesture:

| Gesture | Command | What is copied |
|---|---|---|
| `L` on a text selection | Copy | the text you selected |
| `L` on a link | Copy link | that link's URL |
| `L` on an image | Copy image | **the picture itself**, as a PNG image; if that fails, the image's address (or, for an inline SVG, its markup) instead, and the on-screen message says which |

Copy and Copy link use the browser's Clipboard API (`navigator.clipboard.writeText`), with the older
`document.execCommand('copy')` as a fallback if the first is refused. Copy image uses
`navigator.clipboard.write` with an `image/png` item.

**How Copy image gets the picture, including its one network request:**

- For an inline SVG figure, a `data:` image, or an image the page itself generated (`blob:`), the
  picture is drawn and converted to PNG **inside the page**, with no request of any kind.
- For an ordinary image on a website (`http://` or `https://`), the extension's background worker
  makes **one request to re-fetch that image** from the same address the page loaded it from. The
  request sends **no cookies**, and asks the browser to answer from its HTTP cache where it can.
  Chrome keeps a separate cache per site, so the answer may come from the cache or may be a fresh
  request to the image's server; that server sees an ordinary image request from your browser, as
  it did when the page first loaded the picture. The worker decodes the image, re-encodes it as a
  PNG and hands the bytes back to the content script, which writes them to your clipboard. **The
  bytes go only to your local clipboard.** Nothing is stored, logged or sent anywhere else.
- If any step fails — the request fails, the address does not return an image, the page's image is
  unreachable, or the browser refuses the clipboard write — the image's address is copied instead,
  and the message says so (for example "Copied the image address — this site blocked copying the
  picture"). The extension never says it copied a picture when it did not. The clipboard is an ordinary
part of copying, but it is disclosed here rather than omitted: after one of these commands, page
content is on your clipboard and can be pasted anywhere, including into other applications.

**The extension never reads the clipboard.** It holds no `clipboardRead` permission and writes only
what the command you ran is defined to write.

---

## 6. The two commands that open a third-party website

Two commands in the selection context open a new tab at a public website, with the text you selected
placed in the URL:

| Gesture | Command | Opens |
|---|---|---|
| `D` on a text selection | Define | `https://en.wiktionary.org/wiki/Special:Search?search=<your selection>` |
| `U` on a text selection | Search this | `https://duckduckgo.com/?q=<your selection>` |

The selection is URL-encoded and trimmed to 200 characters. The resulting request is made **by your
browser to that website**, exactly as it would be if you typed the address yourself or used Chrome's
own "Search with…" menu item. Those sites' own privacy policies then apply to that visit. PRISM has
no API key, no account and no relationship with either site, and it never sees or reads what comes
back.

**These two gestures and Copy image's re-fetch of the picture you gestured on (§5) are the only
times a request goes out because of PRISM, and each happens only when you deliberately draw that
gesture.** Save image's download (§3) is likewise a request for a file you chose, made by the
browser as any download is.

---

## 7. What PRISM never does

- No analytics, telemetry, metrics, usage statistics, crash reports or "anonymous diagnostics".
- No advertising, no ad networks, no trackers, no fingerprinting, no cookies.
- No user accounts, sign-in, email collection or newsletters.
- No selling, renting, sharing or transferring of data to anyone — there is no data to transfer.
- No remote code: every line of JavaScript it runs ships inside the package.
- No background activity when you are not gesturing: the service worker only wakes to answer a
  message from the content script.
- No study data. The research instrument that records trial data is a **separate** local web page
  that is not part of this extension; the extension has no logger, no trial engine and no
  questionnaires, and records nothing about how you use it.

---

## 8. How you can check these claims

- **The installed code is readable.** A Chrome extension ships its JavaScript to your computer in
  readable form, so anyone who installs PRISM can inspect exactly what it runs — for example via
  `chrome://extensions` → PRISM → **Details**, on disk in the `Extensions` folder of your Chrome
  profile, or with a CRX viewer. Start with `manifest.json`; the permission reasoning is written in
  the header comments of `background.js`, `commands-bridge.js` and `bindings.js`, next to the lines
  that spend them. (The project's development repository is not public; you do not need it to check
  any claim here.)
- **A comment-stripped static scan** of the 21 source files of the shared engine and study
  instrument looked for network calls, remote URLs, remote `@import`, `localStorage` /
  `sessionStorage` / `indexedDB` / `document.cookie`, and analytics or telemetry names, and found
  **0 hits in every category**. The extension's own files are searched for the same primitives —
  `fetch`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`, `EventSource`, `importScripts`, and the
  common analytics names — by an automated source audit that is part of the project's checks. The
  **one** permitted match is Copy image's `fetch` in `background.js` (§5); the audit allows it only
  in that file, only inside that handler, and only once, and fails on any other `fetch` anywhere in
  the extension.
- **A DevTools network capture of a full session** in a throwaway Chrome profile, taken on
  22 September 2026, recorded **20 requests, every one to `http://localhost:8080`, none external**.
- **Scope note, stated rather than glossed over:** that capture was taken with `--disable-extensions`
  so that other installed extensions stayed out of the measurement. It is therefore evidence about
  the engine, guide and command modules this extension loads — not a recording of a browsing session
  with PRISM installed. No such capture has been taken yet. The network claims for the extension's
  own files rest on the source review above, and anyone can verify them in a minute with the service
  worker's Network panel open while gesturing: the only request it makes is Copy image's re-fetch of
  the picture.

---

## 9. Retention, and getting rid of it

There is nothing to retain, because nothing is sent to PRISM and there is no server. A picture you
copy stays on your clipboard until you copy something else; an image you save stays in your
Downloads folder until you delete it. Your custom
bindings stay on your computer until you delete them on the options page or uninstall the extension,
which removes them with it. There is no account to close and no deletion request to file — but if
you want to ask about any of this, the contact is in §11.

## 10. Children, sensitive data, and changes to this policy

The extension is not directed at children and collects nothing from anyone, so it holds no sensitive
category of data — no health, financial, location, authentication or personal-communications data.

If this policy changes, the updated version will be published at the same URL with a new date at the
top. An extension that started collecting data would be a materially different one, and this page
would say so plainly before it shipped.

## 11. Contact

Questions, corrections, or anything in this policy that does not match what you observe:

**Manush Patel**
Department of Computer Science, Stony Brook University
manushsachin.patel@stonybrook.edu
