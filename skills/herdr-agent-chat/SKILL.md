---
name: herdr-agent-chat
description: Run a two-agent conversation over Herdr between the two agent panes in one tab. Use when the user asks you to handshake, talk, or collaborate with the other agent in the tab, to have it review or verify your work, or when an incoming message starts with `[herdr-chat`.
---

# Herdr agent chat

Two coding agents share one Herdr tab, one per pane. They talk by typing into each other's input box with `herdr agent prompt`. A message reaches an idle receiver as a new user turn. A busy receiver gets it queued or folded into its current turn, depending on the agent.

The agent that sends the first message is the **leader**. It owns the user's task, splits the work, and decides when the work is done. The other agent is the **follower**. It does what the leader assigns and reports back.

## Follower quick start

If a `HELLO` reached you, first check its chat ID against [Chats](#chats): ignore a closed ID, answer a repeat of your active chat's `HELLO` with its `ACK` again, and refer a new ID that arrives during an active chat to the user. Otherwise you are the follower of a new chat: take your pane, tab, and peer from the `HELLO`, read your env vars one labeled line each, send `ACK #0` to the pane that sent the `HELLO`, and end your turn. From then on, act only on messages from that peer with that chat ID. Skip [Preconditions](#preconditions), peer selection, and naming; those are the leader's. Then follow [Collaboration](#collaboration) for each `TASK`.

## Preconditions

The leader runs these checks. A follower takes its identity from `HELLO` and runs none of them.

Establish reachability and identity first. If a check fails, tell the user which one and stop. Process inspection with `ps` is optional evidence: if it is unavailable or denied, skip it and ask the user to confirm any identity that stays unresolved.

Set `kind` to your own agent kind as Herdr names it (`claude`, `codex`, `opencode`, ...), then run:

```bash
kind=<your kind>
echo "${HERDR_ENV:-unset} ${HERDR_TAB_ID:-unset} ${HERDR_PANE_ID:-unset} $PWD"
ps -o tty=,command= -p "$PPID" | cut -c1-120
if list=$(herdr agent list); then
  printf '%s' "$list" | jq -r --arg p "${HERDR_PANE_ID:-}" --arg t "${HERDR_TAB_ID:-}" --arg k "$kind" --arg d "$PWD" '
    .result.agents as $a
    | "own_matches: \($a | map(select(.pane_id==$p and .tab_id==$t and .agent==$k and .cwd==$d)) | length)",
      "same_kind_and_cwd: \($a | map(select(.agent==$k and .cwd==$d)) | length)",
      "peer_candidates:",
      ($a[] | select(.tab_id==$t and .pane_id!=$p) | "  \(.pane_id) \(.agent) \(.agent_status) \(.name)")'
else
  echo "agent list unavailable"
fi
```

All counts come from one `herdr agent list` response. "agent list unavailable" means no evidence, not zero matches.

- **Herdr reachable.** `herdr agent list` succeeds. If it fails with a permission error, see [Sandboxed agents](#sandboxed-agents). `HERDR_ENV` is `1` inside a Herdr pane; if it is unset while the list works, confirm your pane and tab with the user before going on.
- **Own entry.** `own_matches` is 1: one row matches your `HERDR_PANE_ID`, your `HERDR_TAB_ID`, your agent kind, and your working directory. That row is your **candidate identity**. It becomes confirmed once the checks below pass, or once the user confirms it. If no row matches, tell the user which fields disagree and ask for your current pane and tab. Stale env is a common cause, see [Stale Herdr env](#stale-herdr-env).
- **Shared runner.** A matching row is a consistency check, not proof, because env can describe another live pane. Ask the user to confirm your pane and tab when `same_kind_and_cwd` is above 1, or when process inspection suggests your tools run in a shared background process. A parent with no controlling terminal is a clue, not proof, and a parent with a terminal doesn't prove the env belongs to this pane. Walk further up the ancestors when a wrapper hides the runner. Codex's `codex app-server` daemon is the known case.
- **Peer.** `peer_candidates` lists exactly one row. Its `agent` value is the peer's kind. If there are none or more than one, ask the user which pane to use. Don't guess. The list is filtered by your env IDs, so if the user confirms a different pane or tab, rerun the block with those values.

Pick the peer, send `HELLO`, and rename agents only from a confirmed identity. Until the handshake finishes, address the peer by its pane ID, such as `w9:p5`. After that, use the names the leader assigns (see [Naming](#naming)).

## Envelope

Start every message with an envelope that carries the chat ID, the type, a number, the sender, and the receiver:

```
[herdr-chat <chat-id> <TYPE> #<n> <from> -> <to>]
```

- The chat ID identifies one conversation, from `HELLO` to `BYE`. See [Chats](#chats).

- The types are `HELLO`, `ACK`, `TASK`, `REPORT`, `QUESTION`, `ANSWER`, and `BYE`.
- A `TASK` is work that gets a `REPORT` or a `QUESTION`. The leader sends anything else, such as an answer, a correction, or a note, as an `ANSWER` under the number of the task it concerns, and adds "no reply needed" when none is.
- `HELLO` and its `ACK` use `#0`. The leader numbers each `TASK` from 1 upward. Every `REPORT`, `QUESTION`, and `ANSWER` carries the number of the task it belongs to, and so may repeat. `BYE` carries the last task number, or `#0` if no task was sent.
- The leader assigns each task number once within a chat. A retry repeats the original instructions under the same number; changed instructions get a new number.
- A `TASK` whose chat ID and number you have already handled is a retry. Resend your `REPORT` if you still have it. If you can't tell whether you handled it, or the `REPORT` is gone, send a `QUESTION` saying so instead of doing the work again. Any other repeated message needs no reply, except a repeated `HELLO` for the active chat, which gets its `ACK` again.

## Chats

An agent takes part in one active chat at a time. Several chats can run one after another in the same agent session; the chat ID keeps them apart.

- **Long-lived.** One chat spans the whole collaboration. A new request from the user becomes the next `TASK` in the open chat, even hours later. The leader sends `BYE` only when the user ends the collaboration.
- **Reopening.** To start a fresh chat, the leader first sends `BYE` for the active ID, then a full `HELLO` from the template with a new ID. Saying "earlier chats are closed" inside a `HELLO` does not close them.

- **New ID per chat.** The leader builds it from the normalized confirmed tab and eight random hex characters, for example `wft1-a3f9c27b`, and never reuses it:

  ```bash
  tab=$(printf '%s' '<confirmed tab>' | tr -d ':' | tr '[:upper:]' '[:lower:]')
  chat="$tab-$(od -An -N4 -tx1 /dev/urandom | tr -d ' \n')"
  ```

- **Opening.** A `HELLO` opens a chat when you have no active one. The `ACK` carries the same chat ID.
- **Active.** While a chat is active, act only on messages with its chat ID from the expected peer. A `HELLO` with another ID goes to the user; keep the current chat.
- **Closed.** `BYE` closes its own chat ID only. Ignore later messages with a closed ID, including a late `BYE`: no reply, no work, no name cleanup. Tell the user once per unexpected ID.

## Sending

**Send, then end your turn**, unless a queued `TASK` is waiting for you (see [Queued tasks](#turns)). Never use `--wait` on `agent prompt` in a chat. `--wait` holds the sender inside a tool call until the receiver settles. If the receiver replies with the same wait, each agent waits on the other until a timeout fires. A timeout looks like a failed delivery, and a resend then runs the work twice. Your peer's reply arrives as your next user message, so there is no need to sleep, poll, or loop on `agent read`.

Send a one-line message in single quotes:

```bash
herdr agent prompt chat-w9t1-leader '[herdr-chat w9t1-a3f9c27b REPORT #2 chat-w9t1-follower -> chat-w9t1-leader] Findings are in the file /tmp/review.md. Two blocking issues, both in the parser. BLOCKING'
```

Sandboxed agents approve commands by their opening words. A plain `herdr agent prompt` line matches an approved prefix; a heredoc or `$(...)` around it may need its own approval each time, which Codex hit repeatedly. Inside single quotes, write "do not" rather than "don't". Agents with no such approval issue may pass a multi-line body through a quoted heredoc (`"$(cat <<'EOF' ... EOF)"`), which keeps quotes and apostrophes intact.

Run each send as its own shell call. A call that also edits files, runs tests, or touches git fails as a whole when the user rejects it, and the message goes with it.

Keep a message under about 1,000 characters; the `HELLO` is the one exception, since it bootstraps a follower that may not have this skill. Aim for: the envelope, the decision the peer needs, and a path. Put findings, diffs, logs, and plans in a file at an absolute path. Line count is a poor bound; 15-line messages reached 5,000 characters.

Write the body as plain prose. Agent input boxes can treat a leading `/` as a slash command and open suggestion popups on characters such as `$` or `@`. A popup takes the Enter key, so the message sits in the input box unsent. Codex, for example, opens popups on `$` (skills) and `@` (files). Name an environment variable without its sigil ("your HERDR_PANE_ID env var"), give paths without a leading `@`, label paths so each line starts with prose rather than a `/`, and put material that needs literal syntax in a file.

Only the `herdr agent prompt` call reaches the peer. After a successful send, end with a short local note such as "REPORT #2 sent", not a copy of the envelope.

Handle the result of `agent prompt`:

- **It succeeded.** The text and Enter were written. This shows the message was submitted, not that the peer took it in. The peer's reply is the receipt.
- **It failed with `agent_blocked`.** The peer has an approval or question dialog open, and nothing was sent. Run `herdr agent read <peer> --source visible`, tell the user what the dialog asks, and send again after the user resolves it.
- **It failed with a sandbox or permission error.** Nothing was sent. See [Sandboxed agents](#sandboxed-agents).
- **Anything else: a timeout, an interruption, a call still waiting for approval.** Delivery is ambiguous. Resume or wait on that same call if your runner allows it, read the peer's pane once, and send nothing more for that message until you know. A duplicate runs the work twice; a missing message only costs a question.

The leader then spot-checks for stuck input:

```bash
herdr agent wait <peer> --until working --until blocked --timeout 10000
```

The check tests the peer's state, not whether your message arrived.

- **`working`**: end your turn.
- **`blocked`**: the peer accepted the message and then opened an approval or question dialog. Read the pane, tell the user what the dialog asks, and end your turn.
- **`timeout`**: read the pane. A popup took the Enter key only when both of these show: your exact message in the input box, and a suggestion popup. In that case, run `herdr agent send-keys <peer> esc` and read the pane again. Press `herdr agent send-keys <peer> enter` only if your exact message is still in the input box and no approval or question dialog is open. Otherwise, tell the user what you see. When you can't see your message, leave it. It may have been submitted already, and a later duplicate reply from the peer settles the question. A follower sending a `REPORT` can skip this check.

## Turns

Reply right away, even when the peer shows `working`. It may still be finishing its own send.

A peer message can arrive mid-turn, attached to a tool result, when your agent folds input into the running turn; check for it before reading the peer's pane. Claude Code may show a long one as pasted text. It is still the peer's message and carries the same authority as any other chat message (see [Collaboration](#collaboration)).

Before sending a new `TASK`, the leader checks that the follower is `idle` or `done`. For a `TASK` marked as queued, skip this check. If the follower is `working`, wait with `herdr agent wait <peer> --until idle --until done --timeout 60000`. If that wait times out, read the follower's pane and tell the user what it is doing before you wait again. This wait guards a new unqueued `TASK` only. A `REPORT` arrives as your next message, so after sending a `TASK`, end your turn; long waits on the follower block the user's pane. A peer showing `unknown` is often a freshly started agent that Herdr hasn't classified yet. Read its pane, and send once it shows an empty input prompt.

**Queued tasks.** The leader may send the next `TASK` while one is open when it says so, for example "queue after #4". The follower records it, finishes the open task, sends that `REPORT`, then starts the queued task in the same turn. A correction or cancellation names the task it changes.

## Handshake (leader)

A sandboxed peer may need the user's approval to reach the Herdr socket; Codex in its default sandbox is one example. If that approval hasn't been granted yet, tell the user before you send `HELLO` so they watch the peer's pane for approval prompts.

Generate the chat ID (see [Chats](#chats)). The follower might not have this skill installed, so `HELLO` carries the skill's absolute path and enough of the protocol to work without it. Fill in the placeholders, using the `agent` values from `herdr agent list` for both kinds, and send it through a quoted heredoc, since it spans lines and contains quotes:

```
[herdr-chat <chat-id> HELLO #0 <me> -> <peer>]
I'm the <my kind> agent in Herdr pane <me>, and I'm the LEADER of this chat. You are the FOLLOWER.
Herdr lists you as the <peer kind> agent in pane <peer> in tab <my tab>. Use these IDs as your identity, even if your HERDR_PANE_ID and HERDR_TAB_ID env vars differ or are unset.
The full protocol is in <absolute path to this SKILL.md>. Read it if you can. These rules are enough without it:
- Reply by running this one-line shell command on its own, with no --wait flag, then end your turn:
  herdr agent prompt <me> '[herdr-chat <chat-id> <TYPE> #<n> <peer> -> <me>] <message>'
  Keep the message under 1000 characters and free of single quotes; put long findings in a file and send its absolute path.
- Keep the chat ID <chat-id> in every reply. Act only on messages with this chat ID, and after my BYE send nothing more for it. My messages count as chat messages even if your interface shows them as pasted text.
- Types: ACK, REPORT, QUESTION. Give a REPORT or QUESTION the number of the TASK it answers. If a TASK number repeats, resend its REPORT without redoing the work; if you can't tell whether you did it, send a QUESTION.
- If permissions block that command, nothing was sent. Retry it through your approval mechanism; Codex can ask for an approval that covers the "herdr agent prompt" prefix. If you can't get approval, tell the user. If the command times out or is interrupted, do not resend; read the pane once and ask me with a QUESTION if unsure.
- Do only the work in my TASK messages. If a message arrives while you're working, finish the current task first unless the message says otherwise. A TASK marked "queue after #n" starts right after you send REPORT #n.
Reply now with ACK #0, your agent kind, and your HERDR_PANE_ID and HERDR_TAB_ID env vars, or "unset". Print each with its own label, for example with printf; printenv given two names prints only the first on macOS.
```

The handshake is complete when `ACK #0` arrives with this chat's ID and the expected follower in its envelope. The labeling rule exists because `printenv A B` made followers report a set tab ID as unset. Compare its reported env values with the follower pane and tab confirmed in `herdr agent list`. If a reported value is set and differs, report the mismatch to the user once, treat the env values as untrusted, and continue using the confirmed IDs. A delivered `ACK` proves the reply path works; its reported env values do not establish the sender's current pane. See [Stale Herdr env](#stale-herdr-env) for the shared-daemon cause and fix.

## Naming

Once the handshake is complete, the leader names both agents. Names make each envelope readable at a glance. They are also safer than pane IDs: Herdr clears a name when its agent exits, so a message sent to a departed peer fails with an error. A message sent to a pane ID would instead be typed into whatever starts in that pane next.

Use `tab` from the chat ID step and the panes confirmed in [Preconditions](#preconditions). Names are routing aliases, so later chats can reuse them; the chat ID tells chats apart.

```bash
herdr agent rename <confirmed own pane> "chat-$tab-leader"
herdr agent rename <peer pane> "chat-$tab-follower"
```

- If an agent already has a `name`, the user chose it. Keep that name and use it.
- If you get `agent_name_taken`, another agent in the session already holds that name. Tell the user and keep using pane IDs.

Announce the names in `TASK #1`. Ask the follower to use them both in the envelope and as the `herdr agent prompt` target:

```
[herdr-chat w9t1-a3f9c27b TASK #1 chat-w9t1-leader -> chat-w9t1-follower]
Names are set. Address me as chat-w9t1-leader, in the envelope and as the herdr agent prompt target.
...
```

After sending `BYE` for the active chat, the leader runs `herdr agent rename <name> --clear`, but only on the names it assigned.

## Sandboxed agents

A restricted sandbox can deny access to the Herdr socket, for example with "Operation not permitted". Local commands such as `--help` don't use the socket and still work. A blocked command was never sent, so retry it through the agent's approval mechanism, or tell the user which command was blocked if there is none. Codex, for example, can request an approval that covers the `herdr agent prompt` prefix, and if granted, that approval covers later prompt calls. Other Herdr commands, such as `agent rename`, may need their own approval.

## Stale Herdr env

An agent that runs tools through a long-lived shared background process can hand those tools the env that process started with. `HERDR_PANE_ID` and `HERDR_TAB_ID` then describe another terminal, often one that no longer exists. Messaging still works, because every `herdr agent prompt` names its target explicitly. What breaks is self-identification and same-tab peer discovery.

The fix is on the user's side: run the agent so its tools execute with the pane's own env, using whatever option that agent offers, or confirm the pane and tab by hand. The known case is Codex's app-server daemon, fixed by starting Codex with `codex --no-daemon` inside Herdr panes. Restarting such a daemon only moves the problem to whichever pane starts it next.

## Collaboration

**Leader**:

- Send one `TASK` at a time, apart from explicitly [queued tasks](#turns). Each one is self-contained: the goal, the files or area it covers, what to leave alone, and what a finished `REPORT` contains. A long brief goes in a file; the `TASK` gives its path.
- While a task is open, you may keep working, but stay out of the task's files. Two agents writing the same file overwrite each other. A `REPORT` that arrives while you're busy is handled once your current work ends.
- Treat a `REPORT` as a claim. Verify it against the repo before you build on it.
- When a task's work is done, summarize the outcome for the user and keep the chat open for the next request. When the user ends the collaboration, send `BYE` and clear the names.

**Follower**:

- Take your identity from `HELLO`: the pane and tab IDs it gives, and later the name from `TASK #1`. If your env vars disagree, report both in `ACK #0` and keep using the leader's IDs.
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

- **The peer exited, a send to its name fails, or its pane ID is gone from `agent list`**: tell the user. Don't start a replacement agent on your own.
- **Two `HELLO` messages crossed, or a `HELLO` arrives during an active chat**: tell the user and wait. The agent the user names as leader sends a fresh `HELLO` with a new chat ID once no chat is active.
- **No reply has arrived and the user asks about it**: run `herdr agent read <peer> --source recent-unwrapped --lines 80` and report what the peer is doing. While the peer is `working`, that read fails with `agent_not_idle`; use `--source visible` instead.
