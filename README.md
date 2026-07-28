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
`/usr/local/bin`.

## Usage

```sh
ironclad scan <path>                      # human-readable table
ironclad scan <path> --json               # machine-readable, for CI or jq
ironclad scan <path> --third-party        # only findings involving a non-Magento module
ironclad scan <path> --strict             # fail on any conflict, not just high-risk
ironclad scan <path> --quiet              # hide informational (Low) findings
ironclad scan <path> --ignore-vendor Foo  # hide findings entirely within one vendor (repeatable)
```

`<path>` can be a Magento root, a bare `vendor/` directory, or a single unzipped extension.

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
| **Medium** | Plugins on one class share a `sortOrder`, so their order can shift between deploys |
| **Medium** | Two modules declare one layout block with a different class or template |
| **Medium** | Two modules move the same layout element to different destinations |
| **Low** | The same, but `<sequence>` pins the outcome — Magento's intended override mechanism |

### Why `<sequence>` matters so much

Severity hinges on it. When a module overrides another's default, it declares a `<sequence>` on the
module it is overriding, which pins the load order and makes the result deliberate. `Magento_Review`
overrides `Magento_Catalog`'s review renderer exactly this way.

A dangerous conflict looks different: two unrelated extensions claim the same thing, neither knows
about the other, and the winner is decided by incidental load order. Ironclad checks for an explicit
or transitive `<sequence>` between the modules involved and ranks accordingly. Without that
distinction a stock install reports around twenty "critical" conflicts that are all core working as
intended.

## Vetting an extension before you install it

Ironclad only reads XML, so you can test a candidate against a real codebase without touching it:
copy the tree, drop the extension in, scan both, and diff the findings. That is how we caught a module
that shared a `sortOrder` with `Magento_Persistent` on the object driving full-page-cache variation.

## Known limitations

Worth reading before you trust a clean report:

- **Plugins are matched per class, not per method.** Two plugins on one class may never touch the same
  method. Findings say to verify manually rather than asserting breakage — method-level analysis needs
  PHP parsing.
- **`around` plugin behaviour is not analysed.** The worst real-world case — an `around` plugin that
  never calls `$proceed()` and kills every plugin after it — cannot be seen in XML.
- **No cross-area precedence.** `global` and each specific area are analysed separately, so a genuine
  global-versus-frontend conflict is missed.
- **Every module on disk is assumed enabled.** `app/etc/config.php` is not read, so a disabled module
  is still analysed.
- **Layout coverage is module-level.** Theme overrides under `app/design/**` are not merged, and
  handles are compared in isolation.
- **Nothing is inferred from generated code.** `var/` and `generated/` are skipped by design — you
  want warnings *before* `setup:di:compile`, not after.

## Privacy

Ironclad reads only the path you give it. It makes no network calls and collects no telemetry, with
one exception: `update` and `check-update` contact the GitHub releases API, and only when you run
them.

## Licence

MIT.
