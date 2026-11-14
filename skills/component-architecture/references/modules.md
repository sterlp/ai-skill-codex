# Modules: cutting the build by domain

A module is a build unit; a domain component is a domain unit (see `SKILL.md`, "Terms"). Modules exist so the
compiler enforces the dependency direction physically and an agent can work on one module with a small
context. Cutting a module is **always a proposal to the user**, done as its own preparation iteration.

## When to propose a module

1. **Domain threshold:** ≥ 10 domain components in one module — with every new domain component, check whether
   it (and related existing ones) form a domain module.
2. **Foreign capability:** the code has nothing to do with the host's domain (audit, docs hosting, docs
   assistant, a generic stub framework). Extract it early, even when small — it is reusable and keeps the
   domain module clean.
3. **Order:** (a) new module directly, if the new feature is clearly separable; (b) tidy up first, if the
   existing module is unclear and the new feature does not need a module yet; (c) both: tidy up, then cut.

## Module kinds

| Kind | Example | Contains | Depends on |
|---|---|---|---|
| Domain module | `customer` | related domain components of one domain (resource, service, functions, repository, connector, entities) | own api module, api modules and services of other domain modules, `shared` |
| api module | `customer-api` | API classes of exactly one domain module + its version-stable value types (enums, strong IDs) | nothing but validation/Jackson annotations |
| Shared module | `shared` | technical tools without domain logic | nothing domain-specific |
| Plug-in module | `docs-assistant` | one capability, auto-configured | framework only — never the host |
| Build module | `frontend`, `docs-site` | builds an artifact the app ships (copied into `static/`) | its toolchain |
| App module | `backend`/`app` | configuration and wiring, security, application class | all of the above |

Rules:

1. **Same names everywhere:** a domain component has the same name as backend package and UI folder; a
   module is named after its domain (case per language).
2. **Acyclic between modules.** If two domain modules need each other, the cut is wrong, or one direction
   becomes an event/ID (see `SKILL.md`, access rule 6).
3. **One api module per domain module** as soon as there is more than one domain module. Before that, a single
   `api` module with one package per domain component is fine (`<base>.api.<component>`).
4. **Reactor order:** api/shared modules → build modules → domain/plug-in modules → app module.
5. ArtifactId and folder prefixes are a project convention (Step 0).

## Generate the UI types from the api module

The UI never hand-writes server types. One `typescript-generator-maven-plugin` execution **per domain
component**: `classPatterns` = its package in the api module, `outputFile` = its UI folder. An api module
with three domain components has three executions. A changed API class then breaks the UI build, not
production.

```xml
<plugin>
  <groupId>cz.habarta.typescript-generator</groupId>
  <artifactId>typescript-generator-maven-plugin</artifactId>
  <executions>
    <execution>
      <id>generate customer-api</id>
      <goals><goal>generate</goal></goals>
      <configuration>
        <classPatterns><pattern>com.example.app.api.customer.**</pattern></classPatterns>
        <jsonLibrary>jackson2</jsonLibrary> <!-- match the project's Jackson setup -->
        <outputKind>module</outputKind>
        <outputFile>${project.basedir}/../frontend/src/customer/customer-api.d.ts</outputFile>
        <noFileComment>true</noFileComment>
        <primitivePropertiesRequired>true</primitivePropertiesRequired>
        <requiredAnnotations>
          <requiredAnnotation>jakarta.validation.constraints.NotNull</requiredAnnotation>
          <requiredAnnotation>jakarta.validation.constraints.NotBlank</requiredAnnotation>
          <requiredAnnotation>jakarta.validation.constraints.NotEmpty</requiredAnnotation>
        </requiredAnnotations>
      </configuration>
    </execution>
  </executions>
</plugin>
```

- `requiredAnnotations`: a field validated as required is non-optional in TypeScript — the UI does not
  have to null-check what the server guarantees.
- Framework types the UI needs (e.g. Spring `Page`/`PageRequest`) get their own execution into `src/shared/`.
- Legacy only: a project may still generate one single file for everything (e.g.
  `src/shared/types/server-api.d.ts`). Move to one file per domain component when touching the generator
  or at the first module split.
- Generated files are never edited by hand. Committing them or not is a project convention.

## Plug-in modules

A plug-in module brings a capability into a host app without knowing the host. Reference: the docs assistant
of the skill `publish-docs-with-spring-boot` (`assets/docs-assistant/`).

1. **Own package root**, not the host's (`de.company.utilities.<plugin>`), so the host's component scan does
   not pick it up by accident.
2. **Registers itself** via Spring Boot auto-configuration
   (`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`). No `@Component`,
   no component scan, no `@Import` needed in the host.
3. **The host owns** security (matchers, roles), configuration and secrets, and the data the plug-in reads.
   The plug-in brings none of these.
4. **Extension points are beans with a default** (`@ConditionalOnMissingBean`), overridden by the host
   (e.g. how the reader's name is resolved).
5. **Own tests** with its own test application, test resources and stubs — `mvn -pl <plugin> verify` runs
   without the host. The host only adds a wiring test (security, status codes).
6. **Owner reports back:** the plug-in returns results/reasons; the host counts, retries and maps errors.
7. **Copied plug-ins stay identical to their source.** Change the source (e.g. the skill asset), then copy
   again — never fork silently.

## Module docs live in the module

Each module has `<module>/docs/` with its feature doc, architecture doc and ADRs. The root docs only
mention the module by path. The architecture doc names the responsibilities ("one owner per
responsibility"), not the code line by line. Spring Modulith's PlantUML output
(`Documenter.writeModulesAsPlantUml()`) goes there as well.

Example of "one owner per responsibility" for docs hosting (from `publish-docs-with-spring-boot`):
the `docs-site` build module owns the build, a `DocsConfig` owns delivery, the security config owns access,
the UI only links. No responsibility is split across two places.
