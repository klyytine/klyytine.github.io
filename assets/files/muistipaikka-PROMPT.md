<role>
You are a senior frontend engineer who specializes in rich text editors, desktop-style productivity apps, and careful UX. You write small, readable React components and you test what you build.
</role>

<context>
Build "Muistipaikka" (Finnish for "a place for notes"): a Notion-style note-taking app with block-based editing, a clean minimal UI, and a desktop-app feel. The interface is in Finnish by default, with English as a second language (section 13). It runs locally in the browser. A small local file service (built into the dev server) lets it open and save real files on disk, keep the whole workspace in a folder, and behave like a native app with a File menu.

Everything below is the full specification. You don't need any other material. For reference on Notion's behavior and look, you may read:
- Notion Help Center: https://www.notion.com/help
- Notion overview: https://en.wikipedia.org/wiki/Notion_(productivity_software)
</context>

<tech-stack>
- Frontend: React 18 + Vite
- Editor: Slate.js (`slate`, `slate-react`, `slate-history`) with a flat, block-based document model
- Hotkeys: `is-hotkey`
- Routing: React Router 6 (`/p/:id` for pages, `/share` for shared pages, `/help` for the help page)
- State: React Context (pages + autosave, preferences, theme, file actions)
- Persistence: `localStorage` by default; optionally a JSON workspace file in a folder on disk
- Styling: plain CSS with CSS variables for theming (light, dark, print)
- Localization: a small built-in i18n module (no library): dictionaries per language, `{name}` placeholders, plural forms with `Intl.PluralRules`
- Images: FileReader API, stored as base64 data URLs
- Local file service: a Vite plugin that adds `/api/*` middleware to the dev and preview servers (Node `fs`, no extra dependencies)
</tech-stack>

<features>

## 1. Block editor
- Block types: Text, Heading 1/2/3, Bulleted list, Numbered list, To-do (checkbox), Quote, Callout (with an emoji), Code, Divider, Image.
- Lists and to-dos can be indented up to 3 levels with Tab / Shift+Tab.
- Inline marks: bold, italic, underline, strikethrough, inline code.
- Slash menu: type `/` to open it, keep typing to filter by name or keyword, move with ↑ ↓, insert with Enter, close with Esc.
- Markdown shortcuts at the start of a block: `#`, `##`, `###`, `-` or `*`, `1.`, `[]`, `"`, ```` ``` ````, `---`. Inline: `**bold**`, `*italic*`, `` `code` ``, `~strike~`.
- Hover toolbar: selecting text shows a floating toolbar with the five marks.
- Image blocks: upload a file or paste a URL, same picker as the cover image.
- Shift+Enter inserts a line break inside a block. Ctrl/⌘+Enter toggles a to-do or leaves a code block.
- Undo and redo work for every edit.

## 2. Page header (Notion style)
- Inline-editable page title.
- Icon picker: 30 emojis in an 8-column grid, opened as an absolutely positioned dropdown; click outside to close. Click an existing icon to change it. Icon size 78px, with a background highlight on hover.
- Cover image, added two ways side by side:
  ```
  [Upload Image]
       OR
  [URL input] [Add Link] [Cancel]
  ```
- Uploaded images are downscaled to at most 1600px (WebP) so they fit in storage and share links. GIFs and SVGs are kept as is.
- Subtle action buttons above the title: "😀 Add icon" and "🖼️ Add cover". They show only when the header is hovered, use gray text that darkens on hover, have no border or background, and put the emoji before the text with CSS `::before`.
- "Change cover" and "Remove cover" buttons appear when hovering the cover.

## 3. Top bar
From left to right:
- Sidebar toggle button.
- **File** menu button (see section 5).
- Breadcrumb: page icon and title. If the page is linked to a file on disk, a chip shows the file name, with a ● dot when the file has unsaved changes. Hovering the chip shows the full path.
- Save status (see section 6).
- **Share** button (see section 9).
- Dark/light mode toggle.

## 4. Sidebar
- Page list with icon and title, active page highlighted.
- "+ New Page" button.
- Delete a page from its row, with a confirmation dialog.
- Footer buttons: **Shortcuts** (opens a dialog listing every keyboard shortcut) and **Help** (opens the `/help` page).
- Collapsible with Ctrl/⌘+\\. On narrow screens it slides over the page with a backdrop.

## 5. File menu
A desktop-style dropdown menu. It works with the keyboard: focus moves to the first item when it opens, ↑ ↓ move between items, → opens the Open Recent submenu, ← closes it, Tab or a click outside closes the menu. Each item shows its shortcut on the right.

| Item | Shortcut | Behavior |
| --- | --- | --- |
| Open File… | Ctrl/⌘+O | Opens the in-app file dialog. Without the file service, falls back to the browser's file picker. |
| Open from URL… | – | Dialog to paste a link to a web page, Markdown, text or `.note` file. Adds `https://` if missing. Opens as a new page. |
| Open Recent ▸ | – | Submenu with the last 10 files and links (title plus a short location hint, full path on hover). "Clear Recent" at the bottom. If a file no longer exists, it is removed from the list and the user is told. |
| Save | Ctrl/⌘+S | Saves right away (the workspace and any linked file). Works even when a dialog is open. |
| Save As… | Ctrl/⌘+Shift+S | In-app save dialog with file name and "Save as type": Note (`.note`) or Markdown (`.md`, text only). Asks before overwriting. After saving, the page is linked to that file. Without the file service, downloads a `.note` file instead. |
| Rename… | F2 | Renames the page title. If the page is linked to a file, a checkbox "Also rename the file on disk" shows `old.note → new.note`. |
| Print… | Ctrl/⌘+P | Browser print. A print stylesheet hides the sidebar, top bar, status bar and buttons, and always prints in light colors. |
| Preferences… | Ctrl/⌘+, | Opens the Preferences dialog (section 7). |
| Quit | Ctrl/⌘+Q | If there are unsaved changes and "Ask to save" is on, shows Save / Don't Save / Cancel with a "Don't ask again" checkbox. Then closes the app window. |

### File formats
- `.note`: the native, lossless format. JSON: `{ "format": "notes-page", "version": 1, "title", "icon", "cover", "content", "createdAt", "updatedAt" }`. `.json` opens the same way.
- `.md` / `.markdown`: read and written. Headings, lists, to-dos (`- [ ]`), quotes, code fences, dividers, images and inline marks are converted both ways. A leading `# Title` becomes the page title.
- `.txt`: opens as plain paragraphs, as a new page (not linked to the file).
- HTML from a URL: converted to blocks (headings, paragraphs, nested lists, checkboxes, quotes, code, images with absolute URLs). The `<title>` becomes the page title. Limit the number of blocks.
- Opening a file that is already open switches to its existing page instead of making a duplicate.
- Pages linked to a `.note` or `.md` file are written back to that file on every save.

### In-app file dialog
Used for Open, Save As, and choosing a folder in Preferences, so every file has a real path:
- Left: "Places" (Home, Desktop, Documents, Downloads, their OneDrive versions when present, and each drive).
- Top: Up-one-folder button, editable path bar, Go.
- List: folders first, then files, with size and date. Click selects, double-click opens. Keyboard: ↑ ↓, Enter, Backspace goes up.
- Open mode shows only supported files, with a "Show all files" checkbox and a "Use the system dialog instead…" link.
- Save mode rejects names with `\ / : * ? " < > |` and adds the extension automatically.

## 6. Saving and autosave
- Debounced autosave after the user stops typing (default 600 ms, adjustable).
- Status text in the top bar: "Saving...", "Saved just now", "Saved 3 min ago", "Unsaved changes" (autosave off), or an error such as "Not saved" with details on hover.
- Saves run one at a time in a queue, so they never overlap.
- If storage is full (`QuotaExceededError`), say so instead of failing silently.
- When the window closes: do a last save (use `fetch` with `keepalive` for the file service), and if there are unsaved changes and the preference is on, show the browser's "Leave site?" prompt.

## 7. Preferences dialog
- **Language**: Suomi (default) or English, as radio buttons, each label written in its own language. Changes apply immediately, with no reload.
- **Storage**: "This browser" (localStorage, about 5 MB) or "A folder on this computer" (saved as `notes-workspace.json` in that folder; no size limit, easy to back up or sync). The folder option has a path field and a Browse… button and is disabled, with an explanation, when the file service isn't running. Switching saves first. If the chosen folder already contains notes, ask: "Open those notes", "Replace them with my current notes", or Cancel. Show the current location. If the folder can't be read at startup, show a screen with "Try again" and "Use browser storage instead".
- **Saving**: Autosave on/off, Autosave delay (100–60,000 ms), "Ask to save unsaved changes when closing".
- **Editor**: "Show line numbers" (numbers each block in the left margin), "Show word count" (a bottom status bar with words, characters and paragraphs).
- Store preferences in localStorage and validate them when loading.

## 8. Local file service
A Vite plugin (`configureServer` and `configurePreviewServer`) that adds POST-only JSON endpoints under `/api/`:

| Endpoint | Purpose |
| --- | --- |
| `fs/ping` | Platform, path separator, home folder, quick places and drives. The app uses this to detect that the service is available. |
| `fs/list` | Folder contents (hide system files such as `$Recycle.Bin`, `desktop.ini`). |
| `fs/stat`, `fs/read`, `fs/write`, `fs/rename`, `fs/mkdir` | File operations on absolute paths. `write` refuses to overwrite unless asked. |
| `fetch-url` | Fetches a URL on the server so "Open from URL" isn't blocked by CORS. 15 s timeout, 10 MB limit. |
| `app/quit` | Closes the launcher's app window (see section 12). |

Security: accept a request only if it has the header `X-Notes-App: 1`, the Host is loopback (`localhost`, `127.0.0.1`, `[::1]`), and the Origin, if present, matches the Host. Limit request and read sizes. Turn Node errors (`ENOENT`, `EACCES`, `EBUSY`…) into short, plain messages. Every error response carries a `code` (for example `notFound`) and `params` (for example `{ path }`) so the app can show it in the user's language, plus an English `error` message as a fallback. Quick places carry an `id` (`home`, `desktop`, `documents`, `downloads`…) so the app can translate their names; drives keep their letter.

Without the service (for example, a static deployment), the app still works: storage stays in the browser, Open uses the browser's file picker, and Save As downloads a file.

## 9. Public sharing
- The Share popover builds a link of the form `/share#<base64url>`, where the fragment holds the whole page snapshot. Nothing is uploaded.
- Copy button with "Copied!" feedback, an "Open preview ↗" link, and the link size in KB. Warn when the link is over 8 KB (usually because of images).
- The shared page is read-only, with a button to duplicate it into the viewer's own notes.

## 10. Help and shortcuts
- `/help` page: basic usage, every slash command, Markdown shortcuts, and keyboard shortcuts.
- Shortcuts dialog from the sidebar.
- Both read from one shared list of shortcuts. Show ⌘ and ⌥ on Mac, Ctrl and Alt elsewhere.

## 11. Theme
- Light and dark themes with CSS variables. Follow the OS setting by default; toggle with the top-bar button or Ctrl/⌘+Shift+L, and remember the choice.

## 12. Desktop launcher (Windows, optional)
- `run.bat` starts a hidden PowerShell script that: checks for Node.js, runs `npm install` on first run, starts Vite hidden on port 5173 (or reuses an instance that is already running), and opens Edge or Chrome in app mode (`--app=URL`, its own `--user-data-dir` profile, no tabs or address bar).
- It writes the window's process ID to a file so File → Quit can close the window, and it stops the server when the window closes.
- `create-shortcut.bat` adds a "Muistipaikka" shortcut with the app icon to the Desktop and Start menu.
- Show plain error message boxes (Node.js missing, port in use, server didn't start). The launcher can't read the in-app language, so these messages follow the Windows display language (`(Get-UICulture).TwoLetterISOLanguageName`): Finnish for `fi`, English otherwise. Save the script as UTF-8 **with BOM**, or Windows PowerShell 5.1 garbles ä and ö.
- App icon: a black rounded tile with a white 3D extruded "M" (grey extrusion sides), drawn by a small Pillow script into `.ico` and `public/icon.png`.

## 13. Languages (Finnish and English)
- Finnish is the default for new and existing users; the choice is stored with the other preferences and applied before the first render, so nothing flashes in the wrong language. Set `<html lang>` to match.
- Translate everything the user sees: menus, dialogs, buttons, placeholders, tooltips and `aria-label`s, status texts, error messages (including those from the file service), the help page, the shortcuts dialog, the welcome page and the browser tab title. Never translate the user's own page content.
- Use the Finnish terms from Notion's own Finnish help (https://www.notion.com/fi/help), for example: block = *lohko*, Text = *Teksti*, Heading 1 = *Otsikko 1*, Bulleted list = *Luettelo*, Numbered list = *Numeroitu luettelo*, To-do list = *Tehtävälista*, Quote = *Lainaus*, Callout = *Huomiolohko*, Code = *Koodi*, Divider = *Jakaja*, Image = *Kuva*, slash commands = *Vinoviivakomennot*, Add icon = *Lisää kuvake*, Add cover = *Lisää kansi*, Share = *Jaa*, dark mode = *tumma tila*, Preferences = *Asetukset*, Untitled = *Nimetön*.
- Follow Finnish typography: closing quotes on both sides (”…”), an en dash with spaces ( – ), a space before units (5 Mt), and a non-breaking space as the thousands separator (60 000).
- Plurals: "1 sana / 2 sanaa", "1 sivu / 3 sivua". Format numbers and dates with the current locale (`fi-FI` / `en-US`).
- Slash commands match keywords in both languages, so `/tehtävä` and `/todo` both work in either language.
- The welcome page exists in both languages; a new user gets it in the current language. The help page has an "Add a welcome page" button that adds a fresh copy in the current language (never overwrite existing pages).
- Plain modules (file parsers, the file-service client, image helpers) translate through a module-level `t()`; components use a `useT()` hook that re-renders them when the language changes.
- Keep internal identifiers (storage keys, file formats, the launcher's profile folder, the `X-Notes-App` header) in English, so changing the language or the app name never moves anyone's data.

</features>

<keyboard-shortcuts>
Use ⌘ and ⌥ on Mac in place of Ctrl and Alt. The block shortcuts (0–8) are ⌘ ⌥ 0–8 on Mac; on Windows and Linux, accept both Ctrl Shift and Ctrl Alt.

| Shortcut | Action |
| --- | --- |
| Ctrl B / I / U / E | Bold / italic / underline / inline code |
| Ctrl Shift X | Strikethrough |
| Ctrl Shift 0 | Text |
| Ctrl Shift 1 / 2 / 3 | Heading 1 / 2 / 3 |
| Ctrl Shift 4 | To-do |
| Ctrl Shift 5 / 6 | Bulleted / numbered list |
| Ctrl Shift 7 / 8 | Quote / code |
| Tab / Shift Tab | Indent / outdent list item |
| Ctrl Enter | Toggle to-do, leave code block |
| Shift Enter | Line break inside a block |
| Ctrl Alt N | New page |
| Ctrl Shift L | Toggle dark mode |
| Ctrl \\ | Toggle sidebar |
| Ctrl Z / Ctrl Shift Z | Undo / redo |
| Ctrl O | Open file |
| Ctrl S / Ctrl Shift S | Save / save as |
| F2 | Rename page |
| Ctrl P | Print |
| Ctrl , | Preferences |
| Ctrl Q | Quit |

Only Save works while a dialog is open; other File shortcuts wait until it closes.
</keyboard-shortcuts>

<design-specifications>
- Font: system fonts (`-apple-system, BlinkMacSystemFont, "Segoe UI", …`).
- All colors are CSS variables, redefined for dark mode and for print.
- Border radius 4–6px. Subtle shadows (`0 1px 3px rgba(0,0,0,0.1)`); popovers and menus slightly stronger.
- Action buttons: 13px, gray text that darkens on hover, `background: none`, no border.
- Smooth transitions on every interactive element.
- Menus, popovers and modals close on Esc and on clicking outside. Modals trap focus and return it when closed.
- Use simple inline SVG icons for the interface (menu, file, folder, globe, help, keyboard, etc.) and emojis for page icons.
- Messages are short and plain, and say what to do next (for example: "Sivu on yli 10 Mt." / "That page is larger than 10 MB.", "Tässä kansiossa on jo muistiinpanoja (3 sivua). Mitä tehdään?").
- Accessible: proper roles (`menu`, `menuitem`, `listbox`, `dialog`, `status`), `aria-expanded`, labels on icon-only buttons, visible focus.
- Responsive down to phone width.
</design-specifications>

<data-safety>
- Treat everything loaded from localStorage, files, URLs and share links as untrusted. Keep only known block types, marks and fields, cap indent levels, and allow image URLs only if they are `http(s)` or `data:image/...;base64`.
- A new user starts with a welcome page that demonstrates the main features.
- If saved data is corrupted, start fresh instead of crashing.
</data-safety>

<project-structure>
```
server/      fileApi.js (Vite plugin: local file service)
launcher/    launch.ps1, create-shortcut.ps1, icon
src/
  i18n/      index.js (t, useT, current language), fi.js (default), en.js — same keys in both
  context/   PagesContext (pages, autosave, file sync), PrefsContext, ThemeContext, welcome page (fi + en)
  editor/    BlockEditor, Slate plugin (withNotion), schema + sanitizer, commands, elements, leaf,
             slash menu, hover toolbar
  files/     FileActions (File menu logic, shortcuts, dialogs), documents (.note/.md/.txt/HTML),
             fileApi (client), workspace, recent
  components/ Topbar, FileMenu, FileDialog, PreferencesDialog, AppDialogs, Modal, Sidebar,
             PageHeader, IconPicker, ImageSourcePicker, StatusBar, ShortcutsDialog, Icon
  pages/     Workspace (/p/:id), SharedPage (/share), HelpPage (/help)
  help/      shortcuts.js (single source of shortcut data), HelpParts.jsx (shared pieces),
             content.fi.jsx / content.en.jsx (help text per language)
  utils/     image (FileReader + downscaling), share (base64url), platform
run.bat, create-shortcut.bat, README.md
```
</project-structure>

<output>
Deliver a complete, working project that runs with `npm install && npm run dev`, plus a README covering how to run it, features, shortcuts and storage. It must include:

✅ Block editor with all 12 block types, slash menu, Markdown shortcuts and hover toolbar
✅ Notion-style page header: inline title, emoji icon picker, cover from file or URL
✅ Sidebar with page management, Shortcuts dialog and Help page
✅ File menu: Open, Open from URL, Open Recent, Save, Save As, Rename, Print, Preferences, Quit
✅ In-app file dialog; `.note` and `.md` save and open; `.txt` and web pages open
✅ Preferences: browser or folder storage, autosave and delay, close prompt, line numbers, word count
✅ Secure local file service, with a graceful fallback when it isn't running
✅ Autosave with a clear status, and an unsaved-changes prompt on quit
✅ Public sharing with links that contain the whole page
✅ Dark mode, print styles, keyboard shortcuts, accessible menus and dialogs
✅ Finnish (default) and English interface, switchable in Preferences, with Notion's Finnish terminology
✅ Optional Windows launcher that opens the app in its own window

Before finishing, run the app and check: create and edit a page with every block type; save it as `.note` and `.md` and reopen both; open a URL; switch storage to a folder and back; quit with unsaved changes; open a share link in a new window; switch the language to English and back and confirm no Finnish or English text is left behind; check that every key exists in both language files.
</output>
