# Script and agent events

`runrook event` gives scripts, cron jobs and agent hooks a small local queue. It
needs no source key and opens no network connection. On your Mac, `runrook` is
`/Applications/RunRook.app/Contents/MacOS/runrookd` (add an alias or a link named
`runrook` on your PATH); on a computer connected to RunRook over SSH it is the
helper, `~/.local/bin/runrook`. Call it as that computer's SSH account. The Mac reads its own queue and connected SSH helpers with
the `events` capability, at most once every five seconds per computer. An older
helper must be updated using its existing Update action first. Linked computers
that report over HTTPS do not drain this queue yet.

The first valid event creates a custom source called `Scripts on this Mac` or
`Scripts on <computer name>`. Runs belong to that computer. Removing the source
stops delivery: events that computer queues afterwards are discarded, and the
source isn't created again. Sources then shows a note for that computer with
Receive them again; its next event creates the source again.

## Commands

```text
runrook event start [--run ID] [--label TEXT] [--ttl SECONDS]
runrook event update [--run ID] [--label TEXT] [--ttl SECONDS]
runrook event wait [--run ID] [--label TEXT] [--question TEXT] [--ttl SECONDS]
runrook event end [--run ID] [--label TEXT] --outcome succeeded|failed|cancelled [--ttl SECONDS]
runrook event run [--run ID] [--label TEXT] [--ttl SECONDS] -- COMMAND [ARGS...]
```

`start`, `update`, `wait` and `end` map to `run.started`, `run.updated`,
`run.waiting` and `run.ended`. Only `wait` accepts `--question`; only `end` accepts
`--outcome`, and requires it. See `runrook event --help` for individual examples.
The run ID comes from `--run`, then `RUNROOK_RUN_ID`. Without either, `run`
generates a random ID; the other forms fail with an instruction to provide one.
IDs contain 1 to 128 ASCII letters, digits, dots, colons, underscores or dashes,
starting with a letter or digit. Labels and questions have a 200-byte limit.

TTL is 30 to 86400 seconds, default 1800. Send an update before it expires for
long-running work; otherwise RunRook eventually shows unknown. Waiting questions
are owner-only: viewers see the waiting state and a generic timeline entry.

`run` directly executes the argument list, with the same stdin, stdout, stderr and
environment plus `RUNROOK_RUN_ID`. It queues the start and end, preserves the
child's exit status, and reports success, failure or signal cancellation. On Unix
a signal exit uses the usual `128 + signal` code. Interrupt and termination
signals received by the wrapper are forwarded to the child. Put shell syntax in
an explicit script if you need it.

Each ordinary event call returns 0 after a local write, or 1 with one plain
sentence explaining a fix. A wrapper whose initial event cannot be queued returns
1 before starting the command. If its final event write fails, it prints the fix
and preserves the child's exit code. Queue writes time out after 60 ms so an agent
hook does not wait on a stalled filesystem. Calls take no network or database locks.
A call takes about 10 to 20 ms (20 ms measured on a small Linux server). The first
run of a newly installed binary can take longer while the operating system checks
it. Atomic rename prevents readers seeing
partial writes; writes do not wait for a disk sync, so power loss can lose a recent
event. Delivery retries after store failures and deduplicates repeated events.

The queue is `$HOME/.local/state/runrook/events` on Unix, including macOS, and
`%LOCALAPPDATA%\RunRook\events` on Windows. Its directory is private (0700), with
0600 event files. It holds at most 500 events and 2 MiB, dropping oldest events
when full while still accepting the new event. Temporary files from interrupted
writes are ignored and removed after one minute. This is bounded, best-effort status reporting.

## Cron job

Make sure cron's PATH includes the installed `runrook`. For example, place the
following command in a nightly job (or use it as the command field of your cron
entry):

```sh
runrook event run --label "Nightly backup" -- /usr/local/bin/backup.sh
```

The backup's input, output and exit status pass through. For example, a crontab
line for 02:00 is `0 2 * * *` followed by the command above. Tests substitute an
executable backup fixture for the site-specific backup path.

## Long script with a wait

Save this as a script and run it in a terminal. Its PID gives concurrent runs
different IDs. For longer work, repeat `update` at an interval shorter than TTL.
The event calls use `|| true` so status reporting failures cannot stop your work.

```sh
#!/bin/sh
RUNROOK_RUN_ID="long-script-$$"
export RUNROOK_RUN_ID
runrook event start --label "Long script" --ttl 3600 || true
runrook event update --label "Preparing files" --ttl 3600 || true
runrook event wait --question "Continue with the prepared files?" --ttl 3600 || true
printf 'Press Enter to continue: '
read -r answer
runrook event update --label "Finishing files" --ttl 3600 || true
runrook event end --outcome succeeded || true
```

## Hermes Agent plugin

**Not run against a real Hermes yet.** The recipe is tested with a stub hook
registry and real local CLI calls. It follows Hermes'
[plugin registration](https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins)
and [human-input hooks](https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks).
`on_session_end` runs at the end of a conversation turn, so this recipe ends that
turn's run; a later wait or update can refresh the same session ID.

Create `~/.hermes/plugins/runrook-events/plugin.yaml`:

```yaml
name: runrook-events
version: "1.0"
description: Send local run status to RunRook
provides_hooks:
  - on_human_input_request
  - on_human_input_resolved
  - on_session_end
```

Create `~/.hermes/plugins/runrook-events/__init__.py`:

```python
import hashlib
import os
import re
import subprocess

_FALLBACK = "hermes-process-" + str(os.getpid())


def _run_id(session_id):
    if not session_id:
        return _FALLBACK
    candidate = "hermes-" + str(session_id)
    if re.fullmatch(r"[A-Za-z0-9][A-Za-z0-9._:-]{0,127}", candidate):
        return candidate
    return "hermes-" + hashlib.sha256(str(session_id).encode()).hexdigest()


def _send(kind, session_id=None, extra=()):
    argv = ["runrook", "event", kind, "--run", _run_id(session_id)]
    argv += ["--label", "Hermes Agent", "--ttl", "3600"] + list(extra)
    try:
        subprocess.run(argv, stdin=subprocess.DEVNULL,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
                       timeout=0.2, check=False)
    except (OSError, subprocess.TimeoutExpired):
        pass


def register(ctx):
    def waiting(session_id=None, prompt="", **kwargs):
        # Count UTF-8 bytes, the same limit as custom events.
        question = str(prompt).encode("utf-8")[:200].decode("utf-8", "ignore")
        _send("wait", session_id, ("--question", question))

    def resumed(session_id=None, **kwargs):
        _send("update", session_id)

    def ended(session_id=None, completed=False, interrupted=False, **kwargs):
        outcome = "cancelled" if interrupted else ("succeeded" if completed else "failed")
        _send("end", session_id, ("--outcome", outcome))

    ctx.register_hook("on_human_input_request", waiting)
    ctx.register_hook("on_human_input_resolved", resumed)
    ctx.register_hook("on_session_end", ended)
```

Enable it with `hermes plugins enable runrook-events` and restart Hermes. Keep
`runrook` on the Hermes process's PATH. Hooks use list argv, suppress reporting
errors, and impose a 200 ms subprocess timeout. A supplied session ID becomes
`hermes-<session id>`; invalid or oversized IDs are hashed, and missing IDs share a
process-scoped fallback. No answer or password is sent. A wait longer than an hour
needs a refresh or a larger TTL.

## OpenClaw plugin

**Not run against a real OpenClaw yet; OpenClaw marks its plugin APIs experimental.**
OpenClaw's metadata-only lifecycle hooks `session_start` and `session_end` need no
conversation access, so this plugin reports when a session starts and ends. OpenClaw
has no hook for "waiting for you", so waits aren't reported.

`runrook-events/package.json`:

```json
{ "name": "runrook-events", "type": "module", "openclaw": { "extensions": ["./index.ts"] } }
```

`runrook-events/openclaw.plugin.json`:

```json
{ "id": "runrook-events", "name": "RunRook events", "description": "Tells RunRook when sessions start and end." }
```

`runrook-events/index.ts`:

```ts
import { execFile } from "node:child_process";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

function report(args: string[]) {
  // Never block or fail the agent: a short timeout, errors ignored.
  execFile("runrook", ["event", ...args], { timeout: 2000 }, () => {});
}

function runId(ctx: { sessionId?: string; sessionKey?: string }) {
  const id = (ctx.sessionId ?? ctx.sessionKey ?? "").replace(/[^A-Za-z0-9._:-]/g, "-").slice(0, 100);
  return id ? "openclaw-" + id : "";
}

export default definePluginEntry({
  id: "runrook-events",
  register(api) {
    api.on("session_start", (_event, ctx) => {
      const run = runId(ctx);
      if (run) report(["start", "--run", run, "--label", "OpenClaw", "--ttl", "3600"]);
    });
    api.on("session_end", (_event, ctx) => {
      const run = runId(ctx);
      if (run) report(["end", "--run", run, "--outcome", "succeeded"]);
    });
  },
});
```

Install it with `openclaw plugins install --link ./runrook-events` and
`openclaw plugins enable runrook-events`, with `runrook` on the PATH of the
OpenClaw gateway.

## Any agent with hooks

Adapt these Python callbacks to your agent's hook registry. Have the agent supply
one stable run ID per session. Calls return after a short, local queue write, and
reporting failure does not fail the hook.

```python
import subprocess


def report(kind, run_id, *extra):
    try:
        subprocess.run(["runrook", "event", kind, "--run", run_id, *extra],
                       stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL,
                       stderr=subprocess.DEVNULL, timeout=0.2, check=False)
    except (OSError, subprocess.TimeoutExpired):
        pass


def on_start(run_id):
    report("start", run_id, "--label", "My agent")


def on_input_needed(run_id):
    report("wait", run_id, "--question", "Review the agent's request?")


def on_input_resolved(run_id):
    report("update", run_id)


def on_finished(run_id, outcome):
    report("end", run_id, "--outcome", outcome)
```

Use a fresh run ID for a new job, and map your hook's result to `succeeded`,
`failed` or `cancelled`. Refresh TTL during long jobs. These hooks only report
status; they do not carry answers back to the agent.
