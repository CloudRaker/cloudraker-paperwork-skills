# CloudRaker Paperwork skills

This repo ships agent skills for the `paperwork` CLI (CloudRaker Paperwork API). The skills live in `skills/<name>/SKILL.md` (agentskills.io format) and are what you should read — start with `skills/paperwork/SKILL.md`, the umbrella.

If you are working *in* this repo: skills are hand-written docs, not generated. Keep frontmatter to spec keys (`name`, `description`, `allowed-tools`), keep the house doctrine consistent across skills (wait for digital-text PDFs, `--wait 0` + poll every 10s otherwise; documents never enter agent context; one space per job, id remembered in `.paperwork/space`), and verify command names against `paperwork <resource> --help` before documenting them.
