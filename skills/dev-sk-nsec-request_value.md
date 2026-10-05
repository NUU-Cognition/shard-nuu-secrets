---
description: "Ask the operator for a missing value or key the right way: the variable, the secret name, the environment, and the exact command for the operator's own terminal; never a value in the chat"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start nsec` if you haven't already.

# Skill: Request a Value or a Key

Ask the operator to write a missing value, or to make or add a key. The operator runs the command in their own terminal. The value never goes through you.

# Input

- The source of the request: the `✘ <n> variable(s) have no value in env <env>` error of `secret`, a `fail` line of `secrets doctor`, or a key problem
- The project: its name and the path of its `secrets.toml`
- The reason: what the value or the key is for

# Actions

1. **Collect the facts.** For each missing value, get these public facts from the error or from `secrets doctor`:
   - the variable (for example `CLERK_SECRET_KEY`);
   - the secret name (for example `clerk/secret-key`, from `secret://clerk/secret-key`);
   - the environment (`dev`, `preview`, `prod`, or `deploy`);
   - the store URL: the `store` field of `[project]` in `secrets.toml`;
   - the local store name: the `NAME` of the `secrets store list` row whose `URL` is that store URL. The missing-value error also names the store (`… of store <store>`). The `secret:` line names a revision, not a store.
2. **Remove what is not necessary.**
   - `preview` reads the dev value when it has no own value. Request a preview value only when it must be different from dev.
   - Request `prod` or `deploy` values only when the task needs them.
   - A platform credential is in the environment `deploy`: `vercel/token`, `vercel/team-id`, `convex/<deployment>/<env>` (a typed deploy key), or `railway/token`. Use the table of [[dev-sk-nsec-plan_push]] (step 7).
   - Request each secret name one time for each environment, also when two variables use it.
3. **Write the request for a value.** Use this shape. Put one command for each value:

   ```text
   NUU Secrets needs values that only you can enter.
   Project: <project> (<path of secrets.toml>)
   Why: <one sentence: what the task needs>

   Missing values:
   - <VAR> → <name>, env <env>
   - (one line for each value)

   In your own terminal, in the project folder, run each command and type the value at the hidden prompt:
     secrets set <name> --env <env>

   Do not send me the values. Tell me when each command reports "set" or "unchanged".
   ```

   - Outside the project folder, the operator adds `--store <local store name>` or `--mapping <path>` to the command.
   - A successful `secrets set` prints `set  <name>  env <env>  store <store> @ <revision>`, or `unchanged  …` when the store already has that value. Both exit 0. Neither shows a value.
4. **Write the request for a key.** Use this shape when `secret` stops with `✘ No usable age key: …`, or when `secrets doctor` says `no age key file at <path>` or `this machine is not a recipient of the store`. Do not use it for `this machine as a recipient: unknown (…)`: that line is not proof of a missing key (see [[dev-sk-nsec-diagnose]]). First collect the public facts yourself:
   - When doctor reports a missing key file and no other key source, use step 1a of the request.
   - When the file exists but cannot be used, use step 1c. Do not ask the operator to replace it with `secrets keygen`.
   - A public-key line from doctor can use another SOPS key source. It does not prove that the CLI key file exists.
   - Run `secrets whoami` to get the id. Run `secrets keygen --public` to get the full public key of the CLI key file. Use step 1b only when that query succeeds.
   - If the query fails while doctor reports another key source, use step 1c. Do not request a new key or infer missing store access from whoami.
   - `keygen --public` prints only the public key. With more identities, stdout has the first one and stderr names the count. Never open the key file, use `--file`, read `SOPS_AGE_KEY`, or run `SOPS_AGE_KEY_CMD`.
   - The store URL and the local store name (step 1).

   ```text
   NUU Secrets cannot open the store <store URL> on this machine.
   Observation: <the ✘ line of secret, or the fail line of secrets doctor>

   1a. Doctor reports a missing key file and no other key source. In your own terminal, run:
         secrets keygen
       It prints the id and the public key. It never prints the private key or replaces an existing key file.
   1b. The public-key query succeeded. Do not make a new key. The public facts of this machine:
         id:         <the id from secrets whoami>
         public key: <the line from secrets keygen --public>
   1c. The CLI cannot read a public key from its selected key file. In your own terminal, select your existing age identity file:
         export SOPS_AGE_KEY_FILE="<path of your existing age identity file>"
         secrets keygen --public
       Keep the private key out of the chat. Tell me which key-file setting the agent session must use. Do not continue until the public-key query succeeds in that session.
   2. Send the id and the public key to a store admin. Never send the key file.
      The admin compares the fingerprint (the last 8 characters of the public key) with you through a second channel, for example a call.
   3. The admin confirms the store URL <store URL>. Then the admin runs, in a project folder of this store (or with --mapping <path> or --store <the admin's local store name>):
         secrets recipient add "<id>" <age-public-key> --groups <groups>
   4. Then, on this machine (I can run this step):
      - first: secrets doctor --store <local store name> (omit --store when no local store exists)
      - keep the lines about local changes, unpushed commits, and links;
      - when `secrets store list` has no row with the URL <store URL>: secrets store clone <store URL>
      - else: secrets store sync <local store name>

   Do not send me the key file or the private key.
   ```

   - Give the groups that the task needs: `dev` for development values, `prod` and `deploy` only when the operator decides.
   - The key file is `$SOPS_AGE_KEY_FILE`, else `~/.config/sops/age/keys.txt`. The `doctor` Key section shows the path.
   - When the key is in another file, the operator can set `SOPS_AGE_KEY_FILE` to that file in place of step 1a.
   - After the admin adds the key, run step 4 yourself. Its plain doctor must run before clone or sync. Then run `secrets doctor --online` with the same mapping and store options.
5. **Send the request.**
   - In an interactive session, write it in your reply and wait for the operator.
   - In a headless session, follow [[dev-hinit-nsec]]: put the request in your result as a blocker. Do not guess and do not wait in a loop.
6. **Never accept a value.** Never ask the operator to paste a value into the chat, a file, a message, or a command. If the operator pastes a value anyway:
   - do not repeat it and do not use it;
   - tell the operator that the value is now in the session transcript;
   - follow [[dev-knw-nsec-rules]] § If a value leaks (the operator replaces the value at its provider).
7. **Check the result.** After the operator acts, run plain `secrets doctor` with the task's mapping, store, and environment options. Keep the lines about local changes, unpushed commits, and links. Then run `secrets doctor --online` with the same options, or run `secret <cmd>` again. The cached clone can be older than a remote change. A successful write on this machine already updates its clone. The `fail` lines for these names must be gone.

# Output

- One request to the operator with each missing value or key, the reason, and the exact commands
- No value in the request, in your context, or in a file
- After the operator acts: a `secrets doctor` result or a `secret <cmd>` run without the missing values
