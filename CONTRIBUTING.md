# Contributing

Thanks for considering a contribution. This skill is maintained by one person and is specific to AAU AI Cloud, so the bar for scope changes is high — but small fixes, additions, and corrections are very welcome.

## Reporting issues

Open an issue at https://github.com/kuff/aicloudskill/issues. Useful info to include: what you asked Claude, what it produced, and what the cluster did with it (error message, `squeue` output, log excerpt).

## Making changes

- The skill lives at `skills/ai-cloud/`. Edit there.
- After editing, mirror to `~/.claude/skills/ai-cloud/` so the installed copy stays in sync — or symlink it once: `ln -snf "$(pwd)/skills/ai-cloud" ~/.claude/skills/ai-cloud`.
- `SKILL.md` is **model-facing**, not a tutorial. Keep it terse — every sentence should either change what Claude generates or prevent a failure mode.
- Templates in `templates.md` must actually run. Verify with `bash -n` over each fenced bash block before committing.
- Cross-check any policy or hardware claim against https://hpc.aau.dk/ai-cloud/ (or the Service Portal for things not in the public docs). The cluster changes; the docs are the source of truth.

## Style

- Match the existing tone. No emoji. No marketing language.
- Commit messages: imperative mood, short subject line, describe the *why* if it isn't obvious from the diff.
