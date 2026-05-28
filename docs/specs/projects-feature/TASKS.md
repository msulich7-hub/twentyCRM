# Tasks: Projekty i zadania (Projects & Tasks)

Rozbicie pracy na zadania. Statusy: `[ ]` todo, `[~]` w toku, `[x]` zrobione.

## Faza 1 — MVP: obiekt Project + powiązanie z Task

### Backend — metadane standardowego obiektu
- [ ] **T1.1** Dodać wpis `project` do `STANDARD_OBJECTS`
  (`packages/twenty-shared/src/metadata/constants/standard-object.constant.ts`):
  universalIdentifier obiektu, pól (name, bodyV2, status, startDate, dueDate, owner,
  position, tasks, attachments, timelineActivities, searchVector, createdBy, updatedBy,
  + pola bazowe), indeksów (ownerId, searchVector GIN) i widoków (allProjects + Kanban).
- [ ] **T1.2** Utworzyć `ProjectWorkspaceEntity`
  (`packages/twenty-server/src/modules/project/standard-objects/project.workspace-entity.ts`)
  + eksport `SEARCH_FIELDS_FOR_PROJECT`.
- [ ] **T1.3** Builder pól `compute-project-standard-flat-field-metadata.util.ts`
  (wszystkie pola, opcje SELECT statusu, relacje owner/tasks/attachments/timeline).
- [ ] **T1.4** Zarejestrować builder pól w `build-standard-flat-field-metadata-maps.util.ts`.
- [ ] **T1.5** Dodać builder obiektu w `create-standard-flat-object-metadata.util.ts`
  (ikona, labelSingular/Plural, labelIdentifier = `name`, isSearchable, shortcut).
- [ ] **T1.6** Builder widoków `compute-standard-project-views.util.ts`
  (widok listy „All Projects" + widok Kanban grupowany po `status`)
  + rejestracja w `build-standard-flat-view-metadata-maps.util.ts`.
- [ ] **T1.7** (jeśli wymagane) builder indeksów dla `ownerId` i `searchVector`.

### Backend — powiązanie Task ↔ Project
- [ ] **T1.8** Dodać do `TaskWorkspaceEntity` pola `project` (MANY_TO_ONE) + `projectId`.
- [ ] **T1.9** Dodać metadane relacji w buildery pól Taska + odpowiednie
  universalIdentifiers w `STANDARD_OBJECTS.task` i stronę ONE_TO_MANY `tasks` w `project`.

### Walidacja
- [ ] **T1.10** `npx nx typecheck twenty-server` + `npx nx lint:diff-with-main twenty-server`.
- [ ] **T1.11** Reset bazy + sync; potwierdzić w metadanych obecność `project`,
  jego widoki, Kanban i pole `Task.project` (Postgres MCP / GraphQL).
- [ ] **T1.12** Smoke-test UI: nowy obiekt „Projects" w nawigacji, tworzenie projektu,
  przypisanie zadania do projektu, Kanban po statusie.

## Faza 2 — Powiązanie z klientami
- [ ] **T2.1** `ProjectTargetWorkspaceEntity` (wzór: `TaskTarget`) + metadane.
- [ ] **T2.2** Relacja `Project.projectTargets`.
- [ ] **T2.3** Powiązani klienci widoczni na karcie projektu i odwrotnie
  (projekty na karcie Company/Person) — weryfikacja, że metadata-driven UI to pokazuje.
- [ ] **T2.4** Testy integracyjne CRUD + powiązań przez GraphQL.

## Faza 3 — Zaawansowane (opcjonalnie)
- [ ] **T3.1** Milestones (nowy obiekt powiązany z Project) lub pole SELECT fazy.
- [ ] **T3.2** Podzadania / zależności zadań.
- [ ] **T3.3** Postęp % projektu liczony z ukończonych zadań.
- [ ] **T3.4** Szablony projektów + prefill danych demo (`prefill-projects.util.ts`).
