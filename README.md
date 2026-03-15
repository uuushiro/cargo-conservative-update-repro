# cargo-conservative-update-repro

Minimal reproduction for `cargo update -p` conservative update behavior that caused Dependabot `unknown_error: null`.

## The Problem

`cargo update -p actix-web` stops at version 4.10.2 instead of reaching 4.13.0.

```console
$ cargo update -p actix-web --verbose
    Updating actix-web v4.9.0 -> v4.10.2 (available: v4.13.0)
   Unchanged once_cell v1.20.2 (available: v1.21.4)
```

## Root Cause

`once_cell` is a shared transitive dependency of both `actix-web` and `sentry`. When `cargo update -p actix-web` runs, Cargo's conservative update strategy does not update `once_cell` because it is also depended on by `sentry` (which is not being updated).

- actix-web **4.10.2** requires `once_cell ^1.5` — satisfied by locked 1.20.2
- actix-web **4.11.0** requires `once_cell ^1.21` — **not** satisfied by locked 1.20.2

Since `once_cell` cannot be updated without also updating `sentry`'s dependency graph, `cargo update -p actix-web` stops at 4.10.2.

## Reproduction

```console
$ git clone https://github.com/uuushiro/cargo-conservative-update-repro
$ cd cargo-conservative-update-repro
$ cargo update -p actix-web
    Updating actix-web v4.9.0 -> v4.10.2 (available: v4.13.0)
```

### Workaround

Update the transitive dependency together:

```console
$ cargo update -p actix-web -p once_cell
    Updating actix-web v4.9.0 -> v4.13.0
    Updating once_cell v1.20.2 -> v1.21.4
```

Or update everything:

```console
$ cargo update
```

## How the Cargo.lock was created

1. Pinned `actix-web = "=4.9.0"` and `once_cell = "=1.20.2"` in Cargo.toml
2. Ran `cargo generate-lockfile` to lock these versions
3. Relaxed constraints to `actix-web = "4"` and removed `once_cell` from direct dependencies

This simulates a project that was last resolved when `once_cell` 1.20.x was latest.

## Cargo's documentation

From [`cargo update`](https://doc.rust-lang.org/cargo/commands/cargo-update.html):

> Its transitive dependencies will be updated only if SPEC cannot be updated without updating dependencies.

Cargo's test suite documents this behavior in [`transitive_minor_update`](https://github.com/rust-lang/cargo/blob/master/tests/testsuite/update.rs) with the comment:

> Also note that this is probably **counterintuitive and weird**. We may wish to change this one day.

## Impact on Dependabot

Dependabot's `VersionResolver` determines the latest version (e.g., 4.11.0) using modified Cargo.toml constraints. But `LockfileUpdater` runs `cargo update -p` with the original Cargo.toml, hitting this conservative behavior. The version mismatch causes `RuntimeError: "Failed to update actix-web!"`, surfaced as `unknown_error: null`.

Fixed in [dependabot-core PR #12487](https://github.com/dependabot/dependabot-core/pull/12487).

## Related

- [dependabot-core PR #12487](https://github.com/dependabot/dependabot-core/pull/12487) — Dependabot fix to accept partial updates
- [Cargo `transitive_minor_update` test](https://github.com/rust-lang/cargo/blob/master/tests/testsuite/update.rs) — Cargo's own test for this behavior
- [rust-lang/cargo#14446](https://github.com/rust-lang/cargo/issues/14446) — Related: Cargo downgrades transitive dependency
