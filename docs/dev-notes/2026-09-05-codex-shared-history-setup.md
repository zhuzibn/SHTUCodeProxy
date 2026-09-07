# Codex shared history and ShanghaiTech proxy setup

**Date**: 2026-09-05
**Type**: docs
**Branch**: `dev/docs-codex-shared-history`
**Status**: ✅ Complete
**Related issues**: #018, #019, #020

## Objective

Configure normal `codex` and ShanghaiTech `codex-uni` to share one Codex session history without allowing SHTUCodeProxy to overwrite the normal Codex OAuth login. Document the complete setup and API-key refresh procedure.

## Acceptance criteria

- [x] `codex` and `codex-uni` use the same default `CODEX_HOME` and session store; the current provider-filtered picker limitation and UUID workaround are documented.
- [x] Normal Codex OAuth credentials remain in `~/.codex/auth.json`.
- [x] The university API key remains separate and is supplied through a custom-provider `env_key`.
- [x] The SHTUCodeProxy GUI paths and API-key refresh command are recorded.
- [x] The old isolated university sessions remain accessible through `codex-uni-old`.
- [x] The current GUI writer's external-profile limitation and safe staging-path workaround are recorded.

## Why the histories were separate

The original Bash wrapper set a different state root:

```bash
codex-uni() (
    export CODEX_HOME="/mnt/c/Users/zzf-m/.codex_shanghaitech"
    command codex "$@"
)
```

`CODEX_HOME` contains Codex configuration, authentication, sessions, and other state. Consequently, normal Codex read sessions from `/home/zhuzibn/.codex`, while `codex-uni` read a separate session collection from `C:\Users\zzf-m\.codex_shanghaitech`.

The corrected design keeps the default `CODEX_HOME` for both launchers and switches only the provider configuration and university credential.

## Final path layout

| Purpose | Path | Important rule |
|---|---|---|
| Shared Codex state and history | `/home/zhuzibn/.codex` | Both `codex` and the new `codex-uni` use this root. |
| Normal Codex authentication | `/home/zhuzibn/.codex/auth.json` | Owned by normal Codex login. Never select this file in SHTUCodeProxy. |
| ShanghaiTech profile | `/home/zhuzibn/.codex/shtu_proxy.config.toml` | Selected by `codex --profile shtu_proxy`. Contains no API-key value. |
| University key bridge | `/home/zhuzibn/.config/codex/shtu-api-key` | Raw key copied from the GUI-generated university `auth.json`; permission must be `600`. |
| GUI-generated staging config | `C:\Users\zzf-m\.codex_shanghaitech\config.toml` | Safe target for **Write Client Config**. Do not point the current GUI writer at the shared profile. |
| GUI-managed university authentication | `C:\Users\zzf-m\.codex_shanghaitech\auth.json` | Keep this path in SHTUCodeProxy. It must not point to normal Codex `auth.json`. |
| Legacy university state | `C:\Users\zzf-m\.codex_shanghaitech` | Retained for old sessions through `codex-uni-old`. |

## SHTUCodeProxy GUI settings

Use these values in the client configuration panel:

| Field | Value |
|---|---|
| Host | `127.0.0.1` |
| Port | `8082` |
| Current Main Model | `gpt-5.6-sol` or the currently selected campus model |
| Codex `config.toml` Path | `C:\Users\zzf-m\.codex_shanghaitech\config.toml` |
| Codex `auth.json` Path | `C:\Users\zzf-m\.codex_shanghaitech\auth.json` |

Do not point either GUI field into `/home/zhuzibn/.codex`:

- The GUI's `auth.json` writer would replace the normal Codex OAuth mode with API-key mode.
- The current GUI's `config.toml` writer emits `requires_openai_auth = true` and the obsolete inline `[profiles.shtu_proxy]` table. Writing that output over `shtu_proxy.config.toml` would undo the external-profile and `env_key` setup.

Issue #018 tracks this writer limitation. Until it is fixed in SHTUCodeProxy, treat `.codex_shanghaitech/config.toml` and `.codex_shanghaitech/auth.json` as GUI-managed staging files. The manually maintained WSL profile consumes the staged API key through the bridge file.

## ShanghaiTech profile

The active profile is `/home/zhuzibn/.codex/shtu_proxy.config.toml`. Its relevant structure is:

```toml
model = "gpt-5.6-sol"
model_provider = "shtu_proxy"

[shell_environment_policy]
inherit = "all"

[shell_environment_policy.filters]
SHTU_CODEX_API_KEY = "exclude"

[model_providers.shtu_proxy]
name = "SHTUClaudeProxy"
base_url = "http://127.0.0.1:8082/v1"
wire_api = "responses"
env_key = "SHTU_CODEX_API_KEY"
```

The profile must use `env_key`; do not add `requires_openai_auth = true`. The environment filter prevents agent-spawned shell commands from receiving the university key.

Codex CLI 0.134.0 and later loads named profiles from `$CODEX_HOME/<name>.config.toml`. Do not restore the obsolete `[profiles.shtu_proxy]` table.

For a fresh installation, first let the GUI write its two staging files, copy `.codex_shanghaitech/config.toml` to `/home/zhuzibn/.codex/shtu_proxy.config.toml`, and make these one-time edits:

1. Replace `requires_openai_auth = true` with `env_key = "SHTU_CODEX_API_KEY"`.
2. Remove the complete `[profiles.shtu_proxy]` table.
3. Add the `[shell_environment_policy.filters]` table shown above.
4. Set permission `600` with `chmod 600 "$HOME/.codex/shtu_proxy.config.toml"`.

For an API-key-only change, do not recopy the staging config; refresh only the bridge file as described below. If the proxy host, port, or selected Codex model changes, also update `base_url` or `model` in the WSL profile manually.

## Bash launchers

The following definitions belong in `/home/zhuzibn/.bashrc`:

```bash
# ShanghaiTech Codex: legacy state, retained for old university sessions
codex-uni-old() (
    export CODEX_HOME="/mnt/c/Users/zzf-m/.codex_shanghaitech"
    command codex "$@"
)

# ShanghaiTech Codex: university proxy with shared history
codex-uni() (
    unset CODEX_HOME
    export SHTU_CODEX_API_KEY="$(<"$HOME/.config/codex/shtu-api-key")"
    command codex --profile shtu_proxy "$@"
)
```

After editing `.bashrc`, open a new WSL terminal or reload it:

```bash
source "$HOME/.bashrc"
```

## Refresh the university API key

There are two required university-key locations:

1. `C:\Users\zzf-m\.codex_shanghaitech\auth.json`, written by SHTUCodeProxy.
2. `/home/zhuzibn/.config/codex/shtu-api-key`, read by the `codex-uni` wrapper.

The profile itself contains only the environment-variable name and never contains the secret.

Whenever the API key is changed or regenerated:

1. In SHTUCodeProxy, update the API key, save the configuration, and click **Write Client Config**. Confirm that both GUI paths still point to the `.codex_shanghaitech` staging directory; never target the shared WSL profile or normal Codex `auth.json`.
2. In WSL, run the following command. It copies the `OPENAI_API_KEY` value without printing it and replaces the bridge file atomically.

```bash
mkdir -p "$HOME/.config/codex"

codex_uni_key_tmp="$HOME/.config/codex/shtu-api-key.tmp"

(
    umask 077
    node -e '
        const fs = require("fs");
        const auth = JSON.parse(fs.readFileSync(process.argv[1], "utf8"));
        if (typeof auth.OPENAI_API_KEY !== "string" || auth.OPENAI_API_KEY.length === 0) {
            throw new Error("OPENAI_API_KEY not found");
        }
        process.stdout.write(auth.OPENAI_API_KEY);
    ' "/mnt/c/Users/zzf-m/.codex_shanghaitech/auth.json" > "$codex_uni_key_tmp"
)

chmod 600 "$codex_uni_key_tmp"
mv "$codex_uni_key_tmp" "$HOME/.config/codex/shtu-api-key"
unset codex_uni_key_tmp
```

3. Verify the copy without displaying the key:

```bash
node -e '
    const fs = require("fs");
    const auth = JSON.parse(fs.readFileSync(process.argv[1], "utf8"));
    const bridge = fs.readFileSync(process.argv[2], "utf8");
    console.log(auth.OPENAI_API_KEY === bridge ? "API key synchronized" : "API key mismatch");
' "/mnt/c/Users/zzf-m/.codex_shanghaitech/auth.json" \
  "$HOME/.config/codex/shtu-api-key"

stat -c '%a %n' "$HOME/.config/codex/shtu-api-key"
```

Expected output includes `API key synchronized` and file mode `600`.

4. Exit any already-running `codex-uni` process and start a new one. A running process retains the previous environment-variable value.

Never print the key with `cat`, add it directly to `.bashrc` or the TOML profile, commit it, or copy it into `/home/zhuzibn/.codex/auth.json`.

## Verification

Start SHTUCodeProxy and confirm that port 8082 is listening:

```bash
ss -ltn | rg ':8082'
```

Validate the university provider with a minimal request:

```bash
codex-uni exec --skip-git-repo-check "Reply with exactly: PROFILE_OK"
```

Check shared history from both launchers:

```bash
codex resume --all
codex-uni resume --all
```

Both commands read sessions from `/home/zhuzibn/.codex`, but Codex CLI 0.153.4 filters the picker by the active `model_provider`. `--all` disables current-working-directory filtering; it does not disable provider filtering. Therefore, `codex-uni resume --all` lists `shtu_proxy` sessions but omits ordinary `openai` sessions.

To continue an ordinary Codex session through the university provider, copy its session UUID from `codex resume --all`, exit the picker, and pass it explicitly:

```bash
codex-uni resume <SESSION_ID>
```

Direct UUID resume bypasses the picker filter. An isolated test with Codex CLI 0.153.4 successfully loaded an `openai` session while the active profile used `shtu_proxy`. This resumes the conversation with the current `codex-uni` profile/provider; it does not modify or duplicate the original rollout file merely by opening it.

Access the original isolated university sessions with:

```bash
codex-uni-old resume --all
```

## Troubleshooting

| Symptom | Check |
|---|---|
| `Connection failed` or repeated reconnects | Start SHTUCodeProxy and confirm that `127.0.0.1:8082` is listening. |
| HTTP 401/403 after changing the campus key | Run the API-key refresh procedure, then restart `codex-uni`. |
| `Profile shtu_proxy not found` | Confirm `/home/zhuzibn/.codex/shtu_proxy.config.toml` exists and the wrapper uses `--profile shtu_proxy`. |
| `failed to decode models response: missing field models` appears, but requests continue | Codex 0.153.4 attempted to parse the proxy's OpenAI-style `/v1/models` response as a Codex model catalog. This warning was non-fatal in live testing; verify the final model response before treating it as a connection failure. |
| Shared profile reverted to `requires_openai_auth = true` | The GUI config path targeted the shared profile. Restore the external-profile edits and change the GUI config path back to `.codex_shanghaitech\config.toml`. |
| Normal history is missing from `codex-uni resume --all` | This is the Codex 0.153.4 provider-filtered picker limitation tracked as #020. Obtain the UUID from `codex resume --all`, then run `codex-uni resume <SESSION_ID>`. |
| Normal `codex` login stops working | Confirm the GUI still writes to `.codex_shanghaitech\auth.json`, not `/home/zhuzibn/.codex/auth.json`. |
| Need an old university-only session | Use `codex-uni-old resume --all`. |

## Rollback

The original wrapper was backed up at:

```text
/home/zhuzibn/.bashrc.codex-history-backup-20260905-220305
```

The original `.codex_shanghaitech` directory, its `auth.json`, its `config.toml`, and its 11 historical sessions were not modified. To temporarily bypass the shared setup, use `codex-uni-old` rather than deleting or merging state files.

## Implementation and validation record

The local migration made these changes:

- Added `/home/zhuzibn/.codex/shtu_proxy.config.toml`.
- Added the mode-`600` bridge file `/home/zhuzibn/.config/codex/shtu-api-key`.
- Updated `/home/zhuzibn/.bashrc` with `codex-uni` and `codex-uni-old`.
- Added a concise README recipe for obtaining a normal session ID with `/status` and resuming it through `codex-uni`.
- Preserved `/home/zhuzibn/.codex/auth.json` and the entire `.codex_shanghaitech` state root.

Validation completed on Codex CLI 0.153.4:

| Check | Result |
|---|---|
| Bash syntax | Pass |
| Profile parsed and started a thread | Pass |
| Bridge credential matches university `auth.json` | Pass |
| Profile and bridge permissions | `600` |
| Shared session store | 220 session files at validation time |
| Legacy university store | 11 session files at validation time |
| Cross-provider picker | Codex 0.153.4 filters picker rows by active `model_provider`; `--all` only disables cwd filtering |
| Cross-provider UUID resume | Pass: an `openai` session loaded through `--profile shtu_proxy` in an isolated state-store copy |
| Live provider response | Pass: Codex CLI 0.153.4 returned the requested exact text through both `codex-uni` and `codex-uni-old` using GPT-5.6-Sol |
| Model-catalog refresh | `codex-uni` logged a non-fatal schema warning because `/v1/models` returned an OpenAI `data` list rather than the Codex catalog's `models` field; the generation request still passed |
| Current writer behavior | Source and smoke-test assertions confirm `requires_openai_auth` plus inline-profile output; workaround documented under issue #018 |
| Documentation links | README and project-index targets resolve |
| Secret scan | No API-key literal or bearer token found in the changed documentation |
| Whitespace validation | `git diff --check` passed |

## References

- [OpenAI Docs: Environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
- [OpenAI Docs: Advanced configuration and profiles](https://learn.chatgpt.com/docs/config-file/config-advanced)
- [OpenAI Docs: Codex developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)

## Change summary

Documented the local design and repeatable procedure for sharing Codex history while maintaining separate normal OAuth and ShanghaiTech proxy credentials. No proxy runtime behavior changed.
