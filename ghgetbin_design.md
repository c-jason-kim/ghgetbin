Create a Python script called `ghgetbin` that manages single-binary GitHub release downloads
with multi-version support.

## Requirements
- Python 3.11+ only
- No third-party dependencies — stdlib only
- Single script file at ~/bin/ghgetbin, chmod +x, with proper shebang

## Note on TOML writing
tomllib is read-only. Implement a simple write_toml() helper that serializes the known
shallow data structures — no third-party lib needed.

write_toml() rules:
- Only support the data shapes actually used: a dict of tables, each containing
  string-valued keys. No nested tables, no arrays, no integers, no booleans.
- Always quote all string values with double quotes.
- Escape the following characters inside string values: backslash (\\), double quote (\"),
  newline (\n), carriage return (\r), tab (\t), backspace (\b), and form feed (\f).
- Validate package names and key names against bare-key rules (alphanumeric, dash,
  underscore only). Refuse to write keys that require quoting rather than trying to
  quote them.
- Validate repo format as "owner/repo" with non-empty owner and repo parts.
- Write tables in sorted order for deterministic output.

## File locations
All paths are configurable via [settings] in ghgetbin.toml. Defaults:
- Config: ~/.config/ghgetbin.toml — override with GHBIN_CONFIG env var
- Binaries: ~/bin/

Expand ~ in all paths using pathlib.Path.expanduser().

## No separate state file
All state is derived from the filesystem at runtime:
- Installed versions: glob {bin_dir}/{name}-* and check they are regular files + executable
- Active version: read symlink target of {bin_dir}/{name}, parse tag from filename
- If symlink is missing or broken, no active version

## Config format (ghgetbin.toml)
The reserved [settings] table controls global behaviour. All settings are optional and
fall back to defaults. Everything else is treated as a package entry.

[settings]
bin_dir = "~/bin"               # where versioned binaries and symlinks are placed

[gh]
repo = "cli/cli"
binary = "gh"           # optional: override installed binary name, defaults to section key

[fzf]
repo = "junegunn/fzf"

[bat]
repo = "sharkdp/bat"
asset_pattern = "bat-*-x86_64-unknown-linux-musl.tar.gz"  # optional: override asset selection

When loading config, parse [settings] first, then treat all other top-level tables as
package entries. The binary name used for filenames and symlinks is: binary field if set,
otherwise the section key (e.g. "gh", "fzf", "bat").

## Config loading
Resolve config path in this order:
1. GHBIN_CONFIG env var if set
2. ~/.config/ghgetbin.toml

Expand ~ in the resolved path. If the file does not exist, warn the user and continue
with an empty in-memory config. The user is responsible for creating the config file
and its parent directory with proper permissions.

## CLI interface
Use argparse with custom subparser organization. All commands reference packages by
their short name from config.

Config actions:
  ghgetbin add <name> <repo>       # add a new package entry to config
                                #   optional flags: --binary, --asset-pattern
  ghgetbin remove <name>           # remove the package entry from config only;
                                #   does NOT delete binaries or symlinks from disk

Inspection action:
  ghgetbin list                    # table: name | repo | binary | active version | other installed versions

State actions:
  ghgetbin install <name> [tag]    # install latest for one package, or specific tag if given
                                #   use "all" as name to install latest for all packages
  ghgetbin use <name> <tag>        # switch active symlink to an already-installed version

Global flags:
  -v / --verbose                # show API calls, asset selection reasoning, file ops
  -n / --dry-run                # print what would happen, no writes

## GitHub API
- Base URL: https://api.github.com/repos/{owner}/{repo}/releases/latest
- For specific tags: https://api.github.com/repos/{owner}/{repo}/releases/tags/{tag}
- Use urllib.request — set User-Agent: ghgetbin/1.0
- If GITHUB_TOKEN env var is set, add Authorization: Bearer {token} header
- Parse response as JSON
- Rate limit detection: on 403/429, print a clear error message including the
  X-RateLimit-Reset time (if present) formatted as a human-readable timestamp.
  Do not retry automatically; the user should re-run the command after the reset.

## Asset selection (in priority order)
1. If asset_pattern defined, match with fnmatch against asset names — take first match
2. Detect current arch via platform.machine(); map aliases:
   x86_64 / amd64, aarch64 / arm64
3. Filter assets containing "linux" and current arch (try both alias forms)
4. Score remaining candidates:
   - Prefer musl over gnu (+2)
   - Prefer .tar.gz (+3), .tar.xz (+2), .zip (+1), raw binary (0)
   - Deprioritize sha256, .sig, .sbom, checksums (-10)
5. Take highest scoring asset; warn with selection reason if ambiguous

## Download and extraction
- Download asset to tempfile.NamedTemporaryFile (deleted automatically on close)
- For .tar.gz / .tar.xz: use tarfile.extractfile(member) to stream the target member
- For .zip: use zipfile.open(member) to stream the target member; extract executable bit
  from external_attr (bits 16-18) to properly detect executable members
- For raw binary: stream download response directly to destination
- In all cases, stream content directly into the destination file handle (see Installing)
- Identify target binary member by matching the filename component of each member
  path (i.e. member.name.split('/')[-1]), not the full archive path:
  1. Exact match on binary field or package key
  2. Any member with executable bit set, excluding dirs and scripts (prefer no extension)
- If multiple candidates found, list them, pick best match, warn user

## Installing a version
- Destination path: {bin_dir}/{binary}-{tag}  e.g. ~/bin/fzf-v0.55.0
- Error if bin_dir does not exist — user is responsible for creating it
- Open destination with open(dest, 'xb') — this is atomic (O_CREAT|O_EXCL); if
  FileExistsError is raised, skip download entirely and just update symlink if needed
- Stream extracted content into the open file handle
- chmod +x after write is complete
- Create/update symlink {bin_dir}/{binary} -> {binary}-{tag} (relative target, not
  absolute) using os.symlink to a temp name + os.replace for atomic symlink swap

## Cleanup on failure
If an error occurs or SIGINT is received after the destination file has been opened
but before installation is complete:
- Delete the partially-written destination file
- Delete the NamedTemporaryFile (if still open)
- On SIGINT, exit 130 after cleanup

## Error handling
- Custom exception classes: GhbinError, AssetNotFoundError, ApiError
- All errors print [ERROR] to stderr, exit non-zero
- Warnings print [WARN] to stderr, continue
- Network errors include the URL that failed
- Missing config on first run: warn and use empty in-memory config
- Missing bin_dir or config parent directory: error with clear message asking user to create it
- Broken symlinks: warn and report, don't crash
- Unknown package name in any command: suggest running `ghgetbin list` or `ghgetbin add`

## Code structure
Flattened architecture with minimal classes:

Classes:
- Config: holds path, bin_dir, packages dict; provides load() and save() methods
  - Callers mutate packages dict directly and call save() explicitly
  - No auto-save methods like add_package/remove_package
- PackageEntry dataclass: name, repo, binary, asset_pattern
- InstalledPackage dataclass: name, binary, active_version, all_versions (derived from fs)

Module-level functions:
- TOML serialization: _write_toml(), _escape_toml_string(), _validate_bare_key()
- GitHub API: fetch_release(repo, tag) → release JSON dict
- Asset selection: select_asset(assets, entry) → (asset, reason)
- Binary extraction: extract_binary(archive_path, archive_type, binary_name) → bytes
  - Helper: _find_binary_in_archive() for member selection
- Installation: install_binary(bin_dir, binary, tag, content) — writes versioned file
- Symlink management: update_symlink(bin_dir, binary, tag) — atomic swap
- Filesystem queries: list_installed(bin_dir, binary) → InstalledPackage
- Commands: cmd_install, cmd_remove, cmd_list, cmd_use, cmd_add

## Code quality
- Type hints throughout
- Docstrings on all public functions/classes
- Handle SIGINT gracefully (see Cleanup on failure)
- Compatible with ruff linting
