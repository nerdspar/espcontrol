# Kickoff Prompt for Claude Code

Paste this as the first message after `cd`-ing into your fork of espcontrol
and starting Claude Code. The CLAUDE.md in the repo root has the full context;
this prompt just gets the session moving.

---

I'm porting EspControl to the Guition JC8048W550C (5" ESP32-S3, 800×480, GT911
touch). Read CLAUDE.md in the repo root first — it has the hardware specs,
porting decisions, and task order.

I've already scaffolded the new device folder at
`devices/guition-esp32-s3-jc8048w550c/` and the build profiles in `builds/`.
Verify the scaffold looks right against the reference at
`devices/guition-esp32-s3-4848s040/`, then take me through Task 1 from
CLAUDE.md: wire the new device into the build system by updating
`scripts/device_slots.json` (a snippet is provided at
`device_slots.snippet.json` in the repo root) and running
`scripts/generate_device_slots.py`.

After that's done, do a Docker compile of the factory build YAML. Don't flash
yet — I want to see the compile is clean first. If the compile fails, show me
the full error output and propose a fix; don't apply the fix until I confirm.

When you're ready to flash, ask me to plug in the panel and tell you the
serial port.
