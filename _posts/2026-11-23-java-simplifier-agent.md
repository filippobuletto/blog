---
title: "A code simplifier agent for java-toolkit"
excerpt: "java-toolkit 0.3.0 ships its first Claude Code subagent: a Java simplifier that loads the SOLID, modernize and secure-coding skills, checks every edit with the project's build and tests, and was tested with evals in a sandbox that can't see my home directory."
permalink: /java-simplifier-agent/
categories:
  - Programming
tags:
  - java
  - claude-code
  - llm
header:
  overlay_image: /assets/images/java-simplifier-agent.jpg
  overlay_filter: rgba(0, 0, 0, 0.5)
  caption: "Photo: [Simone Girlanda](https://www.flickr.com/photos/simonegirlanda/8146061432/), [CC BY-NC-ND 2.0](https://creativecommons.org/licenses/by-nc-nd/2.0/)"
toc: true
toc_sticky: true
---

[java-toolkit](https://github.com/filippobuletto/java-toolkit), my Claude
Code plugin for Java projects, has been a set of skills since the
[first post](/claude-code-plugins/). Version 0.3.0 adds its first
subagent, `java-simplifier`. It cleans up Java code I've just written and
leaves its behavior alone.

I started from a generic code simplifier prompt written for TypeScript and
React projects. Its principles carry over to any language: never change
what the code does, prefer readable code to short code, avoid nested
ternaries, keep the abstractions that help. Its project rules were about
ES modules, the `function` keyword and React `Props` types, so I rewrote
those for Java.

## Why an agent

The plugin already had the rules in three skills. `java-solid` knows when
an interface or a class split pays for itself and when it doesn't.
`java-modernize` knows which language features each JDK release allows
and when not to convert. `java-secure-coding` knows which defensive copies
and checks a cleanup must not remove. A simplifier that ignored them would
undo their work.

A subagent can preload skills with the `skills` field in its frontmatter,
so all three are in its context from the first turn:

```yaml
---
name: java-simplifier
model: opus
skills:
  - java-solid
  - java-modernize
  - java-secure-coding
---
```

Bare skill names work for a plugin agent. The debug log showed
`Preloaded skill 'java-solid'` and the other two, and the three files add
about 15 KB, roughly 4,000 tokens, to each run. The agent also runs in
its own context, so reading every changed file and running several builds
doesn't fill up my main session.

## What it does

The agent works through eight steps in order:

1. Scope: the Java files changed since `HEAD`, staged, unstaged or
   untracked, or the paths I name. With neither, it stops instead of
   sweeping the repository. Test sources are read-only, because they are
   the proof that behavior didn't change.
2. Gate: it reads the project's release with the plugin's `detect.sh`,
   and every edit must compile on it.
3. Mode: plain simplification, or simplification plus modernization when
   the request asks for it.
4. Baseline: it builds and runs the tests first with the project's
   wrapper. A red build stops it before the first edit.
5. Snapshot: it copies each file into a temporary directory before
   editing it.
6. Simplify: it flattens nesting, replaces nested ternaries, removes dead
   private code, wrappers that only delegate, and comments that repeat the
   code.
7. Verify: it builds and tests again. If something breaks, it restores
   only its own edits to the failing files from the snapshot.
8. Report: what changed and why, what it reverted, and what it chose not
   to do.

The snapshot step exists because the code the agent edits is usually
uncommitted. Restoring with `git checkout` or `git stash` would throw away
my work along with its mistakes, so the agent never uses them, and it
never commits.

Records, pattern `switch` and text blocks are optional, as they are in
`java-modernize`. A plain "simplify" leaves them out and lists the places
where they would fit. Changes to public signatures and serialized forms
also go in the report as candidates, because only I can approve them.

## Calling it

It runs only when I ask for it by name:

```text
Use the java-simplifier agent to simplify my recent changes
Use the java-simplifier agent to simplify and modernize src/main/java/com/example/orders
```

The request has to name the agent. In my first eval the prompt only said
"simplify my recent changes", and the main session did the work itself
without calling the agent. It got most of it right but kept a wrapper
class the agent removes. Adding the name to the prompt fixed that.

## Testing it with evals

The plugin has a small Maven project for this, built on JDK 11. It
contains a nested ternary, an unused private method, a class that only
forwards calls to another one, an `Order` that copies its item list on
the way in and on the way out, and a `Money` class shaped like a record.
Eleven JUnit tests pin the behavior, including those two copies.

Three cases run through `claude plugin eval`:

| Case | JDK | Request | Expected |
| :- | :- | :- | :- |
| `java-simplifier-preserves` | 11 | simplify | clutter gone, copies kept, no record |
| `java-simplifier-no-modernize` | 17 | simplify | `Money` stays a class |
| `java-simplifier-modernize` | 17 | simplify and modernize | `Money` becomes a record |

Most graders are regular expressions on the files after the run, plus a
check that the Agent tool was called with `java-toolkit:java-simplifier`.
The prompt also asks for `./mvnw -q verify` at the end, with the exit code
written to a file that a grader reads. Every case also starts with an
untracked test file of mine in the workspace, and a grader checks it is
still there afterwards.

Getting a Maven build to run inside an eval took most of the effort. When
a case grants Bash, every command runs in Claude Code's sandbox, and the
sandbox can't read my home directory. My JDKs live in `~/.sdkman` and my
Maven repository in `~/.m2`, so neither was reachable. The scaffold
script runs as me, outside the sandbox, so it does the preparation:

- it copies the JDK and Maven into the workspace with `cp -c`, which on
  APFS makes copy-on-write clones that take no extra disk, so the whole
  scaffold runs in about three seconds;
- it fills a Maven repository inside the workspace, using `~/.m2` as a
  `file://` mirror;
- it writes a small `./mvnw` that runs that Maven offline with that JDK.

The mirror needed one fix. Maven records which repository each cached
artifact came from, and an offline build refused artifacts recorded under
my mirror's name. Naming the mirror `central` solved it.

Git was the other surprise. Inside the eval, `git` resolved to the Xcode
shim in `/usr/bin`, which needs to write a cache file the sandbox doesn't
allow, so every git call failed. The agent noticed, said so in its report
and worked on the paths in the request instead. That fallback is fine,
but the eval never exercises the "changed since `HEAD`" scope. I checked
that command by hand, and a Linux container with git installed would let
the eval cover it.

With the agent named in the prompt, all three cases scored 1.00, in one
run each without the no-plugin baseline, for $1.26 in total. In the JDK 11
case the agent replaced the ternary with early returns, removed the unused
method and the forwarding class, kept both list copies with the comment
explaining them, and left `Money` alone because records need JDK 16. Its
report said the code had no modernization candidates on that release
anyway.

## Try it

Install the plugin from my marketplace, or update it to 0.3.0:

```text
/plugin marketplace add filippobuletto/claude-plugins
/plugin install java-toolkit@filippobuletto
```

The [v0.3.0 release](https://github.com/filippobuletto/java-toolkit/releases/tag/v0.3.0)
lists the changes. If the agent changes behavior it shouldn't, or keeps
code it should remove, open an
[issue](https://github.com/filippobuletto/java-toolkit/issues) with the
diff.
