# skills

Personal Claude Code skills. One directory per skill, each with a `SKILL.md` and optional
`references/`.

| Skill | Scope |
|---|---|
| `code-comments` | What a comment should say, in any language: the test, what to keep, what to delete, why history belongs in the commit message |
| `dotnet-service-skills` | .NET backend services: DDD layering, records and discriminated unions, pattern matching, Orleans, EF Core persistence and migration safety |

## Installing

Claude Code discovers skills in `~/.claude/skills/` (all projects) and
`<repo>/.claude/skills/` (that repo only). Link rather than copy, so edits here take
effect without syncing.

Windows:

```powershell
New-Item -ItemType Junction `
  -Path   "$env:USERPROFILE\.claude\skills\<skill>" `
  -Target "<clone path>\skills\<skill>"
```

macOS / Linux:

```bash
ln -s "<clone path>/skills/<skill>" ~/.claude/skills/<skill>
```

A skill is not loaded on every turn. Claude sees each skill's `name` and `description`
every session, and reads the body only when the task matches. To force one into every
turn of a project, reference it from that project's `CLAUDE.md` instead.

## Changing a skill

Skills are conventions that get applied to code nobody is re-reading at the time, so they
change deliberately: propose, discuss, then commit. Each skill states its own rules for
this — see the `Maintaining this skill` section in its `SKILL.md`.
