---
title: "Three Claude Code plugins and a marketplace to ship them"
excerpt: "How I built kubectl-guard, toon-mcp-squeeze and java-toolkit, and why they now ship from a single marketplace repository."
permalink: /claude-code-plugins/
categories:
  - Programming
tags:
  - claude-code
  - llm
  - kubernetes
  - mcp
  - java
toc: true
toc_sticky: true
---

I use Claude Code every day at work. For my personal projects I run models
locally instead. Over time, at work, I wrote a few hooks and scripts for
problems I kept running into. Until recently they lived in my local
configuration, which meant copying files around by hand on every
machine. Claude Code plugins solve that: a plugin bundles hooks, skills and
options, and a marketplace lists plugins so anyone can install them with
one command.

I packaged three of them as plugins and published a marketplace,
[filippobuletto/claude-plugins](https://github.com/filippobuletto/claude-plugins),
that lists all of them:

```text
/plugin marketplace add filippobuletto/claude-plugins
/plugin install kubectl-guard@filippobuletto
/plugin install toon-mcp-squeeze@filippobuletto
/plugin install java-toolkit@filippobuletto
```

## kubectl-guard: read-only Kubernetes for the agent

[kubectl-guard](https://github.com/filippobuletto/kubectl-Guard-for-Claude-Code)
started in October as a plain Bash script. I wanted Claude to inspect a
cluster (pods, logs, events, rollout status) without being able to change
it, read secrets or switch identity, even when it is wrong or
prompt-injected.

```text
$ kubectl get pods -n prod             allowed
$ kubectl -n prod delete pod web-0     ask: kubectl verb 'delete' is not on the allowlist
$ kubectl get secret db -o yaml        BLOCKED: reads secrets
```

It is a `PreToolUse` hook on the `Bash` tool. Before every call, the script
reads the command with `jq` and returns one of three outcomes: exit `0` lets
it run, a JSON answer with `permissionDecision: "ask"` shows me a
confirmation prompt, and exit `2` blocks it and tells the agent why.

The script works on an allowlist. `get`, `describe`, `logs`, `top`,
`events`, `rollout status`, `auth can-i` and a few others run freely, and
`apply` passes only with an explicit `--dry-run`. For Helm, the read-only
commands such as `list`, `status`, `template` and `lint` pass. Anything
else is a verb violation. Some rules always deny, whatever the mode:

- identity and connection flags such as `--as`, `--token`, `--context` and
  `--kubeconfig`;
- reading secrets through `get`, `describe` or `diff`;
- shell indirection (`$VAR`, `$(...)`, backticks, `eval`, `sh -c`) next to
  Kubernetes tools, because the hook can't know what they expand to;
- `KUBECONFIG=` and `KUBECTL_*=` overrides;
- `curl` or `wget` straight at the API server.

The command is split on `&&`, `||`, `;`, `|` and `&`, so a `kubectl delete`
hidden after a pipe or `xargs` gets checked too. The verb is the first
non-flag token, so `kubectl -n prod --context x delete` can't hide it
behind flags.

Packaging it as a plugin raised one design question. Verb violations can
either be denied or sent to me as a prompt, and that choice became the
`verb_violation_mode` option. The hook passes it to the script as an
argument, because the agent sets the environment of the commands it runs
and could flip an environment variable with `VERB_VIOLATION_MODE=ask
kubectl ...`. Any value other than `ask` fails closed to `deny`. In unattended runs such as `claude -p` or CI there is nobody to
answer a prompt, so `deny` is the right mode there.

The README calls the hook a convenience guard, because it only reads the
command string. A `make deploy` that calls `kubectl` is invisible to it, and so are other
Kubernetes clients. The real control is RBAC: give the agent its own
read-only identity, and let the hook catch mistakes before they reach the
API server.

## toon-mcp-squeeze: fewer tokens from MCP tools

MCP servers usually return JSON, and a list of pods or issues in JSON
repeats every key on every record. [TOON](https://toonformat.dev/) writes
uniform arrays as a table with the keys listed once.
[toon-mcp-squeeze](https://github.com/filippobuletto/toon-mcp-squeeze) is a
`PostToolUse` hook on `mcp__.*` that re-encodes those results before Claude
reads them:

```text
[MCP JSON output re-encoded as TOON; decode with `toon --decode` if JSON needed]
[3]{id,name,role}:
  1,alice,admin
  2,bob,dev
  3,carol,dev
```

The hook converts only text blocks holding a JSON object or array of at
least `min_bytes`. Images, plain text and small blocks pass through. Two
options, `include_tools` and `exclude_tools`, take regular expressions on
the tool name, so I can convert GitHub and Kubernetes tools but leave file
contents alone. Claude copies those back verbatim, and they should stay
JSON. If `toon` is missing, a regex is invalid or encoding fails, Claude
gets the original output.

Testing against
[mcp-server-kubernetes](https://github.com/Flux159/mcp-server-kubernetes)
on minikube turned up two problems.

`kubectl_logs` returns JSON that is mostly one long string. TOON saves
nothing there, and with the label line on top the result grew by 1.5 to
10%. The hook now replaces a block only when label plus TOON is smaller
than the original.

The second one hit the largest payloads. Claude Code saves MCP results
over 50,000 characters to a file in the session's `tool-results` directory and hands
the hook only a message naming that file, so the results where TOON saves
the most were never converted. Now the hook writes a `.toon` copy
next to the file and points the message at it, and the label names the
original JSON for `jq` queries. It only reads regular files inside the
current session's `tool-results` directory and rejects paths with `/../`
and symlinks.

With those fixes, all 17 tools the server exposes in read-only mode keep
working, and every converted result decodes to exactly the original JSON.
Pod lists shrink by 56 to 58%. A 123,289-byte `kubectl_get` events result
became a 46,665-byte `.toon` file, 62% smaller. A ConfigMap saves only 11%,
which matches what TOON promises: it pays off on uniform arrays and much
less on nested or irregular data.

The plugin also has a `/toon-mcp-squeeze:setup` skill. It checks for `jq`
and the `toon` CLI, warns when `toon` sits under a Node version manager
the hook may not see, helps pick which tools to convert, measures the
savings and verifies the hook.

## java-toolkit: upgrading Java one LTS at a time

The third plugin has no hooks. [java-toolkit](https://github.com/filippobuletto/java-toolkit)
is a set of skills that upgrade Java projects from JDK 8 to JDK 27:

```text
/java-toolkit:java-upgrade 21
```

`java-upgrade` is the entry point and routes the work to four other skills.
`java-assess` runs a read-only audit with a baseline build, `jdeps
--jdk-internals` and `jdeprscan --for-removal`. `java-build` handles Maven
and Gradle: `release` and toolchains, wrapper and plugin versions, and
libraries that block the build, such as Lombok, JaCoCo, Byte Buddy and
Error Prone. `java-migrate` fixes what each hop breaks (8 to 11, 11 to 17,
17 to 21, 21 to 25, 25 to 27). `java-modernize` adopts newer features like
records, pattern matching and virtual threads, and only if I ask for it.

I didn't want the model to guess the mechanical parts, so two scripts do
them. `detect.sh` reports the build tool, wrapper, Java release and where it
is set, plugin versions, the JDKs pinned in CI and Docker, and the JDKs
installed on the machine. `scan.sh` runs a catalog of patterns over
sources, build files and launch scripts and reports each hit with the JDK
where it matters and the fix:

```text
$ scan.sh . --from 8 --to 17 --summary
jaxb      removed  11  2  JEP 320: JAXB removed from JDK. Add jakarta.xml.bind-api + ...
cms-gc    flag     14  1  JEP 363: CMS removed in 14. Use G1 (default) or ZGC
...
```

The skills follow a few ground rules. They make no edits before I confirm
the plan, start from a green baseline and move one hop at a time. They
never weaken tests and never install JDKs for me. The plan goes in
`.java-upgrade/plan.md` and progress in `progress.md`, so an interrupted
upgrade can resume.

Language models easily get version facts wrong, so all of
them live in one file, `jdk-matrix.md`, checked against openjdk.org. It
also lists common wrong advice the skills must avoid, such as String
Templates (removed in 23), `-XX:+ZGenerational` (obsolete in 24) and
`StructuredTaskScope.ShutdownOnFailure` (gone in 25). A test script
compiles every marked code snippet in the references with the JDK it
claims. With a `fails-below` marker it also checks that the previous
release rejects the snippet, which proves the minimum JDK stated in the
text. Behavior evals run through `claude plugin eval`, and each case also
runs a baseline without the plugin.

## One marketplace for all of them

At first every plugin repository was also its own single-plugin
marketplace, so each one needed its own `/plugin marketplace add`. With
three plugins that got silly. I moved them all to one marketplace named
`filippobuletto`. It was a breaking change for the two plugins already
out: `kubectl-guard@kubectl-guard` and
`toon-mcp-squeeze@toon-mcp-squeeze` became
`kubectl-guard@filippobuletto` and `toon-mcp-squeeze@filippobuletto`.

The marketplace repository holds only the catalog,
`.claude-plugin/marketplace.json`. Each plugin keeps its own repository
with its README, options and releases. The entries use unpinned `github`
sources, so they track each plugin's default branch, and Claude Code picks
up a new release when the `version` in the plugin's `plugin.json` changes.

Claude Code can't read `plugin.json` from a remote source before
installing it, so each catalog entry repeats the plugin's description,
homepage, license and keywords, and those copies can drift. The
marketplace has a small test, `tests/run.sh`, that fetches each plugin's
`plugin.json` from GitHub and checks that name, description, homepage and
license match the entry. Together with `claude plugin validate .`, it runs
every time I add a plugin.

The move also broke the toon setup skill. It writes options under the
plugin's config key, which includes the marketplace name, so after the
rename it wrote settings that the installed plugin never read. The fix
was to point it at the `toon-mcp-squeeze@filippobuletto` key.
