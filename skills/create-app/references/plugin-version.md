# Plugin version: update check and project stamp

Every Kinotic skill and command runs this check once per session, before its first other
step, and stamps a project's `.config/` with the plugin version whenever it changes that
project's files.

## Check for a newer plugin

1. **Installed version.** Read `version` from this plugin's own manifest. The plugin's root
   is two directories above this skill's directory (`skills/<skill>/`); its manifest is
   `.claude-plugin/plugin.json` in Claude Code and the Claude desktop app,
   `.codex-plugin/plugin.json` in Codex and `.grok-plugin/plugin.json` in Grok Build. All
   three carry the same version.
2. **Latest version.** Fetch
   `https://raw.githubusercontent.com/kinotic-ai/agent-plugin/main/.claude-plugin/plugin.json`
   and read its `version`.
3. **Compare** the two as semantic versions. Also compare against the version stamped in
   the project's `.config/agent-plugin.json` (below), when the working directory is a
   Kinotic project that has one: a stamp newer than the installed plugin means a newer
   plugin already worked on this project.
4. **When a newer version exists**, ask the user, before doing anything else, whether they
   want to update the plugin, naming the installed and the newer version. If they say yes,
   run their host's update commands yourself:
   - **Claude Code:** `claude plugin marketplace update kinotic`, then
     `claude plugin update kinotic@kinotic`. The new version loads after the user runs
     `/reload-plugins` or starts a new session; ask them to do that, then continue the task.
   - **Codex:** `codex plugin marketplace upgrade kinotic`, then
     `codex plugin add kinotic@kinotic`. The new version loads in a new session.
   - **Grok Build:** `grok plugin update kinotic`. The new version loads in a new session.

   If a command fails or the shell cannot run it, give the user the commands to run
   instead. If they decline the update, carry on with the installed version and do not ask
   again in this session.
5. **When no newer version exists**, or either version cannot be read (no network, no
   manifest), carry on without mentioning the check.

Never edit the installed plugin's files; update it only through the host's commands above.

## Stamp the project

A Kinotic project records the plugin version that last changed it in
`.config/agent-plugin.json` at the repository root:

```json
{
    "version": "0.7.8"
}
```

Whenever you change a Kinotic project's files, write the installed plugin version
(step 1 above) to that file, creating it if it is missing, and commit it with the change.
Never lower the stamp: when it already holds a newer version than the installed plugin,
leave it as it is (the check above has already offered the user the update).
