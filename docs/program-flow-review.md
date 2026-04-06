# Reality-EZPZ Code Review & Program Flow

## Repository overview

This repository contains two executable entry points:

1. `reality-ezpz.sh` — the main installer/configurator/manager script.
2. `tgbot.py` — an optional Telegram bot for user management.

At runtime, the shell script is responsible for host setup, config generation, Docker orchestration, and user lifecycle actions. The Telegram bot is a remote control layer that invokes the shell script commands.

---

## What `reality-ezpz.sh` does when run

### 1) Parse CLI and validate inputs

Execution starts by parsing CLI options with `parse_args`, validating transport/security/domain/server/port/bot values and setting operation flags (`--backup`, `--restore`, `--restart`, `--menu`, user actions, etc.).

### 2) Enforce root and pre-flight operations

The script requires root privileges (`EUID == 0`).

If backup/restore flags are supplied, it executes those flows first:
- `backup`: package state and upload to temp.sh.
- `restore`: download/read backup and restore config/users.

### 3) Install/upgrade/setup foundations

Normal run path performs:
- `generate_file_list`
- `install_packages`
- `install_docker`
- `configure_docker`
- `upgrade`
- `parse_config_file`
- `parse_users_file`
- `build_config`
- `update_config_file`
- `update_users_file`
- `tune_kernel`

### 4) Interactive and service orchestration

If menu mode is enabled (`--menu`), `main_menu` is launched (dialog-based TUI).

Then service lifecycle logic ensures the core compose stack is running and, when enabled, the Telegram bot compose stack is also running:
- `restart_docker_compose`
- `restart_tgbot_compose` (conditional)

### 5) Output-oriented user actions

Finally it handles output actions:
- `--show-server-config` → `show_server_config`
- `--list-users` → print users
- `--show-user`, `--add-user` → resolve username and call `print_client_configuration`

If no hard error occurs, it prints `Command has been executed successfully!`.

---

## Program flow chart (main shell script)

```mermaid
flowchart TD
    A[Start script] --> B[parse_args]
    B -->|invalid| B1[show_help + exit]
    B --> C{Root user?}
    C -->|No| C1[Exit with error]
    C -->|Yes| D{--backup?}
    D -->|Yes| D1[backup]
    D1 --> D2[print backup URL + exit]
    D -->|No| E{--restore?}
    E -->|Yes| E1[restore]
    E1 --> E2[mark restart=true]
    E -->|No| F[generate_file_list]
    E2 --> F

    F --> G[install_packages]
    G --> H[install_docker]
    H --> I[configure_docker]
    I --> J[upgrade]
    J --> K[parse_config_file]
    K --> L[parse_users_file]
    L --> M[build_config]
    M --> N[update_config_file]
    N --> O[update_users_file]
    O --> P[tune_kernel]

    P --> Q{--menu?}
    Q -->|Yes| Q1[main_menu]
    Q -->|No| R{restart requested?}
    Q1 --> R

    R -->|Yes| R1[restart_docker_compose]
    R1 --> R2{tgbot ON?}
    R2 -->|Yes| R3[restart_tgbot_compose]
    R2 -->|No| S[health checks for compose]
    R3 --> S
    R -->|No| S

    S --> T{--show-server-config?}
    T -->|Yes| T1[show_server_config + exit]
    T -->|No| U{--list-users?}
    U -->|Yes| U1[print users + exit]
    U -->|No| V[resolve username from args]
    V --> W{username set?}
    W -->|Yes| X[print_client_configuration]
    W -->|No| Y[success message]
    X --> Y
    Y --> Z[Exit 0]
```

---

## Function interconnection chart

### `reality-ezpz.sh` (high-level)

```mermaid
flowchart LR
  parse_args --> show_help
  parse_args --> backup
  parse_args --> restore
  parse_args --> build_config

  restore --> parse_config_file
  restore --> parse_users_file

  build_config --> generate_keys
  build_config --> generate_engine_config
  build_config --> generate_config
  build_config --> generate_docker_compose
  build_config --> generate_haproxy_config
  build_config --> generate_certbot_script
  build_config --> generate_certbot_deployhook
  build_config --> generate_certbot_dockerfile
  build_config --> generate_tgbot_compose
  build_config --> generate_tgbot_dockerfile
  build_config --> download_tgbot_script
  build_config --> generate_selfsigned_certificate

  main_menu --> configuration_menu
  main_menu --> add_user_menu
  main_menu --> delete_user_menu
  main_menu --> view_user_menu
  main_menu --> list_users_menu
  main_menu --> backup_menu
  main_menu --> restore_backup_menu

  configuration_menu --> config_core_menu
  configuration_menu --> config_server_menu
  configuration_menu --> config_transport_menu
  configuration_menu --> config_sni_domain_menu
  configuration_menu --> config_security_menu
  configuration_menu --> config_port_menu
  configuration_menu --> config_safenet_menu
  configuration_menu --> config_warp_menu
  configuration_menu --> config_tgbot_menu
```

### `tgbot.py` (event/callback flow)

```mermaid
flowchart TD
  A[Telegram update] --> B{restricted decorator}
  B -->|unauthorized| B1[send denial message]
  B -->|authorized| C{update type}

  C -->|/start| D[start]
  D --> E[show inline menu]

  C -->|button callback| F[button]
  F --> G{callback data}
  G -->|show_user| H[users_list]
  G -->|delete_user| I[users_list]
  G -->|add_user| J[add_user]
  G -->|show_user!name| K[show_user]
  G -->|delete_user!name| L[delete_user]
  G -->|approve_delete!name| M[approve_delete]
  G -->|cancel/start| N[cancel or start]

  C -->|text input| O[user_input]
  O --> P{expected_input=username?}
  P -->|yes| Q[validate username]
  Q -->|invalid/existing| J
  Q -->|valid| R[add_user_ezpz]
  R --> K

  H --> S[get_users_ezpz]
  I --> S
  K --> T[get_config_ezpz]
  L --> U[get_users_ezpz]
  M --> V[delete_user_ezpz]

  S --> W[run_command]
  T --> W
  U --> W
  V --> W
  R --> W
```

---

## Code review

## Strengths

- Clear operational phases in the shell script: parse, bootstrap, build, persist, orchestrate, output.
- Extensive input validation in `parse_args` and regex guards for common fields.
- Useful recovery/portability features: backup/restore, defaults restoration, menu-based management.
- Telegram bot uses a clean `restricted` decorator for authorization and keeps handlers concise.

## Risks / Improvement opportunities

1. **Remote execution pattern in bot**
   - `tgbot.py` executes a shell command that fetches a remote script URL at runtime (`bash <(curl -sL ...)`).
   - Even with username validation, runtime remote fetch introduces supply-chain and availability risk.
   - Recommendation: execute a pinned local script path instead and update by explicit deployment.

2. **Potential command injection surface (defense-in-depth)**
   - Bot command construction uses string concatenation and `bash -c` in `run_command`.
   - Current username regex blocks special characters, but safer design is `subprocess.run([...], shell=False)` with argument vectors.

3. **Error propagation in bot subprocess wrapper**
   - `run_command` discards stderr and return code, so handlers may present success even if command failed.
   - Recommendation: check return code and surface stderr to logs/user-safe error messages.

4. **Large monolithic shell script**
   - The shell script is feature-rich but long and dense; maintenance/testing burden is high.
   - Recommendation: split into sourced modules (args, renderers, docker, warp, ui, backup) while keeping one entrypoint.

5. **`set -e` + command substitution subtleties**
   - In large bash scripts, `set -e` can behave unexpectedly inside conditionals/subshells; this may create edge-case continuation paths.
   - Recommendation: add explicit `|| return/exit` for critical external calls and more targeted failure handling.

## Suggested next hardening steps (priority)

1. Replace bot remote-exec command with local script execution.
2. Refactor bot subprocess calls to argument arrays and explicit error handling.
3. Add basic automated checks (shellcheck, bashate or shfmt check, Python lint/type checks).
4. Add smoke tests for high-value flags (`--list-users`, `--add-user`, `--show-user`, `--backup`, `--restore`).

