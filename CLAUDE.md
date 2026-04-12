# CLAUDE.md — ob-redirect

## Project Overview

`ob-redirect` is a minimal, single-file static web application that bridges browser-based links to the **OnBase** desktop document management system via custom URI schemes (`onbase://`). It is used internally at Mutual of Enumclaw.

The entire codebase is a single file: `index.html`.

---

## Repository Structure

```
ob-redirect/
└── index.html   ← the entire application
```

There are no dependencies, no build tooling, no package manager, and no test framework. The file can be served as-is from any static web host.

---

## How It Works

When the page loads, the `redirect()` function (defined in an inline `<script>`) reads URL query parameters and routes the user to one of three OnBase deep-link targets:

| Priority | Required Parameters | Resulting `onbase://` URI |
|----------|--------------------|-----------------------------|
| 1 | `DocID` | `onbase://document/view/?DocID=<DocID>` |
| 2 | `classid` + `objectid` | `onbase://wv/object/?classid=<classid>&objectid=<objectid>` |
| 3 | `QueueID` (+ optional `LifeCycleID`) | `onbase://wf/lc/?LifeCycleID=<id>&QueueID=<QueueID>` |

- **Priority order matters**: `DocID` is checked first, then `classid`/`objectid`, then `QueueID`.
- `LifeCycleID` defaults to `250` if not supplied in the URL.
- All parameter values are passed through `encodeURIComponent()` before being placed into the target URI.
- If no valid parameter combination is found, the page displays: `Required parameters are missing.`
- After the auto-redirect attempt (`window.location.href = target`), the page renders a fallback link in case the OS does not handle the `onbase://` scheme automatically.

### Example URLs

```
# Open a document by DocID
https://<host>/index.html?DocID=12345

# Open an object (contact, etc.)
https://<host>/index.html?classid=10&objectid=99

# Open a workflow queue (uses default LifeCycleID=250)
https://<host>/index.html?QueueID=7

# Open a workflow queue with explicit LifeCycleID
https://<host>/index.html?LifeCycleID=300&QueueID=7
```

---

## Development Workflow

### Editing

Open `index.html` in any text editor. There is no compilation or build step.

### Testing Locally

Serve the file with any local HTTP server (browsers block some APIs on `file://`):

```bash
# Python 3
python3 -m http.server 8080

# Node (if npx is available)
npx serve .
```

Then navigate to `http://localhost:8080/?DocID=12345` (or another parameter combination) and verify the redirect fires correctly. Testing the `onbase://` URI itself requires a machine with the OnBase Unity Client installed.

### No Automated Tests

There is no test suite. Manual verification in a browser is the only testing mechanism.

---

## Code Conventions

- **Language**: Vanilla HTML5 + ES6 JavaScript (no frameworks, no TypeScript).
- **Style**: Inline `<script>` in `<head>`; logic runs via `onload` on `<body>`.
- **Parameter names**: Match OnBase API exactly — `DocID`, `classid`, `objectid`, `LifeCycleID`, `QueueID` (case-sensitive).
- **URL encoding**: Always use `encodeURIComponent()` for any value inserted into a URI.
- **Fallback UX**: After setting `window.location.href`, always replace `document.body.innerHTML` with a human-readable fallback link so the user can proceed manually if the OS does not handle the custom scheme.
- **Error state**: Use `document.body.textContent` (not `innerHTML`) for plain error messages to avoid XSS.
- **No external dependencies**: Keep the file self-contained. Do not add CDN links, npm packages, or build steps.

---

## Key Constraints

1. **Single-file requirement**: The application must remain a single deployable `index.html`. Do not split logic into separate `.js` or `.css` files unless the deployment target explicitly supports multi-file hosting and the change is approved.
2. **No server-side logic**: All routing is done client-side in JavaScript.
3. **Parameter names are fixed by OnBase**: Do not rename or normalize query parameter names — the calling systems pass them verbatim.
4. **Default LifeCycleID = 250**: This is a business default tied to Mutual of Enumclaw's OnBase configuration. Do not change it without confirming with the OnBase administrator.
5. **Security**: The fallback anchor (`<a href="${target}">`) embeds a user-supplied-derived value. Ensure any modifications continue to route only to `onbase://` URIs, never to arbitrary user input, to prevent open-redirect or XSS issues.

---

## Git Conventions

- Branch naming: `claude/<short-description>-<id>` for AI-assisted work; otherwise direct commits to `main`.
- Commit messages historically follow: `"Update index.html"` or a brief description of the change.
- There is no CI/CD pipeline; deployment is manual.
