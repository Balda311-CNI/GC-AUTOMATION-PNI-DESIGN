# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`GC-AUTOMATION` is a jBPM **kjar** (Kie archive) — a Maven project whose deliverable is BPMN2 process definitions, not Java code. It is authored in jBPM Business Central (the Process Modeler) and deployed to a Kie Server / jBPM runtime. The `src/main/java` and `src/test/java` trees are placeholders (`.gitkeep` only); all behavior lives in `.bpmn` files under `src/main/resources/com/gc/gc_automation/`.

## Build

```bash
mvn clean install   # produces target/GC-AUTOMATION-1.0.0-SNAPSHOT.jar (kjar)
```

`kie-maven-plugin` validates all BPMN2/DMN/DRL resources at package time — a broken process fails the build. There is no separate lint step and no test command (the `test` scope deps exist but no tests are written).

To run a single jBPM test you would add it under `src/test/java/com/gc/gc_automation/` and invoke `mvn test -Dtest=ClassName#method`; the `kie-api`/`junit`/`xstream` deps already in `pom.xml` are the intended harness.

## Architecture

### Everything is an `SdcApiCall`

The processes never call REST endpoints directly. Every service task has `drools:taskName="SdcApiCall"` and is dispatched to the custom work item handler `cz.sykora.SdcApiWorkItemHandler` (from the `cz.sykora:sdc-jbpm` dependency), wired in `src/main/resources/META-INF/kie-deployment-descriptor.xml`.

The handler is a generic API-call router. Each task passes two conceptual parameter families:

- `routeMap.*` — tells the handler *where* to go: `domain` (e.g. `TMF_STORE`), `operation` (`GET` / `POST`), `transformation` (e.g. a JSONata template name), `source`.
- `dataMap.*` — the payload. Keys beginning `dataMap.jsonata.*` feed into the transformation template.

The endpoint itself comes from the process-scoped global `_apiCallUrl`, which is injected from the deployment descriptor via an MVEL resolver (currently `http://gc-demo.cross-ni.com:8085/api/jbpm/apiCall`). Every SdcApiCall task reads `_apiCallUrl` into its `apiUrl` input. When changing environments, edit the `<global>` in `kie-deployment-descriptor.xml`; do **not** hardcode URLs in the BPMN.

### Work item definitions

`global/WorkDefinitions.wid` and `global/SdcApiCall.wid` register work items so Business Central shows them in the palette. These are metadata only — the runtime binding is the `<work-item-handlers>` block in `kie-deployment-descriptor.xml`. Keep the two in sync when adding a new work item.

### Processes

- `resourceAvailabilityCheck.bpmn` — main synchronous flow. Chains SdcApiCall tasks: `TMF_STORE: get data` (resolve routing data via JSONata) → `REST_API_CALL: get address list by masterId` → validates uniqueness (embedded Java `conditionExpression` on outgoing flows) → `REST_API_CALL: check equipment availability`. Errors set `errorMessage` via `kcontext.setVariable(...)` inside gateway conditions.
- `resourceAvailabilityCheckAsync.bpmn` — async variant of the same flow.

Script tasks and gateway conditions use inline Java (`language="http://www.java.com/java"`). Types allowed without FQN come from `project.imports` (`String`, `Integer`, `Long`, `Double`, `List`, `ArrayList`, `Map`, `LinkedHashMap`, `Collection`, `Number`, `Boolean`).

### Runtime assumptions

`kie-deployment-descriptor.xml` declares `runtime-strategy=SINGLETON` and JPA persistence against `org.jbpm.domain` / `java:jboss/datasources/ExampleDS` — the kjar is intended for a jBPM on JBoss/WildFly environment, not Kogito/Quarkus.

## Editing BPMN files

Each process is a **pair**: `foo.bpmn` (the model) and `GC-AUTOMATION.foo-svg.svg` (the rendered diagram Business Central displays). Business Central regenerates the SVG when you save through the modeler. If you edit `.bpmn` XML by hand, the SVG will go stale — preferred workflow is to round-trip through Business Central so the diagram stays in sync, and commit both files together (recent history shows this pattern: BPMN + matching SVG in each commit).

BPMN element IDs like `_8CB7A114-8CEA-4461-8718-36C5581E42D1` are used as prefixes across dozens of `<itemDefinition>` / `<dataInput>` / `<dataInputAssociation>` entries for a single task. When renaming or duplicating a task by hand, you must rewrite every occurrence of the UUID consistently or the kjar build will fail.
