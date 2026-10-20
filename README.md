# CrushOn Exporter PLUS

A userscript for exporting [CrushOn.AI](https://crushon.ai/) conversations and characters, with SillyTavern-compatible output, better chat capture, and a workaround for broken character images.

**Current version:** 2.5.2  
**Based on:** [CookieLoverAnon's CrushOnAI Chat Exporter](https://github.com/CookieLoverAnon/CrushOnAI-Chat-Exporter)

> **Note:** This is an independently modified fork, not an official CrushOn.AI tool. CrushOn's site and APIs can change without notice. Some functions are best-effort and may require further updates.

## Features

### Export conversations

- **TXT:** Readable plain-text transcripts with `[user]` and `[ai]` labels.
- **HTML:** Offline-friendly, dark-themed transcripts that preserve supported message formatting. Exported message HTML is sanitized to remove scripts, event handlers, and other unsafe embedded elements.
- **SillyTavern JSONL:** Chat files in SillyTavern's JSON Lines format, with speaker roles and available original send times.
- **History-aware capture:** Reads CrushOn's own message tree (`chat.getMessageHistory`, the call the chat page itself makes) and exports the branch shown on the page when it can be identified. Otherwise it tries a history request observed in the tab, then the messages rendered on the page. Exports disclose which source was used, whether the history is complete, and how many pages were fetched.
- **Export all branches:** switch the **ONE / ALL** button next to **↓** to **ALL** (red), then press **↓**. It saves one SillyTavern `.jsonl` per branch (greeting → that branch's last message, including the history it shares with other branches) plus a `manifest.json` describing every fork, which branch each file splits from, and where. Regenerated replies are written as standard SillyTavern swipes. Saved as one `.zip` (best on phones); **⚙ → This chat: all branches (separate files)** saves them individually instead.
- **Export ALL chats:** **⚙ → ALL chats: every branch** lists every chat in your Recent list and saves every branch of every chat into one `.zip`: a folder per character plus a `manifest.json` with one entry per file (character, chat, branch, message count, completeness). Requests go one at a time, about every 2 seconds, with back-off if CrushOn asks to slow down. You can stop at any point and keep what was collected.
- **Regenerations as branches (off by default):** With this off, a regenerated reply that nothing continues from stays a swipe on its branch instead of getting its own branch file. When every version of a reply is a dead end (e.g. the last message), the newest is the main reply. Alternatives you kept playing from are always branches. Toggle in **⚙**.
- **Skip "continue" messages (on by default):** User messages that only say "continue" (any case, optionally with `.`, `!`, `…`, asterisks, quotes or brackets) are left out of every export. The following message's `parent_message_id` points to the nearest exported ancestor, and the skipped IDs are kept in `extra.crushon.skipped_continue_ids`. Toggle in **⚙**.
- **Dump history response:** Saves the raw `chat.getMessageHistory` response for troubleshooting. It contains full message text.
- **Speaker identification:** Automatic calibration, plus manual speaker calibration when the page layout is ambiguous. A manually selected signature is used when classifying rendered messages.

### Export characters

- **SillyTavern V2 character cards:** Generates `.png` files with embedded character data, including any available description, personality, scenario, greeting, and example dialogue.
- **Avatar handling:** Fetches the character image directly when possible, converts it to PNG, and uses a generated placeholder if the avatar cannot be retrieved.
- **Profile lookup:** Tries CrushOn's existing character APIs, including an owner-specific lookup when authorized, rather than relying solely on visible page text.
- **Visibility-aware:** Allows public and unlisted character exports, as well as characters belonging to your account. Private characters belonging to others are blocked. Unavailable or explicitly hidden definition fields are not fabricated.
- **Data inspection:** Includes a **Dump character data** tool for troubleshooting.

### Restore missing images

The script works around a known kind of image-loading failure in which CrushOn's image-transforming proxy (`img.cocdn.co`) cannot serve an image that is still available from `cdn.crushon.ai`.

It rewrites matching image URLs locally, including images loaded after scrolling or changing pages. The fix is **on by default** and can be toggled in **⚙ → Restore missing pictures**. It doesn't change, delete, or reupload images on your CrushOn account.

This only helps when the original image is still accessible. It cannot recover files that are actually gone, nor can it fix unrelated image URLs.

### Interface

A compact draggable toolbar offers **PNG**, **JSONL**, and **TXT** output toggles, a **↓** export button, a **ONE / ALL** scope switch (purple: the branch on screen; red: every branch of this chat), and a **⚙** settings menu. The settings menu also includes HTML export, the optional status bar, manual calibration, UI resizing, character-data diagnostics, and a hide-panel option. Preferences and panel placement are saved locally when the userscript environment permits it.

## Installation

Install a userscript manager in a supported browser, then install the JavaScript userscript from this fork:

**[`CrushOn-Exporter-Plus.user.js`](./CrushOn-Exporter-Plus.user.js)**

- **Desktop Firefox:** Use a userscript manager such as Violentmonkey, then install or paste the contents of the `.user.js` file.
- **Copy and paste into Violentmonkey:** Open [`CrushOn-Exporter-Plus.user.md`](./CrushOn-Exporter-Plus.user.md), a plain-text copy of the script, in any text editor and copy everything (⌘A / Ctrl+A, then copy). In Violentmonkey's dashboard, click **+** → **New**, select all of the placeholder template and replace it with the copied script, then **Save & Close**. To update, open the existing script in Violentmonkey, replace its contents the same way, and save.
- **iPhone Safari:** Use a Safari userscript extension such as [Userscripts](https://github.com/quoid/userscripts) and enable it for `crushon.ai`. Save the script as a `.user.js` file in the extension's configured script directory.
- **Other browsers:** A compatible userscript manager may work, but behavior depends on the browser and the manager's supported APIs.

The script matches `https://crushon.ai/*` and `https://www.crushon.ai/*`. Reload CrushOn after installation.

`CrushOn-Exporter-Plus.user.md` is the same script as `CrushOn-Exporter-Plus.user.js`, byte for byte: **the contents are JavaScript**, not Markdown. Paste it into your userscript manager or save it as a `.user.js` file. The `.md` extension itself won't run as a userscript. Both files are updated together.

## Usage

**Scope at a glance:** with **ONE** (purple) selected, **↓** exports the current chat (the branch on screen). With **ALL** (red), **↓** exports every branch of the current chat. The choice persists. **⚙ → ALL chats: every branch** exports every chat.

1. Open a CrushOn conversation or character page.
2. In the floating toolbar, select **TXT**, **JSONL**, and/or **PNG**. The selected formats persist between visits and apply to every export: **↓**, all branches, and all chats. In branch and bulk exports, TXT adds a readable `.txt` per branch and PNG adds the character card (one per chat, or one per character folder in the bulk export).
3. Click **↓** to capture the current conversation and/or character, then export the selected formats.
4. For an HTML transcript, open **⚙ → Export .html (bonus)**.
5. If message speakers were mislabeled, use **⚙ → Calibrate speakers (fallback)** and select one user bubble followed by one AI bubble.
6. If images are missing, check **⚙ → Restore missing pictures**; it is enabled by default.

TXT and JSONL exports require chat messages. PNG export requires an accessible character record with usable data. On a character-only page, chat formats may be skipped; on a chat-only page, the character export may be unavailable.

### Output formats

| Format | Purpose | Notes |
| --- | --- | --- |
| `.txt` | Readable conversation backup | Includes capture source and coverage note. |
| `.html` | Styled offline reading | Message HTML is sanitized; linked external images may still require network access. |
| `.jsonl` | SillyTavern chat import | Preserves genuine timestamps when available; leaves unknown times empty. API captures keep CrushOn's text verbatim. |
| `.zip` | Branch / bulk archives | One `.jsonl` per branch + `manifest.json`. File names are deterministic: `<Character> - <chat id> - branch-NN.jsonl`. |
| `.png` | SillyTavern V2 character card | Includes data actually accessible to your session; may contain a placeholder avatar. |

### JSONL metadata (optional extras, ignored by SillyTavern)

The header line's `chat_metadata` keeps `source_url` (`/character/<id>/chat/<chat id>`) and `exported_at`, and adds `chat_metadata.crushon`:

| Field | Meaning |
| --- | --- |
| `character_id`, `chat_id`, `exporter_version` | Identifiers. |
| `capture` | `message_tree`, `observed_request`, or `visible_dom`. |
| `complete` | `true` only when CrushOn's history ran to its end with no gaps. |
| `pages_fetched`, `stopped_early` | Pagination record. |
| `branch_id`, `parent_branch_id` | This branch and the earlier branch it shares the longest history with. |
| `fork_index` | 0-based index (among message lines) of the first message unique to this branch. |
| `fork_kind` | `regen` or `branch`, inferred from CrushOn's data (AI alternative = regen; otherwise branch). CrushOn's edited flag is not used here: it marks text changed later, on original and alternative replies alike, and is reported per message as `edited`. With regenerations off, `regen` only appears on alternatives that were continued. |
| `regens_as_branches`, `skip_continue` | The export options in effect. Manifests also record `options` (with `folded_regens`, the number of regenerations kept as swipes only). |

Each message's `extra.crushon` has `message_id`, `parent_message_id`, `created_at`, `time_source` (`server`, `page`, or `none`), `edited`, and `greeting: true` on the opening message. With `message_id` and `parent_message_id` the tree can be rebuilt from any set of exports.

## Current limitations

- **History depends on CrushOn's API.** The message-tree call is undocumented and was verified in October 2026. If CrushOn changes it, the branch and bulk exports say so and download nothing, and the regular ↓ export falls back to the older methods. Each file's `complete` flag tells you whether the whole history came back.
- **Only what CrushOn returns.** Branches hidden in the UI are included because the server returns them. Deleted messages, or anything the server does not send, cannot be recovered and are never invented.
- **DOM fallback may be partial.** If history retrieval is unavailable, only rendered messages are captured. Emphasis is converted back to `*`/`**`, but paragraph spacing can differ slightly from the original text, and the page's "(edited)" label only appears on some edited messages.
- **Which branch is "current" is a best guess.** CrushOn keeps the branch selection in the browser only. The exporter marks the branch whose messages match the page; if it can't tell, it says so.
- **Bulk export covers your Recent chats list** (`character.queryRecentCharacters`). Group chats are not included. Large archives are built in memory, so use a desktop browser for the bulk export.
- **Character definitions can be limited.** Information not supplied by CrushOn cannot be reconstructed. This includes fields hidden from the current account.
- **Browser support varies.** Mobile userscript managers may not implement all `GM_*` APIs. In particular, character-avatar downloading and file saving can behave differently on iOS; a placeholder avatar or alternate download path may be used.
- **Site changes can break selectors and API lookups.** A valid-looking download does not necessarily mean the entire conversation was captured; check its **Source** and **Coverage** note.

## Privacy and access

The script runs in your browser using your existing CrushOn session. It does not require credentials to be entered into the script, and it does not operate a separate collection service. Its export and diagnostic features may retrieve data from CrushOn's own APIs and image-hosting services. **Exports are local files, but they can contain private conversations or character details, so handle them accordingly.**

The character exporter does not bypass server-side access permissions. It does not create missing or restricted character definitions.

## Attribution

**Original work:** [CookieLoverAnon / CrushOnAI-Chat-Exporter](https://github.com/CookieLoverAnon/CrushOnAI-Chat-Exporter), which provided the foundation for chat capture, speaker calibration, and TXT/HTML export.

**This fork:** Extends the original with SillyTavern chat and character-card output, API-assisted character and history retrieval, branch-tree and bulk exports, image restoration, expanded UI controls, export safety improvements, and additional compatibility handling.

Please retain attribution to the original author when redistributing modifications. This README does not assert a license that the upstream repository has not explicitly provided.

## Disclaimer

This project is unofficial and is not affiliated with, endorsed by, or maintained by CrushOn.AI or SillyTavern. Use it only for data you are authorized to access, and follow applicable platform terms.
