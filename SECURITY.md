# Security

## Reporting a problem

Please report security problems privately, not in a public issue: use **[Report a vulnerability](https://github.com/Adineh55/RunRook/security/advisories/new)** on this repository. Include the RunRook version (shown at the bottom of Settings), what you did, and what happened. You'll get a reply within a week, and a fix or a plan within 30 days. Once a fixed release is out, you're welcome to publish your findings, and you'll be credited unless you'd rather not be.

Supported: the latest release. Updates install automatically, so fixes reach everyone through the normal update.

## What's in scope

- The RunRook app and its service on the Mac.
- The helper RunRook installs on your other computers, and the setup commands it shows.
- The update feed and the release files in this repository.

Reports about the agents RunRook watches (Claude Code, Codex, Cursor, Ollama, LM Studio) belong with their makers.

## How releases are protected

- Every release is signed with Developer ID (Horizon Path SPC, team B96DZZW55C) and notarized by Apple.
- Updates are signed with a separate key that never leaves the maker’s Mac. The app refuses an update that this key didn't sign, even one served from this repository.
- Each release's `MANIFEST.txt` lists the SHA-256 of every file. Releases here can't be changed after they're published.
