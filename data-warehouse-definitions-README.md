# Project Definitions

The `definitions/` directory contains the Dataform actions that implement individual DIR data projects and the shared audit relations used by those projects.

Each project must be isolated under its own lowercase folder:

```text
definitions/
├── audit/
├── letf/
├── <project-a>/
└── <project-b>/
```

The [LETF implementation](letf/README.md) is the current reference implementation. It demonstrates Bronze declarations, Silver current-state upserts, Gold reporting tables, assertions, audit logging, tags, and operational patterns. New projects should reuse the framework conventions but must select data patterns appropriate to their own requirements rather than copying LETF logic unchanged.

## Required project structure

```text
definitions/<project>/
├── README.md
├── bronze/
├── silver/
└── gold/
```

### `bronze/`

Contains declarations for externally managed source-aligned BigQuery tables or views. Bronze transformations should remain minimal and preserve source lineage.

### `silver/`

Contains reusable standardized business entities. Each entity must document its grain, persistence model, business key, deduplication, watermark, late-data, replay, and deletion behavior.

### `gold/`

Contains governed business data products and reporting-ready tables or views. Gold logic must have an identified business owner and documented definitions.

## What belongs outside a project folder

Only genuinely reusable capabilities should be placed at repository scope:

- generic environment resolution;
- generic audit generation;
- generic assertion builders;
- shared CI and dependency configuration;
- shared framework documentation.

Project-specific column dictionaries, source mappings, business rules, SQL helpers, and exception lists should remain with the project implementation or use an explicitly project-namespaced include.

## Project README requirements

Every project README must contain:

1. purpose and scope;
2. ownership and consumers;
3. architecture and datasets;
4. source/Bronze catalog;
5. Silver entity catalog and grain;
6. Gold data-product catalog and rules;
7. pipeline tags, dependencies, schedules, and execution order;
8. audit, assertions, and reconciliation;
9. sensitive-data controls;
10. onboarding and maintenance procedures;
11. backfill, recovery, rollback, and incident runbooks;
12. known limitations and improvement priorities.

## Project onboarding checklist

- [ ] Create the project folder and README.
- [ ] Add independent environment/dataset configuration.
- [ ] Register source contracts as declarations.
- [ ] Choose persistence patterns per entity.
- [ ] Add target creation/migration code.
- [ ] Add audit and quality controls.
- [ ] Add tags and explicit dependencies.
- [ ] Add integration fixtures and test cases.
- [ ] Document schedules, SLAs, owners, and support paths.
- [ ] Add the project to the root project catalog.
