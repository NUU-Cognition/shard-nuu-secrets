---
description: "Run a program with the values of the store in its process environment (secret <cmd>), read the secret: line and the exit status, and act on a failure"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Run an App with Secrets

Run a program (a dev server, a test, a script) with the variables of its project from the store. The values go only into the process environment of the program. The wrapper writes no file with a plaintext value. A refresh or a first clone of the store changes the Git files and the index of the store clone.

# Input

- The program and its arguments, for example `pnpm dev` or `node server.js`
- The project folder (it has `secrets.toml`, or a parent folder has it)
- (Optional) The environment. The default is `dev`.
- (Optional) Network state: offline or online

# Actions

1. **Check the rules.** You read [[dev-knw-nsec-rules]]. The program must not print its process environment or a value. Do not start it with debug or dump options.
2. **Go to the app folder.** Run the command in the folder of the app, or in a child folder of the `secrets.toml`. When you cannot, add `--mapping <path>` or `--project <name>` (not both).
3. **Select the environment.**
   - Use the default `dev` for development and tests.
   - Use `--env preview` only when the task needs preview values.
   - Do not use `--env prod` or `--env deploy` unless the operator asks for it. These are production values and platform credentials.
4. **Build the command.** Put the options of `secret` before the program:
   ```sh
   secret pnpm dev
   secret --env preview node server.js
   secret --offline pnpm test
   secrets run --env preview -- node server.js
   ```
   - Use `--offline` when there is no network, or when the task forbids network access. `--offline` uses the cached clone, names its revision on stderr, and makes no network call. When the store has no cached clone, it stops with `✘ --offline: the store of <project> (<url>) has no cached clone on this machine. Nothing was cloned.` (exit 1). Then:
     - when the task allows network access, run once without `--offline` (it clones the store), or run `secrets store clone <url>`;
     - else report a blocker.
   - Use `--keep-env` only when a shell variable must win over the store value, or must stay when an `optional` variable has no store value. Normally the store value wins, stderr names the replaced variable, and the shell value of an absent `optional` variable is removed.
   - `--` ends the options of `secret`. The first word that is not an option also ends them. Each word after that goes to the program without a change.
5. **Do not put `secret` in project files.** Do not add `secret` to the scripts of `package.json`, to a `Makefile`, or to CI files. The person or the agent adds `secret` in front of the command at run time. A project file with `secret` fails on a machine without NUU Secrets.
6. **Turborepo.** When the program is `turbo run …` and Turborepo uses strict env mode, Turborepo can drop the variables. Run `secret` in the app folder with the app command (for example `secret next dev`). If the task needs `turbo run`, ask the operator to add the variables to `passThroughEnv` or `globalPassThroughEnv` in `turbo.json`. Do not change `turbo.json` yourself unless the task says so.
7. **Long-running programs.** A dev server does not stop. Run it in the background of your harness. Stop it when the task is done. `secret` gives SIGTERM to the program and waits for it.
8. **Read the `secret:` line.** Before the program starts, stderr has one line:
   `secret: <project> · env <env> · store <revision> (<date>) · <n> variables`
   - Check the project, the environment, and the count.
   - `· <n> optional not set` means that the store has no value for `<n>` `optional` entries. The CLI does not give these variables. When your shell has a variable of the same name, the CLI removes it from the process environment of the program, and stderr says `secret: the store has no value for the optional <VAR>: the shell value is removed (--keep-env keeps it)`. With `--keep-env`, the shell value stays, and stderr says so.
   - `store not read (no ref entries)` means that the mapping has only literals for this target.
   - `secrets: cloning the store of <project> from <url>` before the line means a first clone (a network access). Check that the secrets home is the intended one ([[dev-knw-nsec-model]] § The store). Only the operator can remove a clone (`secrets store remove`).
9. **Check a variable without its value** when the task needs proof:
   ```sh
   secret sh -c 'test -n "$CLERK_SECRET_KEY" && echo "CLERK_SECRET_KEY is set"'
   ```
10. **Read the exit status.**

    | Status | Meaning | Action |
    |---|---|---|
    | `0` | The program succeeded. | Continue. |
    | `1` with `✘ <n> variable(s) have no value in env <env>` | Values are missing. Nothing ran. | Use [[dev-sk-nsec-request_value]] for each listed name. |
    | `1` with `✘ There is no secrets.toml …` | No mapping here. | Go to the project folder, or use [[dev-sk-nsec-setup_project]]. |
    | `1` with `✘ <path> is not valid:` | A mapping problem. A `[targets.*]` table that is not valid also stops `secret <cmd>`. | Run `secrets doctor` and use [[dev-sk-nsec-diagnose]]. |
    | `1` with `✘ No usable age key: …` | The key file is missing, not a regular file, or cannot be read. Nothing ran. | Tell the operator the path. Do not make a key. Use [[dev-sk-nsec-request_value]] (the key request). |
    | `1` with an identity error: `✘ There is no identity named …`, `✘ The CLI cannot read the identity of project …`, or the errors `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT` ([[dev-knw-nsec-cli]] § Messages that agents see) | The CLI cannot use the identity of the project, and does not use the default in its place. Nothing ran. | Stop. Use [[dev-sk-nsec-diagnose]] (step 6). Do not add `--identity`, and do not run from another folder to pass the check. |
    | `0` (or the status of the program) with `secrets: warning: … has no identity of its own in projects.toml, so the CLI uses …` | The program ran with the default identity, not with the identity of the parent project. | After the run, stop and report the warning ([[dev-sk-nsec-diagnose]], step 6). |
    | `1` with `✘ sops decrypt failed …` | SOPS did not decrypt the file with the available key sources. This error does not prove that the selected key file is valid. | Use [[dev-sk-nsec-diagnose]]. Do not run `sops`. |
    | `1` with `✘ --offline: … has no cached clone …` | `--offline` and no clone. Nothing ran. | See step 4. |
    | `1` with `✘ The stores folder <path> is a symbolic link …`, `✘ The CLI cannot check the stores folder <path> (<code>). …`, `✘ Store <name> has a file that is not a regular file …`, or `✘ Store <name> has a file that the CLI cannot check …` | An unsafe store layout, or a file or folder that the CLI cannot check. Nothing ran. | Report the message to the operator. Do not change modes or folders in the secrets home. Use [[dev-sk-nsec-diagnose]]. |
    | `2` with `✘ No command to run.` | No program after the options. The program did not start. | Give the program. |
    | `1` with `error: …` | The option parser refused an option of `secret` (for example `error: unknown option '--bad'`). | Fix the option, or put it after the program name. |
    | `126` with `✘ Cannot start <cmd> (EACCES).`, `127` with `✘ The program "<cmd>" is not on PATH.` | The program is not executable, or not on `PATH`. | Fix the program name or the path. |
    | `128 + n` | Signal `n` stopped the program (143 is SIGTERM). | Normal when you stopped it. |
    | other | The exit status of the program. | Read the output of the program. |

    After the program starts, `secret` exits with the exit status of the program. That status can also be 1, 2, 3, 126, or 127. Thus read the `✘` or `error:` message, not only the status. The `secret:` line on stderr means that the CLI resolved the variables and then tried to start the program.

11. **If a value shows** in the output of the program, follow [[dev-knw-nsec-rules]] § If a value leaks.

# Output

- The program ran with the variables of its project, or a clear failure with its next step
- No value in your output, in a file, or in a command argument
