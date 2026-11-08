---
title: "SOLID Java, revisited"
excerpt: "Eight of the ten snippets in my 2019 SOLID Java post don't compile. What was wrong, what the corrected code looks like on current Java, and the Claude Code skill that came out of it."
permalink: /solid-java-revisited/
categories:
  - Programming
tags:
  - java
  - claude-code
  - llm
header:
  overlay_image: /assets/images/solid-java-revisited.jpg
  overlay_filter: rgba(0, 0, 0, 0.5)
  caption: "Photo: [Simone Girlanda](https://www.flickr.com/photos/simonegirlanda/8150626452/), [CC BY-NC-ND 2.0](https://creativecommons.org/licenses/by-nc-nd/2.0/)"
toc: true
toc_sticky: true
---

In 2019 I wrote [SOLID Java](/solid-java/), a quick reminder of the five
SOLID principles for Java developers who already knew them. Each principle
had a BAD and a GOOD snippet, a list of code smells and a list of benefits.

This month I wanted a SOLID skill for
[java-toolkit](https://github.com/filippobuletto/java-toolkit), my Claude
Code plugin for Java projects, and I asked Claude Code to start from that
post. Before reusing any snippet, it compiled each one with javac 21 and
25, stubbing only the types the post never defines (`User`, `Engine`,
`Connection` and a few others). Eight of the ten didn't compile.

## What didn't compile

Two of the failures come from placeholder bodies: the BAD SRP
`checkAccess` has a comment where its `return` should be, and the BAD DIP
`search` calls an undefined `query`. The other six are real errors:

| Principle | Snippet | What javac says |
|-----------|---------|-----------------|
| SRP | GOOD `UserSettingService` | `cannot find symbol`: `securityService` is never declared |
| OCP | GOOD `ConsoleLog`, `FileLog` | `log() in ConsoleLog cannot implement log() in Log`, `attempting to assign weaker access privileges; was public` |
| LSP | BAD `public boolean function add(...)` | `';' expected`: `function` is not a Java keyword |
| LSP | GOOD `DataHashSet implements Set` | `DataHashSet is not abstract and does not override abstract method clear() in Set` |
| ISP | `Driver`, `Mechanic` | `cannot find symbol`: the constructors drop `car`, then the methods call `this.car` |
| DIP | GOOD `MySqlConnector implements Connector {}` | `MySqlConnector is not abstract and does not override abstract method open() in Connector` |
| DIP | GOOD `Database` | `cannot find symbol`: `connector` and `finder` are never declared |

JDK 21 and 25 gave the same errors. The GOOD `DataHashSet` implements 2 of
the 13 abstract methods of `Set`. The empty classes in the DIP example
were meant as placeholders, but they don't compile either. Only the BAD
OCP `Logger` and the GOOD ISP interfaces compile, and neither has a
method body with real logic.

## What was wrong beyond the syntax

Some examples showed a different problem from the one in their heading,
and some definitions were loose.

### The LSP example is about inheritance, not substitution

With `function` removed, the BAD `DataHashSet` compiles and runs:

```java
DataHashSet s = new DataHashSet();
s.addAll(List.of("x", "y", "z"));
// addCount=6 size=3
```

The comment in the post already said the count doubles. The reason is
that `HashSet.addAll` calls `add`, which the subclass also overrides. That
is the fragile base class problem from Effective Java (3rd edition, Item
18, "Favor composition over inheritance"). A caller holding this object as
a `Set` still gets correct set behavior, so nothing fails to substitute.

A real substitution failure is a subclass that breaks a contract its
callers rely on. `equals` in a class hierarchy is the classic case:

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, "red");
p.equals(cp);  // true
cp.equals(p);  // false
```

`ColorPoint extends Point` adds a field and overrides `equals`, and
symmetry is gone. Collections that contain both kinds behave differently
depending on argument order. On current Java the fix is composition with
records, which are final:

```java
record Point(int x, int y) {}

record ColorPoint(Point point, String color) {}
```

The counting set still deserves a fix, and composition is the answer.
Extending `AbstractSet` makes it short, because `AbstractSet`
inherits from `AbstractCollection` an `addAll` documented to call `add`
once per element:

```java
final class CountingSet<E> extends AbstractSet<E> {
    private final Set<E> delegate;
    private int addCount;

    CountingSet(Set<E> delegate) {
        this.delegate = Objects.requireNonNull(delegate);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return delegate.add(e);
    }

    @Override
    public boolean contains(Object o) {
        return delegate.contains(o);
    }

    @Override
    public Iterator<E> iterator() {
        return delegate.iterator();
    }

    @Override
    public int size() {
        return delegate.size();
    }

    int addCount() {
        return addCount;
    }
}
```

The same `addAll` of three elements now gives `addCount=3 size=3`.

### Mislabeled smells and loose definitions

- SRP: the post said a class implementing feature 1 and feature 2
  violates SRP. Robert C. Martin's
  [2014 article](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)
  ties a reason to change to people: changes should "originate from a
  single person, or rather, a single tightly coupled group of people".
  Two features owned by one group can share a class.
- SRP: the GOOD version moved `checkAccess` into a `static` method. A
  static call can't be replaced or mocked, which contradicts the post's
  own benefit, "can be now tested separately".
- OCP: one listed smell was a class that references another class
  directly instead of receiving it. That is a DIP smell.
- OCP: the `Log` interface has a `log()` method with no parameter, so
  no implementation can log a message.
- LSP: the post called LSP "an extension of open/close principle". LSP
  stands on its own (Liskov, 1987). OCP through subtyping depends on it.
- DIP: the BAD example reads `MSSQLConnection` and the GOOD one has a
  `MySqlConnector`, two different databases. The `Connector` and `Finder`
  interfaces copy the low-level operations, and `Database` only forwards
  calls. In DIP the high-level module owns the interface and names it
  after what it needs.

## What Java added since 2019

The post targeted Java 8. Several features since then change how some of
the principles look in code.

| Feature | Final in | Effect on SOLID code |
|---------|----------|----------------------|
| Records | 16 | Final value types, so no subclass can break `equals` (LSP) |
| Sealed classes | 17 | A closed set of variants the compiler knows (OCP, LSP) |
| `InstantSource` | 17 | The JDK applying ISP to `Clock` |
| Pattern matching for `switch` | 21 | Exhaustive `switch` over a sealed hierarchy (OCP) |

When new variants arrive often, an interface with implementations still
fits the open/closed principle. When the variants are fixed and new
operations arrive often, a sealed interface with an exhaustive `switch`
fits better:

```java
sealed interface Shape permits Circle, Rect {}

record Circle(double radius) implements Shape {}

record Rect(double width, double height) implements Shape {}

static double area(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Rect(double w, double h) -> w * h;
    };
}
```

There is no `default` branch. When a new record joins the `permits` list,
javac rejects every `switch` that misses it.

`InstantSource` is a small JDK example of interface segregation. Code that
only needs the current instant can take an `InstantSource` instead of a
`Clock`, which also carries a time zone. Tests pass a fixed source.

## When not to apply them

The 2019 post listed smells and benefits but never said when to stop.
Applied everywhere, SOLID leaves an interface beside every class and a
class for every method. The skill has rules for that:

- SRP: a long class with one owner is cohesive. Splitting it scatters
  each change across files.
- OCP: one implementation and no second in sight needs no interface.
  Inside one code base, extracting it when the second variant arrives is
  cheap. Published libraries are the exception, because their callers
  can't edit the code.
- LSP: make classes `final` unless they are designed for extension.
- ISP: split interfaces by client group, not by method.
- DIP: depending on `String`, collections or `java.time` values is
  fine. Invert at boundaries: persistence, network, clock, file system.

## The java-solid skill

`java-solid` shipped in java-toolkit
[v0.1.5](https://github.com/filippobuletto/java-toolkit/releases/tag/v0.1.5).
It explains a principle, writes code that follows it, or reviews a code
base:

```text
/java-toolkit:java-solid review src/main/java
```

A review starts from the files that changed most in the last year of
`git log`, because that is where SRP and OCP problems cost the most. Then
it runs a few `rg` searches for candidate sites: catch-all `*Manager` and
`*Util` classes, `instanceof` chains, overrides that throw
`UnsupportedOperationException`, and policy code calling `new` on
repositories and clients or reading `Instant.now()` and `System.getenv`.
A hit is a candidate, not a finding. Each reported finding names the
principle, the evidence (who changes the class, which edit repeats, which
client breaks) and a fix that compiles on the project's Java release. When
a fix costs more than it saves, the report says to leave the code as it
is.

Each Java example in the skill carries a marker with its minimum JDK, such as
`<!-- compile: jdk=21 fails-below -->`. The repository's test script
compiles each marked snippet with that JDK. With `fails-below` it also
checks that the previous release rejects the snippet: the sealed `Shape`
must compile with `--release 21` and fail with `--release 20`.

To try it, add the marketplace and install the plugin:

```text
/plugin marketplace add filippobuletto/claude-plugins
/plugin install java-toolkit@filippobuletto
```

The original post stays online as it was, with a note pointing here. If
the skill gives you a wrong or useless finding, open an issue on
[java-toolkit](https://github.com/filippobuletto/java-toolkit/issues).
