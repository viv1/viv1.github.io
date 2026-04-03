---
layout: post
title: "The Complete Guide to Fixing CVEs in Maven Dependencies"
date: 2026-04-03 08:00
category: Java
tags: ["java", "maven", "pom.xml", "dependencies", "cve", "security", "trivy", "dependency-management", "shaded", "bom"]
description: "Direct, transitive, shaded, BOM-managed, plugin dependencies and everything in between. A practical guide to resolving every type of Maven dependency vulnerability."
---

## Introduction

If you have worked on any Java project long enough, you have dealt with this. A security scan flags a CVE in one of your dependencies. Sometimes it is straightforward, you bump a version and move on. Other times, you spend hours tracing through the dependency tree trying to figure out where a vulnerable library is even coming from.

Remember [CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228){:target="_blank"} (Log4Shell)? That one was a direct dependency for most projects, so the fix was relatively simple. But not every CVE is that clean. Sometimes the vulnerable library is buried three levels deep in your dependency tree, or worse, shaded inside another JAR where Maven can't even see it.

I have been doing this across multiple repositories over the years, and the patterns repeat. The same types of problems, the same types of fixes. This article is my attempt to document all of them in one place so I (and hopefully you) don't have to rediscover the solution every time.

## Finding Vulnerabilities

Before you can fix anything, you need to know what is vulnerable and where it is coming from. There are two broad categories of tools here, and you need both.

### Maven Dependency Tree

`mvn dependency:tree` is your first stop. It shows you the full dependency graph of your project, every direct dependency and everything they pull in transitively.

```bash
mvn dependency:tree
```

This gives you the full tree, but most of the time you already know which library is flagged. Use the `-Dincludes` filter to narrow it down:

```bash
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

```
[INFO] com.example:my-app:jar:1.0.0
[INFO] \- com.some.library:some-lib:jar:3.2.1:compile
[INFO]    \- com.fasterxml.jackson.core:jackson-databind:jar:2.13.0:compile
```

This tells you that `jackson-databind:2.13.0` is coming in as a transitive dependency through `some-lib`. Now you know where to look.

> `mvn dependency:tree` only shows dependencies that Maven resolves, meaning things declared in your `pom.xml` and their transitives. It does **not** detect vulnerable classes inside shaded/fat JARs. More on that later.
{:.prompt-info}

### Trivy, Grype and Other Scanners

Tools like [Trivy](https://github.com/aquasecurity/trivy){:target="_blank"} and [Grype](https://github.com/anchore/grype){:target="_blank"} scan the actual JAR files on disk. They look at bytecode and class names, not just Maven coordinates. This means they catch things `mvn dependency:tree` cannot, like vulnerabilities inside shaded JARs where the classes have been relocated into a different package namespace.

```bash
trivy fs --scanners vuln .
```

There is also [OWASP dependency-check](https://owasp.org/www-project-dependency-check/){:target="_blank"} if you want something more Maven-native:

```bash
mvn org.owasp:dependency-check-maven:check
```

And of course, GitHub Dependabot and Snyk can be set up to alert you automatically on PRs or through CI.

### Read the CVE

This sounds obvious but it is worth saying. Before you start fixing, read the CVE description. Understand which artifact is affected, which version range, and what the actual vulnerability is. Sometimes the CVE applies to a specific feature of the library that you don't even use. That changes your approach entirely.

## Bump the Version (Direct Dependency)

The simplest case. The vulnerable library is something you declared directly in your `pom.xml`. You just update the version.

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.3</version> <!-- was 2.13.0 -->
</dependency>
```

Or if the version is controlled by a property (which it usually should be):

```xml
<properties>
    <jackson.version>2.15.3</jackson.version> <!-- was 2.13.0 -->
</properties>
```

A few things to watch out for:

**Breaking API changes.** Major version bumps can introduce incompatible changes. Your code may not compile, or worse, it compiles fine but behaves differently at runtime. Always run your test suite after upgrading.

**No fixed version available.** Sometimes the CVE is reported but the maintainer hasn't released a patch yet. You are stuck waiting, or you need to evaluate whether the vulnerability is actually exploitable in your usage and suppress the finding temporarily.

**Abandoned libraries.** If the library is archived or unmaintained, no fix is coming. You need to migrate to a fork or an alternative library entirely. This is the most painful scenario and there is no shortcut.

## Override Transitive Dependencies

This is the most common scenario in practice. The vulnerability is not in something you declared directly. It is in a library that one of your dependencies pulls in.

### How Maven Resolves Versions

Maven uses a "nearest wins" strategy. If the same library appears at multiple levels in the dependency tree, Maven picks the version that is closest to the root of the tree. In case of a tie (same depth), the one declared first in the `pom.xml` wins.

This is important to understand because it means your override strategy depends on where things are in the tree.

### Option 1: Upgrade the Parent Dependency

The cleanest fix. If `some-lib:3.2.1` pulls in `jackson-databind:2.13.0`, check if a newer version of `some-lib` already uses a fixed `jackson-databind`. If it does, just bump `some-lib`.

```xml
<dependency>
    <groupId>com.some.library</groupId>
    <artifactId>some-lib</artifactId>
    <version>3.3.0</version> <!-- now pulls jackson-databind 2.15.3 -->
</dependency>
```

This is the ideal fix because you are not fighting Maven's resolution. But it is not always possible. The parent library may not have released a new version yet, or their new version may bring its own set of problems.

### Option 2: Force via dependencyManagement

`<dependencyManagement>` lets you pin a version of any dependency across your entire project, regardless of what your dependencies declare.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.3</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

This is my go-to when the parent dependency hasn't upgraded yet. It is clean and applies globally. But do verify that the parent library actually works with the newer version of its transitive. You are essentially telling Maven "I know better than what `some-lib` asked for", and sometimes `some-lib` genuinely needs the older version.

### Option 3: Promote to Direct Dependency

Add the transitive dependency as a direct dependency in your `pom.xml` with the fixed version. Since direct dependencies are closer to the root, Maven's "nearest wins" rule will pick your version.

```xml
<dependencies>
    <!-- your other dependencies -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.3</version>
    </dependency>
</dependencies>
```

This works but it adds noise to your `pom.xml`. You now have a direct dependency on something you don't actually use directly in your code. If you go this route, leave a comment explaining why.

### Option 4: Exclude and Re-add

Exclude the vulnerable version from the dependency that brings it in, then add the fixed version yourself.

```xml
<dependency>
    <groupId>com.some.library</groupId>
    <artifactId>some-lib</artifactId>
    <version>3.2.1</version>
    <exclusions>
        <exclusion>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.3</version>
</dependency>
```

I use this when I want to be very explicit about what is happening, or when the vulnerable library is coming in through multiple dependencies and `dependencyManagement` alone isn't enough to make it clear.

### The Diamond Dependency Problem

This is when two of your direct dependencies both pull in the same library but at different versions.

```
my-app
├── lib-A → jackson-databind:2.13.0 (vulnerable)
└── lib-B → jackson-databind:2.15.3 (fixed)
```

Maven picks the one declared first in your `pom.xml` (since they are both at the same depth). If `lib-A` is declared before `lib-B`, you end up with the vulnerable version even though a fixed version exists in your tree.

The fix here is `<dependencyManagement>` to force the version you want. Don't rely on declaration order.

### Verify the Fix

After making changes, always verify:

```bash
mvn dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

Make sure only the fixed version shows up.

## Upgrade the BOM

A BOM (Bill of Materials) is a special POM that manages versions for a set of related dependencies. Frameworks like Spring Boot, AWS SDK, and Jackson all publish BOMs. If you use one, the versions are controlled there, not in your individual dependency declarations.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Upgrading the BOM

Often the simplest fix. Bump the BOM version and all managed dependencies update together.

But be aware: a BOM upgrade can bump dozens of libraries at once. This is both a feature and a risk. You fix one CVE but you also change versions of libraries you weren't targeting. Run your tests carefully.

### Overriding a Single Version Within a BOM

Sometimes the BOM hasn't been updated yet, but you need to fix a specific CVE now. You can override a single managed version by adding your own `<dependencyManagement>` entry **after** the BOM import.

```xml
<dependencyManagement>
    <dependencies>
        <!-- BOM import -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.1.5</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Override a specific version from the BOM -->
        <dependency>
            <groupId>org.yaml</groupId>
            <artifactId>snakeyaml</artifactId>
            <version>2.2</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> In `<dependencyManagement>`, the last declaration wins. So if you need to override a version from a BOM, declare your override **after** the BOM import. If you have multiple BOMs managing the same dependency, the one imported last takes precedence.
{:.prompt-info}

Many Spring Boot BOMs also expose version properties. You can often override just the property without touching `<dependencyManagement>` at all:

```xml
<properties>
    <snakeyaml.version>2.2</snakeyaml.version>
</properties>
```

Check the BOM's source POM to see if a property exists for the dependency you want to override.

## Parent POM and Multi-Module Projects

In a multi-module Maven project, dependency versions are usually managed in the parent POM. This is the right thing to do for consistency, but it introduces a few scenarios you need to watch for.

### Parent You Own

If the parent POM is part of your project, update the version there. All child modules inherit the change.

```xml
<!-- parent pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-text</artifactId>
            <version>1.11.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Parent You Don't Own

If the parent POM comes from a corporate shared framework or a third-party, you can't change it. Override in your own `<dependencyManagement>` section. Your local declaration takes precedence over the inherited one.

### Child Module Overriding Parent

Watch out for child modules that declare their own version of a dependency, overriding the parent. When you update the parent, the child's override still takes precedence. This is a common source of "I already fixed this, why is the scanner still flagging it".

```xml
<!-- child pom.xml - this overrides the parent's version -->
<dependencies>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-text</artifactId>
        <version>1.9.0</version> <!-- old, vulnerable -->
    </dependency>
</dependencies>
```

The fix is to remove the version from the child and let it inherit from the parent, or update the child's version too.

### Maven Enforcer Plugin

If your project uses `maven-enforcer-plugin` with rules like `requireUpperBoundDeps` or `bannedDependencies`, your version change may get rejected by the build. This is actually a good thing. It forces you to resolve conflicts properly rather than letting Maven silently pick a version. But it does mean you might need to update the enforcer configuration alongside your dependency change.

## Shaded and Embedded Dependencies

This is the tricky one. Some libraries use the `maven-shade-plugin` to bundle their dependencies inside their own JAR. The classes are copied and often relocated to a different package namespace (e.g., `com.google.common` becomes `com.somelib.shaded.com.google.common`).

The problem: you cannot override a shaded dependency through Maven. It is not a Maven dependency anymore. It is just a bunch of class files baked into a JAR.

```bash
mvn dependency:tree -Dincludes=com.google.guava:guava
```

This will show nothing, even though `guava` classes are sitting inside `some-lib.jar`. This is where `mvn dependency:tree` fails and you need Trivy or Grype to detect it.

### What You Can Do

**Wait for upstream.** The library maintainer needs to re-shade with the fixed version. This is often the only real fix. File an issue, link the CVE, and hope they are responsive.

**Assess exploitability.** Read the CVE. If the vulnerability is in a code path that the shading library never invokes, the risk may be low. For example, a deserialization CVE in a library that is only used for its string utilities. In this case, you might choose to suppress the scanner finding with a note explaining why.

**Re-shade it yourself.** If upstream is unresponsive and you can't wait, there is a practical workaround. Create a small, dedicated repository whose only job is to take the problematic shaded JAR, re-shade it with the updated dependency, and publish the patched artifact to your internal Nexus or Artifactory.

```xml
<!-- reshade-fix/pom.xml -->
<project>
    <groupId>com.yourorg.reshaded</groupId>
    <artifactId>some-lib-reshaded</artifactId>
    <version>3.2.1-patched</version>

    <dependencies>
        <!-- the original shaded JAR -->
        <dependency>
            <groupId>com.some.library</groupId>
            <artifactId>some-lib</artifactId>
            <version>3.2.1</version>
        </dependency>
        <!-- the fixed version of the vulnerable dependency -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.3</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <relocations>
                                <relocation>
                                    <pattern>com.fasterxml.jackson</pattern>
                                    <shadedPattern>com.some.library.shaded.com.fasterxml.jackson</shadedPattern>
                                </relocation>
                            </relocations>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

The `ServicesResourceTransformer` handles merging `META-INF/services` files so service loader registrations from both the original JAR and the updated dependency don't clobber each other. The relocation pattern must match whatever the original library used, otherwise you end up with duplicate classes under different paths.

In your actual project, you replace the original dependency with your reshaded one:

```xml
<dependency>
    <groupId>com.yourorg.reshaded</groupId>
    <artifactId>some-lib-reshaded</artifactId>
    <version>3.2.1-patched</version>
</dependency>
```

This is a maintenance burden, you now own that patched artifact. But the repo is small and single-purpose. When upstream eventually releases a fix, you switch back to the original coordinates and archive the reshade repo.

**Suppress the false positive.** Scanners sometimes flag relocated classes even when the vulnerability is not exploitable. If you have done the analysis, suppress it in your scanner configuration and document your reasoning.

This is genuinely the most frustrating scenario because you have no direct control. But at least understanding why you are stuck helps you communicate the risk to your security team properly.

## Plugin Dependencies

This one trips people up because it looks like a regular transitive dependency problem, but the fix is completely different. Maven plugins like `maven-compiler-plugin`, `maven-surefire-plugin`, etc. have their own dependency trees that are **entirely separate** from your project's dependencies. Your project's `<dependencyManagement>` has no effect on them. A `<dependencyManagement>` override that fixes `commons-compress` in your project dependencies will not touch the `commons-compress` version that `maven-surefire-plugin` uses internally.

### Is It Actually a Risk?

Plugin dependencies run during the build only. They don't end up in your production artifact. So the risk is different. It is a build-time supply chain concern (a compromised build tool), not a runtime exploit. Worth fixing, but usually lower priority.

### How to Fix

**Option 1: Upgrade the plugin itself.** Often the cleanest path. A newer plugin version will usually pull in updated dependencies.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.2</version> <!-- was 3.0.0 -->
</plugin>
```

**Option 2: Override the plugin's transitive dependency.** If the plugin hasn't released a version with the fix, you can override its internal dependency using the `<dependencies>` section inside the `<plugin>` block. This is the plugin equivalent of `<dependencyManagement>` for project dependencies.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
            <dependencies>
                <dependency>
                    <groupId>org.apache.commons</groupId>
                    <artifactId>commons-compress</artifactId>
                    <version>1.26.0</version> <!-- override vulnerable version -->
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

Note that this `<dependencies>` block inside `<plugin>` is a different thing from the project-level `<dependencies>`. It only affects what the plugin uses during execution.

## Scope, False Positives and Suppression

Not every scanner finding needs a code change.

### Test Scope Dependencies

If the vulnerability is in a `test` scoped dependency, it never ships to production. It is only present during your build and test phase. The risk is low.

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.1.210</version>
    <scope>test</scope>
</dependency>
```

Still worth upgrading when possible, but don't lose sleep over it.

### Provided Scope Dependencies

`provided` scope means the library is supplied by your runtime environment (e.g., Tomcat, application server). Your POM declares it so the code compiles, but the actual JAR at runtime comes from the server.

Upgrading it in your `pom.xml` won't fix the vulnerability. You need to upgrade the runtime environment itself. This is an infrastructure change, not a Maven change.

### The CVE Doesn't Apply to Your Usage

This happens more than you'd think. A CVE might be about XML external entity injection, but you only use the library for JSON parsing. Or it is about a specific deserialization gadget that requires a particular class to be on the classpath, which it isn't in your application.

Read the CVE. Understand the attack vector. If it is not reachable in your application, document that and suppress.

### Artifact Confusion

Sometimes the scanner flags the wrong artifact. For example, `log4j-core` had the famous Log4Shell vulnerability, but `log4j-api` was not affected. If you only use `log4j-api`, you may be dealing with a false positive.

Similarly, libraries that change their Maven coordinates over time (like `javax.servlet` to `jakarta.servlet`, or `mysql:mysql-connector-java` to `com.mysql:mysql-connector-j`) can confuse scanners. You might already be using the new, fixed coordinates while the scanner flags the old ones.

## Verification and Closing the Loop

After making your changes:

**Check the dependency tree:**
```bash
mvn dependency:tree -Dincludes=groupId:artifactId
```

**Re-run your scanner:**
```bash
trivy fs --scanners vuln .
```

**Run your test suite** and watch for `NoClassDefFoundError`, `NoSuchMethodError`, or any runtime behavior changes. Binary incompatibilities don't always show up at compile time.

**Build the project end to end:**
```bash
mvn clean verify
```

If your CI pipeline has security scanning integrated (and it should), the next build will confirm whether the vulnerability is resolved. If you use `maven-enforcer-plugin`, it will catch any version conflicts you may have introduced.

## When It's Not in Your POM at All

If you have gone through everything above and still can't find the vulnerable library in your dependency tree, it might not be a Maven problem. Scanners like Trivy don't just scan your `pom.xml`. When run against a Docker image, they scan every JAR on the filesystem, including ones that come from the base image itself.

For example, your `eclipse-temurin:17-jre` or `amazoncorretto:17` base image might ship with JARs in `/usr/lib` or `/opt` that have nothing to do with your application. The fix is to upgrade the base image, not your `pom.xml`.

```dockerfile
FROM eclipse-temurin:17.0.10_7-jre  # was 17.0.8_7-jre
```

If you are staring at a scanner report and `mvn dependency:tree` shows nothing, check what your Docker image is bringing in.

## Conclusion

Most CVE fixes in Maven projects fall into one of these categories. The simple ones are just version bumps. The frustrating ones involve shaded dependencies where you are waiting on upstream. And the sneaky ones are transitive dependencies hiding three levels deep in your tree that you didn't even know existed.

The key takeaway: always start with `mvn dependency:tree` to understand where the vulnerable library is coming from, and use Trivy or Grype to catch what Maven can't see. If neither shows it, check your Docker base image. Once you know the "where", the fix usually becomes obvious.

## Related Articles

If you found this useful, you might also like these:

- [Simplifying the complicated world of Libraries - Part 1](/posts/simplifying-libraries-part-1/) - A deep dive into how library binaries work, static vs dynamic linking, and compatibility across platforms. Useful context for understanding why shaded JARs exist in the first place.
- [When JNI Meets 'Write Once, Run Anywhere'](/posts/when-jni-meets-write-once-run-anywhere-in-multi-arch-world/) - How Java's native library dependencies (JNI) introduce platform-specific packaging challenges, another layer of dependency complexity beyond what Maven manages.

## References

- [Maven Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html){:target="_blank"}
- [Trivy - Vulnerability Scanner](https://github.com/aquasecurity/trivy){:target="_blank"}
- [Grype - Vulnerability Scanner](https://github.com/anchore/grype){:target="_blank"}
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/){:target="_blank"}
- [Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/){:target="_blank"}
- [Maven Enforcer Plugin](https://maven.apache.org/enforcer/maven-enforcer-plugin/){:target="_blank"}
