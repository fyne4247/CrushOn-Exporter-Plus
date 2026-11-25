# CrushOn Exporter Plus

A userscript that backs up [CrushOn.AI](https://crushon.ai/) chats, memories, and character cards for offline reading or import into SillyTavern.

## Install

Import [CrushOn-Exporter-Plus.user.js](./CrushOn-Exporter-Plus.user.js) into a userscript manager such as Violentmonkey, then reload CrushOn. To install manually, paste the script into a new userscript. To update, replace the contents of the existing entry and save.

On iPhone or iPad, use a Safari extension such as [Userscripts](https://github.com/quoid/userscripts). Enable it for CrushOn and save the `.user.js` file in the extension's script directory. Browser and download support vary.

## Export chats

Open a character chat, select **TXT**, **JSONL**, and/or **PNG** in the floating toolbar, then press **↓**. A *storyline* is a separate chat with the same character.

For just the character card, select **PNG** alone and press **↓**. You get one `.png` with the SillyTavern character JSON embedded. This skips chat history in both **ONE** and **ALL**, even with **Include branches** on. A character chat address ending in `/chat?from=DETAIL` is enough.

When you select **TXT** or **JSONL**, the chat settings apply.

| Export | Include branches off (default) | Include branches on |
| --- | --- | --- |
| **ONE** | Visible branch of the open storyline | Every branch of the open storyline |
| **ALL** | Newest branch of each storyline for this character | Every branch of each storyline for this character |
| **⚙ → Master export** | Newest branch of each chat in your Recent list | Every branch of each chat in your Recent list |

**TXT** saves a readable transcript. **JSONL** saves a SillyTavern chat with original timestamps where available. **PNG** saves a SillyTavern V2 character card with the profile fields available to your account. Bulk ZIPs include one PNG per character when available and a `manifest.json` describing the export. **ALL** and **Master export** add `WARNINGS-AND-ERRORS.txt` when there are problems or the export stops early.

For a styled, sanitized transcript, use **⚙ → HTML export (this chat)**. Linked images may still require internet access.

## Optional: send straight to SillyTavern

[CrushOn-Exporter-Plus-D2ST.user.js](./CrushOn-Exporter-Plus-D2ST.user.js) is the same exporter with one addition: it can send cards and chats directly into a running SillyTavern instead of (or as well as) saving files. If you don't use SillyTavern, stay with the plain script above. Install one or the other, not both.

In **⚙ → SILLYTAVERN**, set **Destination** (download only, SillyTavern only, or both), set **SillyTavern address** (default `http://127.0.0.1:8000`), and use **Test SillyTavern connection**. Select **PNG** alone to send just the card to your chosen destination, or turn on **JSONL** as well to include chats.

- The card goes first. A character whose CrushOn ID is already on a card in SillyTavern is skipped, not duplicated.
- Each chat is saved under that character with SillyTavern's own `Name - YYYY-MM-DD@HHhMMmSSsMMMms` naming. Branches become `... - Branch #n` chats linked to their parent. Sending the same export again adds nothing.
- Your active SillyTavern persona name is used for your messages, and `{{user}}`/`{{char}}` are filled in.
- Memories are not sent; they stay file-only.

It uses SillyTavern's HTTP API through your userscript manager (tested design: Violentmonkey), so it needs `GM_xmlhttpRequest` and works wherever you can open SillyTavern's address from the same browser. SillyTavern with basic auth or user accounts is not supported yet.

## Settings and memories

**Include branches** and **Keep regenerations** are independent. Regenerations are off by default. Turn them on to include older alternate replies as SillyTavern swipes in JSONL. An alternate reply can also be a branch if the chat continued from it.

**Skip “continue” messages** and **Restore missing pictures** are on by default. The image fix uses the original image when CrushOn's image proxy fails and the original is still available.

On a memory page, **↓** exports that memory. On your account page, **↓** exports all your memories as a ZIP. You can also use **⚙ → Memories only**. Master export includes memories by default, controlled by **Include memories in Master export**. CrushOn stores up to 100 messages per memory, even when its list shows a higher count.

## Troubleshooting and privacy

Check the export's source and coverage if messages are missing. The exporter uses CrushOn's history when available. **ONE** with branches off can fall back to the messages rendered on the page, which may be partial. Deleted messages and hidden profile fields are only exportable if CrushOn returns them to your session.

If speakers are mislabeled, use **⚙ → Debug → Calibrate speakers**. Debug dumps can contain full conversations or character data. Exports are saved locally and may contain private information.

Master export covers your Recent list and skips group chats. Use a desktop browser for large archives. Bulk exports run requests sequentially, and you can stop and keep what was collected.

## Credits

An unofficial fork of [CookieLoverAnon's CrushOnAI Chat Exporter](https://github.com/CookieLoverAnon/CrushOnAI-Chat-Exporter). Retain the original attribution when redistributing modifications.
