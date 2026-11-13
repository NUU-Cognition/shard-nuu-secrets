# Release notes

The notes of each version of the shard `nsec` (`@nuucognition/nuu-secrets`). The notes of the versions before 0.6.0 are in the Git history of this repository.

## 0.6.0

For the CLI of Mission 002 (`even-macbook`, not yet on canon): the CLI of canon `471e270` with the identities, the project index, and the local web page `secrets ui`. The npm version `0.1.0-beta.1` does not have these parts.

- **Rule 11: `secrets ui` is for the operator.** Agents never run `secrets ui` (also when the operator asks, and with no way around the terminal check), never open its URL, login link, page, or API, and never read values from it. A login link is a credential. The rule is in the rules, the init, and the headless init.
- **Rule 12: identities.** The agent uses the identity that the CLI selects. It does not add `--identity` or set `SECRETS_IDENTITY` or `SOPS_AGE_KEY_FILE` unless the operator names the identity. `identity add|use|rm`, `project identity <name>`, `project identity --clear`, and `project forget` are operator commands. `identity list`, `project list`, and `project identity` (show) are agent commands. On `LINKED_MAPPING`, `MAPPING_CHANGED`, `OUTSIDE_PROJECT`, `DUPLICATE_PROJECT`, `UNKNOWN_IDENTITY`, `PROJECT_INDEX`, the parent-identity warning, or the link warning, the agent stops and gives the operator the exact message and the remedy of the CLI. In a headless session, these are blockers.
- **The model:** the selection order of the key (`--identity`, `SECRETS_IDENTITY`, `SOPS_AGE_KEY_FILE`, the project identity, the default, the default key file), where the project identity applies, and the project index.
- **Request a value:** the operator sets a value with `secrets set` or in the page of `secrets ui`; then the agent continues with `secret <cmd>`. The key request covers the key file of an identity.
- **The setup guide:** the sections Identities (make the identity with the least access the default), The project index (`project list|forget`), and The UI: `secrets ui` (the start, the login link, the identity choice, the lock, the hidden values, the writes, Quit).
- **The CLI reference:** `--identity` on each command that decrypts or reads the key, `identity list`, `project list`, `project identity`, the operator commands `identity add|use|rm`, `project identity <name>`, `project forget`, and `secrets ui`, the `doctor` lines of the key selection and the project index, the identity messages, and the new known limits (items 23 to 27).
- **Diagnose:** the identity rows of the `doctor` table, the `Projects (projects.toml)` section, the states `missing`, `linked`, and `duplicate` of `project list`, and step 6 (stop on an identity problem). `run_app` has the identity errors in its exit table.
