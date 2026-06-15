# heroku-buildpack-ruby-duckdb

This buildpack installs the DuckDB native library (`libduckdb.so`) into a Heroku slug so the `duckdb` Ruby gem can compile and run. It is used by `stern-etl`.

Global preferences in `~/.claude/CLAUDE.md` (writing style, commit/PR conventions) apply here too.

## When to make changes

**Only when upgrading DuckDB in stern-etl.** No other work happens in this repo. If the `duckdb` gem version in stern-etl's `Gemfile` is being bumped, the native library version in this buildpack must match.

## Upgrade checklist

1. Find the target version (e.g. `1.4.3`) from the `duckdb` gem release notes or stern-etl's `Gemfile`.

2. Get the SHA256 for `libduckdb-linux-amd64.zip` at that version:
   ```bash
   curl -fsSL https://github.com/duckdb/duckdb/releases/download/v<VERSION>/libduckdb-linux-amd64.zip \
     | sha256sum
   ```
   Always verify against the checksum published on the DuckDB GitHub release page before using it.

3. Update `bin/compile`:
   - `DUCKDB_VERSION` on line 23
   - `EXPECTED_SHA256` on line 28

4. Commit, push, open a PR. The PR title must begin with the ClickUp ticket ID (from the branch name).

5. After merging, update stern-etl's buildpack reference if the URL or branch changed.

## How the buildpack works

`bin/compile` runs during `heroku build`:

- Downloads `libduckdb-linux-amd64.zip` from the DuckDB GitHub releases for `$DUCKDB_VERSION`.
- Verifies the SHA256 before extracting (build fails hard on mismatch).
- Installs `libduckdb.so` and headers into `$BUILD_DIR/vendor/duckdb/`.
- Caches the tarball in `$CACHE_DIR` so subsequent deploys skip the download.
- Copies the library to `$HOME/vendor/duckdb/` at runtime.

The stern-etl `Procfile` expects `BUNDLE_BUILD__DUCKDB` to point at this vendor path:
```
--with-duckdb-include=/app/vendor/duckdb/include --with-duckdb-lib=/app/vendor/duckdb/lib
```

## Files

- `bin/compile` — the only file that ever changes. Version and checksum live here.
- `bin/detect` — always returns true (no detection logic needed).
- `bin/release` — empty; no runtime config exported.
