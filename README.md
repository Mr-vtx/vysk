# Vysk — Rust-Powered Image Optimization CLI

Optimize images across your web projects without manually converting

files one by one.

Vysk scans configured directories for PNG and JPEG images, converts them

to WebP (or AVIF for images that exceed WebP's dimension limit), and

caches the results so unchanged images don't need to be processed again.

When you're ready, Vysk can rewrite image references in your source code

to use the optimized files, remove originals that appear to be unused,

and restore rewritten files from backups.

-   **Optimize:** Convert images and preserve transparency.

-   **Cache:** Skip unchanged images on subsequent runs.

-   **Rewrite:** Update supported source-code references to optimized

    files.

-   **Clean up:** Review and remove originals that appear to be unused.

-   **Undo:** Restore files changed by the last rewrite.

Vysk works with web projects using React, Next.js, and other frameworks.

It operates on image files and supported source-code references rather

than requiring a framework-specific integration.

## Get started

``` bash

npm install -D vysk

npx vysk -c

```

The published npm package includes prebuilt native binaries for

supported platforms. You don't need to install Rust to use Vysk.

**Requirements:** Node.js 18+ and a supported platform. See [Supported platforms](#supported-platforms) and [Limitations](#limitations) before

running it in your project.

``` text

public/                     public/

├── hero.png                ├── hero.png

├── logo.png        →       ├── hero.webp

└── banner.jpg              ├── logo.png

                            ├── logo.webp

                            ├── banner.jpg

                            └── banner.webp

```

The encoder is a native addon built with [napi-rs](https://napi.rs) and

distributed as a prebuilt binary inside the package.

## Supported platforms

-   Linux x64 (glibc)

-   Windows x64

-   macOS Intel

-   macOS Apple Silicon

Node.js 18+ is required.

Not yet supported: Linux arm64, Alpine/musl, and Windows arm64. See the

[Roadmap](#roadmap).

## Commands

| Command | What it does |
| --- | --- |
| `vysk -c` | Scan configured paths and convert images to WebP |
| `vysk rewrite` | Update source code to reference converted files |
| `vysk clean` | Delete originals that no longer appear to be referenced |
| `vysk undo` | Restore files changed by the last `rewrite` |
| `vysk doctor` | Check the Node.js version, native binding, project root, and cache |

A typical flow:

``` bash

npx vysk -c          # 1. Convert images; originals remain untouched

npx vysk rewrite     # 2. Update references to .webp/.avif files; asks first

npx vysk clean       # 3. Optionally remove originals that appear unused; asks first

```

If something goes wrong after step 2, run `npx vysk undo`. If something

isn't working after installation, run `npx vysk doctor` first.

## Options for `vysk -c`

| Flag | Description | Default |
| --- | --- | --- |
| `-c`, `--compress` | Scan configured paths and optimize images | n/a |
| `-q`, `--quality <n>` | Encode quality, 0–100 (WebP, or AVIF for oversized images) | `82` |
| `--lossless` | Force lossless WebP encoding | `false` |
| `--no-cache` | Ignore the cache and reprocess every image | n/a |
| `--dry-run` | Show what would happen; write no files and leave the cache untouched | n/a |
| `--json` | Print the report as JSON instead of colored text for CI and scripting | n/a |
| `-h`, `--help` | Show help | n/a |

## Config file

Vysk reads `vysk.config.js` from the project root. The file uses

CommonJS, similar to `next.config.js` and `tailwind.config.js`.

``` js

module.exports = {

  quality: 75,

  lossless: false,

  paths: ['public', 'static'], // Default: ['public']

};

```

Precedence: CLI flag > config file > built-in default. A missing or

broken config file logs a warning and falls back to defaults instead of

failing.

## Caching

Results are cached in `.vysk/cache.json` (which is gitignored). An image

is reprocessed only if its content, the `quality`/`lossless` settings,

or the installed Vysk version has changed. If a cached output file was

deleted, Vysk treats it as a cache miss and reprocesses the image.

Use `--no-cache` to bypass the cache.

## Run it on every build

You can add Vysk to your `package.json` scripts:

``` json

{

  "scripts": {

    "prebuild": "vysk -c"

  }

}

```

With npm, `npm run build` runs the `prebuild` script first. The cache

helps repeat builds process only new or changed images.

**Before using this in CI or deployment:** make sure the environment has

a compatible Vysk native binary and can write to the configured image

directories. If your build environment treats assets as read-only or

expects builds to be reproducible without modifying source assets, run

Vysk as a separate preparation step instead.

## Rewrite, clean, and undo

### `vysk rewrite`

Vysk reads `.vysk/cache.json` to find converted images, then scans

`.tsx`, `.jsx`, `.js`, `.css`, and `.scss` files for references to the

originals. Supported patterns include JSX `src="..."`, CSS `url(...)`,

and local `import`/`require` specifiers. It rewrites those references to

the `.webp` or `.avif` output.

This is a full replacement, not a `<picture>` fallback: after rewriting,

your code no longer references the original file.

**Safety measures:**

-   Refuses to run on an unclean Git working tree (or a non-Git project)

    unless you pass `--force`. Git is the recommended rollback path.

-   Prints the complete plan and asks for explicit `y`/`N` confirmation.

-   Piped or non-interactive input defaults to No.

-   Backs up every touched file to `.vysk/rewrite-backups/<timestamp>/`

    before writing.

### `vysk clean`

Vysk finds converted originals that no longer appear in scanned source

files, shows the proposed deletion list and file sizes, and asks before

deleting.

``` bash

npx vysk clean             # Show the plan and ask for confirmation

npx vysk clean --dry-run   # Show the plan and delete nothing

```

The same safety model as `rewrite` applies: a clean Git working tree or

`--force`, plus explicit confirmation.

**Important:** "Not referenced" is evidence, not proof. Vysk uses text

matching rather than a full parser, so dynamic paths and references in

unsupported file types may be missed. Always inspect the deletion plan

before confirming. See [Limitations](#limitations).

### `vysk undo`

Restores files from backups created by `rewrite`.

``` bash

npx vysk undo              # Restore the most recent backup

npx vysk undo --list       # List backups, newest first

npx vysk undo <timestamp> # Restore a specific backup

```

Vysk shows which files it will overwrite and asks for confirmation. It

refuses to run on an unclean Git working tree unless `--force` is

passed.

### `vysk doctor`

Checks whether Node.js is a supported version, the native addon loads on

the current platform, the project root can be found, and

`.vysk/cache.json` (if present) contains valid JSON.

This command is read-only and safe to run at any time.

## How it works

-   **Node.js/TypeScript** (`src/scanner.ts`, `cli.ts`, `report.ts`)

    finds the project root, walks configured paths for supported images,

    runs the optimization loop, and prints the report.

-   **Rust** (`src/lib.rs`) decodes each image and encodes it to WebP.

    Images exceeding WebP's 16,383-pixel-per-side limit are encoded to

    AVIF using the pure-Rust [`ravif`](https://docs.rs/ravif) encoder.

    The native functionality is connected to Node.js with

    [napi-rs](https://napi.rs).

Images are encoded through their RGBA representation, preserving alpha

transparency.

## Limitations

-   **Input and paths:** Vysk accepts PNG, JPG, and JPEG inputs, and

    scans only configured `paths` (default: `public/`).

-   **Text-based matching:** `rewrite` and `clean` use text matching,

    not an AST. Dynamic or computed paths---for example,

    ``src={`/images/${slug}.jpg`}``---are not detected. References in

    file types outside `.tsx`, `.jsx`, `.js`, `.css`, and `.scss` (such

    as plain `.html` files) are not scanned. As a result, `clean` may

    flag an original that is still in use. Always review its plan.

-   **No automatic fallback:** `rewrite` replaces references outright.

    It does not generate a `<picture>` fallback for browsers without

    WebP support.

-   **Platform support:** Prebuilt binaries cover the four platforms

    listed above.

## Roadmap

### Shipped

-   One-shot PNG/JPG to WebP optimization, with AVIF for oversized

    images; originals remain untouched during conversion.

-   Hash-based cache, config file, `prebuild` workflow, and `--json`

    output.

-   Source-reference tools: `rewrite`, `clean`, and `undo`.

-   `doctor` diagnostics.

### Next

-   Responsive variants and `srcset` generation.

-   Oversized-image detection.

-   Linux arm64 and Alpine/musl prebuilt binaries.

-   Bundler integration, such as a Next.js or webpack plugin, beyond the

    `prebuild` workflow.

## Development

Building Vysk from source requires Node.js 18+, a Rust toolchain

([rustup](https://rustup.rs)), and `nasm` (needed to build `rav1e`, the

AVIF encoder).

Install `nasm` with your platform's package manager, for example:

``` bash

sudo apt install nasm

brew install nasm

```

CI can use [ilammy/setup-nasm](https://github.com/ilammy/setup-nasm).

End users installing the published package do not need these build

tools.

``` bash

npm install

npm run build        # Build the native addon, then compile TypeScript

npm run typecheck    # Run tsc --noEmit

npm run test:unit    # Run TypeScript unit tests (no Rust needed)

```

Try it on a real project:

``` bash

cd /path/to/some/project

node /path/to/vysk/dist/cli.js -c

```

## Releasing

Do not publish from a local machine: one machine can build the native

binary only for its own platform. Bump `version` in `package.json`, then

push a matching tag for that new version.

For example, if you changed the package version to `0.1.2`:

``` bash

git tag v0.1.2

git push origin main v0.1.2

```

Use the actual new version number; do not reuse a version or tag that

already exists.

The tag triggers `.github/workflows/release.yml`, which builds the addon

on Linux, Windows, and macOS (Intel and Apple Silicon) runners, checks

that each binary loads, and publishes one package containing all four.

The workflow requires an `NPM_TOKEN` repository secret.

`.github/workflows/ci.yml` runs builds and tests on each OS for every

push and pull request; it does not publish.

## License

MIT. See [LICENSE](./LICENSE).
