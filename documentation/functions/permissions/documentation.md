# Ultimate Mac — Permission Functions
**Technical Documentation**

- **Project:** Ultimate Mac
- **Category:** Permission
- **Version:** 0.1
- **Language:** Bash
- **Platform:** macOS
- **Status:** Initial implementation

---

## 1. Purpose

The Permission category provides reusable functions for checking access rights, determining whether administrator privileges are available, requesting authorization, and running approved commands with elevated privileges.

These functions are designed to be called by other Ultimate Mac functions, the installer, interactive menus, and automated workflows.

### Design principles

1. **Exit codes are authoritative.** Callers must use return statuses to determine whether an operation succeeded.
2. **No unnecessary privilege elevation.** Operations should run with ordinary privileges whenever possible.
3. **No security bypasses.** Authorization must use supported system mechanisms.
4. **Reusable functions.** Functions must not depend on interactive menu navigation.
5. **Explicit dependencies.** Required functions must be loaded before dependent functions are called. A dedicated dependency-management file will be implemented later.
6. **Minimal side effects.** A permission check must not modify the target it checks.

## 2. File inventory

All five files belong in the repository's `functions/` directory and, once installed, in Ultimate Mac's installed functions directory.

| File | Public function | Purpose |
|---|---|---|
| `check_admin.sh` | `check_admin` | Checks effective root privileges |
| `check_file_access.sh` | `check_file_access` | Checks file and directory properties |
| `request_permission.sh` | `request_permission` | Requests administrator authorization |
| `run_as_admin.sh` | `run_as_admin` | Executes a command through `sudo` |
| `check_permission.sh` | `check_permission` | General-purpose permission dispatcher |

These files define functions. They should be loaded with `source` rather than executed as standalone programs.

## 3. Standard exit-code convention

The following convention applies to the functions in this category.

| Exit code | Meaning |
|---:|---|
| `0` | The requested check or operation succeeded |
| `1` | The check failed, access was unavailable, or an operation failed |
| `2` | Invalid arguments or unsupported operation |
| Other nonzero codes | May be returned by an underlying command, particularly through `run_as_admin` |

A return code of `1` does not necessarily mean a system error. For example, `check_admin` returning `1` simply means the current process is not running as root.

**Important:** callers must not interpret a successful function call as proof that a later, separate operation will succeed. Filesystem state and permissions can change between checking and acting.

## 4. Function reference

### 4.1 `check_admin`

**File:** `check_admin.sh`

**Signature:**

```bash
check_admin
```

**Description:**

Checks whether the current Bash process has effective user ID `0`, indicating root privileges.

**Arguments:** None.

**Return codes:**

- `0` — The process is running as root.
- `1` — The process is not running as root.

**Example:**

```bash
if check_admin; then
    echo "Administrator privileges are available."
else
    echo "Administrator privileges are unavailable."
fi
```

**Limitations:**

This checks effective root privileges, not whether the user account belongs to the macOS administrator group. It also does not establish that macOS privacy permissions, such as Full Disk Access, are available.

---

### 4.2 `check_file_access`

**File:** `check_file_access.sh`

**Signature:**

```bash
check_file_access <path> <access_type>
```

**Description:**

Checks whether a path exists or whether the current process has a specified type of access.

**Arguments:**

- `path` — The target file or directory.
- `access_type` — One of the supported types below.

| Access type | Test performed |
|---|---|
| `read` | `-r`: readable |
| `write` | `-w`: writable |
| `execute` | `-x`: executable or searchable, depending on the path |
| `exists` | `-e`: path exists |
| `directory` | `-d`: path identifies a directory |
| `file` | `-f`: path identifies a regular file |

**Return codes:**

- `0` — The requested test passed.
- `1` — The requested test failed.
- `2` — The path is missing or the access type is unsupported.

**Examples:**

```bash
check_file_access "$STATE_FILE" read
check_file_access "$STATE_FILE" write
check_file_access "$ULTIMATE_MAC_DIR" directory
```

**Limitations:**

- These tests reflect access as evaluated by the current process.
- A successful check does not guarantee that a subsequent operation will succeed.
- The function does not create missing files or directories.
- The `file` test follows symlinks when determining whether their targets are regular files.

---

### 4.3 `request_permission`

**File:** `request_permission.sh`

**Signature:**

```bash
request_permission
```

**Description:**

Ensures that the current process has root privileges. If it does not, the function attempts to validate administrator authorization using `sudo -v`.

**Dependencies:**

- `check_admin` must already be loaded.
- The system must provide `sudo`.

**Arguments:** None.

**Return codes:**

- `0` — Root privileges are already available or `sudo -v` succeeds.
- `1` — A terminal is unavailable when required, or authorization fails.

**Example:**

```bash
if request_permission; then
    echo "Authorization is available."
else
    echo "Authorization failed."
fi
```

**Limitations:**

- This function does not permanently elevate the current shell.
- Successful authorization does not grant Full Disk Access or bypass System Integrity Protection.
- `sudo -v` validates or refreshes authorization; it does not guarantee that a later elevated command will succeed.
- The current implementation checks for terminal availability. It is not a graphical authorization dialog.

---

### 4.4 `run_as_admin`

**File:** `run_as_admin.sh`

**Signature:**

```bash
run_as_admin <command> [arguments...]
```

**Description:**

Requests authorization through `request_permission` and then invokes the specified command using `sudo`.

**Dependencies:**

- `request_permission` must already be loaded.
- `check_admin` must be loaded for `request_permission` to work.

**Return codes:**

- `0` — The command succeeds.
- `1` — Authorization fails.
- `2` — No command was supplied.
- Other nonzero codes — The executed command or `sudo` failed.

**Example:**

```bash
run_as_admin mkdir -p "$ULTIMATE_MAC_DIR"
```

**Security requirements:**

- Only invoke commands explicitly approved by the calling workflow.
- Pass the command and its arguments separately. Do not build an executable command string from untrusted input.
- Validate paths and arguments before invoking this function.
- Never use elevated execution as a substitute for proper macOS authorization.

**Limitations:**

The function does not independently decide whether a command is safe. That responsibility belongs to the calling function or workflow.

---

### 4.5 `check_permission`

**File:** `check_permission.sh`

**Signature:**

```bash
check_permission <type> [path]
```

**Description:**

Provides a common entry point for the permission checks implemented by this category.

**Dependencies:**

- `check_admin` must already be loaded.
- `check_file_access` must already be loaded.

**Supported types:**

| Type | Required argument | Delegates to |
|---|---|---|
| `admin` | None | `check_admin` |
| `read` | Path | `check_file_access` |
| `write` | Path | `check_file_access` |
| `execute` | Path | `check_file_access` |
| `exists` | Path | `check_file_access` |
| `directory` | Path | `check_file_access` |
| `file` | Path | `check_file_access` |

**Return codes:**

- `0` — The selected check succeeds.
- `1` — The selected check fails.
- `2` — The type or arguments are invalid.

**Examples:**

```bash
check_permission admin

check_permission write "$STATE_FILE"

check_permission directory "$ULTIMATE_MAC_DIR"
```

**Limitations:**

This dispatcher does not implement separate macOS privacy-permission checks. Additional types may be added in the future without changing the public interface of existing checks.

## 5. Loading and dependencies

For the initial implementation, functions must be sourced in dependency order:

```bash
source "$FUNCTIONS_DIR/check_admin.sh"
source "$FUNCTIONS_DIR/check_file_access.sh"
source "$FUNCTIONS_DIR/request_permission.sh"
source "$FUNCTIONS_DIR/run_as_admin.sh"
source "$FUNCTIONS_DIR/check_permission.sh"
```

Sourcing defines the functions in the current shell. It does not automatically execute their bodies.

A dedicated dependency-management file is planned for a future implementation. It should centralize dependency declarations, load-order management, and missing-dependency detection.

Until that exists, callers must ensure dependencies have been loaded before invoking dependent functions.

## 6. Automated workflows

Automated workflows should use conditional checks rather than relying on printed messages.

**Recommended:**

```bash
if check_permission write "$STATE_FILE"; then
    echo "State file is writable."
else
    echo "State file is not writable."
    return 1
fi
```

**Avoid:**

```bash
check_permission write "$STATE_FILE"
echo "Done!"
```

The second example prints success regardless of whether the check succeeded.

Functions should propagate meaningful failures to their callers. A workflow that performs several operations should stop or recover appropriately when a required operation fails.

## 7. Testing checklist

Before integrating the category into the installer or other workflows, test the following:

- [ ] `check_admin` returns the correct status for root and non-root processes.
- [ ] `check_file_access` handles readable, writable, executable, nonexistent, and invalid targets.
- [ ] `check_file_access` rejects unsupported access types with code `2`.
- [ ] `request_permission` handles existing root privileges, successful authorization, failed authorization, and unavailable terminals.
- [ ] `run_as_admin` rejects missing commands and propagates command failures.
- [ ] `check_permission` dispatches all supported types correctly.
- [ ] All functions work when sourced in the documented order.
- [ ] No function reports success when its requested operation fails.

Tests involving authorization should be performed with harmless commands and should not require weakening system security.

## 8. Known limitations and future work

The current implementation is an initial foundation, not a complete macOS permission-management framework.

Potential future improvements include:

1. A central dependency-management file.
2. More consistent error reporting and error-code definitions.
3. Explicit validation of missing arguments in all functions.
4. Dedicated handling for macOS privacy permissions, including Full Disk Access where applicable.
5. A consistent logging interface for permission failures.
6. Automated tests for every documented return code.
7. Clear separation between checking authorization, requesting authorization, and executing privileged operations.

## 9. Compatibility and maintenance

Changes to a function's arguments, return codes, or documented behavior can break callers. Such changes should be documented and tested before integration.

New functions should follow the same principles: predictable arguments, explicit return codes, limited side effects, and no unconditional success messages.

**Core rule:** Ultimate Mac's automation must determine success from reliable return statuses, not from the presence of a reassuring message in the terminal.
