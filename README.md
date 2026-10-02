# portmaster-tauri-tao-patch

Patched copy of [tauri-apps/tao](https://github.com/tauri-apps/tao) 0.32.8,
the windowing library of the Tauri UI framework, used by the
[Portmaster](https://github.com/safing/portmaster) desktop UI on Windows.

This is a temporary fix for a reference-count race that crashes the Portmaster
UI. It is not a maintained fork and is only needed until an upstream release
contains a fix.

The upstream README is kept as [README.upstream.md](README.upstream.md).

## Why this exists

Portmaster issues:

- <https://github.com/safing/portmaster/issues/2185> (v2.1.19)
- <https://github.com/safing/portmaster/issues/2255> (v2.2.3)

Symptom: `portmaster.exe` (the Tauri UI process) dies randomly on Windows with
`Illegal instruction (0xc000001d)`, often preceded by
`STATUS_HEAP_CORRUPTION (0xc0000374)`.

Mechanism: `tauri_runtime_wry::Context` derives `Clone` and contains tao's
`EventLoopWindowTarget`. On Windows that struct holds the event-loop runner
behind `Rc<EventLoopRunner>`, a plain non-atomic reference count. Tauri clones
`Context` on every `AppHandle`, `Window` or `Webview` clone, from any thread.
Two threads cloning at the same instant lose an update, the runner is freed
while still in use, and the next clone aborts.

Upstream tracking:

- Issue: <https://github.com/tauri-apps/tauri/issues/15408>
- Tauri-side fix (open): <https://github.com/tauri-apps/tauri/pull/15411>
- tao-side fix, same change as here (closed unmerged):
  <https://github.com/tauri-apps/tao/pull/1334>

## What is changed

Branch `portmaster/windows-atomic-refcount-0.32.8` is upstream tag
`tao-v0.32.8` plus our commits: the code change and this documentation. The authoritative
list of changes is the diff against the tag:

<https://github.com/safing/portmaster-tauri-tao-patch/compare/tao-v0.32.8...portmaster/windows-atomic-refcount-0.32.8>

Code changes, all marked with `// PORTMASTER PATCH` comments:

| File | Change |
|------|--------|
| `src/platform_impl/windows/event_loop/runner.rs` | `type EventLoopRunnerShared<T> = Rc<EventLoopRunner<T>>` becomes `Arc<...>`; import `rc::Rc` becomes `sync::Arc` |
| `src/platform_impl/windows/event_loop.rs` | `Rc::new(EventLoopRunner::new(..))` becomes `Arc::new(..)`; unused `rc::Rc` import removed |
| `src/lib.rs` | adds `pub const PORTMASTER_PATCH_LEVEL: u32 = 1;` (marker, see below) |

Only the reference count becomes atomic. The runner keeps its `Cell` and
`RefCell` interior and stays non-`Send` and non-`Sync`. Nothing else in tao is
touched. Linux and macOS code paths are unchanged.

`version` in `Cargo.toml` stays `0.32.8` on purpose: Cargo only accepts a
`[patch]` whose version matches the one in the consumer's lock file.

## How Portmaster uses it

In `desktop/tauri/src-tauri/Cargo.toml`, pinned to a commit so the built code
cannot change under a force push:

```toml
[patch.crates-io]
tao = { git = "https://github.com/safing/portmaster-tauri-tao-patch", rev = "<commit>" }
```

Portmaster also asserts the marker constant at compile time, reached through
`tauri_runtime_wry::tao` so it is guaranteed to be the tao that Tauri compiles
against:

```rust
const _: () = assert!(tauri_runtime_wry::tao::PORTMASTER_PATCH_LEVEL == 1);
```

If a dependency update makes Cargo drop the patch, the constant disappears and
the Portmaster build fails instead of silently shipping the unpatched crate.

## Is this patch still needed?

1. Check whether <https://github.com/tauri-apps/tauri/pull/15411> or an
   equivalent change is in a released `tauri-runtime-wry` or `tao`:
   <https://v2.tauri.app/release/tauri-runtime-wry/all-versions/>,
   <https://v2.tauri.app/release/tao/all-versions/>.
2. Practical test in the Portmaster repository: remove the `[patch.crates-io]`
   entry and the marker assertion, build the debug UI with
   `cargo build --profile dev` (Cargo re-resolves tao from crates.io by
   itself) and start it with the command-line flag `--verify-tao-patch`. The
   debug build contains a stress test that clones the app handle from four
   threads. An unpatched tao crashes within seconds; a fixed one keeps logging
   `still alive` lines. The guard, the stress hook and the full description
   live in `desktop/tauri/src-tauri/src/windows_tao_patch.rs` in the Portmaster
   repository.

## Making a patch branch for a newer tao

Needed when Portmaster upgrades Tauri and the new Tauri requires a newer tao,
while upstream still has no fix. The idea: start from the new upstream release
tag and put our commits (the code change and this documentation) on top of it. The example uses
tao 0.37.1; replace the version with the one Tauri needs.

1. Get the upstream release tag into this repository. The fork does not track
   upstream tags, so fetch the one tag explicitly (`--no-tags` keeps all other
   upstream tags out).

   ```bash
   git remote add upstream https://github.com/tauri-apps/tao.git   # once per clone
   git fetch upstream tag tao-v0.37.1 --no-tags
   ```

2. Create the new patch branch from that tag.

   ```bash
   git checkout -b portmaster/windows-atomic-refcount-0.37.1 tao-v0.37.1
   ```

3. Copy the patch commits from the previous patch branch onto it. The previous
   branch is its upstream tag plus our commits, so the range "everything after
   the old tag up to the old branch" is exactly those commits, no hashes needed:

   ```bash
   git cherry-pick tao-v0.32.8..portmaster/windows-atomic-refcount-0.32.8
   ```

   If the cherry-pick reports a conflict, the upstream lines moved; apply the
   change by hand (it is two lines, see "What is changed") and continue with
   `git cherry-pick --continue`.

4. Check that it compiles for Windows, then tag the result.

   ```bash
   cargo check
   git tag tao-v0.37.1-portmaster.1
   ```

5. Push three things in one go: the branch, our new tag, and the upstream
   release tag. The upstream tag is pushed because the fork does not have it
   otherwise, and GitHub's compare view against it needs it.

   ```bash
   git push origin portmaster/windows-atomic-refcount-0.37.1 tao-v0.37.1-portmaster.1 tao-v0.37.1
   ```

6. On GitHub, set the new branch as the default branch so this README stays the
   one visitors see. In Portmaster, point `[patch.crates-io]` at the new commit.

Bump `PORTMASTER_PATCH_LEVEL` (here and in Portmaster's guard) only if the
patch content itself changes, not for a plain re-base.

## When upstream ships the fix

Portmaster drops the `[patch]` entry, the marker assertion and the
`windows_tao_patch.rs` module, and upgrades to the fixed Tauri release.
Portmaster commits made before that keep referencing this repository at a
pinned commit.

## License

tao is licensed under Apache-2.0 (see [LICENSE](LICENSE)). The changes in this
repository are published under the same license.
