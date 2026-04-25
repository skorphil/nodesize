# 📦 nodesize

A fast CLI tool to analyze node_modules disk usage across different package managers.

Instantly see which projects use **real disk space** vs **shared storage** (pnpm APFS clones, Yarn global cache).

## Why?

On macOS with APFS, `du` reports misleading sizes for pnpm's cloned node_modules (shows 1.1GB when actual disk usage is ~0GB). This tool shows you the **actual** disk usage so you can focus on real space hogs.

## Installation

### Homebrew

```bash
brew tap skorphil/tap
brew install nodesize
```

### Manual

```bash
curl -O https://raw.githubusercontent.com/skorphil/nodesize/main/nodesize
chmod +x nodesize
sudo mv nodesize /usr/local/bin/
```

## Usage

```bash
# Scan current directory (depth 3)
nodesize

# Scan specific directory
nodesize ~/github

# Deeper scan (5 levels)
nodesize -d 5 ~/projects

# Unlimited depth (slow, searches entire tree)
nodesize --unlimited ~/work

# Show help
nodesize -h
```

## Output Explained

```
┌──────────┬──────────┬──────────────┬────────────────────────────────────────┐
│   Size   │  Status  │   Manager    │                Project                 │
├──────────┼──────────┼──────────────┼────────────────────────────────────────┤
│ 850M     │ ⚠️ REAL  │ npm          │ old-project                            │
│ 600M     │ ⚠️ REAL  │ Yarn Classic │ legacy-app                             │
│ 1.1G     │ ✅ FREE  │ pnpm         │ modern-app                             │
└──────────┴──────────┴──────────────┴────────────────────────────────────────┘

📊 Summary: 2 real · 1 shared
```

## Status Indicators

- ✅ FREE - Uses shared storage via:
  - pnpm APFS clones/hardlinks from global store
  - Yarn Berry PnP with global cache enabled
  - Zero additional disk space (files are deduplicated)

- ⚠️ REAL - Uses actual disk space:
  - npm (always copies files)
  - Yarn Classic (copies files)
  - Yarn Berry PnP with local cache (zero-installs mode)

## Supported Package Managers

| Package Manager          | Disk Usage Strategy                  | Status   |
|--------------------------|--------------------------------------|----------|
| npm                      | Full copies per project              | ⚠️ REAL |
| pnpm                     | APFS clones or hard links from global store | ✅ FREE |
| Yarn Classic             | Full copies per project              | ⚠️ REAL |
| Yarn Berry (PnP)         | Global cache (default)               | ✅ FREE |
| Yarn Berry (PnP)         | Local cache (zero-installs)          | ⚠️ REAL |
| Yarn Berry (node-modules)| Various modes                         | Varies   |

## Important Notes

### About pnpm Detection

pnpm projects are marked FREE assuming default settings (package-import-method=auto). This means pnpm tries:

- APFS clones (macOS) - instant, copy-on-write
- Hard links (all platforms) - shared inodes
- Copy (fallback) - only if above fail

If you've manually configured:

```bash
pnpm config set package-import-method copy
```

Then pnpm will use real disk space instead of shared storage, but nodesize will still mark it as FREE.

### About APFS Clones

On macOS, pnpm uses APFS copy-on-write clones by default. These:

- ✅ Take zero additional disk space (share physical blocks)
- ✅ Are instantly created (metadata operation only)
- ✅ Become independent only when modified
- 👎 Appear as full-size files to du and Finder

This is why `du -sh node_modules` might show 1.1GB when actual disk usage is ~0GB. nodesize helps you see through this.

## How It Works

- Finds all node_modules directories (and .pnp.cjs for Yarn PnP)
- Detects package manager by inspecting project structure:
  - .pnpm/ folder → pnpm
  - .pnp.cjs + .yarnrc.yml → Yarn Berry PnP
  - yarn.lock → Yarn Classic
  - package-lock.json → npm
- Checks Yarn Berry config for enableGlobalCache setting
- Reports disk usage with clear FREE/REAL indicators

## FAQ

**Q: Why does my pnpm project show 1.1GB in Finder but nodesize says FREE?**

A: APFS clones share physical disk blocks but appear as separate files to standard tools. The 1.1GB is "apparent size", real usage is ~0GB.

**Q: How accurate is this?**

A: Very accurate for detecting npm/Yarn Classic (always real). pnpm detection assumes default settings (which is true 99% of the time).

**Q: Can I check if pnpm is actually using clones?**

A: Run `pnpm store path` to see the global store, then check `df` before/after deleting node_modules. If freed space << apparent size, it's using clones.

**Q: What about Yarn Berry hardlinks-global mode?**

A: Not currently detected. Will show as REAL even though it uses shared storage. PR welcome!

## Contributing

Contributions welcome! Please open an issue or PR.

## License
MIT
