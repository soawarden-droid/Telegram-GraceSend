# Grace Send: a 10-second undo window before your message truly leaves

You know the moment: you tap Send, and in the same second you see the typo,
the wrong chat, the sentence that sounded better in your head.

Telegram lets you edit a message and delete it for everyone — but both
happen *after* the other side may already have seen it. Email has had an
undo window for years. Telegram has never had one before the send.

**Grace Send** adds it. When you send a message, it does not leave your
phone instantly. For 10 seconds it waits, while a slim bar above the
keyboard counts down: **"Sending in 0:07 — Undo"**. Tap Undo and the
message is cancelled completely, with your text back in the composer
exactly as you typed it. Nothing was sent, nothing to delete, nobody saw
anything. Do nothing, and at zero the message goes out through Telegram's
normal send path, unchanged.

## What it does

### 1. Sending

Press Send as always. Instead of the message appearing in the chat at
once, the countdown bar appears above the composer. The message is fully
prepared — text, formatting, reply, link preview — it simply has not been
dispatched yet.

### 2. Undo

Tap **Undo** (or the bar itself) at any point in the 10 seconds:

- the send is cancelled; the prepared message is thrown away on the
  device, so nothing ever reaches a server;
- your text returns to the composer, ready to fix and send again.

### 3. The send

When the countdown reaches zero, the message is dispatched through the
exact same code path Telegram uses today, and appears in the chat with
its real timestamp.

### 4. Settings

**Settings → Chat Settings → Grace Send** turns it on or off (on by
default). The delay is fixed at 10 seconds — long enough to catch a
mistake, short enough to forget it exists.

## Never delayed

Grace Send stays out of the way where a delay would be wrong:

- messages you schedule yourself,
- edits to existing messages,
- a comment sent together with forwarded messages,
- paid (stars) messages,
- voice messages and picked photos/files — this version covers what you
  type; media keeps today's behaviour.

## How it works

- **One seam.** Everything typed in a chat is dispatched from a single
  place in the app. Grace Send parks the finished send request there —
  it does not re-implement sending, so replies, entities, effects and
  link previews all behave exactly as stock.
- **Client-side only.** The waiting message exists only in memory, on
  your device. There is no server support and none is needed.
- **Small.** Five files, about 360 lines: a controller, a countdown bar,
  one hook, one settings toggle, three strings.

## Applying the change

`changes.patch` is the difference from the official Telegram for Android
master at commit `21f7d109da6ab3506c9e9577f98c6a38771ab335`
("update to 13.2.2"). From a checkout of that commit:

```
git apply --check changes.patch
git apply changes.patch
```

Then build as described in the Telegram Android README.

The feature lives in two new files —
`org/telegram/messenger/GraceSend.java` (the controller) and
`org/telegram/ui/Components/GraceSendBar.java` (the bar) — plus a hook in
`ChatActivityEnterView.processSendingText`, a toggle in `ThemeActivity`,
and three strings.

## Status

A design-and-code patch by **ADIN**, published for discussion. The patch
is verified to apply cleanly to the pinned base commit; it has not yet
been compiled into an APK (that build is long and heavy, and is the next
step). Honest notes on scope and compromises live in `PATCH-NOTES.md`.

*Not affiliated with Telegram.*
