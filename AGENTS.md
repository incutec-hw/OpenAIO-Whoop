# Agent notes

Facts for AI agents working in this repo.

- KiCad 10 project. Top schematic `hardware/OpenAIO-Whoop.kicad_sch`, sub-sheets under `hardware/schematics/` (fc, esc, elrs), board `hardware/OpenAIO-Whoop.kicad_pcb`.
- Early design: the ESC channel sheet (`hardware/schematics/esc/esc_channel.kicad_sch`) and the board layout are inherited donor content, not the Bluejay whoop design. Do not treat them as current.
- Clone with `git clone --recursive`; the `libs/KiCad-Library` submodule is referenced by the project lib tables for shared parts.
- Never edit `.kicad_*` files as text. Use kicad-skip or the pcbnew API, and only for metadata (text variables, symbol BOM/doc fields). Never change nets, placement, or component values.
- Checks and exports:

```
kicad-cli sch erc --exit-code-violations hardware/OpenAIO-Whoop.kicad_sch
kicad-cli pcb drc --exit-code-violations hardware/OpenAIO-Whoop.kicad_pcb
kicad-cli sch export netlist --format kicadsexpr -o /tmp/OpenAIO-Whoop.net hardware/OpenAIO-Whoop.kicad_sch
```

- Fabrication Toolkit exports land in `hardware/production/` (gitignored). Datasheets live in `hardware/datasheets/` (gitignored, local only).
- Docs are deterministic: current fact only, no TODOs or plans.
- `main` is protected; push feature branches and open PRs.
