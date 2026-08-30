# m2-integrity-db

The **Magento 2 integrity database**: the SHA-256 of every file that every
`magento/*` package version actually shipped.

Each digest here represents the clean, as-shipped content of a Magento source
file. Comparing the digests against the files actually on disk tells you whether
anything under `vendor/magento/` has been modified at all — and, once you fold in
the patches an installation is meant to be carrying, which of those modifications
have an explanation and which do not.

Nothing in Magento answers that question today. `composer show` does not say.
`bin/magento` does not say. Composer records exactly one `dist.shasum` per package
*archive* and no per-file hashes at all, so the reference data has to be built
separately. That is what this repository holds. An unexplained modification to a
core file is the finding worth having: a botched deploy, an abandoned hand-edit, or
an injected backdoor.

This repository is **generated data only**. The builder that produces it lives in
[`m2-integrity-db-builder`](https://github.com/m2-integrity-db/m2-integrity-db-builder).

## What is here

| | |
|---|---|
| Package CSV files | 5,132 |
| Rows (files hashed) | 2,074,555 |
| Distinct packages | 390, all under the `magento/` vendor |
| Community releases mapped | 74 (2.4.0 through 2.4.9) |
| Enterprise releases mapped | 82 (2.4.0 through 2.4.9) |
| Size on disk | about 274 MB; largest single CSV 1.3 MB |
| Hash | SHA-256, lowercase hex |

```
db/packages/<vendor>/<package>/<version>.csv    the database
db/releases/<edition>/<release>.json            release tag => exact package versions
manifest.json                                   provenance + the SHA-256 of every CSV
state.json                                      what has been built
```

The summary below is enough to consume the data; field-by-field detail is in
[docs/schema.md](https://github.com/m2-integrity-db/m2-integrity-db-builder/blob/main/docs/schema.md).

## The CSV format

```csv
path,sha256,size
App/AbstractAction.php,09dbc040d5aff585421f4e4892059f8089b7a725620a21e8d676873c0da0c74d,11307
App/Action.php,e8c5edde8e68706ffc6b7002e4af639d5489a8bbf9a2c5caec8ca53744ba9be1,325
```

`path` is relative to the package root, which is also the path relative to
`vendor/<vendor>/<package>/`, so a row can be compared against a walk of the
installed directory without any mapping. Rows are sorted by path, so a rebuild
produces a byte-identical file and a genuine upstream change produces a small,
readable diff. Directory entries are excluded.

A handful of paths contain a comma and are quoted per RFC 4180, so parse with a
real CSV reader rather than splitting on commas.

There is deliberately **no URL column**. It would cost roughly 90 bytes per row
across more than two million rows to store derivable data; `manifest.json` carries
`source_url_templates` instead.

## Why it is keyed by package version

The database is keyed by *package version*, not by Magento release. The 74 stable
Community 2.4.x releases contain 17,461 package requirements between them, but only
about 3,400 distinct package versions, because the same package version ships in
many releases: storing each once is roughly five times smaller.

More importantly, it removes a guess. A consumer reads
`vendor/composer/installed.php` for exact versions and looks each one up directly,
never having to work out which Magento release an installation corresponds to —
which matters, because real installations drift from their metapackage. A package
pinned or upgraded on its own is normal, and a release-keyed database would
mis-compare it.

## Using the data

Fetch the CSV for an installed package version, walk the directory, compare. One
file, to see the shape of it:

```bash
curl -fsS https://raw.githubusercontent.com/m2-integrity-db/m2-integrity-db/main/db/packages/magento/module-backend/102.0.7.csv | grep -F 'etc/di.xml,'
shasum -a 256 vendor/magento/module-backend/etc/di.xml
```

A whole package, in PHP:

```php
<?php
// php verify.php magento/module-backend 102.0.7 vendor/magento/module-backend
[, $package, $version, $dir] = $argv;

$url = "https://raw.githubusercontent.com/m2-integrity-db/m2-integrity-db"
     . "/main/db/packages/{$package}/{$version}.csv";

$csv = @fopen($url, 'r');
if ($csv === false) {
    exit("no reference data for {$package} {$version}\n"); // report, never pass silently
}
fgetcsv($csv); // header

$expected = [];
while (($row = fgetcsv($csv)) !== false) {
    [$path, $sha256] = $row;
    $expected[$path] = $sha256;
    $file = "{$dir}/{$path}";
    if (!is_file($file)) {
        echo "removed   {$path}\n";
    } elseif (!hash_equals($sha256, hash_file('sha256', $file))) {
        echo "modified  {$path}\n";
    }
}
fclose($csv);

$files = new RecursiveIteratorIterator(
    new RecursiveDirectoryIterator($dir, FilesystemIterator::SKIP_DOTS)
);
foreach ($files as $file) {
    $path = substr($file->getPathname(), strlen($dir) + 1);
    if ($file->isFile() && !isset($expected[$path])) {
        echo "added     {$path}\n";
    }
}
```

Cache the CSVs locally; a full installation is several hundred packages.

### Verdicts

Five outcomes, not four:

| Verdict | Meaning |
|---|---|
| `modified — explained` | Digest mismatch, and a patch the installation carries names this path |
| **`modified — unexplained`** | **Digest mismatch with nothing accounting for it — the finding this database exists for** |
| `added` | Present in `vendor/<package>/`, absent from the CSV |
| `removed` | In the CSV, missing on disk |
| `no reference data` | The package version is not in the database — third-party, or newer than the last build. **Must be reported, never silently passed** |

The explained/unexplained split needs a list of the patches an installation is
meant to be carrying: `m2-hotfixes`, Composer patch declarations, and the patch
sets described below. Without that, everything modified is unexplained and the
signal is lost in noise.

### Known-benign noise

Account for these before treating a difference as a finding:

- `.DS_Store` and editor swap files — real `added` rows, no signal.
- Installations built with `composer install --prefer-source`. A `.git` directory
  is present, and `.gitattributes` `export-ignore` rules mean the checked-out file
  set differs from the dist archive this database was built from.
- Adobe Commerce Cloud auto-applied patches, which land without a local record.
- Patches applied through `magento/quality-patches` or
  `magento/magento-cloud-patches`, via `vendor/bin/magento-patches`.

## Provenance and trust

Every archive is verified against the digest its source published *before* it is
opened. An archive whose digest does not match is never hashed, and a package
version with no published digest is recorded as a gap rather than trusted. The six
packages that publish no digest (see below) are verified against the git tree of the
exact commit named in `dist.reference` instead — a stronger attestation, not a
weaker one.

`manifest.json` carries the SHA-256 of all 5,132 CSVs, plus `source_url_templates`,
the release list per edition and the build timestamp, so a consumer that trusts the
manifest can verify the whole database without re-fetching anything upstream.

### Verifying the database

`manifest.json` is signed with Ed25519 in minisign format, so one signature covers
every CSV it lists:

```bash
minisign -Vm manifest.json -p minisign.pub
```

Key ID `B72F78714F9831AA`. Verification needs no external binary if you would rather
not install one — `ext-sodium` is enough.

**[SECURITY.md](SECURITY.md)** has the full instructions: installing minisign,
verifying without it, what to do if you are writing a consumer, what signing does
and does not cover, and how to report a vulnerability.

## Scope and limits

- **Both editions are covered**, in this one tree: 74 Community release maps and 82
  Enterprise ones. A package version is stored once regardless of which edition ships
  it, so an Enterprise release resolves its Community packages from the same files.
- **Community release maps cover 74 of Adobe's 82 stable 2.4.x releases.** The eight
  without one — `2.4.4-p14` through `p18` and `2.4.5-p15` through `p17` — are Adobe
  extended-support releases behind Marketplace authentication, which the Mage-OS
  mirror does not carry. Their *package* data is present, because the Enterprise
  build reaches those packages through an authenticated source; only the
  Community-side release map is absent. A consumer that reads exact versions from
  `installed.php` is unaffected.
- **`magento/*` only.** Not Hyvä, not Amasty, not any other third-party vendor. The
  schema is deliberately vendor-agnostic and the same machinery would work for
  them; this database does not cover them today.
- **2.4.x only.** No 2.3.x, no `2.4-develop`.
- **Files outside `vendor/magento/` are out of scope**, including the root files
  `magento2-base` copies into place on installation.

## Where the data comes from

Community packages come from the [Mage-OS mirror](https://mirror.mage-os.org)
(Composer v2, no authentication), which serves the same dist archives Composer
installs. Hashing the artefact Composer actually installs is correct by
construction: no path mapping and no guesswork.

Packagist is a fallback for six `magento/*` packages the mirror does not carry:
`zendframework1`, `zend-cache`, `zend-db`, `zend-pdf`, `magento-zf-db` and
`magento-zf-mvc`. All six are genuine `vendor/magento/` directories on a real
installation, so without the fallback every install would report `no reference
data` for real directories. They are also the six that publish no digest, and are
verified against their git tree instead. Enterprise packages come from `repo.magento.com`, which needs Marketplace
credentials, and which also serves the extended-support Community releases the
mirror omits.

## Builds and bundles

The database is built from a scheduled nightly job. A build resolves the
metapackage metadata, compares it against `state.json`, and hashes only package
versions it has not seen, so most nights add nothing at all. The initial contents
were seeded from a local run, because a cold build of several thousand archives does
not fit inside a CI job.

Flat per-release bundles are a generated join of everything a release ships — one
row per file, paths relative to `vendor/` — for consumers that would rather not
reassemble several hundred per-package CSVs. They ship as **GitHub Release assets
rather than git blobs**, so they do not count toward repository size and clones stay
small.

Assets are named by edition and release: `ce-2.4.7.csv.gz`, `ee-2.4.7.csv.gz`. The
prefix is not decoration. Both editions publish a bundle for the same release
numbers, and without it the two collide — a Community bundle can be replaced by the
Enterprise one of the same name, which is not an obvious failure but a confident
wrong answer: a Community installation checked against an Enterprise bundle reports
every Enterprise package as a missing file.

**The per-package CSVs under `db/packages/` are the authoritative form**, and they
are what the signed manifest covers. Bundles are a convenience derived from them; if
you are unsure, use the per-package files.

## What this is not

This repository contains file paths, SHA-256 digests and byte sizes. It contains no
Magento or Adobe source code, and none can be reconstructed from it: SHA-256 is a
one-way function. Archives are downloaded during a build, streamed through the
hasher, and discarded. Magento and Adobe Commerce are trademarks of Adobe Inc.;
this project is not affiliated with or endorsed by Adobe.

## Licence

[BSD 3-Clause](LICENSE.md).

## Security

Found something? Email scott@dor.ky rather than opening a public issue. See
[SECURITY.md](SECURITY.md).
