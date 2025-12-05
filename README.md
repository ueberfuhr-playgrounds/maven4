# Maven 4 - What's New

> [!IMPORTANT]
> This repository was tested with the Maven 4 Release Candidate (RC5).
> Things might change until the final release.

> [!NOTE]  
> This repository contains a simple multi-project setup. To build it, just download Maven4 and invoke
> ```bash
> mvn clean install \
>   -Dmaven.consumer.pom.flatten=true \
>   -b concurrent
> ```

Now let's find out what's new:

---
## Model Version 4.1.0

**Maven 4 updates the POM version to `4.1.0`.** It is compatible with version `4.0.0`, but adds elements like `<subprojects>` and marks some elements as deprecated.

The POM now looks like this:

```xml
<project
    xmlns="http://maven.apache.org/POM/4.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.1.0 http://maven.apache.org/xsd/maven-4.1.0.xsd">

  <modelVersion>4.1.0</modelVersion>

  <!-- ... -->

</project>
```

We could omit `<modelVersion>` since Maven derives it from the schema.

**Updating existing projects is optional.** Maven 4 can build projects with version `4.0.0` or `4.1.0`, but new features only work with the updated version.

---
## Build/Consumer Separation

Previously, a library’s POM included build metadata like plugin configurations. This bloated dependency resolution and sometimes exposed internal details.

**Maven 4 [separates the _Build POM_ from the _Consumer POM](https://cwiki.apache.org/confluence/display/MAVEN/Build+vs+Consumer+POM)_.** We keep the Build POM as before, and Maven generates a Consumer POM for dependency users.

A Consumer POM excludes build-specific info, parent POM references, and unused dependencies. It only includes transitive dependencies and managed dependencies that the project uses. Properties are resolved in place.

We call the transformation _"flattening"_. Maven 3 required the [Flatten Maven Plugin](https://www.mojohaus.org/flatten-maven-plugin/), but Maven 4 handles it natively.

> [!TIP]
> Since RC5, we must use `-Dmaven.consumer.pom.flatten=true` to enable it.

> [!NOTE]
> This is only supported by the `maven-install-plugin` version `4.x`, which is currently not the default version in Maven 4 RC5. (Same with deploy plugin.)

---
## New Artifact Types

For dependencies, we could specify a `<type> to reference artifacts with non-default packaging. Maven defines several types in the [Default Artifact Handlers Reference](https://maven.apache.org/ref/3.9.11/maven-core/artifact-handlers.html).

For a normal JAR, Maven adds it to the classpath. If the JAR contains `module-info.class`, Maven adds it to the module path, assuming that we use [Java Modules](https://www.baeldung.com/java-modularity). Maven 4 adds `<type>classpath-jar</type>` and `<type>module-jar</type>` to control this explicitly.

We also get new types to simplify annotation processor registration: `processor`, `classpath-processor`, and `modular-processor`. We just add dependencies instead of configuring plugins such as the `maven-compiler-plugin`.

For example, with Lombok and Maven 4, we could write:

```xml
<project>

  <properties>
    <lombok.version>1.18.42</lombok.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>${lombok.version}</version>
      <type>classpath-processor</type>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>${lombok.version}</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

</project>
```

We need both dependencies because Maven 4 separates API and processor classpaths.

In future releases, Maven might extend the list of types, e.g. for extending the taglet or doclet path.

> [!NOTE]
> This is currently only supported by the `maven-compiler-plugin` version `4.0.0-beta3` and above. Other plugins do not support it at the moment!

---
## Subprojects

Maven Modules got confusing when Java 9 introduced [Java Modules](https://www.baeldung.com/java-modularity). As a consequence, Maven 4 renames the term _Modules_ to _Subprojects_. We now have a `<subprojects>` element in the POM, while `<modules>` is marked as deprecated:

```xml
<subprojects>
  <subproject>project-a</subproject>
  <subproject>project-b</subproject>
</subprojects>
```

And there are some other improvements:

- **Parent Inference:** We can use an empty `<parent/>` with no coordinates. Maven infers it from the Aggregator POM.
- **Subprojects Discovery:** We can omit `<subprojects>`, and Maven finds all subfolders with a `pom.xml`.
- **Consistent Timestamps** in packaged archives for all subprojects
- **Safe Deployment:** If one subproject fails, the others are not deployed to repositories too.

---
## Lifecycle Improvements

Maven 4 introduces a tree-based lifecycle. Each project moves through phases independently, and dependent projects start once dependencies reach their ready phase. This boosts performance for multi-project builds. We enable it with:

```bash
mvn -b concurrent verify
```

Each phase now has `before:` and `after:` hooks. These replace older `pre-` and `post-` phases, that are now aliases for backwards compatibility. For example, `maven-gpg-plugin:sign` can bind to `before:install` instead of `verify`, which would be semantically correct. `after:` hooks only run if the main phase succeeds.

Maven 4 also adds `all` and `each` phases with hooks. These help us run logic once per project tree or around each phase, making multi-project builds more predictable.

---
## Further Changes

Some smaller changes affect upgrades:

- Activating a non-existing profile with `-PmyProfile` fails. Use `-P?myProfile` to make the profile optional and avoid failure.
- We can use `--fail-on-severity` or `-fos` to fail builds at a log level.

We now look at a few more changes that improve flexibility in Maven 4.

### Condition-Based Profile Activation

Maven 4 adds a `<condition>` element for profile activation. This allows to declare complex expressions using functions:

```xml
<project>

  <!-- ... -->

  <profiles>
    <profile>
      <id>conditional-profile</id>
      <activation>
        <!-- use <condition><![CDATA[...]]></condition> to avoid encoding XML entities -->
        <condition>exists('${project.basedir}/src/** /*.xsd') &amp;&amp; length(${user.name}) > 5</condition>
      </activation>
      <!-- ... -->
    </profile>
  </profiles>

</project>
```

### Source Directories

In Maven 3, we declare custom source roots with two dedicated elements. This works, but it is limited when we want multiple directories or fine-grained filtering:

```xml
<project>
  <!-- Maven 3 -->
  <build>
    <sourceDirectory>my-custom-dir/foo</sourceDirectory>
    <testSourceDirectory>my-custom-dir/bar</testSourceDirectory>
  </build>
</project>
```

Maven 4 introduces a unified <sources> section. We can list as many source roots as we need, assign a scope, and prepare more advanced setups without extra plugins:

```xml
<project>
  <!-- Maven 4 -->
  <build>
    <sources>
      <source>
        <scope>main</scope>
        <directory>my-custom-dir/foo</directory>
      </source>
      <source>
        <scope>test</scope>
        <directory>my-custom-dir/bar</directory>
      </source>
    </sources>
  </build>
</project>
```

This new model scales well for multi-release projects, module-based layouts, and cases where we want to apply include or exclude rules directly in the POM.

---
## Maven 4 Upgrade Tool

Maven 4 adds an [Upgrade Tool](https://maven.apache.org/tools/mvnup.html) to migrate projects from Maven 3. It scans POMs, plugins, and structure, then recommends updates.

We can run:

```bash
# dry run, only report
mvnup check
# migrate the POMs
mvnup apply
```

---
## Conclusion

Maven 4 modernizes Java builds with a clearer POM model, separated build and consumer artifacts, and better support for modules and annotation processors. The updated lifecycle improves multi-project and concurrent builds. Hooks give us precise control, and smaller changes improve reproducibility.

The Upgrade Tool eases migration from Maven 3. Overall, Maven 4 boosts performance, clarity, and maintainability, laying a solid foundation for modern Java projects.

We can find more details in the [Maven Documentation](https://maven.apache.org/whatsnewinmaven4.html).
