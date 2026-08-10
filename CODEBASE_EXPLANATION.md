# Codebase Explanation: `senja_teras`

## 1. Overview

**`senja_teras`** is a minimal, from-scratch **Linux container runtime** written in Rust. It creates a fully isolated process environment using Linux kernel namespaces, sets up a private root filesystem via `pivot_root`, mounts a minimal `/dev`, `/proc`, `/tmp`, `/run`, `/dev/shm`, and `/dev/pts`, then drops the user into a `bash` shell inside the container.

The name is Indonesian: *"senja"* (dusk/twilight) + *"teras"* (core) — roughly "twilight core."

**Project artifact type:** Rust binary (not a library), targeting `x86_64-unknown-linux-gnu`.

---

## 2. Tech Stack

| Component        | Detail                                                                 |
|------------------|------------------------------------------------------------------------|
| Language         | Rust, edition **2024** (requires Rust 1.92.0+)                        |
| Core dependency  | `libc` 0.2.183 — raw FFI to Linux kernel syscalls                     |
| Error handling   | `anyhow` 1.0.102 — ergonomic error propagation                        |
| Path resolution  | `path-absolutize` 3.1.1 — resolve rootfs path relative to CWD         |
| Build system     | Cargo workspace (single member `senja_teras`)                          |
| Build profile    | Release: `opt-level=3`, `lto=true`, `codegen-units=1`, `panic=abort`  |
| Code formatting  | `rustfmt` with `max_width=100`, `edition=2024`, import reordering     |
| Linting          | `clippy` (via cargo alias `lint`)                                      |

---

## 3. Project Structure

```
.
├── .cargo/
│   └── config.toml            # Cargo aliases & custom build target-dir
├── .vscode/
│   └── settings.json          # VS Code: format-on-save, rust-analyzer config
├── .gitmodules                # Submodule: senja_teras → https://github.com/puisiteras/senja_teras.git
├── .rustfmt.toml              # Rustfmt configuration
├── Cargo.toml                 # Workspace root manifest
├── Cargo.lock                 # Lockfile
├── rust-toolchain.toml        # Pinned Rust toolchain: 1.92.0, x86_64-unknown-linux-gnu
├── LICENSE                    # GPLv3 (inferred from 35KB size)
└── senja_teras/               # The actual crate (git submodule)
    ├── Cargo.toml             # Package manifest
    ├── .gitignore             # Ignores /target
    └── src/
        ├── main.rs            # Entry point & container orchestration
        └── libcwarper/        # Thin, safe-ish wrappers over raw libc syscalls
            ├── mod.rs         # Module re-exports
            ├── newns.rs       # Linux namespace creation via clone(2)
            ├── mountbuilder.rs # Mount & pivot_root orchestration (builder pattern)
            ├── restrictor.rs  # Hostname & PR_SET_NO_NEW_PRIVS
            └── utils.rs       # Return-code checkers & CWD cache
```

---

## 4. Architecture & Module Responsibilities

### 4.1 `main.rs` — Entry Point & Container Orchestration

**File:** `senja_teras/src/main.rs`

The `main()` function calls `handler()`, which is the complete container setup sequence:

1. **Create Linux namespaces** via `NewNamespace::new(...)` with flags:
   - `CLONE_NEWNS` — mount namespace
   - `CLONE_NEWPID` — PID namespace
   - `CLONE_NEWUTS` — UTS namespace (hostname isolation)
   - `CLONE_NEWIPC` — IPC namespace
   - `CLONE_NEWUSER` — user namespace

2. **Run the child handler** via `newns.run(|| { ... })`. Inside:
   - Set hostname to `"senja_teras"` via `set_hostname()`
   - Build and execute the `MountBuilder` chain (see §4.3)
   - Call `set_no_new_privs()` to prevent privilege escalation
   - `execve("/bin/bash")` into a bash shell

3. **Hardcoded rootfs path:** `/tmp/debugging_teras_rootfs` — this is where the container's root filesystem must be pre-populated.

**Flow diagram (simplified):**
```
main() → handler()
           → NewNamespace::new(flags)     // prepare namespace config
           → newns.run(closure)           // clone(2) child
                [parent]                   [child]
                write uid_map/gid_map      wait on pipe signal
                send signal via pipe       → set_hostname("senja_teras")
                waitpid(child)             → MountBuilder chain
                                           → set_no_new_privs()
                                           → execve("/bin/bash")
```

### 4.2 `newns.rs` — Namespace Creation (`clone`)

**File:** `senja_teras/src/libcwarper/newns.rs`

This module provides `NewNamespace`, the core isolation primitive.

- **`NewNamespace::new(nsflag)`** — stores the clone flags.
- **`NewNamespace::run(handler)`** — the critical method:
  1. Allocates a **2 MiB stack** aligned to 16 bytes (C-ABI requirement).
  2. Creates a **pipe** for synchronization: the parent signals the child after writing UID/GID mappings.
  3. Calls `clone(2)` with `child_entry` as the entry point, passing a `ChildArgs` struct containing the handler closure and pipe fd. The clone flags include `SIGCHLD` so the parent can `waitpid`.
  4. **Parent side:**
     - If `CLONE_NEWUSER` is set, writes UID/GID mappings to `/proc/{pid}/uid_map`, `/proc/{pid}/setgroups`, `/proc/{pid}/gid_map` — maps **host UID 0 → container UID 0** (root).
     - Sends a 1-byte signal through the pipe to unblock the child.
     - Calls `waitpid` and reports exit status.
  5. **Child side:**
     - Blocks on `read(pipe)` until parent finishes mapping.
     - Closes the read end.
     - Executes the user-provided handler closure.

> **Fact:** The UID mapping is hardcoded to `"0 0 1\n"` — this grants full root inside the container. The `setgroups` is set to `"deny"` as required before writing `gid_map`.

### 4.3 `mountbuilder.rs` — Mount & `pivot_root` Orchestration (Builder Pattern)

**File:** `senja_teras/src/libcwarper/mountbuilder.rs` (10,216 bytes — the largest module)

This module implements a **builder pattern** to chain filesystem setup operations. Every method (except the final ones) returns `&Self`, enabling the fluent chain seen in `main.rs`.

#### `MountBuilder::new(rootfs: &Path)`
- Resolves the rootfs path absolutely (using `path-absolutize` and the cached CWD).
- Verifies the path exists and is a directory.
- Stores the path as a `CString`.

#### Method-by-method call chain (as executed in `handler()`):

| Step | Method                      | What it does                                                                 |
|------|-----------------------------|------------------------------------------------------------------------------|
| 1    | `make_root_private()`       | `mount --make-rprivate /` — prevents mounts in the new namespace from propagating back |
| 2    | `bind_mount_rootfs()`       | Bind-mounts the rootfs path onto itself (recursive), then makes it private   |
| 3    | `mount_staging_host_dev()`  | Bind-mounts host `/dev` → `{rootfs}/.host/dev` (to access device nodes later) |
| 4    | `mount_before_pivot_root()` | Bind-mounts `/nix/store` → `{rootfs}/nix/store` (optional, for Nix environments) |
| 5    | `pivot_root()`              | Calls `pivot_root(rootfs, {rootfs}/.old_root)`, then `chdir("/")`            |
| 6    | `create_minimal_dev()`      | Mounts a tmpfs on `/dev` (4 MiB), mounts `/dev/null`, `/dev/zero`, `/dev/full`, `/dev/random`, `/dev/urandom`, `/dev/tty` from host, sets up `/dev/pts`, `/dev/mqueue`, `/dev/shm` (16 MiB), `/tmp` (32 MiB), `/run` (16 MiB) |
| 7    | `mount_restricted_proc()`   | Mounts `/proc` with `hidepid=2,subset=pid`, reads UID/GID from `/proc/self/uid_map` and `/proc/self/gid_map`, writes `/etc/passwd` and `/etc/group` |
| 8    | `link_proc_to_dev()`        | Creates convenience symlinks: `/dev/fd` → `/proc/self/fd`, `/dev/stdin/out/err`, `/dev/core` → `/proc/kcore` |
| 9    | `unmount_staging_dev()`     | Detaches `/.host/dev` from the staging mount                                  |
| 10   | `deatach_old_root()`        | Detaches `/.old_root` (the original root from before `pivot_root`)            |

#### Key implementation details:

- **`validate_target()`** — Prepends the rootfs path to any target that starts with `/` or `\`, enabling mounts inside the new rootfs *before* `pivot_root`.
- **`mount_before_pivot_root()` vs `mount_after_pivot_root()`** — The former prepends the rootfs path; the latter operates directly on the new `/` after pivot.
- **`mount_dev()`** — Creates an empty file at the device path inside the new root, then bind-mounts the corresponding host device node from `/.host/dev`.
- **`mount_restricted_proc()`** — Parses `/proc/self/uid_map` to determine the container UID and creates matching `/etc/passwd` and `/etc/group` entries (so `whoami` etc. work).

### 4.4 `restrictor.rs` — Process Restrictions

**File:** `senja_teras/src/libcwarper/restrictor.rs`

Two small functions:

- **`set_hostname(name: &str)`** — Calls `sethostname(2)` to set the container hostname (works because a UTS namespace was created).
- **`set_no_new_privs()`** — Calls `prctl(PR_SET_NO_NEW_PRIVS, 1, ...)` — prevents the process (and its children) from ever gaining new privileges via setuid/setgid binaries or capabilities.

### 4.5 `utils.rs` — Return-Code Wrappers & CWD Cache

**File:** `senja_teras/src/libcwarper/utils.rs`

- **`warp_ret(ret, context)`** — Converts a libc-style integer return (0 = success, non-zero = error) into `anyhow::Result`. On failure, includes the context string and `errno`.
- **`warp_io_call(result, context)`** — Wraps `std::io::Result` with context.
- **`get_cwd()`** — Returns a cached `&'static Path` of the current working directory (lazily initialized once via `OnceLock`). This is important because after `pivot_root` + `chdir("/")`, the CWD as seen by the process changes — the cache preserves the *pre-pivot* CWD for path resolution.

---

## 5. External Dependencies & Integrations

**This project has no external service dependencies** — no databases, no message queues, no cloud APIs. The only external dependency is the **host Linux kernel** (via its syscall interface through `libc`).

The only filesystem dependency is:
- **Bash** must exist at `/bin/bash` inside the rootfs (`/tmp/debugging_teras_rootfs/bin/bash`).
- The rootfs directory `/tmp/debugging_teras_rootfs` must be pre-populated with a minimal Linux userspace.
- (Optional) `/nix/store` on the host is bind-mounted into the container if it exists.

---

## 6. Build, Run & Test

### Build commands

```bash
# Build (must be run from workspace root)
cargo build                    # Debug build
cargo build --release          # Optimized release build

# Note: .cargo/config.toml sets target-dir to /tmp/teras
```

### Run

```bash
# Requires:
#   1. Linux kernel with user namespaces enabled
#   2. A populated rootfs at /tmp/debugging_teras_rootfs
#   3. Run as a user with CLONE_NEWUSER capability (or root)

cargo run
# or
./target/release/senja_teras    # after building
```

### Lint & Format

```bash
cargo lint    # alias: clippy --workspace --all-targets --all-features -- -D warnings
cargo rapi    # alias: fmt --all   ("rapi" = Indonesian for "neat/tidy")
cargo sapu    # alias: clippy --workspace --fix --allow-dirty --allow-staged  ("sapu" = "sweep")
```

### Test

**There are no tests in this codebase.** The crates `main.rs` and all modules in `libcwarper/` contain zero `#[test]` annotations and no `#[cfg(test)]` blocks.

---

## 7. Configuration & Environment

### Required: Rootfs

The path `/tmp/debugging_teras_rootfs` is **hardcoded** in `main.rs` (line where `PathBuf::from_str` is called). This directory must:
- Exist and be a directory
- Contain a minimal Linux root filesystem including at minimum `/bin/bash`

### Required: Environment Variables

Inside the container, the environment is hardcoded to:
```
PATH=/bin:/sbin
HOME=/home
TERM=true
```

### No configuration files, no secrets, no environment variables are read by the binary at runtime.

---

## 8. CI/CD & Deployment

**There are no CI/CD pipeline configuration files in this repository** — no `.github/`, no `.gitlab-ci.yml`, no `Dockerfile`, no `docker-compose.yml`, no Kubernetes manifests. This project appears to be a personal/experimental tool with no automated deployment pipeline.

---

## 9. Notable Risks & Observations

### 9.1 Hardcoded Paths — Production Blocker
The rootfs path `/tmp/debugging_teras_rootfs` is hardcoded. The path name `debugging_teras` strongly suggests this is a **development/debugging artifact** and not intended for production use as-is.

### 9.2 Hardcoded UID 0 Mapping — Total Isolation Escape Risk
The UID mapping `"0 0 1\n"` maps host UID 0 to container UID 0. If the process is run as root, the container process is **true root** inside the container. Combined with the hardcoded user namespace, this is expected — but the lack of capability restrictions (`capabilities(7)`) means the container process can do anything inside its namespaces.

### 9.3 No Seccomp / No Cgroup — Incomplete Isolation
This runtime does not apply:
- **seccomp** filters (no `prctl(PR_SET_SECCOMP, ...)`)
- **cgroup** restrictions (no cgroupv1 or cgroupv2 operations)
- **Capability bounding** (no `prctl(PR_CAPBSET_DROP, ...)`)

The isolation relies entirely on namespaces + `PR_SET_NO_NEW_PRIVS`. An attacker who escapes the namespace (e.g., via a kernel exploit) faces no further barriers.

### 9.4 Error Handling — Process Aborts Silently on `execve` Failure
If `execve("/bin/bash")` fails (e.g., bash not installed in rootfs), the error is caught by `warp_ret` and propagated as `anyhow::Error`, causing the child handler to return `255` and the parent reports "child exited with code 255". There is no fallback shell.

### 9.5 Pipe Synchronization — Potential Deadlock on Error
In `newns.rs`, the parent writes UID/GID maps, writes to the pipe, then `waitpid`s. The child first reads from the pipe, then runs the handler. If the parent fails *before* writing to the pipe (e.g., UID map write fails), the child blocks forever on `read()`. The code does not handle partial pipe synchronization failures.

### 9.6 No Cleanup on Failure
If the `MountBuilder` chain fails partway through, there is no rollback. Partial mounts, directories, and symlinks created by prior steps remain. This is acceptable for a disposable container but could leak resources in long-running scenarios.

### 9.7 Safety: Mutable Static via `OnceLock` — Actually Fine
`CWD_CACHE` in `utils.rs` uses `OnceLock<PathBuf>`, which is a safe lazy-initialized static. This is correct and not a concern — but it is worth noting that after `pivot_root`, any code calling `get_cwd()` will still return the *pre-pivot* CWD (which is the intended behavior for path resolution).

### 9.8 Memory Layout in `newns.rs`
The 2 MiB stack is allocated as a `Vec<u8>` and aligned to 16 bytes. There are comments in Indonesian explaining the alignment logic. The stack is passed to `clone(2)` as the child's stack — correct usage, but the stack size is not configurable.

### 9.9 No Logging
All status output uses `println!` and `eprintln!` — there is no structured logging, no log levels, no `log` or `tracing` crate. This is acceptable for a debug tool but insufficient for production.

### 9.10 Drops bash into an environment that may have no `passwd`/`group` entries initially
The `mount_restricted_proc()` step generates `/etc/passwd` and `/etc/group` based on the UID/GID map. However, if this step is reached *after* `pivot_root` (as it is in the current chain), the files are written inside the new root. This is correct — but it also means the files are ephemeral (on tmpfs mounts if `/etc` is not part of the rootfs).

---

## 10. Open Questions

1. **What is the intended use case?** The name "senja_teras" and the hardcoded debugging path suggest this is an experimental/learning project for Linux container primitives. It resembles a simplified version of tools like `unshare`, `chroot`, or minimal Docker-like runtimes. (Inference from code, not confirmed.)

2. **What should populate `/tmp/debugging_teras_rootfs`?** There is no script, Dockerfile, or documentation describing how to prepare the rootfs. The user must supply their own Linux userspace at that path.

3. **Is `/nix/store` bind-mount intentional or leftover debugging?** The step `mount_before_pivot_root(Some(c"/nix/store"), c"/nix/store", ...)` unconditionally attempts to bind-mount `/nix/store` from the host. If the host is not a NixOS/Nix system, this directory likely does not exist, and `fs::create_dir_all` + `mount` would create an empty directory inside the rootfs and fail to mount (returning an error). This could break the entire container setup if the host lacks `/nix/store`.

4. **Why is `CWD_CACHE` used instead of just resolving the path at `MountBuilder::new` time?** The `path-absolutize` crate already resolves relative to CWD when called. The cache may be a premature optimization or a safeguard against CWD changing during the builder chain. The code comment in Indonesian does not explain the rationale.

5. **Is there a companion project or orchestration layer?** The GitHub organization `puisiteras` and the repository name suggest there may be other related projects (e.g., a rootfs builder, a container manager). This codebase alone cannot be used without a pre-built rootfs.

---

## 11. Summary

`senja_teras` is a **minimal, educational Linux container runtime** that demonstrates:
- User, mount, PID, UTS, and IPC namespace creation via `clone(2)`
- UID/GID mapping for user namespaces
- `pivot_root`-based root filesystem switching
- Builder-pattern mount orchestration for `/dev`, `/proc`, `tmpfs` filesystems
- Process hardening via `PR_SET_NO_NEW_PRIVS`

It is **not production-ready** due to hardcoded paths, missing seccomp/cgroups, lack of tests, absence of CI/CD, and incomplete error recovery. It is best understood as a **learning tool or proof-of-concept** for understanding how container runtimes like `runc`, `crun`, or `youki` work at the syscall level.

The code is well-structured and idiomatic Rust (fluent builder pattern, `anyhow` for errors, proper `unsafe` scoping with safe wrappers), with inline comments in Indonesian suggesting a developer from Indonesia.
