# ghgetbin

A lightweight tool for managing single-binary GitHub releases with multi-version support.

## What it does

`ghgetbin` downloads and manages binaries from GitHub releases, allowing you to:
- Install specific or latest versions of tools
- Keep multiple versions installed side-by-side
- Switch between installed versions instantly
- Automatically select the right binary for your system

## ⚠️ Disclaimer

This project was vibe coded with AI assistance. While functional, it may contain bugs or edge cases. Please review the code and use at your own risk. Consider testing in a safe environment before relying on it for critical workflows.

## Requirements

- Python 3.11+
- No third-party dependencies (stdlib only)

## Installation

1. Copy `ghgetbin` to `~/bin/` (or anywhere in your `$PATH`)
2. Make it executable: `chmod +x ~/bin/ghgetbin`
3. Create config directory: `mkdir -p ~/.config`

## Quick start

Add a package to track:
```bash
ghgetbin add fzf junegunn/fzf
ghgetbin add gh cli/cli
```

Install the latest version:
```bash
ghgetbin install fzf
ghgetbin install all          # install all configured packages
```

List installed versions:
```bash
ghgetbin list
```

Install a specific version:
```bash
ghgetbin install fzf v0.54.0
```

Switch to a different installed version:
```bash
ghgetbin use fzf v0.54.0
```

## Configuration

Configuration lives in `~/.config/ghgetbin.toml`:

```toml
[settings]
bin_dir = "~/bin"           # where binaries are installed

[fzf]
repo = "junegunn/fzf"

[gh]
repo = "cli/cli"
binary = "gh"               # optional: override binary name

[bat]
repo = "sharkdp/bat"
asset_pattern = "bat-*-x86_64-unknown-linux-musl.tar.gz"  # optional: force specific asset
```

## How it works

- Installed binaries are named `{binary}-{tag}` (e.g., `fzf-v0.55.0`)
- Active version is a symlink: `fzf` → `fzf-v0.55.0`
- Multiple versions can coexist; switch between them with `ghgetbin use`

## GitHub API rate limits

Set `GITHUB_TOKEN` environment variable for authenticated requests (higher rate limits):
```bash
export GITHUB_TOKEN=ghp_your_token_here
```

## Global flags

- `-v` / `--verbose` — show API calls and asset selection reasoning
- `-n` / `--dry-run` — show what would happen without making changes

## Examples

```bash
# Add and install a new tool (binary name differs from package name)
ghgetbin add ripgrep BurntSushi/ripgrep --binary rg
ghgetbin install ripgrep

# Install a specific older version
ghgetbin install ripgrep 14.0.0

# Switch back to latest
ghgetbin install ripgrep

# Remove package from config (doesn't delete installed binaries)
ghgetbin remove ripgrep
```
