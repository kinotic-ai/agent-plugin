# Plugin version: update check and project stamp

Every Kinotic skill and command runs this check once per session, before its first other
step, and reads a project's stamp (step 3 and step 5) again whenever it starts work in another
Kinotic project in the same session. It stamps a project's `.config/` with the plugin version whenever it changes that
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
3. **Project stamp.** When the working directory is a Kinotic project with a
   `.config/agent-plugin.json` (below), read its `version`.
4. **When the latest version is newer than the installed one**, ask the user, before doing
   anything else, whether they want to update the plugin, naming both versions. If they say
   yes, run their host's update commands yourself:
   - **Claude Code:** `claude plugin marketplace update kinotic`, then
     `claude plugin update kinotic@kinotic`. The new version loads after the user runs
     `/reload-plugins` or starts a new session; ask them to do that, then continue the task.
   - **Claude desktop app:** run the Claude Code commands above if the shell has the
     `claude` CLI. Otherwise ask the user to update the Kinotic plugin from the app's plugin
     settings and start a new conversation.
   - **Codex:** `codex plugin marketplace upgrade kinotic`, then
     `codex plugin add kinotic@kinotic`. The new version loads in a new session; ask the user
     to start one and continue the task there.
   - **Grok Build:** `grok plugin update kinotic`. The new version loads in a new session;
     ask the user to start one and continue the task there.

   If a command fails or the shell cannot run it, give the user the commands to run
   instead. If they decline the update, carry on with the installed version and do not ask
   again in this session.
5. **When the project stamp is newer than the version you will work with** (the installed
   one, unless an update from step 4 is now loaded), and the update step 4 offered, if any,
   would not have brought the stamp's version or was declined, tell the user that a newer
   plugin version (the stamp's) last changed this project than the one installed, and ask
   whether to go on with the installed version.
6. **Otherwise**, including when the installed version cannot be read, carry on without
   mentioning the check.

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
(step 1 above) to that file, creating it if it is missing. When the change is committed,
the stamp goes in the same commit. Never lower the stamp: when it already holds a newer
version than the installed plugin, leave it as it is (the check above has already told the
user).
