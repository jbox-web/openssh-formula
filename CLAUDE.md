# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Context

Hard fork of `saltstack-formulas/openssh-formula`, owned by `jbox-web`. The
parent directory's `CLAUDE.md` (`../CLAUDE.md`) describes the conventions shared
by all the forks (CI, Kitchen platforms, lint, commit rules); this file only
covers what is specific to openssh, and where it departs from the generic
layout described there.

## Commands

```bash
bundle install
bin/kitchen list                                # 3 instances: default-<platform>
bin/kitchen converge default-debian-13-3008-2
bin/kitchen verify   default-debian-13-3008-2
bin/kitchen test     default-debian-13-3008-2   # full cycle for one instance
pre-commit run --all-files                      # all linters
pre-commit run salt-lint --all-files            # a single hook
```

There is one suite, `default`; it applies `openssh._mapdata`, `openssh.config`
and `openssh.known_hosts`. `kitchen.vagrant.yml` is an upstream leftover (ERB,
excluded from yamllint) and is not used by the CI.

## Architecture — departures from the generic template

### `map.jinja` is the v5 "mapstack" variant

There are no `defaults.yaml` / `os*map.yaml` files at the state root. Parameters
live under `openssh/parameters/` and are layered by `libmapstack.jinja`:

```
parameters/defaults.yaml
  <- parameters/osarch/<osarch>.yaml
    <- parameters/os_family/<os_family>.yaml
      <- parameters/os/<os>.yaml
        <- parameters/osfinger/<osfinger>.yaml
          <- config.get openssh:lookup, openssh
             sshd_config:lookup, sshd_config
             ssh_config:lookup, ssh_config
            <- parameters/id/<minion id>.yaml
```

The source list itself is data: `parameters/map_jinja.yaml` (a `mapstack`
input, overridable per minion like any other layer). Missing files are simply
skipped; only `os_family/*.yaml` and `osfinger/CentOS-6.yaml` exist.
`C:SUB@<key>` merges the pillar/config key under `mapdata.<key>`, so `mapdata`
has **three** roots — `openssh`, `sshd_config`, `ssh_config` — matching the three
top-level pillar keys. Merge behaviour is tunable through the
`openssh:strategy` and `openssh:merge_lists` config keys. `map.jinja` also
includes `post-map.jinja` if present (none today). Full reference:
`docs/map.jinja.rst`.

The `_mapdata` dump therefore contains a `map_jinja: sources:` block in
addition to the merged values; the reference files under
`test/integration/default/files/_mapdata/` must reflect it.

### Flat states

States are single files under `openssh/` (no `package/` `config/` `service/`
tree). `init.sls` declares the `openssh` ID with both `pkg.installed` and
`service.running` (or `service.dead` when `sshd_enable` is not `true`); every
other state `include`s it and hooks onto `service: {{ openssh.service }}`.

- `config.sls` — renders `sshd_config` (validated with `sshd -t -f` via
  `check_cmd`) and `ssh_config`, each only when its mapdata root is non-empty;
  then manages host keys per algorithm in `openssh:host_key_algos`, driven by
  the `provide_<algo>_keys` / `generate_<algo>_keys` / `absent_<algo>_keys`
  flags (checked in that order of precedence).
- `config_ini.sls` — alternative to `config.sls` that edits the existing file
  key by key with `ini.options_present` instead of templating it.
- `known_hosts.sls` — builds `/etc/ssh/ssh_known_hosts` from the Salt mine
  (`mine_functions` must be set, see the test pillar); `gather_host_keys.sls`
  is a master-side state for salt-ssh rosters, paired with
  `_pillar/known_hosts_salt_ssh.sls`.
- `auth.sls` / `auth_map.sls`, `banner.sls`, `client.sls`, `moduli.sls` —
  independent leaves; see `docs/README.rst` for their pillar keys.

`*_src` parameters (`sshd_config_src`, `ssh_config_src`, `banner_src`) are
passed through `files_switch()` unless they contain `://`, in which case they
are used as a literal source (backward compatibility).

### The `sshd_config` template

`files/default/sshd_config` renders known keywords in a fixed order through the
`option()` / `option_string_or_list()` macros, each appending to
`processed_options`. Any keyword not yet processed is then emitted by a
catch-all loop after them, and `matches` always comes last as `Match` blocks.
Because sshd keeps the **first** value it reads for a keyword, position in the
output matters: a keyword that must win over `Include`d snippets (or that
`Include`s them) has to be rendered explicitly near the top, not left to the
catch-all loop. Booleans render as `yes`/`no`; lists render as repeated lines
unless the keyword is collapsed with a separator.

## Tests

- `test/salt/pillar/default.sls` is rendered through Jinja: the
  `# {%- if … %}` lines are live Jinja hidden from yamllint behind `#`.
  Its `sshd_config` is deliberately minimal — a setting that blocks the
  `kitchen` user's SSH login makes `kitchen verify` fail with a transport
  error, not a test failure.
- The host-key fixtures in that pillar are `NOT_DEFINED` placeholders, and all
  `provide_*` / `generate_*` flags are `false`.
- `_mapdata/` holds reference dumps for many upstream platforms; only
  `debian-11.yaml` and `debian-13.yaml` are exercised by `kitchen.yml`, and
  they are the ones to update when a default or the test pillar changes.
