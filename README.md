# Vysk

Optimize website images so pages load faster and use less bandwidth.
Vysk scans your project for PNG/JPG/JPEG files and converts them to WebP,
leaving your original files untouched. Then it can rewrite your code to use
the new files, remove the originals you no longer need, and undo the rewrite.

```bash
npm install -D vysk
npx vysk -c
```

```
public/                     public/
├── hero.png                ├── hero.png
├── logo.png        →       ├── hero.webp
└── banner.jpg              ├── logo.png
                            ├── logo.webp
                            ├── banner.jpg
                            └── banner.webp
```

No Rust required. The encoder is a native addon (built with
[napi-rs](https://napi.rs)) that ships inside the package as a prebuilt binary.

**Supported platforms:** Linux x64 (glibc), Windows x64, macOS Intel, macOS
Apple Silicon. Node.js 18+.
Not yet supported: Linux arm64, Alpine/musl, Windows arm64 (see [Roadmap](#roadmap)).

## Commands

| Command | What it does |
| --- | --- |
| `vysk -c` | Scan configured paths and convert images to WebP |
| `vysk rewrite` | Update source code to reference the converted files |
| `vysk clean` | Delete originals that no longer appear to be referenced |
| `vysk undo` | Restore files changed by the last `rewrite` |
| `vysk doctor` | Check Node version, native binding, project root and cache |

A typical flow:

```bash
npx vysk -c          # 1. convert (originals untouched)
npx vysk rewrite     # 2. point your code at the .webp files (asks first)
npx vysk clean       # 3. optionally remove the now-unused originals (asks first)
```

If something goes wrong after step 2, run `npx vysk undo`. If something isn't
working after install, run `npx vysk doctor` first.

### Options for `vysk -c`

| Flag | Description | Default |
| --- | --- | --- |
| `-c`, `--compress` | Scan configured paths and optimize | n/a |
| `-q`, `--quality <n>` | Encode quality, 0-100 (WebP, or AVIF for oversized images) | `82` |
| `--lossless` | Force lossless WebP encoding | `false` |
| `--no-cache` | Ignore the cache and reprocess every image | n/a |
| `--dry-run` | Show what would happen: no files written, cache untouched | n/a |
| `--json` | Print the report as JSON instead of colored text (CI/scripting) | n/a |
| `-h`, `--help` | Show help | n/a |

### Config file

`vysk.config.js` in the project root (CommonJS, same convention as
`next.config.js` / `tailwind.config.js`):

```js
module.exports = {
  quality: 75,
  lossless: false,
  paths: ['public', 'static'], // default: ['public']
};
```

Precedence: CLI flag > config file > built-in default. A missing or broken
config file logs a warning and falls back to defaults instead of failing.

### Caching

Results are cached in `.vysk/cache.json` (already gitignored). An image is only
reprocessed if one of these changed since the last run: the file's content, the
`quality`/`lossless` settings, or the installed Vysk version. If the cached
output file was deleted, it counts as a miss and is reprocessed. Use
`--no-cache` to bypass the cache.

### Run it on every build

```json
{
  "scripts": {
    "prebuild": "vysk -c"
  }
}
```

`npm run build` now runs Vysk first. Thanks to the cache, repeat builds only
process new or changed images.

## Rewrite, clean and undo

### `vysk rewrite`

Reads `.vysk/cache.json` to find every image that has been converted, scans
`.tsx` / `.jsx` / `.js` / `.css` / `.scss` for references to the originals
(JSX `src="..."`, CSS `url(...)`, local `import`/`require` specifiers), and
rewrites them to the `.webp` / `.avif` output. This is a full replace, not a
`<picture>` fallback: once rewritten, your code no longer references the
original.

Safety:
- Refuses to run on an unclean git working tree (or a non-git project) unless
  you pass `--force`. Git is your real rollback path.
- Always prints the complete plan first and asks for explicit `y`/`N`.
  Piped or non-interactive input defaults to No.
- Backs up every touched file to `.vysk/rewrite-backups/<timestamp>/` before
  writing.

### `vysk clean`

Finds originals that have a recorded conversion and no longer appear in any
scanned source file, shows the list with sizes, and asks before deleting.

```bash
npx vysk clean             # show the plan, ask to confirm
npx vysk clean --dry-run   # show the plan, delete nothing
```

Same safety model as `rewrite` (clean git tree or `--force`, explicit
confirmation). "Not referenced" is evidence, not proof: see
[Limitations](#limitations) and read the list before confirming.

### `vysk undo`

Restores files from the backups `rewrite` made.

```bash
npx vysk undo              # restore the most recent backup
npx vysk undo --list       # list backups, newest first
npx vysk undo <timestamp>  # restore a specific one
```

Shows the files it will overwrite, asks for confirmation, and refuses on an
unclean git tree unless `--force` is passed.

### `vysk doctor`

Confirms Node.js is a supported version, the native addon loads on this
platform, the project root can be found, and `.vysk/cache.json` (if present) is
valid JSON. Read-only, safe to run any time.

## How it works

- **Node/TypeScript** (`src/scanner.ts`, `cli.ts`, `report.ts`) finds the
  project root, walks the configured paths for supported images, drives the
  optimization loop and prints the report.
- **Rust** (`src/lib.rs`) decodes each image and encodes it to WebP. Images
  that exceed WebP's 16,383px-per-side limit are encoded to AVIF instead (via
  the pure-Rust [`ravif`](https://docs.rs/ravif) encoder). It's connected to
  Node with [napi-rs](https://napi.rs).

Images are always encoded through their RGBA representation, so alpha
transparency survives regardless of the source format.

## Limitations

- Only PNG, JPG and JPEG inputs, and only inside the configured `paths`
  (default: `public/`).
- `rewrite` and `clean` use text matching, not an AST. Dynamic or computed
  paths (for example `` src={`/images/${slug}.jpg`} ``) are not detected, and
  references in file types outside `.tsx`/`.jsx`/`.js`/`.css`/`.scss` (plain
  `.html`, for example) are not seen, so `clean` could flag an original that is
  still in use. Always read the printed plan.
- `rewrite` replaces references outright. There is no `<picture>` fallback
  for browsers without WebP support.
- Prebuilt binaries cover four platforms (see above).

## Roadmap

**Shipped**
- Optimize: one-shot PNG/JPG → WebP (AVIF for oversized images), originals
  untouched
- Automate: hash-based cache, config file, `prebuild` pattern, `--json` output
- Website intelligence: `rewrite`, `clean`, `undo`
- `doctor` diagnostics

**Next**
- Responsive variants and `srcset` generation
- Oversized-image detection
- Linux arm64 and Alpine/musl prebuilt binaries
- Bundler integration (a Next.js / webpack plugin) beyond the `prebuild` pattern

## Development

Requires Node.js 18+, a Rust toolchain ([rustup](https://rustup.rs)), and
`nasm` (needed to build `rav1e`, the AVIF encoder: `apt install nasm`,
`brew install nasm`, or [ilammy/setup-nasm](https://github.com/ilammy/setup-nasm)
in CI). End users of the published package need none of this.

```bash
npm install
npm run build        # builds the native addon, then compiles TypeScript
npm run typecheck    # tsc --noEmit
npm run test:unit    # TypeScript unit tests (no Rust needed)
```

Try it on a real project:

```bash
cd /path/to/some/project
node /path/to/vysk/dist/cli.js -c
```

## Releasing

Don't publish from a local machine: a single machine can only build the native
binary for its own platform. Bump `version` in `package.json`, then push a tag:

```bash
git tag v0.1.1
git push origin main v0.1.1
```

The tag triggers `.github/workflows/release.yml`, which builds the addon on real
Linux, Windows and macOS (Intel and Apple Silicon) runners, checks that each
binary loads, and publishes one package containing all four. It needs an
`NPM_TOKEN` repository secret. `.github/workflows/ci.yml` runs build and tests on
each OS for every push and PR and never publishes.

## License

MIT. See [LICENSE](./LICENSE).