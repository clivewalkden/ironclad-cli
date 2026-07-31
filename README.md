# Ironclad

**Find Magento 2 extension conflicts before they break a store — without installing anything.**

Two extensions can quietly claim the same class, stack plugins on the same method in an undefined
order, or remove a layout block the other one needs. Magento does not warn you. The last module to
load simply wins, and behaviour can change on the next deploy.

Ironclad reads `di.xml`, `module.xml` and layout XML straight off disk and tells you first. It never
boots Magento, needs no database, and does not care whether the code is installed — so you can point
it at a client's codebase, a `vendor/` directory, or an extension you are still deciding whether to
buy.

```
$ ironclad scan /srv/magento --third-party
Scanned /srv/magento
  429 modules, 1618 preferences, 1364 plugins, 4102 layout ops
  114 finding(s) hidden: every involved module belongs to Magento

  [1] HIGH — Magento\Sales\Model\Order\Pdf\Invoice (adminhtml)
      2 modules replace this class with different implementations (Amasty_Orderattr ->
      Amasty\Orderattr\Model\Order\Pdf\Invoice; Oscprofessionals_Vatexempt ->
      Oscprofessionals\Vatexempt\Model\Order\Pdf\Invoice), and none declares a <sequence> on the
      others. Oscprofessionals_Vatexempt happens to load last, so
      Oscprofessionals\Vatexempt\Model\Order\Pdf\Invoice wins and the rest are silently discarded
      — but nothing pins that outcome.
```

That is a real finding from a production store: one of those two extensions' changes never appears on
an invoice, and nobody knew.

## Install

Download a binary from the [latest release](https://github.com/clivewalkden/ironclad-cli/releases/latest).
One file. No PHP, no Composer, no runtime.

| Platform | Asset |
|---|---|
| Linux x86-64 | `ironclad-v<version>-linux-amd64.tar.gz` |
| macOS Apple Silicon | `ironclad-v<version>-macos-arm64.tar.gz` |
| Windows x86-64 | `ironclad-v<version>-windows-amd64.zip` |

**Linux / macOS**

```sh
tar -xzf ironclad-v<version>-linux-amd64.tar.gz
sudo install ironclad-v<version>-linux-amd64/ironclad /usr/local/bin/
ironclad --version
```

The Linux build is statically linked against musl, so it runs on any distribution regardless of glibc
version — including inside slim containers.

**macOS note:** the binary is not notarised. If Gatekeeper blocks it, clear the quarantine flag:

```sh
xattr -d com.apple.quarantine /usr/local/bin/ironclad
```

**Windows:** extract the zip and put `ironclad.exe` somewhere on your `PATH`.

Every release ships a `SHA256SUMS` file:

```sh
sha256sum -c SHA256SUMS --ignore-missing
```

## Staying up to date

Ironclad can replace itself:

```sh
ironclad check-update   # report the latest published version, change nothing
ironclad update         # download and install it (add -y to skip the prompt)
```

`update` needs write permission to wherever the binary lives — use `sudo` if you installed it to
`/usr/local/bin`. It replaces the running binary in place and verifies the download before doing so.

## Usage

```sh
ironclad scan <path>                      # human-readable table
ironclad scan <path> --json               # machine-readable, for CI or jq
ironclad scan <path> --third-party        # only findings involving a non-Magento module
ironclad scan <path> --strict             # fail on any conflict, not just high-risk
ironclad scan <path> --quiet              # hide informational (Low) findings
ironclad scan <path> --ignore-vendor Foo  # hide findings entirely within one vendor (repeatable)
ironclad scan <path> --html report.html    # also write a self-contained HTML report
```

`<path>` can be a Magento root, a bare `vendor/` directory, an unzipped extension, or a `.zip`
archive — Ironclad extracts the XML it needs to a temp directory, scans, and cleans up:

```sh
ironclad scan ~/Downloads/Vendor_Extension-1.4.2.zip
```

Findings point at paths inside the archive, so they stay meaningful after the temp copy is gone.

**Start with `--third-party`.** Magento's own modules replace each other's defaults constantly, and
that is by design — those findings dominate an unfiltered report and no one is going to patch core.
On a typical install this cuts the report roughly in half and leaves the things somebody can act on.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | No high-risk conflicts (or, with `--strict`, none at all) |
| `1` | Conflicts found that should fail a build |
| `2` | Usage error, or the path could not be read |

Medium and Low findings do not fail a build by default, so dropping Ironclad into CI does not
immediately go red on things needing human judgement. Anything hidden by a filter also stops counting
toward the exit code — and every filter says how many findings it dropped, so a shrunken report never
passes itself off as a clean one.

## What it finds

| Risk | Finding |
|---|---|
| **High** | Two modules replace the same class with different implementations and nothing pins which wins |
| **High** | One module removes a layout element another module defines |
| **Medium** | Plugins share a `sortOrder` and a method, so their order can shift between deploys |
| **High** | An `around` plugin never calls through, so plugins after it on that method never run |
| **Medium** | Two modules declare one layout block with a different class or template |
| **Medium** | Two modules move the same layout element to different destinations |
| **High** | Two modules define the same virtual type on a different base class, unpinned |
| **High** | Module `<sequence>` declarations form a loop, so Magento will not boot |
| **Low** | The same, but `<sequence>` pins the outcome — Magento's intended override mechanism |

### Grade and score

Every report carries a headline grade from **A** to **F**, plus a weighted score (High counts 10,
Medium 3, Low 1) and that score per 100 modules so installs of different sizes can be compared.

The grade is driven by high-risk findings, because those are the ones where nothing pins the outcome;
Medium only moves the needle once it piles up. There is deliberately no "score out of 100" — there is
no meaningful total to divide by, and inventing one would imply precision that is not there.

Filters and the grade stay in step: run with `--third-party` and the grade describes only the findings
you can act on.
**Read the grade with `--third-party`.** Unfiltered, it is dominated by unpinned collisions inside
Magento core and MSI, which nobody is going to fix — across ten production installs the unfiltered
grade clustered at E/F and told us almost nothing. The same ten with `--third-party` graded C to E and
ranked in an order that matched how messy those installs actually are. The unfiltered number is honest
about absolute risk; the filtered one is the one you can act on.

The HTML report is one self-contained file with inline CSS, no external requests and no JavaScript, so
it renders from an email attachment and can go straight to a client.

### Plugins are compared per method, not per class

Magento declares plugins against a *class*, but two plugins on one class only interfere if they
intercept the same *method*. Ironclad reads each plugin class's PHP to find out which, so plugins
that provably touch different methods are not reported at all. On ten production installs this cut
the medium-risk findings by around a third — 109 to 66 on the worst install — because those were the
ones that used to say "verify method overlap manually". Correcting the default `sortOrder` in v0.9.1
removed a further handful of false ties per install, taking that worst case to 62.

It also makes the most damaging plugin mistake visible: an `around` plugin that never calls the
callable it is given replaces the method outright, and every plugin with a higher `sortOrder` on that
method silently never runs. On one store that was an extension's `around` on `getPagerUrl()` blocking
another vendor's SEO pager rewriting entirely.

Reading the PHP is lexical, not a full parse — Magento names interceptor methods after their target,
so `afterGetName` is all the information needed. A plugin that inherits its methods from a parent
class reports none, and those findings fall back to class-level wording.

### Why `<sequence>` matters so much

Severity hinges on it. When a module overrides another's default, it declares a `<sequence>` on the
module it is overriding, which pins the load order and makes the result deliberate. `Magento_Review`
overrides `Magento_Catalog`'s review renderer exactly this way.

A dangerous conflict looks different: two unrelated extensions claim the same thing, neither knows
about the other, and the winner is decided by incidental load order. Ironclad checks for an explicit
or transitive `<sequence>` between the modules involved and ranks accordingly. Without that
distinction a stock install reports around twenty "critical" conflicts that are all core working as
intended.

## Adopting it on an existing codebase

A real install has dozens of findings on day one, none of them caused by the change you are
reviewing. Gating CI on that means a permanently red build, and a permanently red build gets
ignored. So record what is already there and gate on what comes next:

```bash
ironclad scan . --third-party --write-baseline .ironclad-baseline.json
```

Commit that file. From then on:

```bash
ironclad scan . --third-party --baseline .ironclad-baseline.json
```

Accepted findings are not shown and cannot fail the run; anything new is reported normally. The run
still says how many were carried over, so a green build never pretends the codebase is clean.

A finding is matched on its kind, area, target and the modules involved — deliberately not on its
wording, its source file paths, or its risk level, so rewording a finding between releases, moving a
vendor's files, or tightening a severity does not resurrect something you already accepted.

Entries carry an occurrence count, because one class can legitimately produce two findings of the
same shape (a plugin tie at two different `sortOrder`s, for instance). Accepting two does not accept
a third.

When entries stop matching anything, the run says so — those findings look fixed, and regenerating
the baseline keeps it honest.

## Use it in CI

Ironclad ships a composite GitHub Action. Add this to a Magento repo:

```yaml
name: Conflict scan
on: [pull_request]

permissions:
  contents: read
  pull-requests: write   # only needed for the PR comment

jobs:
  ironclad:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      # Ironclad reads vendor/, so it must exist.
      - run: composer install --no-progress --no-interaction
      - uses: clivewalkden/ironclad-cli@v0.5.0
        with:
          path: .
          third-party: true
```

It downloads the pinned binary, verifies its checksum, scans, writes a job summary table, and posts
(or updates) one pull request comment with the high-risk findings.

| Input | Default | |
|---|---|---|
| `path` | `.` | Magento root, `vendor/` directory, or a `.zip` |
| `version` | `latest` | Pin a version for reproducible findings |
| `third-party` | `false` | Only findings involving a non-Magento module |
| `ignore-vendor` | — | Space-separated vendors to hide internal findings for |
| `strict` | `false` | Fail on any conflict, not just high-risk |
| `quiet` | `false` | Hide informational findings |
| `fail-on-conflicts` | `true` | Set `false` to report without blocking a merge |
| `comment` | `true` | Post a PR comment |
| `html-report` | — | Also write a self-contained HTML report here |

Outputs `grade`, `high`, `medium`, `low` and `json` (a path to the full findings) for later steps.

**When adopting on an existing store, start with `fail-on-conflicts: false`.** A first scan of a mature
install typically finds a few high-risk items and dozens of mediums; blocking merges on day one just
gets the check disabled. Read the report, fix or accept, then turn enforcement on.

There is no diff-scoped mode, and that is deliberate: a conflict is a relationship *between* two
modules, so knowing whether a changed module now collides with something needs the whole tree. A full
scan takes a few seconds, so scoping to a diff would add complexity and miss findings.

## Vetting an extension before you install it

Ironclad only reads XML, so you can test a candidate against a real codebase without touching it:
copy the tree, drop the extension in, scan both, and diff the findings. That is how we caught a module
that shared a `sortOrder` with `Magento_Persistent` on the object driving full-page-cache variation.

## Known limitations

Worth reading before you trust a clean report:

- **Plugins that inherit interceptor methods fall back to class-level comparison.** Method names are
  read from the plugin's own PHP, so a plugin whose `afterGetName` lives in a parent class reports no
  methods and its findings say the overlap is unverified rather than guessing.
- **No cross-area precedence.** `global` and each specific area are analysed separately, so a genuine
  global-versus-frontend conflict is missed.
- **Every module on disk is assumed enabled.** `app/etc/config.php` is not read, so a disabled module
  is still analysed.
- **Layout coverage is module-level.** Theme overrides under `app/design/**` are not merged, and
  handles are compared in isolation.
- **Virtual types are not resolved transitively.** A name collision is reported, but a preference or
  plugin reaching a class through a chain of virtual types is not followed to its concrete target.
- **Nothing is inferred from generated code.** `var/` and `generated/` are skipped by design — you
  want warnings *before* `setup:di:compile`, not after.

## Privacy

Ironclad reads only the path you give it. It makes no network calls and collects no telemetry, with
one exception: `update` and `check-update` contact the GitHub releases API, and only when you run
them.

## Licence

MIT.
