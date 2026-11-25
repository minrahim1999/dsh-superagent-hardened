# Local fork: what changed from upstream

Upstream: `deepseek-ai/dsh-superagent-prototype`, commit `8c32a9dc5d72c033816c6a8110b37f40cf94fba4`
(2026-10-31), MIT. The `origin` remote was removed on purpose: this checkout is a fork we edit, so
it does not follow upstream. To pull a later upstream state, add it back and rebase `local` onto it:

```sh
git remote add upstream https://github.com/deepseek-ai/dsh-superagent-prototype
git fetch upstream && git rebase upstream/main
```

Keep `LICENSE` and the upstream README's research-prototype notice.

## Patches in this fork

1. **Sessions run under a configurable permission preset instead of hardcoded `danger-full-access`.**
   - `src/deployment.ts`: new plugin config keys `sessionPermission` (default `workspace-write`, which
     every stock DSH profile defines) and `sessionApproval` (default `never`), carried on `Machine`.
     `stateDir` now also defaults to `<DSH_HOME>/state/superagent`, so a profile needs no configuration
     at all to start the bundle.
   - `src/app/sessions.ts`: `ensure()` applies `this.host.permission` rather than the literal
     `danger-full-access`, then overrides the approval knob with `ctx.approval.setPolicy(...)` so an
     unattended Session is never left waiting on a prompt; `SessionsHost` gains `permission`/`approval`.
     The `approval` service is now a module dependency (`@deepseek-ai/dsh-user-approval`).
   - `src/app/runtime.ts`: passes `machine.sessionPermission` into `Sessions`.
   - `src/app/setup.ts` + a new `processEnvironment` config key: once setup succeeds, `shims` (our
     default) only puts `<stateDir>/worker-bin` first on this DSH process's `PATH` (the agent `gh` and the
     blocked commands) and leaves git configuration alone; `shims-and-git` is the upstream behaviour that
     also rewrites `GIT_CONFIG_GLOBAL`; `none` touches the process environment at all. The git-config
     verification runs only in `shims-and-git`, so `gitTooOld` can no longer arise in the other modes.
   - `src/index.ts`: validates the named preset with `ctx.permissionPresets.resolve(...)` at startup and
     fails the entry with a message naming the preset if the profile does not define it.
   - Workers then write only inside their ticket workspace instead of the operator's whole OS account.
   - Escape hatch: if a ticket legitimately needs broader write access, set `sessionPermission` to
     `danger-full-access` in the plugin configuration.

2. **Safer default blocked commands.** `src/policy.ts` exports `DEFAULT_BLOCKED_COMMANDS`
   (`sudo`, `doas`, `su`, `osascript`, `open`, `security`, `defaults`, `launchctl`, `diskutil`, `shutdown`,
   `reboot`, `halt`, `ssh`, `scp`, `sftp`, `nc`, `telnet`) and `ExecutorConfigSchema.blockedCommands`
   defaults to it. They become failing shims in `<stateDir>/worker-bin`. Edit them in the panel's
   **Execution** settings when a ticket needs one of them.

3. **Tests updated for both defaults** (`tests/policy.spec.ts`, `tests/deployment.spec.ts`,
   `tests/fixtures/deployment.ts`).

## Deliberately NOT changed

- Worker Sessions still inherit this process's environment: `SessionCreateRequest` has no env field and
  `ctx.shellEnv` only carries `DSH_*` facts, so there is no per-Session environment. That is why the
  scope is a config choice rather than something this fork can remove outright.
- With the default `processEnvironment: shims`, worker `git` therefore uses the operator's own git
  configuration and credentials, while `gh` still resolves through the agent shim and the blocked
  commands still refuse. Set `shims-and-git` if the agent must use a separate git identity.
- The built-in Delivery prompt still describes auto-merge; it only activates when the Prompts settings
  name a merge CLI, and the default names none. Leave that section without a merge CLI.
- The management API stays on the profile's loopback Web server (`/orchestrator/api`, same-origin plus
  the `x-orchestrator-request` header). Keep the profile bound to `127.0.0.1`.
