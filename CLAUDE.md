# homelab-infrastructure

Documentation and config templates for a 7-VLAN home network on OPNsense + Suricata + WireGuard + Unbound + TIG monitoring. Maintained by Bastiaan ([@bastiaan365](https://github.com/bastiaan365)).

This file scopes Claude's behaviour for this repo. The global `~/.claude/CLAUDE.md` covers personal conventions; everything below is repo-specific.

## What this repo is and is not

- **Is**: a public-facing reference describing the architecture, with template configs that another homelab operator could adapt to their own network.
- **Is not**: a runnable deployment, a copy of my live OPNsense config, or anything containing my real IPs / hostnames / MACs / WireGuard public keys.
- **Is not**: a substitute for the running configuration on the actual OPNsense box. The truth is on the device; this repo is the *intent* and the *teachable summary*.

Anonymisation is the load-bearing property. If a config in this repo could be applied to my network as-is by someone reading it, that's a leak. If it requires substituting placeholders to work, that's correct.

## Repo conventions

### Structure

```
configs/
├── opnsense/        firewall-rules.md, vlan-assignments.md (markdown tables + commentary)
├── suricata/        custom-rules/*.rules, suricata-overrides.yaml
└── wireguard/       *.conf.example (templates, never real keys)
diagrams/            network-topology.md (ASCII / Mermaid; no embedded internal hostnames)
docs/                hardware.md and any deeper write-ups
README.md            the public landing page
```

Add a new top-level directory only when an existing one no longer fits. Prefer adding files inside the current categories.

### Anonymisation rules (non-negotiable)

- **No real IPs.** The repo's accepted placeholder convention is **`10.10.X0.0/24`** (e.g. `10.10.10.0/24` LAN, `10.10.20.0/24` IoT, `10.10.30.0/24` Guest, ...) — these are documentation placeholders chosen specifically because they don't overlap with my real subnets. Never use my actual subnets: `192.168.178.x` (IOT_TIG), `192.168.10.x` (IOT_HA), `192.168.100.x` (IoT VLAN), `192.168.1.x` (LAN_FRITZBOX), `192.168.254.x` (ADMIN).
- **No real hostnames.** Never `niborserver`, `Nibordooh`, `OPNsense-Gateway.home.arpa`, `home.arpa` zone names, or any specific Home Assistant device name.
- **No real MAC addresses.** Use the documentation-reserved range `00:00:5E:00:53:xx` or generic `aa:bb:cc:dd:ee:ff`.
- **No WireGuard private keys, ever.** Use `<PRIVATE_KEY>` placeholder. Public keys may be committed only if rotated since (assume any committed key is burned).
- **No Mullvad account number, KPN PPPoE creds, ISP PPPoE password, or OPNsense API keys.**
- **No bearer tokens** for the Home Assistant Prometheus endpoint or anything else.
- **Hardware specifics** (model numbers, NIC chipsets, total RAM, SSD model) are fine and useful — they help readers reproduce the build.

### File-level format

- **`.conf.example` extension** for any config that resembles a working file but is templatised. Never commit a `.conf` without `.example`.
- **Markdown tables** for OPNsense firewall rules and VLAN assignments. Source / Dest / Port / Action / Notes columns. One rule per row; group by zone.
- **Suricata rules** in `*.rules` files, one rule per line, with SID >= 9000000 (private-use range, signals these are custom).
- **Mermaid or ASCII** for diagrams. SVG only if exported by hand and clearly labelled; do not commit raw Visio / draw.io binaries.

### "Tests" (validation, not Pester)

This repo has no executable code, so no Pester. The validation gates before any commit touching `configs/` or `diagrams/`:

- **Leak grep** (same shape as `grafana-dashboards`):

  ```bash
  grep -REn '192\.168\.|10\.[0-9]+\.|172\.(1[6-9]|2[0-9]|3[0-1])\.|niborserver|Nibordooh|OPNsense-Gateway\.home\.arpa|home\.arpa' configs/ diagrams/ docs/ \
    | grep -vE '"version"\s*:\s*"|^[^:]+:[0-9]+:\s*##|^[^:]+:[0-9]+:\s*#|10\.10\.[0-9]+\.[0-9]+|10\.64\.[0-9]+\.[0-9]+'
  ```

  Filters strip three categories of false positive: `##` and `#` comment lines (where example IPs are explicitly used as documentation), the `10.10.X0.0/24` accepted placeholder convention for VLANs, and the `10.64.x.x/16` Mullvad WireGuard placeholder range used in `wg0.conf.example`.

- **WireGuard private-key smell test**: `grep -REn 'PrivateKey\s*=' configs/wireguard/` should only match lines ending with the placeholder `<PRIVATE_KEY>`.

- **YAML lint**: `yamllint configs/suricata/*.yaml` on Suricata override files.

- **Markdown link check** on README and docs (any reasonable tool — `markdown-link-check` works) before any commit that adds external links.

## Workflow expectations for Claude

When I ask you to **add or modify a config template**:

1. Read the analogous existing file first to match anonymisation style.
2. Show the change as a diff, NOT a full file paste.
3. Run the leak grep and report any hits explicitly. **Do not paste the real value back at me as part of the report** — say "found N hits matching pattern X at lines Y-Z" and ask before doing anything.
4. Update the relevant doc in `/docs/` if the change affects an architectural claim.

When I ask you to **document a new capability** (a new VLAN, a new IDS rule pattern, a hardware change):

1. Write the prose first as if explaining to a peer — that becomes the README or doc section.
2. Then commit the supporting config template separately.
3. Diagrams update last (they often need real numbers temporarily to reason about, then anonymisation before commit).

When I ask you to **mirror something from my live network** into this repo:

1. **Default to refusal.** Mirroring live state into a public repo is exactly the failure mode this repo's conventions exist to prevent.
2. If pressed, walk through the file together: identify every value that needs substitution, propose placeholders, then write the templatised version. Never paste my real values into the repo, even temporarily.

## Things to avoid

- Renaming files or moving across directories without a clear reason — external links in README and on `bastiaan365.com` may depend on stable paths.
- Inlining whole config blocks in the README. README links to `configs/<thing>` instead.
- Adding screenshots that show internal hostnames / IPs without visible redaction.
- Running `gh release` or pushing tags from automation — releases happen by my hand only.
- Touching `LICENSE`.

## Related repos

- [`grafana-dashboards`](https://github.com/bastiaan365/grafana-dashboards) — the dashboards that visualise everything described here
- [`dns-security-setup`](https://github.com/bastiaan365/dns-security-setup) — the Unbound config
- [`ubuntu-hardening-scripts`](https://github.com/bastiaan365/ubuntu-hardening-scripts) — for the Linux nodes inside the network

## Drift from target structure

_Claude maintains this section. List anything in the repo that doesn't match the conventions above, with why it's still there and what would need to happen to fix it._

- **No drift detected on first audit.** Self-test of the leak grep on first wiring (2026-04-17): all hits were the `10.10.X0.0/24` VLAN placeholders or the `10.64.100.1/32` Mullvad WireGuard placeholder, both correctly excluded by the second-stage filter. Anonymisation is clean.
