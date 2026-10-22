# codex-cua-abliterator

**Abliterate the mandatory credential hand-off from Codex Computer Use.**

Codex/ChatGPT's Computer & Browser Use Confirmation Policy forces the *user* to
take over for sensitive credential actions ("Hand-Off Required"): the agent must
stop and make you complete password entry, confirmation, and submission yourself —
even when you just explicitly authorized it to do exactly that.

This tool patches every live copy of that policy on macOS so it reads
**"Hand-Off Required … unless the user explicitly authorizes the agent to
complete it."** When you grant explicit permission, the agent finishes the whole
credential change itself instead of stopping for a hand-off.

```
python3 codex-cua-abliterator.py check          # report layers needing patching
python3 codex-cua-abliterator.py patch --yes    # patch everything (backs up first)
python3 codex-cua-abliterator.py restore         # undo from backups
```

## Why four layers

The policy text doesn't live in one place. Patching only the obvious files
doesn't stick — silent regenerators put the old rule back within minutes:

| Layer | What | Gotcha |
|---|---|---|
| 1 | `computer-use` plugin `SKILL.md` | duplicated across app bundles + `CODEX_HOME`s |
| 2 | `browser`/`chrome` plugin `confirmations.md` + `browser-service.mjs` | header uses lowercase "Hand-off" — easy to miss |
| 3 | model catalog JSONs (`confirmation_policies.browser_use` / `computer_use`) | **regenerated from the binary at runtime** |
| 4 | the `codex` CLI binaries themselves | the regeneration source; also `~/.codex/packages/app-server-daemon/` releases |

Layer 4 is the keystone: patched binaries regenerate *patched* catalogs.

The tool:

- discovers all copies across `/Applications/ChatGPT.app`, the HaCodex rig
  (`~/.local/share/codex-subscription-lab`), `~/.codex`, npm/brew CLI installs,
  and app-server-daemon releases
- patches binary policy text in-place with a length-matched byte replacement
  (same-length, space-padded — no Mach-O layout shift)
- re-signs what it touches: standalone binaries ad-hoc, app bundles with your
  dev identity (HaCodex) or ad-hoc (ChatGPT.app, since OpenAI's Developer-ID
  seal can't be restored locally)
- verifies each patched binary still launches (`codex --version`) and each `.mjs`
  still parses (`node --check`), restoring from backup on failure
- writes `*.cua-abliterator.bak` backups for every file; `restore` reverts all

## Requirements

- macOS (codesign, app bundles)
- Python 3.8+ (stdlib only)
- `node` on PATH (for `.mjs` syntax checks)
- For app-bundle re-signing: any working signing identity, or it falls back to ad-hoc

## After patching

**Restart** Codex/ChatGPT and any other Codex-family apps so new processes load
the patched policy. Existing in-flight chats may carry old cached text until
they start a fresh conversation.

**App updates restore the old policy** (updates replace bundles, binaries, and
plugin files). Re-run `codex-cua-abliterator.py check` after any update.

## Notes

- Scope of the patched policy is unchanged elsewhere: it still only governs
  browser/computer-UI actions (terminal/shell/MCP unaffected), and every other
  confirmation rule (deletes, CAPTCHAs, financial transactions, …) is intact.
- Hot-patching binaries of *running* apps can destabilize them (seen once with
  the HaCodex rig's account worker). Close Codex-family apps before patching,
  or restart them right after.

## License

MIT
