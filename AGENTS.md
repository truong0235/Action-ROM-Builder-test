# AGENTS.md

## What This Is
GitHub Actions repo that builds Android custom ROMs. Fork → dispatch workflow → get ROM zip. No application code, tests, linting, or local build system.

## Workflows
- `build-arrowos.yml` — ArrowOS. Uses `actions/setup-java@v4` (Oracle Java 22), `repo sync`, `m otapackage`.
- `build-lineageos.yml` — LineageOS. Uses OrangeFox setup scripts. Patches known symlink bug via `sed` on `install_android_sdk.sh`.
- `build-pph-aosp.yml` — PHH/Treble GSI. Clones `treble_experimentations`, runs `build-rom.sh`. Outputs `.img` not `.zip`.
- `build-lineageos-gsi.yml` — LineageOS 18.1 GSI for Poco C40 (`frost`). Clones treble manifest, applies patches from `treble_experimentations` zip, runs `generate.sh` for device config, builds `systemimage` only, compresses to `.img.xz`. No owner-only check.

All: `workflow_dispatch` only, owner-only execution, 12 GB swap; ccache varies (ArrowOS/LineageOS: 50 GB, GSI: 10 GB, PHH: disabled).

## Gotchas
- **PHH syntax error** — `build-pph-aosp.yml:77` has space in `${{ }}` for `CUSTOM_BUILD`: `$ {{ github.event.inputs.CUSTOM_BUILD }}`. Will pass empty string. Fix: remove space.
- **Owner-only lock** — Each workflow checks `sender.login == owner.login`. Forks fail without editing this condition. `build-lineageos-gsi.yml` removed this check.
- **continue-on-error on build** — `m otapackage` step won't fail workflow. Upload runs even on build failure → empty/partial release.
- **ccache not in build step (ArrowOS)** — ArrowOS sets up ccache in "Prepare" step but doesn't export `USE_CCACHE` in the actual build step. LineageOS does both. PHH has ccache commented out.
- **Java inconsistency** — ArrowOS: Java 22 via `setup-java`. LineageOS/PHH: runner default via `readlink -f $(which java)`.
- **OrangeFox script patched inline** — `sed -i 's/cd -/cd ../g'` workaround. Upstream changes may break it.
- **No auto-triggers** — Nothing on push/PR. Every build is manual dispatch.
- **PHH extra `m otapackage`** — Line 82 runs `m otapackage` after `build-rom.sh` which may be redundant or conflicting.
- **Disk space** — GitHub-hosted runners have ~14 GB. Android source needs 80+ GB. `build-lineageos-gsi.yml` runs on `ubuntu-latest` but will likely fail during repo sync or build due to insufficient space.
- **MediaTek GSI compat** — Poco C40 uses MT6761 (MediaTek). Many GSIs don't boot on MTK. Build may succeed but ROM may not boot.
- **PHH directory mismatch** — `build-rom.sh` runs from `result/` but `m otapackage` (line 82) runs from workspace root. No Android source tree there — likely no-op or error.
- **GSI disk cleanup** — `build-lineageos-gsi.yml` removes Android SDK, .NET, GHC, Boost, Docker to free space. Even with cleanup, 14 GB runner may fail during `repo sync`.
- **GSI PATH_OVERRIDE_SOONG** — `build-lineageos-gsi.yml` must `export PATH_OVERRIDE_SOONG=""` before `make`. The treble device config doesn't include `vendor/lineage/config/BoardConfigSoong.mk` which sets this variable. Without it, soong fails with `unknown variable '$(PATH_OVERRIDE_SOONG)'`.

## Editing Tips
- YAML: 2-space indent throughout.
- `workflow_dispatch` inputs are strings. Quote or interpolate carefully.
- `repo` tool installed from Google storage — version not pinned.
- Release tags use `${{ github.run_id }}` — numeric, auto-increment.
- LineageOS uses `--git-lfs` in `repo init`; ArrowOS does not.
