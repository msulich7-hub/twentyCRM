# Spec: Notatki głosowe + transkrypcja + agent AI → akcje (Voice Notes & AI Actions)

> Status: PROPOZYCJA / DRAFT
> Branch: `claude/crm-projects-tasks-ShVYK`
> Autor: Claude Code (analiza na zlecenie msulich7)
> Data: 2026-05-28

## 1. Cel biznesowy

Umożliwić użytkownikom nagrywanie **roboczych notatek głosowych** bezpośrednio przy
zdarzeniach z klientami — wszędzie tam, gdzie system rejestruje aktywność (karta firmy,
osoby, szansy, zadania, projektu, timeline). System **transkrybuje** nagranie, a w dalszej
roadmapie **agent AI** przekształca transkrypt w konkretne działania:
- tworzenie nowych **zadań** / **akcji**,
- **edycję** istniejących rekordów i zdarzeń,
- tworzenie / aktualizację **wydarzeń** (events) w różnych miejscach CRM.

Notatka głosowa = szybki, mobilny sposób zapisu kontekstu „w biegu" (po spotkaniu,
telefonie, wizycie), który CRM sam zamienia w ustrukturyzowane dane.

## 2. Co już istnieje (fundament — NIE budujemy od zera)

Potwierdzone w kodzie (`packages/twenty-server`, `packages/twenty-front`, `packages/twenty-apps`):

- **Upload i storage plików:** sterowniki Local/S3 (`file-storage-driver.factory.ts`),
  walidacja MIME z zawartości pliku (`extract-file-info.utils.ts`), mutacja
  `uploadFilesFieldFile`, obiekt `Attachment` przypinany do Person/Company/Task/Note/
  Opportunity/Dashboard/Workflow (FK `target*Id`).
- **Edytor rich-text BlockNote** — body notatek/tasków (`bodyV2: RichTextMetadata`).
- **Timeline aktywności** (`timeline-activity.workspace-entity.ts`) — zdarzenia przy
  rekordach, renderowane w `TimelineCard`.
- **System agentów AI** (`engine/metadata-modules/ai/`): `AgentAsyncExecutorService`
  używa `ai` SDK (`generateText` + `ToolSet`), billing tokenów, role/uprawnienia.
- **Narzędzia (tool-calling):** `tool-provider/` — `database_crud` auto-generuje
  `create_*`, `update_*`, `delete_*`, `find_*` dla każdego obiektu → agent może
  tworzyć/edytować taski i dowolne rekordy. Plus HTTP, Email, Code Interpreter, MCP.
- **Providerzy LLM:** OpenAI / Anthropic / Google / Mistral / xAI (`ai-providers.json`,
  klucze przez `*_API_KEY`).
- **Silnik workflow:** akcja `ai-agent.workflow-action.ts` (uruchamia agenta),
  `create/update/delete/upsert-record.workflow-action.ts`, `tool-executor-workflow-action.ts`.
- **Wzorzec referencyjny audio+transkrypcja:** apka `twenty-apps/internal/call-recording`
  (`recordingFile`, `transcriptFile`, `transcript` RICH_TEXT, `summary`, `status`)
  + skill `call-transcript-summarization.ts`; integracja Fireflies przez `SecureHttpClient`.
- **SecureHttpClient** (`secure-http-client.service.ts`) — bezpieczne wywołania API
  zewnętrznych (z ochroną SSRF) → do wywołania API STT.
- **Kolejki BullMQ** — przetwarzanie zadań w tle (idealne do asynchronicznej transkrypcji).

## 3. Główne luki (co dobudowujemy)

1. **Nagrywanie głosu w UI** — komponent `MediaRecorder` (web) + upload; brak obecnie.
2. **Silnik transkrypcji (STT)** — brak natywnego; potrzebna **abstrakcja providera**
   (driver pattern jak storage): OpenAI Whisper / `gpt-4o-transcribe`, AssemblyAI, Deepgram.
3. **Generyczny obiekt `VoiceNote`** — przypinany wszędzie (wzór `Note`/`call-recording`).
4. **Pętla „transkrypt → proponowane akcje → akceptacja człowieka → wykonanie"** przez agenta.

## 4. Najlepsze praktyki / zasady projektowe (cel 10/10)

- **Asynchroniczność:** nagranie → upload → job transkrypcji w tle (BullMQ), UI pokazuje
  status (`PENDING/PROCESSING/COMPLETED/FAILED`) bez blokowania.
- **Abstrakcja providera STT** (jak `file-storage-driver.factory`): łatwa zmiana
  Whisper↔AssemblyAI↔Deepgram przez env, bez zmian w logice.
- **Human-in-the-loop dla AI:** agent **proponuje** działania (draft/preview),
  użytkownik **akceptuje** zanim cokolwiek zostanie zapisane/zmienione w CRM.
  Auto-mutacja danych klienta bez akceptacji = ryzyko; domyślnie wyłączona.
- **Re-use, nie reinvent:** akcje wykonujemy istniejącymi narzędziami `database_crud`
  i akcjami workflow (create/update record), nie piszemy własnego CRUD.
- **Bezpieczeństwo:** audio jako pliki w istniejącym storage z walidacją MIME; STT przez
  `SecureHttpClient`; klucze API STT jak pozostałe sekrety.
- **Prywatność/zgody:** flaga zgody na nagrywanie + retencja audio (konfigurowalna).
- **Idempotencja i retry:** job transkrypcji z limitem prób i statusem błędu.
- **i18n + dostępność:** Lingui, etykiety ARIA dla nagrywania.
- **Koszty/limity:** transkrypcja i tokeny agenta liczone w istniejącym billingu (`AiBillingService`).

## 5. Model danych (DECYZJA: reuse istniejącego `Note` + audio `Attachment`)

> Decyzja produktowa: **nie tworzymy osobnego obiektu VoiceNote.** Notatka głosowa to
> istniejący obiekt **`Note`** (z jego powiązaniami `noteTargets`, body `bodyV2`,
> timeline, attachmentami) wzbogacony o nagranie audio jako **`Attachment`** (FK
> `targetNoteId`). Transkrypt trafia do `Note.bodyV2` (rich text/BlockNote).
>
> Zalety: zero nowego obiektu, notatka od razu przypina się **wszędzie** przez istniejący
> `NoteTarget` (Person/Company/Opportunity/…), działają widoki, wyszukiwanie, timeline.

### Wykorzystanie istniejących obiektów
| Element | Gdzie | Uwagi |
|------|------|-------|
| Treść/transkrypt | `Note.bodyV2` (RICH_TEXT) | wynik transkrypcji wstawiany do body |
| Tytuł | `Note.title` (TEXT) | auto z daty/kontekstu, edytowalny |
| Powiązanie z klientem | `NoteTarget` (Person/Company/Opportunity) | bez zmian — działa „wszędzie" |
| Nagranie audio | `Attachment` z `targetNoteId` | re-use uploadu plików |
| Ślad zdarzenia | `TimelineActivity` | event „voice note transcribed" |

### Minimalne rozszerzenie `Note` (potrzebne do śledzenia transkrypcji)
Aby śledzić stan transkrypcji bez nowego obiektu, dodajemy do standardowego `Note`
dwa **opcjonalne** pola (nowe universalIdentifiers, nieinwazyjne dla istniejących notatek):
| Pole | Typ | Uwagi |
|------|-----|-------|
| `transcriptionStatus` | SELECT (nullable) | `PENDING`/`PROCESSING`/`COMPLETED`/`FAILED`; puste dla zwykłych notatek |
| `isVoiceNote` | BOOLEAN (default false) | odróżnia notatki głosowe od pisanych (filtrowanie/UI) |

> Alternatywa rozważana: trzymać status wyłącznie efemerycznie (w jobie + zdarzeniu
> timeline) bez dodawania pól do `Note`. Minusy: brak trwałego statusu w UI/liście,
> trudniejsze ponowienie transkrypcji. Rekomendacja: dodać 2 powyższe pola.

### `AiSuggestedAction` (faza 3 — propozycje agenta do akceptacji)
| Pole | Typ | Uwagi |
|------|-----|-------|
| `sourceVoiceNote` | RELATION → VoiceNote | źródło |
| `actionType` | SELECT | `CREATE_TASK`/`UPDATE_RECORD`/`CREATE_EVENT`/`CREATE_NOTE`… |
| `payload` | RAW_JSON | proponowane dane (np. tytuł taska, pola do zmiany) |
| `targetObjectMetadataId` / `targetRecordId` | TEXT/UUID | czego dotyczy edycja |
| `status` | SELECT | `SUGGESTED`/`ACCEPTED`/`REJECTED`/`APPLIED`/`FAILED` |
| `confidence` | NUMBER | pewność modelu (0–1) |

(Alternatywa minimalna: trzymać propozycje w `properties` zdarzenia timeline + UI panel,
bez nowego obiektu. Decyzja w fazie 3.)

## 6. Architektura przepływu

```
[UI: nagraj/wgraj audio]  →  uploadFilesFieldFile  →  utwórz Note (isVoiceNote=true,
        │                       transcriptionStatus=PENDING) + Attachment(targetNoteId) + NoteTarget(klient)
        ▼
[BullMQ job: transcribeNote]  →  pobierz audio ze storage
        │                         →  TranscriptionService (driver: Whisper/AssemblyAI/Deepgram)
        ▼
zapis transcript → Note.bodyV2, transcriptionStatus=COMPLETED → emit timeline event
        │
        ▼  (Faza 3 — opcjonalny trigger workflow)
[Workflow: on Note.transcriptionStatus = COMPLETED && isVoiceNote]
        │   → akcja AI Agent (agent "voice-note-to-actions" + skill)
        │   → agent czyta transcript, generuje listę propozycji (NIE zapisuje od razu)
        ▼
[UI: panel "Sugerowane akcje"] → użytkownik akceptuje/odrzuca   (HUMAN-IN-THE-LOOP — wymagane)
        │   → DOPIERO na akceptację: wykonanie przez database_crud / akcje workflow
        ▼
utworzone taski / zmienione rekordy / nowe eventy + ślad na timeline
```

## 7. Wybór technologii STT (rekomendacja)
- **Domyślnie OpenAI** (`gpt-4o-transcribe` / Whisper) — provider OpenAI już skonfigurowany
  w repo (klucz `OPENAI_API_KEY`), więc najmniej tarcia.
- **AssemblyAI / Deepgram** jako alternatywni driverzy (diaryzacja mówców, znaczniki czasu).
- Abstrakcja: `TranscriptionProvider` z metodą `transcribe(audioStream, opts)` → `{ text, language, segments }`.

## 8. Zakres / fazy

**Faza 1 — Nagrywanie + przechowywanie (bez AI):**
- Komponent nagrywania głosu (MediaRecorder) + upload audio jako `Attachment` (targetNoteId).
- Utworzenie `Note` (isVoiceNote=true) + powiązanie `NoteTarget` z aktualnym rekordem.
- Dodanie 2 pól do `Note` (`transcriptionStatus`, `isVoiceNote`).
- Odtwarzacz audio na karcie notatki; body na transkrypt; filtr „voice notes".
- Rozszerzyć dozwolone MIME o audio (`audio/webm`, `audio/m4a`, `audio/mpeg`, `audio/wav`).

**Faza 2 — Transkrypcja:**
- `TranscriptionService` + driver OpenAI (Whisper/gpt-4o-transcribe) przez `SecureHttpClient`.
- Job BullMQ `transcribeVoiceNote` (status, retry, błędy).
- Zapis transkryptu do `VoiceNote.transcript`, event na timeline, podgląd w UI.

**Faza 3 — Agent AI → propozycje akcji (human-in-the-loop):**
- Agent + skill `voice-note-to-actions` (wzór `call-transcript-summarization`).
- Trigger workflow po `COMPLETED`; agent generuje propozycje (`AiSuggestedAction`).
- UI „Sugerowane akcje": akceptuj/odrzuć → wykonanie przez istniejące tools/akcje.

**Faza 4 — Zaawansowane:**
- Auto-dopasowanie notatki do właściwego klienta/rekordu (NER/embeddingi).
- Diaryzacja mówców, znaczniki czasu, wielojęzyczność, podsumowania.
- Komendy głosowe („utwórz zadanie…"), nagrywanie z poziomu mobile.
- Konfigurowalna retencja audio + zgody (compliance).

## 9. Ryzyka / decyzje
- **Model danych (ZDECYDOWANE):** reuse `Note` + audio `Attachment` (bez nowego obiektu);
  dodajemy 2 opcjonalne pola do `Note`. Uwaga: zmiana standardowego `Note` wymaga
  ostrożnych universalIdentifiers i nie może psuć istniejących notatek.
- **Autonomia agenta (ZDECYDOWANE):** wyłącznie **propozycje + akceptacja człowieka**;
  brak auto-mutacji danych. Wykonanie tylko po zatwierdzeniu w UI.
- **Provider STT i koszty** — domyślnie OpenAI; potwierdzić budżet/limit.
- **Retencja i zgody na nagrania** — wymóg prawny zależny od rynku.
- **Mobile / nagrywanie w tle** — czy w zakresie web MVP, czy później.

## 10. Walidacja / testy
- Typecheck + `lint:diff-with-main` (front + server).
- Test uploadu audio i utworzenia `VoiceNote` (integracyjny GraphQL).
- Mock providera STT → test jobu transkrypcji (status, retry, błąd).
- Test agenta na przykładowym transkrypcie → poprawne propozycje akcji (bez auto-zapisu).
- E2E: nagranie → transkrypt → propozycja taska → akceptacja → task utworzony.
