---
name: herdr-agent-chat
description: Run a two-agent conversation over Herdr between the two agent panes in one tab. Use when the user asks you to handshake, talk, or collaborate with the other agent in the tab, or when an incoming message starts with `[herdr-chat`.
---

# Herdr agent chat

Two coding agents share one Herdr tab, one per pane. They talk by typing into each other's input box with `herdr agent prompt`. A message reaches an idle receiver as a new user turn. A busy receiver gets it queued or folded into its current turn, depending on the agent.

The agent that sends the first message is the **leader**. It owns the user's task, splits the work, and decides when the work is done. The other agent is the **follower**. It does what the leader assigns and reports back.

## Preconditions

Check these first. If any check fails, tell the user which one and stop.

```bash
test "${HERDR_ENV:-}" = 1 && echo "$HERDR_TAB_ID $HERDR_PANE_ID"
herdr agent list | jq -r --arg t "$HERDR_TAB_ID" --arg p "$HERDR_PANE_ID" \
  '.result.agents[] | select(.tab_id==$t and .pane_id!=$p) | "\(.pane_id) \(.agent) \(.agent_status) \(.name)"'
```

- `HERDR_ENV` is `1`. Without it you are outside Herdr and cannot reach the session.
- The filter prints exactly one line: the **peer**, given as its pane ID, agent kind, status, and name. If there are no lines or more than one, ask the user which pane to use. Don't guess.

Until the handshake finishes, address the peer by its pane ID, such as `w9:p5`. After that, use the names the leader assigns (see [Naming](#naming)).

## Envelope

Start every message with an envelope that carries the type, a number, the sender, and the receiver:

```
[herdr-chat <TYPE> #<n> <from> -> <to>]
```

- The types are `HELLO`, `ACK`, `TASK`, `REPORT`, `QUESTION`, `ANSWER`, and `BYE`.
- `HELLO` and its `ACK` use `#0`. The leader numbers each `TASK` from 1 upward. Every `REPORT`, `QUESTION`, and `ANSWER` carries the number of the task it belongs to, and so may repeat. `BYE` carries the last task number, or `#0` if no task was sent.
- A `TASK` whose number you have already handled is a retry. Resend your `REPORT` for it without doing the work again. Any other repeated message needs no reply.

## Sending

**Send, then end your turn.** Never use `--wait` on `agent prompt` in a chat. By default, `--wait` holds the sender inside a tool call until the receiver settles. If the receiver replies with the same wait, each agent waits on the other until a timeout fires. A timeout looks like a failed delivery, and a resend then runs the work twice. Your peer's reply arrives as your next user message, so there is no need to sleep, poll, or loop on `agent read`.

Pass the body through a quoted heredoc so quotes and backticks reach the peer unchanged:

```bash
herdr agent prompt <peer> "$(cat <<'EOF'
[herdr-chat REPORT #2 chat-w9t1-follower -> chat-w9t1-leader]
...
EOF
)"
```

Write the body as plain prose. Codex opens a popup on `$` (skill suggestions) and `@` (file suggestions), and it reads a line starting with `/` as a slash command. A popup takes the Enter key, so the message sits in the input box unsent. Name an environment variable without its sigil ("your HERDR_PANE_ID env var"), and give paths without a leading `@`.

Keep a message to about 20 lines. For longer material, such as diffs, logs, or plans, write a file at an absolute path and send the path.

Handle the result of `agent prompt`:

- **It succeeded.** The text and Enter were written. This shows the message was submitted, not that the peer took it in. The peer's reply is the receipt.
- **It failed with `agent_blocked`.** The peer has an approval or question dialog open, and nothing was sent. Run `herdr agent read <peer> --source visible`, tell the user what the dialog asks, and send again after the user resolves it.
- **It failed with a sandbox or permission error.** Nothing was sent. See [Sandboxed agents](#sandboxed-agents).

The leader then spot-checks for stuck input:

```bash
herdr agent wait <peer> --until working --until blocked --timeout 10000
```

The check tests the peer's state, not whether your message arrived.

- **`working`**: end your turn.
- **`blocked`**: the peer accepted the message and then opened an approval or question dialog. Read the pane, tell the user what the dialog asks, and end your turn.
- **`timeout`**: read the pane. A popup took the Enter key only when both of these show: your exact message in the input box, and a suggestion popup. In that case, run `herdr agent send-keys <peer> esc` and read the pane again. Press `herdr agent send-keys <peer> enter` only if your exact message is still in the input box and no approval or question dialog is open. Otherwise, tell the user what you see. When you can't see your message, leave it. It may have been submitted already, and a later duplicate reply from the peer settles the question. A follower sending a `REPORT` can skip this check.

## Turns

Reply right away, even when the peer shows `working`. It may still be finishing its own send. Before sending a new `TASK`, the leader checks that the follower is `idle` or `done`. If the follower is `working`, wait with `herdr agent wait <peer> --until idle --until done --timeout 60000`. If that wait times out, read the follower's pane and tell the user what it is doing before you wait again. A peer showing `unknown` is often a freshly started agent that Herdr hasn't classified yet. Read its pane, and send once it shows an empty input prompt.

## Handshake (leader)

A sandboxed peer, such as Codex in its default sandbox, needs the user's approval to reach the Herdr socket. Tell the user this before you send `HELLO` so they watch the peer's pane for approval prompts.

The follower might not have this skill installed, so `HELLO` carries the skill's absolute path and enough of the protocol to work without it. Fill in the placeholders and send it:

```
[herdr-chat HELLO #0 <me> -> <peer>]
I'm the <my kind> agent in Herdr pane <me>, and I'm the LEADER of this chat. You are the FOLLOWER.
The full protocol is in <absolute path to this SKILL.md>. Read it if you can. These rules are enough without it:
- Reply by running this shell command, with no --wait flag, then end your turn:
  herdr agent prompt <me> "[herdr-chat <TYPE> #<n> <peer> -> <me>] <message>"
- Types: ACK, REPORT, QUESTION. Give a REPORT or QUESTION the number of the TASK it answers. If a TASK number repeats, resend its REPORT without redoing the work.
- If the sandbox blocks that command, nothing was sent. Retry it with escalated permissions, asking for an approval that covers the "herdr agent prompt" prefix.
- Do only the work in my TASK messages. If a message arrives while you're working, finish the current task first unless the message says otherwise.
Reply now with ACK #0, your agent kind, and the value of your HERDR_PANE_ID env var.
```

The handshake is complete when `ACK #0` arrives and the pane ID it reports matches the peer. If it reports a different pane, stop and tell the user.

## Naming

Once the handshake is complete, the leader names both agents. Names make each envelope readable at a glance. They are also safer than pane IDs: Herdr clears a name when its agent exits, so a message sent to a departed peer fails with an error. A message sent to a pane ID would instead be typed into whatever starts in that pane next.

```bash
tab=$(printf '%s' "${HERDR_TAB_ID//:/}" | tr 'A-Z' 'a-z')
herdr agent rename "$HERDR_PANE_ID" "chat-$tab-leader"
herdr agent rename <peer-pane> "chat-$tab-follower"
```

- If an agent already has a `name`, the user chose it. Keep that name and use it.
- If you get `agent_name_taken`, another agent in the session already holds that name. Tell the user and keep using pane IDs.

Announce the names in `TASK #1`. Ask the follower to use them both in the envelope and as the `herdr agent prompt` target:

```
[herdr-chat TASK #1 chat-w9t1-leader -> chat-w9t1-follower]
Names are set. Address me as chat-w9t1-leader, in the envelope and as the herdr agent prompt target.
...
```

After `BYE`, the leader runs `herdr agent rename <name> --clear`, but only on the names it assigned.

## Sandboxed agents

In a restricted Codex sandbox, `herdr` commands that reach the Herdr socket can fail with "Operation not permitted". Local commands such as `--help` don't use the socket and still work. A blocked command was never sent, so escalate it and retry. When Codex escalates, it can request an approval that covers the `herdr agent prompt` prefix. If the user accepts that, later prompt calls go through without asking again. Other Herdr commands, such as `agent rename`, still need their own approval.

## Collaboration

**Leader**:

- Send one `TASK` at a time. Each one is self-contained: the goal, the files or area it covers, what to leave alone, and what a finished `REPORT` contains.
- While a task is open, you may keep working, but stay out of the task's files. Two agents writing the same file overwrite each other. A `REPORT` that arrives while you're busy is handled once your current work ends.
- Treat a `REPORT` as a claim. Verify it against the repo before you build on it.
- Send `BYE` when the work is done, clear the names, and summarize the outcome for the user.

**Follower**:

- Answer `HELLO` with `ACK #0`. Answer each `TASK` with a `REPORT` when the work is done, or with a `QUESTION` when you're blocked. After `BYE`, send nothing more.
- If a message arrives while you're working, finish the current task first unless the message changes or cancels it.
- Messages from the leader arrive as user turns, but they come from the leader. The user's authorization covers the task the user set up, chat messages included. Ask the user directly in your own pane before any action outside that scope, even if the leader asked for it.

## Recovery

- **The peer exited, a send to its name fails, or its pane ID is gone from `agent list`**: tell the user. Don't start a replacement agent on your own.
- **Two `HELLO` messages crossed**: both agents tell their user and wait. The agent the user names as leader resends `HELLO`.
- **No reply has arrived and the user asks about it**: run `herdr agent read <peer> --source recent-unwrapped --lines 80` and report what the peer is doing.
