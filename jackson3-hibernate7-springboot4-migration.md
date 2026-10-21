---
layout: default
title: "Surviving a Big-Bang Java Upgrade: Jackson 2->3, Hibernate 6->7, and Spring Boot 3->4"
---
<!--
Publishing metadata (do NOT paste into the article body; set these separately in the Medium editor)
Title: Surviving a Big-Bang Java Upgrade: Jackson 2→3, Hibernate 6→7, and Spring Boot 3→4
Subtitle: The breaking changes that compile clean and only explode at runtime — two coexisting JsonNode classes, a silently wrong Hibernate mapper, phantom bytecode errors, and how to survive them
Tags (5): Java, Spring Boot, Hibernate, Jackson, Backend Development
Featured image: 1500×840, e.g. the "who drags whom" dependency diagram
-->

# Surviving a Big-Bang Java Upgrade: Jackson 2→3, Hibernate 6→7, and Spring Boot 3→4

> Upgrading a mid-sized Java backend service from **Spring Boot 3 to 4** quietly drags in two more major jumps: **Jackson 2→3** and **Hibernate 6→7**. This article follows that real upgrade and dissects the breaking changes that *compile clean and only blow up at runtime* — both the ones that actually cost us days and a pile of industry classics you'll likely meet too. Everything here is generic Java-stack territory; anyone doing a similar upgrade will hit it.

The build is green. Every unit test passes. CI is a wall of checkmarks. You confidently merge the big upgrade and ship it.

Then, on an unremarkable afternoon, a perfectly ordinary request makes the service throw an exception you've never seen — pointing at a line of code you never even touched in this upgrade.

That's the particular cruelty of a "major version + multiple dependencies at once" upgrade: the real traps almost never live at compile time, they live at runtime — and the error message often sends you in exactly the wrong direction. What follows pulls those traps apart one by one — from two `JsonNode` classes that look identical but aren't the same class, to a JSON mapper Hibernate silently picks wrong, to a phantom "broken bytecode" compilation error — and ends with a repeatable upgrade playbook.

---

## Table of Contents

1. [Why "major version + many dependencies" is the hard kind](#why)
2. [Act I: Jackson 2 → 3](#jackson)
3. [Act II: Hibernate 6 → 7](#hibernate)
4. [Act III: Spring Boot 3 → 4](#springboot)
5. [Act IV: the ripple effects on your tests](#testing)
6. [Cross-cutting traps: toolchain and process](#crosscutting)
7. [A common misconception: is it faster now?](#faster)
8. [Retrospective: a repeatable upgrade playbook](#methodology)

---

<a name="why"></a>
## Why "major version + many dependencies" is the hard kind

A minor upgrade (3.4 → 3.5) is usually "change it and it runs." Major upgrades carry two compounding effects that make them a different beast.

**Breaking changes arrive in batches.** Package names change, API signatures change, the exception hierarchy changes, default behaviors change — and often there's no compile error to catch you. Some traps only detonate at runtime.

**Dependencies are bound together.** Bump Spring Boot to 4 and it drags Hibernate from 6 to 7, Jackson from 2 to 3, and Spring Framework from 6 to 7. You think you're upgrading one thing; you're actually upgrading four — and they're coupled: Hibernate uses Jackson to store JSON, Spring MVC uses Jackson to read and write request bodies.

Keep this "who drags whom" map in your head and every error later will line up against it:

```
Spring Boot 4.0  ──┬──> Spring Framework 7    (core container / Web / tx)
  (the master      ├──> Hibernate 7           (JPA / ORM)
   switch)         ├──> Jackson 3  (default)   (JSON serialization)
                   ├──> Jakarta Persistence 3.2
                   └──> Java 17 baseline (21 recommended)
```

The rest is told in four acts plus a set of cross-cutting traps. The acts are the four main storylines; the cross-cutting traps are the toolchain and process problems that span the whole upgrade.

---

<a name="jackson"></a>
## Act I: Jackson 2 → 3

Jackson is the largest surface area in this upgrade, because serialization is everywhere — reading and writing request bodies, caching, messaging, JSON columns in the database all route through it.

### It's not just "a package rename" — two versions coexist at runtime

The most visible change is the top-level package:

```java
// Jackson 2
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.core.JsonProcessingException;

// Jackson 3
import tools.jackson.databind.json.JsonMapper;
import tools.jackson.databind.JsonNode;
import tools.jackson.core.JacksonException;
```

`core` and `databind` moved from `com.fasterxml.jackson.*` to `tools.jackson.*`. This is deliberate — it lets 2.x and 3.x live on the same classpath without colliding.

**But there's one exception, and it's the source of the trap: the annotations did not move.**

```java
// Note: in Jackson 3, annotations are still under com.fasterxml!
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
```

`jackson-annotations` never shipped a 3.x release; `@JsonProperty`, `@JsonInclude`, `@JsonIgnore` all stay in the old package. So a real class ends up half `tools.jackson`, half `com.fasterxml` — it looks schizophrenic, but it's correct:

```java
import com.fasterxml.jackson.annotation.JsonInclude;  // annotation: old package
import tools.jackson.databind.JsonNode;               // type: new package

@JsonInclude(JsonInclude.Include.NON_NULL)
public class RuleRequest {
    String notificationType;
    JsonNode conditionNode;   // tools.jackson.databind.JsonNode
}
```

#### The real landmine: two `JsonNode` classes that look identical but aren't

Because 2.x and 3.x can coexist, your classpath may well carry both `com.fasterxml.jackson.databind.JsonNode` and `tools.jackson.databind.JsonNode`. **Same name, almost identical API — but two different classes.**

Your own code moved fully to Jackson 3, but a transitive dependency (very common mid-migration: some database migration tool, some API-docs library, some messaging SDK) may still be on Jackson 2. The result:

- **Compile time:** all fine, because everything you import is `tools.jackson`.
- **Runtime:** some component hands a `com.fasterxml...JsonNode` to code expecting `tools.jackson...JsonNode` — `ClassCastException`, or a more cryptic "type mismatch" deserialization failure.

How to debug it: when you hit a bizarre "surely these are the same class?" type error, first read the fully-qualified class name in the stack trace — is it `com.fasterxml` or `tools.jackson`? Then run `mvn dependency:tree` to find who's still pulling Jackson 2 in. Coexistence is fine short-term, but you must know which path uses which.

### `ObjectMapper` can't be `new`-ed anymore, and exceptions went from checked to unchecked

In Jackson 3, `ObjectMapper` became an abstract base class, and the mapper you build is **immutable**. The old style no longer compiles:

```java
// Jackson 2: new it directly, reconfigure any time
ObjectMapper mapper = new ObjectMapper()
        .registerModule(new JavaTimeModule())
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
```

Jackson 3 goes through a builder, configured once:

```java
// Jackson 3: built via builder, frozen afterwards
ObjectMapper mapper = JsonMapper.builder()
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
        .build();
```

A few related changes worth knowing:

- **`java.time` is built in.** No more manual `registerModule(new JavaTimeModule())`; `LocalDateTime` / `Instant` work out of the box.
- **`FAIL_ON_UNKNOWN_PROPERTIES` is off by default.** Jackson 2 threw on unknown fields by default; Jackson 3 silently ignores them. If you relied on that to reject dirty data, that guard quietly disappeared — either turn it back on or accept the new default.
- **`JsonNode.fields()` is gone**, replaced by `properties()`.

```java
// Jackson 2
Iterator<Map.Entry<String, JsonNode>> it = node.fields();
// Jackson 3
for (Map.Entry<String, JsonNode> e : node.properties()) { ... }
```

#### Exceptions: checked → unchecked, and that changes control flow

Jackson 2's `JsonProcessingException` is a **checked** exception — the compiler forces you to `throws` or `try/catch`. Jackson 3's `JacksonException` is a `RuntimeException`, **unchecked**:

```java
// Jackson 2: checked, the signature declares throws, callers must handle it
public String toJson(Object obj) throws JsonProcessingException {
    return MAPPER.writeValueAsString(obj);
}

// Jackson 3: JacksonException is a RuntimeException; drop throws and the compiler stops reminding you
public static String toJson(Object obj) {
    try {
        return MAPPER.writeValueAsString(obj);
    } catch (JacksonException e) {
        log.error("[toJson] serialize failed", e);
        return "";   // this fallback value is a decision you must make deliberately
    }
}
```

During a mechanical migration it's tempting to slap on a `catch` that returns `""` or `null`. But the checked exception used to bubble up the call chain and be handled centrally; now you've swallowed it at the bottom and **the exception semantics changed**. "What should the fallback return, and should it still propagate?" is a business decision, not something to settle offhand while batch-renaming packages (more on this in cross-cutting trap #3).

#### Your tests have to change too

Flipping an exception from checked to unchecked hits the tests directly, in two flavors — one won't compile until you fix it, the other compiles fine but now asserts the wrong thing.

**Flavor one (mechanical): the exception type in assertions / stubs must change.** Old tests that do `assertThrows(JsonProcessingException.class, ...)`, or use Mockito to make the mapper throw that checked exception, all need the type swapped to `JacksonException`. Note that `JsonProcessingException`'s constructor is `protected` in Jackson 2, so tests commonly use an anonymous subclass to get around it — same trick after migration:

```java
// Jackson 2: checked exception, anonymous subclass to construct, assert it's thrown
@Test
void toJson_whenSerializeFails_throwsChecked() throws Exception {
    JsonMapper spyMapper = spy(mapper);
    doThrow(new JsonProcessingException("boom") {})
            .when(spyMapper).writeValueAsString(any());
    assertThrows(JsonProcessingException.class, () -> service.toJson(badObj));
}

// Jackson 3: swap to the unchecked JacksonException; the throws on the signature can go
@Test
void toJson_whenSerializeFails() {
    JsonMapper spyMapper = spy(mapper);
    doThrow(new JacksonException("boom") {})
            .when(spyMapper).writeValueAsString(any());
    // for the assertion, see flavor two
}
```

**Flavor two (behavioral, and the one that matters): if you swallowed the exception at the bottom, the whole direction of the assertion flips.** Once `toJson` catches and returns `""`, it no longer propagates anything at runtime — the old `assertThrows` isn't just using a stale type, the entire assertion is wrong. It must change from "assert it throws" to "assert the fallback value":

```java
// Before: the method propagates the exception, so the test asserts it's thrown
assertThrows(JsonProcessingException.class, () -> service.toJson(badObj));

// After: the method swallows it and returns a fallback, so the assertion flips
String result = service.toJson(badObj);
assertEquals("", result);   // this is "control flow changed" projected onto the test
```

There's also a compile-time knock-on: wherever old code wrapped a now-no-longer-checked call in `try { ... } catch (JsonProcessingException e)`, Java will complain that the caught exception is "never thrown in the corresponding try block" — delete the catch or change it to catch `JacksonException`.

This loops back to cross-cutting trap #3: the moment you change exception semantics, the test assertions must follow — and "I edited an assertion to make the test pass" is itself a signal of a behavior change. Stop and confirm whether the migration required it, or whether you quietly moved the business semantics.

### Industry add-on: `StreamReadConstraints`, the "invisible bomb" that loves production

Here's one we didn't trigger this time, but which has caused countless production incidents in the wild — worth calling out on its own.

Since Jackson **2.15** (and carried into 3.x), the parser enforces a set of default safety limits via `StreamReadConstraints`, to stop oversized or deeply nested JSON from blowing up the service: max single-string length around 20 million characters, a max number length, a max nesting depth (1000 by default), and so on.

Nobody notices — until one day a legitimate but huge payload (a big Base64 blob, a very long text field) comes in and you get:

```
tools.jackson.core.exc.StreamConstraintsException:
  String length (20054321) exceeds the maximum length (20000000)
```

The insidious part: this is a **default-value change** — you didn't touch a line of code, yet behavior shifted after the upgrade — and it only fires on specific large data, which tests rarely cover. The fix is to widen it explicitly (once you've confirmed the data is legitimate):

```java
ObjectMapper mapper = JsonMapper.builder()
        .streamReadConstraints(StreamReadConstraints.builder()
                .maxStringLength(50_000_000)
                .build())
        .build();
```

Lesson: after a major upgrade, audit every library that has a **default safety threshold** (JSON parsing, HTTP body size, regex backtracking, decompression ratios) — did the default move, and will your real traffic cross the line?

---

<a name="hibernate"></a>
## Act II: Hibernate 6 → 7

Hibernate's upgrade is the most dangerous, because it governs **data**. A serialization bug fails one request; an ORM behavior bug can mean dirty data persisted, primary-key collisions, even data loss — and it often only surfaces under production-scale concurrency or volume.

### Our trap: Hibernate auto-picked the wrong mapper for JSON

We store semi-structured data in PostgreSQL `jsonb` columns, mapped like this:

```java
@Entity
public class EventRecord {

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private JsonNode content;   // tools.jackson.databind.JsonNode
}
```

Hibernate delegates the "Java object ⇄ JSON string" conversion to a component called a `FormatMapper` (the string → native `jsonb` step is handled by PostgreSQL itself; the `FormatMapper` only converts between objects and JSON text).

Here's the problem: **Hibernate 7 ships two `FormatMapper`s** — `JacksonJsonFormatMapper` (Jackson 2) and `Jackson3JsonFormatMapper` (Jackson 3).

Hibernate 7 "auto-detects" what's on the classpath and picks one. During the migration window — both Jacksons present — its auto-detection **defaults to the Jackson 2 one**. So your field is a `tools.jackson...JsonNode` but the mapper doing the work is Jackson 2's; they don't match, and at runtime you get:

```
Could not deserialize string to java type: class tools.jackson.databind.JsonNode
```

The error is misleading at first glance: it looks like corrupt JSON data, but it's actually "wrong mapper picked."

The fix: don't let it guess. Force the Jackson 3 `FormatMapper` explicitly, and **reuse the `JsonMapper` bean already configured in the Spring container** (so the whole app shares one set of serialization rules):

```java
@Configuration
public class HibernateJsonConfiguration {

    @Bean
    public HibernatePropertiesCustomizer jsonFormatMapperCustomizer(
            ObjectProvider<JsonMapper> jsonMapperProvider) {
        JsonMapper jsonMapper = jsonMapperProvider.getIfAvailable();
        Jackson3JsonFormatMapper formatMapper = (jsonMapper != null)
                ? new Jackson3JsonFormatMapper(jsonMapper)
                : new Jackson3JsonFormatMapper();
        return props -> props.put(AvailableSettings.JSON_FORMAT_MAPPER, formatMapper);
    }
}
```

There's an architectural bonus here: once Hibernate's JSON mapper and the Web/business layers all share **the same instance**, the entire chain (HTTP deserialization → business logic → persist → read back) runs on one set of rules. No more "parses on the way in, won't parse on the way to the DB."

Lesson: any library that advertises "auto-detect / auto-configure" deserves suspicion during a major-version coexistence window — what it detects may not be what you want. When in doubt, specify explicitly; don't let it guess.

### Industry add-on: the primary-key generator, the easiest place to "hurt the data"

If the trap above is an "obvious error," the primary-key generator change is the "silently wrong" kind — a classic that has burned people across Hibernate upgrades again and again.

The crux is that `@GeneratedValue`'s default behavior has shifted across major versions, especially the sequence generator's `allocationSize`. For the `SEQUENCE` strategy Hibernate defaults to `allocationSize = 50` with a pooled optimizer: it grabs a block (say 50) from the DB sequence and hands them out in memory. Two classic ways this goes wrong:

**ID jumps.** On restart or under multi-instance concurrency, IDs aren't contiguous (1, 2, 3, then suddenly 51). If anyone assumed "IDs are sequential," they're now wrong. It's not a bug — it's the pooled optimizer behaving normally — but it startles people the first time.

**Collisions from mismatched step size.** If the DB sequence is `INCREMENT BY 1` but Hibernate thinks the step is 50, the ranges the two sides compute will overlap — primary-key collisions, possibly overwriting existing data.

The right move is to declare it explicitly so the app side and DB side agree on the step, rather than relying on defaults:

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "event_seq")
@SequenceGenerator(name = "event_seq", sequenceName = "event_seq", allocationSize = 50)
private Long id;
// DB side: CREATE SEQUENCE event_seq INCREMENT BY 50;  -- the two MUST match!
```

Lesson: any ORM upgrade that touches ID generation, optimistic-lock versions, cascade deletes, or `orphanRemoval` must be validated against production-scale data and concurrency — not signed off after one row on a laptop. Mistakes in the data layer are the most expensive.

### Industry add-on: HQL semantics, implicit naming, enum mapping

Hibernate major versions also tend to quietly change defaults here:

- **HQL / Criteria semantics tighten.** "Loose" queries that ran on the old version (implicit cartesian products, fuzzy type coercions) may now error or return different results. Regression-test your complex HQL after upgrading.
- **Implicit naming strategy.** The default entity/field → table/column name mapping has been adjusted across major versions. If you didn't pin names with `@Table` / `@Column`, you might now map to a different column name — showing up as "a field suddenly reads no value." Explicit naming is the cheapest defense.
- **Default storage types for enums, booleans, and UUIDs** have changed (enum default `ORDINAL` vs `STRING`, etc.). These "read/write asymmetry" traps only surface at the data layer — verify with real round-trip data.

---

<a name="springboot"></a>
## Act III: Spring Boot 3 → 4

Spring Boot 4 is the master switch — it's what dragged the first two acts in. Its own breaking changes are relatively restrained (the heavy lifting lives in Spring Framework 7), but a few are unavoidable for every project.

### It decides the first two acts

- **Jackson 3 by default.** This is the root of Act I. SB4's auto-configuration and `@RestController` message converters default to Jackson 3.
- **Brings Hibernate 7 + Jakarta Persistence 3.2.** The root of Act II.
- **Java baseline raised to 17** (and fully embracing 21's virtual threads and friends). Still on Java 11/8? This step blocks you first.
- **Built on Spring Framework 7.** Core container, Web, transactions, validation all come along for the ride.

### Configuration and auto-config: the exit of `spring.factories`

A hard change many only discover mid-migration: the old mechanism of registering auto-configuration via `spring.factories` has been fully removed (deprecated since SB3 with a grace period, finished in SB4). Custom starters / auto-configurations must switch to the new file:

```
# Old (removed): META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration

# New: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.MyAutoConfiguration
```

If a third-party starter you depend on hasn't caught up, its auto-configuration will **fail silently** after the upgrade — the bean isn't registered, there's no error, and you only find out via `NoSuchBeanDefinitionException` when something uses it. Combined with renamed/removed config keys (plenty under `spring.*` changed or disappeared), run a full config validation + startup self-check after upgrading — don't just eyeball "it started."

### Industry add-on: RestClient, observability, nullability annotations

A few things worth modernizing — or that you must adapt — during SB3→4:

- **HTTP client.** `RestTemplate` is in maintenance mode; the official recommendation is `RestClient` (a synchronous, fluent API with the same mental model as `WebClient`). Don't write new `RestTemplate` code.
- **Observability.** Distributed tracing moved from Spring Cloud Sleuth to Micrometer Tracing / Observation long ago. If Sleuth remnants linger, clean them up during the upgrade.
- **Null contracts (JSpecify).** Spring Framework 7 adopted JSpecify's standard `@Nullable` / `@NonNull` annotations in place of its own. If you run strict null static analysis (e.g. NullAway), expect a fresh batch of warnings — a good thing (clearer contracts), but budget time to digest them.
- **Deprecation sweep.** SB4 deleted a large pile of APIs that SB3 had marked `@Deprecated`. Best practice: clear every deprecation warning while still on the current major (SB3) *first*, then jump to SB4 — don't mix the two efforts.

---

<a name="testing"></a>
## Act IV: the ripple effects on your tests

During an upgrade many people only watch whether the main code compiles and the service boots — and the test modules become a blind spot. In this major upgrade the test side changed plenty, and some of it is the hard "won't compile until you fix it" kind.

### Test starters got modularized

Spring Boot 4 split test auto-configuration into finer-grained modules. Previously `spring-boot-starter-test` plus `spring-boot-starter-data-jpa` gave you **test slices** like `@DataJpaTest` out of the box; in SB4 the JPA test-slice support was carved out into a dedicated `spring-boot-starter-data-jpa-test` that you must add explicitly (test scope), or the relevant auto-config is missing and repository-layer tests won't even start:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa-test</artifactId>
    <scope>test</scope>
</dependency>
```

Symptom: after the upgrade, `@DataJpaTest` complains it can't find auto-config / the test context won't come up. Lesson: SB4's test starters are more granular — pull in whichever slice you actually use, and stop assuming "one starter-test rules them all."

### `@MockBean` / `@SpyBean` removed, replaced by `@MockitoBean`

Since Spring Boot 3.4, `@MockBean` / `@SpyBean` were deprecated in favor of Spring Framework's generic bean-override mechanism `@MockitoBean` / `@MockitoSpyBean` (package `org.springframework.test.context.bean.override.mockito`). In SB4 the old annotations are gone and won't compile:

```java
// Old (removed in SB4)
import org.springframework.boot.test.mock.mockito.MockBean;
@MockBean private NotificationService notificationService;

// New
import org.springframework.test.context.bean.override.mockito.MockitoBean;
@MockitoBean private NotificationService notificationService;
```

It's a purely mechanical replacement, but high-volume (every test using a mocked bean must change), and the import package is completely different — don't expect the IDE's auto-import to get it right in one shot.

### Mockito major bump: inline becomes default, `mockito-inline` becomes baggage

The upgrade also pulls Mockito to 5.x. Mockito 5 makes the inline mock maker the default (you can mock `final` classes and `static` methods directly), which makes the separately-declared `mockito-inline` in old projects redundant — in Mockito 5 it's an empty/deprecated shell. You can remove it; keeping it won't error, it's just pointless. Mockito 5 also requires Java 11+ and tightens strict stubbing, so some previously "lax" stubs will now throw `UnnecessaryStubbingException` after the upgrade.

### The test context also needs that JSON-mapper fix

The config from Act II that forces Hibernate's Jackson 3 `FormatMapper` **must also take effect in the test context**. Otherwise the moment a `@DataJpaTest` / repository test touches a `jsonb` column, you reproduce that same `Could not deserialize ... tools.jackson...JsonNode` inside the tests.

Test slices often don't load the `@Configuration` bean you wrote (`@DataJpaTest` boots only a minimal JPA context), so the simplest, most reliable approach is to pin the format mapper directly via Hibernate's native property in `application-test.yml`:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        show_sql: false
        default_schema: notifications
        # Force the Jackson 3 format mapper for @JdbcTypeCode(SqlTypes.JSON) columns.
        # Hibernate 7 defaults to the Jackson 2 mapper, which can't handle tools.jackson.databind.JsonNode.
        type:
          json_format_mapper: org.hibernate.type.format.jackson.Jackson3JsonFormatMapper
```

So: the main code uses a `@Bean` (which can reuse Spring's `JsonMapper`), while the tests use one line of YAML (independent of whether the main config is loaded) — both paths point at Hibernate 7's `Jackson3JsonFormatMapper`. Add a separate `application-test.yml` that also skips DB migration for tests, and your test environment is aligned with runtime.

Lesson: validating a major upgrade can't rely on unit tests alone (all-mocked, they never touch real serialization or the DB). You need integration test slices that **actually hit the database and actually round-trip serialization**, or the Act II class of trap is simply not covered. Also note Spring Framework 7 no longer bundles built-in JUnit 4 runner support — lingering `@RunWith` / vintage tests should migrate to JUnit 5.

---

<a name="crosscutting"></a>
## Cross-cutting traps: toolchain and process

These span all four acts, aren't tied to any one library, and can grind you down just as hard.

### Cross-cutting trap #1: don't trust the IDE's red squiggles — "phantom errors" and "broken bytecode"

A classic symptom: the service throws this at startup:

```
Caused by: java.lang.Error: Unresolved compilation problems:
	log cannot be resolved
	at com.example.config.NettyServer.<init>(NettyServer.java:63)
```

`log cannot be resolved` — but `log` is generated by `@Slf4j`, the source is perfectly fine, and a command-line `mvn compile` passes in one go. So why a "compilation error" at runtime?

The root cause is that **the IDE's built-in compiler and the command-line compiler are not the same**. Many IDEs default to their own incremental compiler, while Lombok and similar libraries generate code at compile time via annotation processors (`@Slf4j` generates `log`, `@Getter` generates getters) and need the compiler to run annotation processing correctly. If the IDE didn't properly enable Lombok's annotation processing, it can't find `log` when compiling the `@Slf4j` class — but instead of stopping, it **bakes an "unresolved compilation problem" into the `.class` bytecode**. Load and execute that broken `.class` and you get `java.lang.Error: Unresolved compilation problems` at runtime.

It's nastier in multi-instance setups: the same code, one instance starts fine and another blows up — because the IDE's "build before launch" step overwrote the shared `target/classes` with its own (broken) bytecode again.

The fix is two steps. First, stop the bleeding: do a clean compile with the build tool (not the IDE) so the real `javac` + Lombok regenerate the bytecode.

```bash
mvn -o -Dmaven.test.skip=true clean compile
```

Then the permanent fix: install the Lombok plugin in the IDE and enable annotation processing, or disable the "build before launch" step that keeps clobbering the output with the bad compiler.

Lesson: during an upgrade, **treat the build tool's compilation as authoritative, not the IDE's red squiggles**. The IDE can both report errors that don't exist (false positives) and silently bake errors into bytecode (false negatives). When the two disagree, `mvn` / `gradle` wins.

### Cross-cutting trap #2: dependency hell and runtime `NoSuchMethodError`

This is the most pervasive "background radiation" of a major multi-dependency upgrade. The symptom is usually "compiles fine, dies at startup," with `NoSuchMethodError` / `NoClassDefFoundError` / `AbstractMethodError` at the top of the stack.

The essence is **version mismatch**: library A wants version 32 of some transitive dependency, library B wants 30, the build tool's "nearest-wins" arbitration picks one, and the other can't find the method signature it expected at runtime. During a major upgrade a whole pile of transitive dependencies get dragged at once, so these conflicts erupt together.

A few battle-tested moves:

1. **Unify versions via a BOM.** `spring-boot-dependencies` already aligns a huge swath for you. Anything a BOM can manage, don't hard-pin in `<dependency>`.
2. **`mvn dependency:tree -Dverbose` to find the culprit.** See exactly which path and which version a class comes from, and who arbitrated the other away.
3. **Use `<exclusions>` and explicit version pins sparingly.** Exclude only when truly necessary, and verify afterward that the excluded side won't be missing something at runtime.
4. **Deliberately manage "two majors coexisting."** Like the dual Jackson in Act I. Coexistence isn't the error; coexisting *without knowing it* is — you must know which path uses which.

### Cross-cutting trap #3: when "mechanical migration" smuggles in "behavior changes"

This isn't a technical trap, it's a process one — but it shows up constantly in big upgrades, especially now that everyone uses AI tools to bulk-edit code.

A major-version migration's edits should ideally be **mechanical and equivalent**: rename packages, swap APIs, adjust signatures, behavior unchanged to the letter. That way, if something breaks the blast radius is contained and review is easy.

But in reality, whether a human edits by hand or an AI does it in bulk, it's very easy to "improve things along the way." Real examples we hit: an exception that used to bubble up gets `catch`-swallowed at the bottom returning `""` / `null` (the Act I example), quietly changing its semantics; things the framework could auto-generate (Lombok getters/setters, builders) get hand-written, adding new surface for bugs; a hand-rolled object-copy routine "replaces" a one-line serialization, adding code nobody tested.

Each looks reasonable in isolation, but none of it relates to "the upgrade" — yet it rode in on the same batch of commits. The fallout: when something breaks you can't tell whether the upgrade caused it or the "drive-by improvement" did; and reviewers can't tell which changes were required and which were smuggled.

Ways to hold the migration boundary:

1. **Upgrade commits do equivalent changes only.** Split behavior changes into separate commits/PRs with titles that say "this is a behavior change, not a migration."
2. **Validate every edit against the baseline.** Is this identical to pre-upgrade behavior? If not, is it *required* by the upgrade, or just "I think it's better"? Split the latter out.
3. **Treat AI as a "fast-typing intern who doesn't know your business intent."** It's great at bulk package renames and signature swaps (the most tedious part of migration), but it also loves to "improve" things — eyeball every non-mechanical edit yourself.
4. **Let tests and the diff guard the boundary.** A pure migration should ideally leave all existing tests passing as-is; if you changed a test's assertion to make it pass, stop and ask: was that required by the migration, or did behavior actually change?

---

<a name="faster"></a>
## A common misconception: is it faster now?

After the upgrade, someone always asks: now that everything's on the new versions and the mappers are unified into one instance, is performance better?

The honest answer: **you probably can't feel it.** In a typical "receive request → process → persist → read back" service, database I/O is the dominant cost; JSON serialization is a tiny slice. Optimize serialization by 20% and, spread across the whole chain, it might be a 0.x% difference — lost in network and disk jitter.

So what's the real payoff of unifying the mapper? Correctness and maintainability, not throughput.

**Eliminating cross-version / cross-instance inconsistency.** Before, you might have had one mapper in the Web layer, a `new`-ed one in a util class, and an auto-configured one in Hibernate — three configs whose details (date format, null policy, unknown-field handling) could differ subtly, planting the "serializes here, won't deserialize there" landmine. One bean, one set of rules across the whole chain.

**Killing the real anti-pattern of `new ObjectMapper()` per call.** Mapper construction isn't cheap; `new`-ing one repeatedly on a hot path is genuine waste. Switching to a reused singleton is actually faster — but that's "fixing a bug," not "the new version is faster."

Lesson: don't rush to claim "performance improved" after an upgrade. Figure out where your bottleneck actually is first; without load-test numbers, just say honestly that the value of this upgrade is correctness, consistency, security patches, and long-term maintainability — not throughput. Mislabeling "the gain from fixing an anti-pattern" as "the new version's doing" misleads everyone downstream.

---

<a name="methodology"></a>
## Retrospective: a repeatable upgrade playbook

1. **Map the dependency topology first.** What does the thing you explicitly upgrade (Spring Boot 4) drag along (Spring Framework 7 / Hibernate 7 / Jackson 3 / the Java baseline)? Draw it with `mvn dependency:tree` up front, and every later error lines up against it.

2. **Anticipate the shapes of breaking changes** and go through them by category: package/coordinate changes (Jackson `com.fasterxml` → `tools.jackson`); construction changes (`new ObjectMapper()` → `JsonMapper.builder()`); exception-hierarchy changes (checked → unchecked, quietly altering control flow); **default-behavior changes** (`FAIL_ON_UNKNOWN_PROPERTIES` flipping, `StreamReadConstraints` thresholds, Hibernate `allocationSize` / implicit naming) — the nastiest, invisible at compile time; and "auto-detect / auto-configure" picking the wrong implementation during coexistence (Hibernate's dual `FormatMapper`, a silently-dead third-party starter).

3. **Treat "found at compile time" and "found only at runtime" differently.** The former you sweep with the compiler, round after round; the latter (Hibernate JSON mapper, ID jumps, `StreamReadConstraints`) you can only catch by running it plus tests that cover that path. So smoke tests must actually exercise writing to the DB, reading back, and large payloads — not just "it started."

4. **Trust the real compiler, not the IDE's red squiggles.** When they disagree, `mvn` / `gradle` decides; for a bizarre runtime "compilation error," `clean` first and confirm Lombok annotation processing is set up right.

5. **Hold the migration boundary.** Upgrade commits do equivalent changes only; behavior changes always split out; with AI assistance, eyeball every non-mechanical edit yourself.

6. **Strategically prefer "incremental" over "big bang."** Clear all deprecation warnings on the current major before jumping to the next; step it where you can (bump the Java baseline → then the framework → then the dragged-in libraries), each step independently compilable and testable; lock versions with a BOM throughout so the build is reproducible.

7. **Be honest about the payoff.** An upgrade's value is usually correctness, consistency, security patches, and long-term maintainability — not "it's faster." No load-test data, no performance story.

---

## Closing

The most draining part of a major upgrade is never "it won't change" — it's the traps the compiler doesn't flag, that only detonate at runtime, with error messages that lead you down the wrong path: why two `JsonNode`s aren't the same class, why Hibernate picked the wrong mapper, why primary keys jump, why the IDE and the command line disagree, why the test module suddenly won't start.

Internalize the *why* behind each, and next time you'll read a symptom straight to its root cause instead of flailing at the top line of a stack trace. The real moat isn't "I memorized which package got renamed to what" — that expires — it's "I understand why it happens," which doesn't.

---

That's it.

Thanks for reading! If you like it or feel it helped, pls click Applaud and share it with your team!
Happy coding.

See you next time :)

---

## References

- [Jackson 3.0 Migration Guide](https://github.com/FasterXML/jackson/wiki/Jackson-Release-3.0)
- [Hibernate ORM 7 User Guide](https://docs.jboss.org/hibernate/orm/7.0/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Boot 4.0 Release Notes](https://github.com/spring-projects/spring-boot/wiki)
- [Spring Framework 7 — Null Safety (JSpecify)](https://docs.spring.io/spring-framework/reference/core/null-safety.html)
