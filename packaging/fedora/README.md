# Fedora RPM packaging

This directory holds the spec and support files used by
[`.github/workflows/rpm_fedora.yml`](../../.github/workflows/rpm_fedora.yml)
to build wxGTK RPMs (with AppIndicator/SNI support) for the current Fedora
release. The workflow builds against whatever this branch's `HEAD` is —
version, tarball, and locale catalogs are all derived from the checkout at
build time, not from the spec's `Version:` field (which is just a
last-committed snapshot).

This lives on `appindicator-sni-taskbar-3.2-rpm-ci`, kept separate from
`appindicator-sni-taskbar-3.2` since that branch is intended for upstream
contribution and shouldn't carry fork-specific packaging/CI concerns.

## 1. Trigger a new RPM build

**Automatic:** pushing to this branch triggers a build automatically, but
only if the push touches `packaging/fedora/**` or the workflow file itself
(see the `paths:` filter in `rpm_fedora.yml`). A push that only changes
feature code (e.g. rebasing onto upstream `appindicator-sni-taskbar-3.2`
changes) will **not** auto-trigger — use the manual method below after such
a rebase.

**Manual**, via `gh`:

```sh
gh workflow run rpm_fedora.yml -R ykne/wxWidgets --ref appindicator-sni-taskbar-3.2-rpm-ci
```

or via the web UI: Actions tab → "Build wxGTK RPM (Fedora)" → "Run workflow"
→ pick `appindicator-sni-taskbar-3.2-rpm-ci` from the branch dropdown.

To follow a run once it's started:

```sh
gh run list -R ykne/wxWidgets --branch appindicator-sni-taskbar-3.2-rpm-ci --limit 5
gh run watch <run-id> -R ykne/wxWidgets
```

`gh run watch` only shows step status, not raw shell output, and `gh run
view --log` refuses to return anything for a step that's still running
("logs will be available when it is complete") — there's no live tail
available through the CLI or API, only through the web UI's live log view.
Once a step finishes, pull its full output with:

```sh
gh run view <run-id> -R ykne/wxWidgets --log --job <job-id>
```

## 2. Copy the results out

Each run uploads one artifact, `wxGTK-rpms`, built via `rpmbuild -ba`, so it
contains both the binary RPMs (`x86_64/`, `noarch/`) **and** the source RPM
(`wxGTK-*.src.rpm`, from `~/rpmbuild/SRPMS/`) in one archive.

`gh run download` always extracts on the fly and **refuses to overwrite
files that already exist** at the destination (it errors out instead of
clobbering them), so re-downloading a run into the same directory you used
before will fail. To force an overwrite, pull the artifact as a raw zip via
the API and extract it yourself with `unzip -o`:

```sh
run_id=<run-id>
destdir=/path/to/destdir

artifact_id=$(gh api repos/ykne/wxWidgets/actions/runs/$run_id/artifacts \
  --jq '.artifacts[] | select(.name=="wxGTK-rpms") | .id')

gh api repos/ykne/wxWidgets/actions/artifacts/$artifact_id/zip > wxGTK-rpms.zip
unzip -o wxGTK-rpms.zip -d "$destdir"
```

`unzip -o` overwrites existing files at the destination without prompting,
which is what you generally want when repeatedly pulling the latest build
into the same `RPMS`/`SRPMS`-style tree.

Alternatively, via the web UI: the run page → "Artifacts" section at the
bottom → download the `wxGTK-rpms` zip (same override-on-extract caveat
applies — that's a plain zip download, not `gh run download`).

Note the container image is `fedora:latest`, which tracks whatever Docker
Hub currently publishes as latest — the `%dist` tag on the resulting RPMs
(e.g. `fc44`) reflects that, and can differ from what a local `mock` build
produces if your local mock config defaults to an older release.
