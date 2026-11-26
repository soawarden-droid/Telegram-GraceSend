# Grace Send — patch notes

## Base

- Project: Telegram for Android — https://github.com/DrKLO/Telegram
- Pinned commit: `21f7d109da6ab3506c9e9577f98c6a38771ab335`
- Why this commit: it is the current master HEAD, and its own message is
  "update to 13.2.2 (7303)" (2026-11-12) — a version-bump tip, not a
  mid-refactor snapshot. The repo's newest tag (release-11.4.2) is far
  behind master, so master HEAD is the honest base.

## Verification

- `git apply --check changes.patch` on a pristine second checkout of the
  base commit: **PASS**
- Full `git apply` on that checkout: **PASS**
- Patch size: 20,155 bytes; 5 files; +361 / −2 lines.

## Files

| File | Change | Lines |
|---|---|---|
| `TMessagesProj/src/main/java/org/telegram/messenger/GraceSend.java` | NEW — the controller: holds fully-built `SendMessageParams` for 10 s, dispatches them unchanged on expiry, drops them on Undo; per-account singleton in SendMessagesHelper's own pattern; preference `grace_send` in the global main settings (default on) | 193 |
| `TMessagesProj/src/main/java/org/telegram/ui/Components/GraceSendBar.java` | NEW — the countdown bar: "Sending in 0:07" + bold **Undo**; tapping either cancels | 62 |
| `TMessagesProj/src/main/java/org/telegram/ui/Components/ChatActivityEnterView.java` | Hook + bar wiring: +87 / −1 | 87 added |
| `TMessagesProj/src/main/java/org/telegram/ui/ThemeActivity.java` | Settings toggle "Grace Send" (Settings → Chat Settings → Other, under Send by Enter), following the `send_by_enter` pattern end to end: row field, reset, row assignment, click handler, cell bind, view type | 16 added |
| `TMessagesProj/src/main/res/values/strings.xml` | 3 strings: `GraceSend`, `GraceSendInfo`, `GraceSendSendingIn` | 3 added |

## Where the hook sits, and why

The single dispatch point for everything typed into the composer is
`ChatActivityEnterView.processSendingText()` — it has exactly one caller
(the send button / Enter path) and it is where the finished
`SendMessageParams` are handed to `SendMessagesHelper.sendMessage(params)`.
The patch swaps that one call for:

```java
if (!tryGraceSend(params, text)) {
    SendMessagesHelper.getInstance(currentAccount).sendMessage(params);
}
```

Because the params are already fully built at that point (text, entities,
reply, web page, notify flag), the delayed send is byte-for-byte the send
the user asked for — just 10 seconds later. `ChatActivity.java` itself
needed no edits at all.

The countdown bar is added by the enter view into its own parent container
(the chat input bubble container, which already hosts bottom-anchored
overlays), positioned directly above the composer; its bottom margin tracks
the composer's height on every 200 ms tick.

## What is never held (by design)

- Scheduled sends (`scheduleDate != 0`, including schedule mode) — already
  delayed by the user's own choice.
- Message edits — edits go out immediately.
- A text sent together with forwarded messages — the pair must stay in sync.
- Paid (stars) sends — money should not wait silently.
- Voice/video messages and picked media (photos, files) — see below.

## Design compromises (honest list)

1. **Text-first.** The grace window covers messages typed in the composer
   (the classic regret case). Voice messages dispatch from a different
   branch of the enter view, and picked media flows through
   `prepareSendingMedia`/`didSelectPhotos` — intercepting those too would
   have tripled the patch and touched the media pipeline. They keep stock
   behaviour in this version.
2. **No fake "waiting" message cell.** A held message does not appear in
   the message list (a local-only placeholder cell would need adapter and
   read-receipt surgery). The countdown bar above the composer is the
   waiting state; the message appears in the list when it really sends,
   with its real timestamp.
3. **Undo restores into an empty composer.** If the user has already typed
   a new draft when they tap Undo, the held send is still cancelled, but
   the old text is not pasted over their new draft.
4. **One bar per chat.** With several sends held in the same chat, the bar
   counts down the one expiring first; Undo cancels that one. (Parts of one
   over-long message are grouped and expire/undo together.)

## How to apply

```bash
git clone https://github.com/DrKLO/Telegram.git
cd Telegram
git checkout 21f7d109da6ab3506c9e9577f98c6a38771ab335
git apply --check /path/to/changes.patch
git apply /path/to/changes.patch
```

## Honest limit

The patch is verified to apply cleanly to the pinned base, and every API
it touches was checked against the base tree (send path, preference store,
settings row machinery, theme keys, string conventions). The full APK
build — Android SDK plus a long, heavy Gradle build of the whole Telegram
app — was **not** run in this workspace. That step belongs on the user's
PC. Until it is built there, treat this as a verified-applies patch, not
a tested binary.
