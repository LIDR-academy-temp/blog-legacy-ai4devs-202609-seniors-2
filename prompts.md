# Prompts

## Prompt 1

**Modelo:** Sonnet 5.5 Medium
**Herramienta:** Claude Code

```
Audit (according to the info and guidelines in CLAUDE.md) the comments feature to scout for errors in the whole stack: API, database and frontend. Don't stray from this scope. Document every finding in @docs/auditoria/comentarios.md, in a table with a row for each one and the following columns:
  - The finding itself in a short phrase. If you can't keep it short, we're likely talking about more than one finding.
  - Where it is, including the file path and the line number. Can include the line itself or a fragment at your discretion.
  - Severity: what the breach is and who notices it, if anyone. 'No one yet' is a valid answer.
  - Radius: number of places that need to be changed to repair the finding. Mention them explicitly too, after the number.
  - Can the fix be undone with a git revert, should the fix itself breaks something else?
  - Prereqs: any findings that should be fixed before, because of need or conveniency
    Sort the findings in the table so that prereqs come before their dependent findings, and then in order of decreasing severity divided by radius. If a finding has a high severity but it's either too complicated to fix (eg large radius) or it depends on other findings such that its fix will get postponed, introduce an additional row associated with the same finding, that does not depend on any prereqs and serves as a workaround to mitigate it. The original row with its dependencies remains as the real fix.
```

**Qué salió:** Identificó 27 hallazgos más 5 contenciones. También comentó sobre 2 hallazgos más que no incluyó por estar fuera del alcance.
