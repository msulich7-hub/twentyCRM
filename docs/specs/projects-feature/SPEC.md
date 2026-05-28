# Spec: Projekty i zadania w stylu Asany (Projects & Tasks)

> Status: PROPOZYCJA / DRAFT
> Branch: `claude/crm-projects-tasks-ShVYK`
> Autor: Claude Code (analiza na zlecenie msulich7)
> Data: 2026-05-28

## 1. Cel biznesowy

Twenty CRM zawiera już obiekt **Task** (zadanie), ale nie ma pojęcia **Projektu**
— kontenera, który grupuje zadania, ma własny status, terminy, właściciela i zespół,
oraz może być powiązany z klientem (firmą/osobą) albo być projektem czysto wewnętrznym.

Celem jest dodanie zarządzania projektami w stylu Asany, działającego:
- **z klientami** — projekt powiązany z Company / Person / Opportunity,
- **wewnętrznie** — projekt bez klienta, tylko z członkami zespołu (workspace members).

## 2. Co już istnieje (nie budujemy od zera)

Analiza kodu (`packages/twenty-server` + `packages/twenty-front`) wykazała:

- **Obiekt Task** już ma: `title`, `bodyV2` (rich text), `dueAt`, `status`
  (SELECT: `TODO` / `IN_PROGRESS` / `DONE`), `assignee` (→ WorkspaceMember),
  `taskTargets` (polimorficzne powiązanie z Person/Company/Opportunity),
  `attachments`, `timelineActivities`, full-text `searchVector`.
- **Wzorzec polimorficznych powiązań** przez encje-mostki (`TaskTarget`, `NoteTarget`):
  jeden rekord-mostek wskazuje na jeden z wielu możliwych celów.
- **System metadanych jest sterowany danymi (metadata-driven):** frontend
  automatycznie wykrywa nowe standardowe obiekty z metadanych GraphQL — widoki listy,
  Kanban (po polu SELECT), karty rekordu, filtry, wyszukiwanie działają „za darmo".
- **Widoki Kanban** powstają automatycznie z pól typu SELECT — dla projektu
  status da nam tablicę Kanban bez dodatkowego kodu frontowego.

Wniosek: **największą luką jest sam obiekt Project + powiązanie Task → Project.**
Większość wartości w stylu Asany (Kanban, przypisania, terminy, widoki) dostajemy
z istniejącego silnika metadanych.

## 3. Architektura — jak dodaje się standardowy obiekt

W tej wersji Twenty encje workspace to **czyste klasy TS bez dekoratorów**;
metadane (UUID, typy pól, opcje SELECT, widoki) są generowane przez „flat metadata
builders". Dodanie obiektu `Project` wymaga (potwierdzone w kodzie):

1. `packages/twenty-shared/src/metadata/constants/standard-object.constant.ts`
   — wpis `project` z `universalIdentifier` obiektu, pól, indeksów i widoków
   (niezmienne UUID; generujemy nowe).
2. `packages/twenty-server/src/modules/project/standard-objects/project.workspace-entity.ts`
   — klasa `ProjectWorkspaceEntity` + `SEARCH_FIELDS_FOR_PROJECT`.
3. (opcjonalnie) `.../project/standard-objects/project-target.workspace-entity.ts`
   — mostek do klientów (Person/Company/Opportunity), wzorowany na `TaskTarget`.
4. Builder pól: `.../twenty-standard-application/utils/field-metadata/compute-project-standard-flat-field-metadata.util.ts`
   + rejestracja w `build-standard-flat-field-metadata-maps.util.ts`.
5. Builder obiektu: wpis w
   `.../utils/object-metadata/create-standard-flat-object-metadata.util.ts`
   (`STANDARD_FLAT_OBJECT_METADATA_BUILDERS_BY_OBJECT_NAME`).
6. Builder widoków: `.../utils/view/compute-standard-project-views.util.ts`
   + rejestracja w `build-standard-flat-view-metadata-maps.util.ts`
   (co najmniej widok „All Projects" + Kanban po statusie).
7. (opcjonalnie) builder indeksów dla pól FK / searchVector.
8. **Modyfikacja Taska:** dodać relację `project` (MANY_TO_ONE) na `Task`
   oraz `tasks` (ONE_TO_MANY) na `Project`. To dotyka `task.workspace-entity.ts`,
   buildera pól Taska oraz wpisów `STANDARD_OBJECTS.task`/`.project`.
9. (opcjonalnie) prefill / dane demo:
   `.../workspace-manager/standard-objects-prefill-data/prefill-projects.util.ts`.
10. Synchronizacja metadanych do workspace'ów następuje przez
    `TwentyStandardApplicationService.synchronizeTwentyStandardApplicationOrThrow()`
    (porównanie stanu „from"→„to" i migracje). W razie potrzeby — instance/workspace
    command zgodnie z `packages/twenty-server/docs/UPGRADE_COMMANDS.md`.

**Frontend: brak ręcznych zmian dla podstawowego CRUD** — obiekt pojawi się
automatycznie w nawigacji, listach, Kanbanie i kartach rekordu.

## 4. Model danych (proponowany)

### Obiekt `Project`
| Pole | Typ | Uwagi |
|------|-----|-------|
| `name` | TEXT | label identifier |
| `bodyV2` | RICH_TEXT | opis projektu |
| `status` | SELECT | `PLANNED`, `IN_PROGRESS`, `ON_HOLD`, `COMPLETED`, `CANCELLED` (zasila Kanban) |
| `startDate` | DATE_TIME | data startu |
| `dueDate` | DATE_TIME | termin |
| `owner` | RELATION (MANY_TO_ONE → WorkspaceMember) | właściciel projektu |
| `position` | POSITION | kolejność |
| `tasks` | RELATION (ONE_TO_MANY → Task) | zadania w projekcie |
| `projectTargets` | RELATION (ONE_TO_MANY → ProjectTarget) | powiązani klienci (opcjonalne) |
| `attachments` | RELATION (ONE_TO_MANY → Attachment) | pliki |
| `timelineActivities` | RELATION | log aktywności |
| `createdBy` / `updatedBy` | ACTOR | audyt |
| `searchVector` | TS_VECTOR | wyszukiwanie po `name` |

### Zmiana w `Task`
| Pole | Typ | Uwagi |
|------|-----|-------|
| `project` | RELATION (MANY_TO_ONE → Project, onDelete SET_NULL) | przynależność zadania do projektu |
| `projectId` | UUID | FK |

### `ProjectTarget` (mostek do klientów — opcjonalny, faza 2)
Wzorowany 1:1 na `TaskTarget`: `project`, `targetPerson`, `targetCompany`,
`targetOpportunity`, `custom`.

## 5. Zakres MVP vs. fazy

**Faza 1 (MVP — rekomendowana na start):**
- Obiekt `Project` z polami: name, bodyV2, status, startDate, dueDate, owner, position.
- Relacja `Task.project` ↔ `Project.tasks`.
- Domyślne widoki: lista „All Projects" + Kanban po `status`.
- Wyszukiwanie + searchVector.
- Działa wewnętrznie i (przez istniejące taskTargets na zadaniach) pośrednio z klientami.

**Faza 2 (klienci):**
- `ProjectTarget` (powiązanie projektu z Company/Person/Opportunity).
- Sekcje na firmie/osobie pokazujące powiązane projekty.

**Faza 3 (zaawansowane, „pełna Asana"):**
- Milestones (kamienie milowe), zależności zadań, podzadania, szablony projektów,
  postęp % liczony z zadań, prefill danych demo.

## 6. Ryzyka / decyzje otwarte
- **Migracje istniejących workspace'ów:** czy potrzebny dedykowany workspace command,
  czy wystarczy sync standardowej aplikacji. Do potwierdzenia testem na czystej bazie.
- **UUID-y:** muszą być globalnie unikalne i niezmienne — generujemy raz.
- **Nazewnictwo i kolory statusów** — do akceptacji produktowej.
- **Czy `owner` to pojedynczy member, czy `members` (wielu)** — MVP: pojedynczy owner;
  współdzielenie i tak realizujemy przez assignee zadań.

## 7. Walidacja / testy
- `npx nx typecheck twenty-server` + `lint:diff-with-main`.
- Reset bazy + sync metadanych: sprawdzić, że `project` pojawia się w metadanych,
  ma widoki, Kanban, i że Task ma pole `project`.
- (Faza 2+) testy integracyjne CRUD przez GraphQL.
