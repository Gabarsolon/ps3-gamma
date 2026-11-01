# ps3-gamma

Question: can a PS3 (CFW Evilnat 4.93 + Cobra, slim CECH-25xx) do gamma / black-level correction on its analog AV Multi-Out, the way `xrandr --gamma` does on a Linux box with a VGA2SCART cable? Owner's own console, own CRT. Research and static analysis only; nothing has been run on the console for this project.

See [docs/RESEARCH.md](docs/RESEARCH.md).

**Short version:** the XMB's "RGB Full Range" and "Super-White" settings are HDMI-only and do nothing on analog output. A per-app route exists today (RetroArch's `image-adjustment.cg` shader). A system-wide route would need an RSX display-engine gamma table; whether the PS3's RSX actually has one at the addresses the report describes is an unverified hypothesis, not a finding, and no such write has been attempted on the console.

Rules: own console, no firmware files in this repository (Sony-derived files stay in a local, gitignored `local_only/`), research and static analysis only unless the owner says otherwise.
