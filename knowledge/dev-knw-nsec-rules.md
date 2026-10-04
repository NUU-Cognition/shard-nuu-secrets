---
description: "The complete agent rules for NUU Secrets: no value in output, files, or arguments; no access to the key or the store files; the operator-only commands; how to ask for a value; what to do after a leak"
---

# Knowledge: Agent Rules

These rules apply to each task that touches NUU Secrets: a project with a `secrets.toml`, a `secret <cmd>` run, a `.env` file, a key, or a store. Read all the rules before you run a `secrets` or `secret` command. When a rule and a task do not agree, the rule wins: stop and tell the operator.

## Words

This shard uses one word for one thing.

| Word | Meaning |
|---|---|
| value | The plaintext of a secret, for example an API key, a token, or a password. |
| secret name | The stable name of a value in the store, for example `clerk/secret-key`. A secret name is not secret. |
| variable | An environment variable of a process, for example `CLERK_SECRET_KEY`. A variable name is not secret. |
| environment | The store environment that `--env` selects: `dev`, `preview`, `prod`, or `deploy`. |
| store | The Git repository of the team that holds the encrypted values (SOPS + age). |
| store clone | The copy of the store on this machine, at `<secrets home>/stores/<store name>/`. |
| key file | The age private key of this machine: `$SOPS_AGE_KEY_FILE`, else `~/.config/sops/age/keys.txt`. |
| operator | The person who owns this machine and its store access. Only the operator enters values and manages keys. |

## Why the rules exist

You run as the operating-system user of the operator. Thus you have the access of that user. You can read the key file. You can decrypt each store file of the groups of this machine. You can read the process environment of each process of the user. NUU Secrets does not stop you. It protects the values at rest and in Git. It does not protect them from a process of the same user. The operator decided this: **an agent is a user.**

These rules prevent accidents. They are not a security boundary. Nothing else stops you, so you must keep them.

A value that you see goes into your context. Your context goes into the session transcript on disk and to the model provider. Thus a value in your context is a leak. The main rule: **a value never reaches your context, and a value never reaches a place that people or tools read.**

## Rule 1 — Never show a value

Never print, log, echo, or type a value. This applies to each of these places:

- A reply, a summary, a turn result, an Orbh message, an update, or a return.
- A commit message, a code comment, a source file, a test fixture, or a test snapshot.
- A Mesh note, a task, a report, or a notepad.
- A log line that you add to an app.
- A command that you type. A value in a command goes into the shell history and into the transcript.

Do not run a command that shows the process environment: `env`, `printenv`, `set`, `export -p`, `echo "$VAR"`, `cat /proc/<pid>/environ`, or a script that prints `process.env`. This rule also applies under `secret`.

To check that a variable has a value, test it and do not print it:

```sh
secret sh -c 'test -n "$CLERK_SECRET_KEY" && echo "CLERK_SECRET_KEY is set"'
```

Use single quotes. Then the inner shell reads the variable, and your shell does not expand it.

Use the safe signals first. The `secret:` line on stderr gives the count of variables. `secrets doctor` names each required secret name that has no value. Neither shows a value.

An app can show a value in its output: debug output, a configuration dump, or an error with a connection URL. Do not start an app with debug or dump options under `secret`. If a value shows, follow [If a value leaks](#if-a-value-leaks).

## Rule 2 — Never write a value to a file

- Do not write `.env`, `.env.local`, `.env.*.local`, or another file that holds values.
- Do not send the output of `secrets export` to a file (`>`, `>>`, or `tee`).
- Do not copy a value into a test fixture, a configuration file, a script, or a temporary file.
- Do not open or show the content of a `.env*` file. This also applies to an example file such as `.env.example`, also when Git tracks it. Any of these files can hold a value, and a file can be a symbolic link to another file. `secrets doctor` names the managed variables in the `.env*` files, and it shows names only. Get the variable names of an app from its source code ([[dev-sk-nsec-setup_project]], step 4).
- If a tool can read its configuration only from a file, stop. Tell the operator the tool and the file. The operator decides.

## Rule 3 — Never pass a value as a process argument

Other processes can read the arguments of a process. A program must read a value from its process environment or from stdin.

- Do not write a command in which a shell expands a value into the arguments of a program. For example, `secret sh -c 'some-cli "$API_TOKEN"'` gives the value to `some-cli` as an argument. Use a program that reads the variable itself.
- `secrets bind --value` is only for a literal that is not secret. Never give a value to `--value`. `bind` refuses some values that look like secrets, but its check does not find all of them.
- Do not give a value to a platform CLI (Vercel, Convex, Railway) as an argument.

## Rule 4 — Never touch the key or the store files

- Do not open, print, copy, move, or change the key file. To check the key, run `secrets doctor` (the path, the mode, and the fingerprint) or `secrets whoami` (the fingerprint and the path). The fingerprint is the last 8 characters of the public key.
- Do not decrypt a store file. Do not run `sops` on a store file. An error of `secret` can say "Run sops yourself to see it". That sentence is for the operator, not for you.
- Do not edit `*.enc.yaml`, `.sops.yaml`, `recipients.toml`, or `stores.toml` by hand.
- Do not run Git commands in a store clone, and do not delete a store clone. To refresh the store clones, run `secrets store sync`.

## Rule 5 — Operator-only commands

The operator enters values and manages keys. Do not run these commands. The one exception: the operator asks you to run that exact command now. (A test of the CLI in a sandbox has its own rule: see [Tests of the CLI itself](#tests-of-the-cli-itself).)

| Command | What it does |
|---|---|
| `secrets set` | Writes a value. The value comes from a hidden prompt or stdin. |
| `secrets edit` | Shows the values of one store file in an editor. For preview, it shows only the preview file. |
| `secrets unset` | Removes a value. |
| `secrets keygen` | Makes an age private key. |
| `secrets recipient add`, `secrets recipient rm` | Change who can open the store. |
| `secrets store init` | Creates a store. |
| `secrets store fix-config` | Writes `.sops.yaml` again after a hand edit. |
| `secrets login`, `secrets logout` | Change the NUU account of the operator. |

"The operator asks" means: the operator (or your dispatcher, for the operator) names this command for this action in this session. A general task, for example "set up the secrets of the app", does not give this permission.

For `secrets set` and `secrets edit`, the operator types the values in their own terminal. Give the operator the command. Do not run it: you would have to type the value. Use [[dev-sk-nsec-request_value]].

## Rule 6 — Commands that you can run

| Command | Use |
|---|---|
| `secret <cmd>`, `secrets run` | Run a program with the values in its process environment. See [[dev-sk-nsec-run_app]]. |
| `secrets doctor` | Check the tools, the key, the stores, the mapping, and the project. It changes nothing. See [[dev-sk-nsec-diagnose]]. |
| `secrets whoami` | Show the recipient id, the key fingerprint, and the stores of this machine. |
| `secrets store list` | List the store clones of this machine (no values). |
| `secrets store sync` | Refresh the store clones. |
| `secrets store clone` | Clone the store of a project. You seldom need it: `secret` clones the store when necessary. |
| `secrets init-project`, `secrets bind` | Write `secrets.toml`: names and targets, no values. See [[dev-sk-nsec-setup_project]]. |
| `secrets export` | Only under the conditions below. |
| `--help`, `--version` | Show the help or the version. |

### Conditions for `secrets export`

`secrets export` writes values to stdout. If you run it in your shell, the values go into your context.

- Use `secret <cmd>` when you can. It covers almost every case.
- Use `secrets export` only when a tool must read the values from stdin. Pipe the output directly into that tool, in the same command. That tool must not print what it reads.
- Never run `secrets export` without a pipe. Never pipe it into a program that shows its input, for example `cat`, `less`, `head`, `jq`, or `tee`.
- Do not use `eval "$(secrets export --format shell)"`. The values then stay in your shell. Use `secret <cmd>`.
- Do not use `secrets export --to vercel`, `--to convex`, or `--to railway` to send values to a platform. Each push of values to a platform needs the approval of the operator for that push.

## Rule 7 — Ask the operator for values and keys

When a value or a key is missing, ask the operator. Give the variable, the secret name, the environment, and the exact command for the operator. Never ask the operator to paste a value into the chat, a file, or a message. Never guess a value, and never use a placeholder value that looks real. Use [[dev-sk-nsec-request_value]].

## Rule 8 — Platform targets need the operator

A platform target (`vercel`, `convex`, `railway`) in `secrets.toml` sends values to a platform in a later version of the CLI. Do not add a `[targets.<target>]` table, and do not add a platform target to the `to` list of an entry, unless the operator or the task names that target.

## Tests of the CLI itself

This shard is for the use of NUU Secrets in projects. A task that tests the `secrets` CLI itself (not an app) needs a sandbox:

- In each command that runs `secrets` or `secret`, set `HOME`, `SECRETS_HOME`, `NUU_HOME`, `XDG_RUNTIME_DIR`, and `TMPDIR` to temporary folders. Set `SOPS_AGE_KEY_FILE` to a file path in the temporary root folder (for example `<temp root>/home/keys.txt`).
- Use test keys (`age-keygen`), a local bare Git repository as the store remote, and made-up test values.
- Never read or change the real key file or the real store clones.
- Delete the temporary folders at the end.

Only the prompt of such a task can allow the operator-only commands, and only inside the sandbox.

## An agent is a user

You have the access of your operating-system user. You can read the key file. You can decrypt each store file of the groups of this machine, also production values when this machine is in the group `prod`. You can read `/proc/<pid>/environ` of a process that `secret` started. The rules are a promise that you keep. They are not a lock.

- Do not test the rules.
- Do not read a value to prove a weakness.
- When you find a way for a value to leak, describe it to the operator without the value.

## If a value leaks

A leak is a value in your context, in output, in a file, in a commit, in a message, or in a log. Act at once:

1. Stop the current work. Do not run a command that can show the value again.
2. Do not repeat the value. Do not quote it in your report, also not a part of it.
3. Tell the operator now. Give the variable, the secret name, the environment, and where the value went: the command, the file path, the commit, the message, or the log. Say that the session transcript holds the value.
4. If you wrote the value to an untracked local file, delete that file and say so. If the value reached a commit, a push, a Mesh note, an Orbh message, or another shared place, do not try to remove it. The operator decides about a change of history.
5. The operator replaces the value at its provider (rotation) and writes the new value with `secrets set <name> --env <env>`. Until then, the old value is public.

In a headless session, report the leak as a blocker (a subagent to its dispatcher, a root session to the operator), then stop. See [[dev-hinit-nsec]].
