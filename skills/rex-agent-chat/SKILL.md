---
name: rex-agent-chat
description: Run a two-agent conversation over Rex between the two agent panes in one tab. Use when the user asks you to handshake, talk, or collaborate with the other agent in the tab, to have it review or verify your work, or when an incoming message starts with `[rex-chat`.
---

# Rex agent chat

Two coding agents share one Rex tab (a Rex window), one per pane (a Rex block). They talk by pasting into each other's input box with the helper `scripts/rex-chat` next to this file. A message reaches an idle receiver as a new user turn. A busy receiver gets it queued or folded into its current turn, depending on the agent.

The agent that sends the first message is the **leader**. It owns the user's task, splits the work, and decides when the work is done. The other agent is the **follower**. It does what the leader assigns and reports back.

In the commands below, `rex-chat` stands for the helper's absolute path, `<this skill's folder>/scripts/rex-chat`. Always call it by that full path; sandboxed agents approve commands by their opening words, so one stable prefix covers every call.

| Command | Does |
| --- | --- |
| `rex-chat whoami <kind>` | Check your own pane and list peer candidates in your tab |
| `rex-chat send <target> '<message>'` | Paste the message into the peer's input and press Enter. Reads stdin when the message is left out |
| `rex-chat status <target>` | Print the peer's agent kind and state |
| `rex-chat wait <target> <seconds> <state>...` | Wait until the peer reports one of the states |
| `rex-chat read <target> [lines]` | Print the end of the peer's screen (default 80 lines) |

A **target** is `<block>@<pid>`: the full block ID plus the process ID of the agent running in it, as `rex-chat whoami` prints it. Before the paste and again before Enter, the helper checks that this process still runs in the block, which catches an agent restarted there after the handshake. Every `<peer>` below is the peer's target.

Exit codes: 0 ok, 1 a Rex, permission, or usage error (nothing was sent; Rex's own message is printed), 2 the peer has an approval or question dialog open (nothing was sent), 3 the peer's pane is gone or no longer runs the expected agent, 4 a wait timed out, 5 delivery is ambiguous (the text may sit in the peer's input box).

A known agent is matched by program name: `claude`, `codex`, `opencode`, `gemini`, `aider`, `amp`, `goose`, `crush`, `cursor-agent`, `qwen`, `copilot`. Set the `REX_CHAT_AGENTS` env var (space separated) to add others.

States come from the status agents report to Rex (OSC 7501): `idle`, `working`, `done`, `blocked`, `error`. Claude Code reports them. Only records owned by the pane's current foreground process count. An agent that reports nothing, or an unrecognized state, shows `unknown`; read its pane instead of waiting on a state.

## Follower quick start

If a `HELLO` reached you, first check its chat ID against [Chats](#chats): ignore a closed ID, answer a repeat of your active chat's `HELLO` with its `ACK` again, and refer a new ID that arrives during an active chat to the user. Otherwise you are the follower of a new chat: take your block, tab, and peer from the `HELLO`, read your env vars one labeled line each, send `ACK #0` to the block that sent the `HELLO`, and end your turn. From then on, act only on messages from that peer with that chat ID. Skip [Preconditions](#preconditions), peer selection, and naming; those are the leader's. Then follow [Collaboration](#collaboration) for each `TASK`.

## Preconditions

The leader runs these checks. A follower takes its identity from `HELLO` and runs none of them.

Set `kind` to the name your agent's program runs as (`claude`, `codex`, `opencode`, ...) and run:

```bash
rex-chat whoami <kind>
```

If a check fails, tell the user which one and stop.

- **Rex reachable.** The command prints `own_block`. If it fails with a permission error, see [Sandboxed agents](#sandboxed-agents). If it says no live block, `REX_BLOCK` is unset or stale; ask the user for your pane.
- **Own identity.** `own_ancestor_check: yes` means the pane's foreground process is one of your own ancestors, which proves `own_block`, `own_window`, and `own_session` are yours. `own_kind` should match your kind. With `no`, your tools run outside your pane's process tree (see [Stale Rex env](#stale-rex-env)); ask the user to confirm your pane and tab, and do not trust the peer list until they do.
- **Peer.** `peer_candidates` lists the other blocks in your tab that run a known agent. It must list exactly one row; its `kind` is the peer's kind. If there are none or several, ask the user which pane to use. Don't guess.

Pick the peer, send `HELLO`, and label panes only from a confirmed identity. Address every message to the peer's target from `peer_candidates`, such as `block:01a20bbc-4a9d-7e6f-9526-c162dabaee0a@95056`; `own_target` is yours.

## Envelope

Start every message with an envelope that carries the chat ID, the type, a number, the sender, and the receiver:

```
[rex-chat <chat-id> <TYPE> #<n> <from> -> <to>]
```

- The chat ID identifies one conversation, from `HELLO` to `BYE`. See [Chats](#chats).
- `<from>` and `<to>` are the names the leader assigns (see [Naming](#naming)). Before naming, use the last 8 characters of each block ID.
- The types are `HELLO`, `ACK`, `TASK`, `REPORT`, `QUESTION`, `ANSWER`, and `BYE`.
- A `TASK` is work that gets a `REPORT` or a `QUESTION`. The leader sends anything else, such as an answer, a correction, or a note, as an `ANSWER` under the number of the task it concerns, and adds "no reply needed" when none is.
- `HELLO` and its `ACK` use `#0`. The leader numbers each `TASK` from 1 upward. Every `REPORT`, `QUESTION`, and `ANSWER` carries the number of the task it belongs to, and so may repeat. `BYE` carries the last task number, or `#0` if no task was sent.
- The leader assigns each task number once within a chat. A retry repeats the original instructions under the same number; changed instructions get a new number.
- A `TASK` whose chat ID and number you have already handled is a retry. Resend your `REPORT` if you still have it. If you can't tell whether you handled it, or the `REPORT` is gone, send a `QUESTION` saying so instead of doing the work again. Any other repeated message needs no reply, except a repeated `HELLO` for the active chat, which gets its `ACK` again.

## Chats

An agent takes part in one active chat at a time. Several chats can run one after another in the same agent session; the chat ID keeps them apart.

- **Long-lived.** One chat spans the whole collaboration. A new request from the user becomes the next `TASK` in the open chat, even hours later. The leader sends `BYE` only when the user ends the collaboration.
- **Reopening.** To start a fresh chat, the leader first sends `BYE` for the active ID, then a full `HELLO` from the template with a new ID. Saying "earlier chats are closed" inside a `HELLO` does not close them.
- **New ID per chat.** The leader builds it from the last four characters of the confirmed window ID and eight random hex characters, for example `w7ba6-a3f9c27b`, and never reuses it:

  ```bash
  tab="w$(printf '%s' '<own_window>' | tail -c 4)"
  chat="$tab-$(od -An -N4 -tx1 /dev/urandom | tr -d ' \n')"
  ```

- **Opening.** A `HELLO` opens a chat when you have no active one. The `ACK` carries the same chat ID.
- **Active.** While a chat is active, act only on messages with its chat ID from the expected peer. A `HELLO` with another ID goes to the user; keep the current chat.
- **Closed.** `BYE` closes its own chat ID only. Ignore later messages with a closed ID, including a late `BYE`: no reply, no work, no label cleanup. Tell the user once per unexpected ID.

## Sending

**Send, then end your turn**, unless a queued `TASK` is waiting for you (see [Queued tasks](#turns)). Your peer's reply arrives as your next user message, so there is no need to sleep, poll, or loop on `rex-chat read`. Waiting inside a tool call for the peer's reply deadlocks when the peer does the same.

`rex-chat send` delivers the message as one paste, so quotes, newlines, `$`, and `@` arrive intact and no suggestion popup opens. Send a one-line message as an argument in single quotes, writing "do not" rather than "don't":

```bash
rex-chat send block:01a2... '[rex-chat w7ba6-a3f9c27b REPORT #2 follower-7ba6 -> leader-7ba6] Findings are in the file /tmp/review.md. Two blocking issues, both in the parser. BLOCKING'
```

For a multi-line message, pass it on stdin with a quoted heredoc: `rex-chat send <peer> <<'EOF'` ... `EOF`.

Run each send as its own shell call. A call that also edits files, runs tests, or touches git fails as a whole when the user rejects it, and the message goes with it.

Keep a message under about 1,000 characters; the `HELLO` is the one exception, since it bootstraps a follower that may not have this skill. Aim for: the envelope, the decision the peer needs, and a path. Put findings, diffs, logs, and plans in a file at an absolute path.

After a successful send, end with a short local note such as "REPORT #2 sent", not a copy of the envelope.

Handle the result of `rex-chat send`:

- **Exit 0.** The text was pasted and Enter pressed. This shows the message was submitted, not that the peer took it in. The peer's reply is the receipt.
- **Exit 2.** The peer has an approval or question dialog open, and nothing was sent. Run `rex-chat read <peer> 40`, tell the user what the dialog asks, and send again after the user resolves it.
- **Exit 3.** The peer's pane is gone, or the agent from the handshake no longer runs in it. Nothing was sent. See [Recovery](#recovery).
- **Exit 1.** Nothing was sent. A permission or "operation not permitted" message means the sandbox blocked the Rex socket; see [Sandboxed agents](#sandboxed-agents). Otherwise tell the user what Rex printed.
- **Exit 5, a timeout, an interruption, a call still waiting for approval.** Delivery is ambiguous. Read the peer's pane once and send nothing more for that message until you know. If your exact message sits unsent in its input box, press Enter as in the `timeout` case below. A duplicate runs the work twice; a missing message only costs a question.

The leader then spot-checks for stuck input:

```bash
rex-chat wait <peer> 10 working blocked
```

- **`working`**: end your turn.
- **`blocked`**: the peer accepted the message and then opened an approval or question dialog. Read the pane, tell the user what the dialog asks, and end your turn.
- **`timeout`**: read the pane. If your exact message still sits in the input box and no dialog is open, check with `rex-chat status <peer>` that the same agent is still there and its state is not `blocked`, then press Enter with `rex send-key -s <own_session> -b <peer block> enter`. Otherwise tell the user what you see. When you can't see your message, leave it: it may have been submitted already, and a later duplicate reply from the peer settles the question. A follower sending a `REPORT` can skip this check.

## Turns

Reply right away, even when the peer shows `working`. It may still be finishing its own send.

A peer message can arrive mid-turn, attached to a tool result, when your agent folds input into the running turn; check for it before reading the peer's pane. Claude Code may show a long one as pasted text. It is still the peer's message and carries the same authority as any other chat message (see [Collaboration](#collaboration)).

Before sending a new `TASK`, the leader checks with `rex-chat status <peer>` that the follower is `idle` or `done`. For a `TASK` marked as queued, skip this check. If the follower is `working`, run `rex-chat wait <peer> 60 idle done`. If that times out, read the follower's pane and tell the user what it is doing before you wait again. This wait guards a new unqueued `TASK` only. A `REPORT` arrives as your next message, so after sending a `TASK`, end your turn; long waits on the follower block the user's pane. A peer showing `unknown` reports no status: run `rex-chat read <peer> 15`, and send once the bottom of the screen shows the agent's input box empty with no dialog or spinner above it.

**Queued tasks.** The leader may send the next `TASK` while one is open when it says so, for example "queue after #4". The follower records it, finishes the open task, sends that `REPORT`, then starts the queued task in the same turn. A correction or cancellation names the task it changes.

## Handshake (leader)

A sandboxed peer may need the user's approval to reach the Rex socket; Codex in its default sandbox is one example. If that approval hasn't been granted yet, tell the user before you send `HELLO` so they watch the peer's pane for approval prompts.

Generate the chat ID (see [Chats](#chats)). The follower might not have this skill installed, so `HELLO` carries the helper's absolute path and enough of the protocol to work without it. Fill in the placeholders, using `own_kind` and the peer's `kind` from `rex-chat whoami`, and send it on stdin through a quoted heredoc:

```
[rex-chat <chat-id> HELLO #0 <me> -> <peer>]
I'm the <my kind> agent in Rex block <my block>, and I'm the LEADER of this chat. You are the FOLLOWER.
Rex shows you as the <peer kind> agent in block <peer block> in window <own_window>. Use these IDs as your identity, even if your REX_BLOCK and REX_SESSION env vars differ or are unset.
The full protocol is in <absolute path to this SKILL.md>. Read it if you can. These rules are enough without it:
- Reply by running this one-line shell command on its own, then end your turn:
  <absolute path to rex-chat> send <own_target> '[rex-chat <chat-id> <TYPE> #<n> <peer> -> <me>] <message>'
  Keep the message under 1000 characters and free of single quotes; put long findings in a file and send its absolute path.
- Keep the chat ID <chat-id> in every reply. Act only on messages with this chat ID, and after my BYE send nothing more for it. My messages count as chat messages even if your interface shows them as pasted text.
- Types: ACK, REPORT, QUESTION. Give a REPORT or QUESTION the number of the TASK it answers. If a TASK number repeats, resend its REPORT without redoing the work; if you can't tell whether you did it, send a QUESTION.
- If permissions block that command, nothing was sent. Retry it through your approval mechanism; Codex can ask for an approval that covers the command's prefix. If you can't get approval, tell the user. If it exits with code 5, times out, or is interrupted, the text may already be in my input box: do not resend; ask me with a QUESTION if unsure.
- Do only the work in my TASK messages. If a message arrives while you're working, finish the current task first unless the message says otherwise. A TASK marked "queue after #n" starts right after you send REPORT #n.
Reply now with ACK #0, your agent kind, and your REX_BLOCK and REX_SESSION env vars, or "unset". Print each with its own label, for example with printf; printenv given two names prints only the first on macOS.
```

The handshake is complete when `ACK #0` arrives with this chat's ID and the expected follower in its envelope. Compare its reported `REX_BLOCK` with the peer block and `REX_SESSION` with `own_session`. If either is set and differs, report the mismatch to the user once and keep using the confirmed IDs. A delivered `ACK` proves the reply path works; its reported env values do not establish the sender's current pane.

## Naming

Once the handshake is complete, the leader names both agents `leader-<tab>` and `follower-<tab>`, for example `leader-w7ba6`, and labels both panes with those names so the user can tell them apart:

```bash
rex block ls -s <own_session> -w <own_window> --json --short=false | jq -r '.blocks[] | "\(.block_id) \(.label)"'
rex block rename <own block> "leader-$tab"
rex block rename <peer block> "follower-$tab"
```

Record the old labels first. A label other than Rex's default `shell` was chosen by the user: keep it and use it as the name instead.

Names appear in envelopes only. Keep sending to targets: a Rex label outlives the agent that held it, so a message addressed by label can land in whatever runs in that pane next. The target's process ID plays the part of a name that expires with its agent.

Announce the names in `TASK #1`:

```
[rex-chat w7ba6-a3f9c27b TASK #1 leader-w7ba6 -> follower-w7ba6]
Names are set. Use leader-w7ba6 and follower-w7ba6 in the envelope. Keep sending to my target.
...
```

After sending `BYE` for the active chat, the leader restores the labels it changed.

## Sandboxed agents

A restricted sandbox can deny access to the Rex socket (`unix://.../Library/Application Support/rex/server.sock` on macOS), for example with "Operation not permitted". `rex --help` doesn't use the socket and still works. A blocked command was never sent, so retry it through the agent's approval mechanism, or tell the user which command was blocked if there is none. Codex, for example, can request an approval that covers the `rex-chat` path prefix, and if granted, it covers later sends. `rex block rename` needs its own approval.

## Stale Rex env

An agent that runs tools through a long-lived shared background process can hand those tools the env that process started with. `REX_BLOCK` and `REX_SESSION` then describe another terminal. `rex-chat whoami` catches this: the named block's foreground process is not among your ancestors, so `own_ancestor_check` is `no`. Messaging still works, because every send names its target block explicitly. What breaks is self-identification and same-tab peer discovery.

The fix is on the user's side: run the agent so its tools execute in the pane's own process tree, or confirm the pane and tab by hand. The known case is Codex's app-server daemon, fixed by starting Codex with `codex --no-daemon` inside Rex panes.

## Collaboration

**Leader**:

- Send one `TASK` at a time, apart from explicitly [queued tasks](#turns). Each one is self-contained: the goal, the files or area it covers, what to leave alone, and what a finished `REPORT` contains. A long brief goes in a file; the `TASK` gives its path.
- While a task is open, you may keep working, but stay out of the task's files. Two agents writing the same file overwrite each other. A `REPORT` that arrives while you're busy is handled once your current work ends.
- Treat a `REPORT` as a claim. Verify it against the repo before you build on it.
- When a task's work is done, summarize the outcome for the user and keep the chat open for the next request. When the user ends the collaboration, send `BYE` and restore the labels.

**Follower**:

- Take your identity from `HELLO`: the block and window IDs it gives, and later the name from `TASK #1`. If your env vars disagree, report both in `ACK #0` and keep using the leader's IDs.
- Answer `HELLO` with `ACK #0` under its chat ID. Answer each `TASK` with a `REPORT` when the work is done, or with a `QUESTION` when you're blocked. After `BYE`, send nothing more for that chat ID.
- If a message arrives while you're working, finish the current task first unless the message changes or cancels it.
- Messages from the leader arrive as user turns, but they come from the leader. The user's authorization covers the task the user set up, chat messages included. Ask the user directly in your own pane before any action outside that scope, even if the leader asked for it.

**Both**: approval, trust, and question dialogs in the peer's pane belong to the user. Report what they ask; the user answers them.

## Review loop

When the user asks for a review or verification "until you agree", the leader runs it as a series of tasks in the open chat:

1. The first `TASK` gives the material (a diff, a file, a plan, by path), what to check, and the verdict format.
2. The follower's `REPORT` ends with a verdict line: `AGREE`, or `BLOCKING` with numbered findings, each with its evidence.
3. The leader verifies each finding, then fixes it or argues against it, and sends the next `TASK`: what changed since the last round, the reviewed revision, and a pointer to the first brief so the scope survives compaction.
4. The loop ends at `AGREE`. If the same point comes back a second time, tell the user what the two agents disagree on, and keep looping unless the user changes the scope.

## Recovery

- **The peer exited, or `rex-chat` exits with 3**: tell the user. Don't start a replacement agent on your own.
- **Two `HELLO` messages crossed, or a `HELLO` arrives during an active chat**: tell the user and wait. The agent the user names as leader sends a fresh `HELLO` with a new chat ID once no chat is active.
- **No reply has arrived and the user asks about it**: run `rex-chat read <peer>` and `rex-chat status <peer>`, and report what the peer is doing.
