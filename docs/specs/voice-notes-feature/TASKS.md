# Tasks: Notatki głosowe + transkrypcja + agent AI

Statusy: `[ ]` todo, `[~]` w toku, `[x]` zrobione.
Powiązany spec: `./SPEC.md`.

## Faza 1 — Nagrywanie + przechowywanie (bez AI)

> Decyzja: reuse `Note` + audio jako `Attachment` (bez nowego obiektu).

### Backend — rozszerzenie Note
- [ ] **V1.1** Dodać 2 pola do `Note` (nowe universalIdentifiers w `STANDARD_OBJECTS.note`):
  `transcriptionStatus` (SELECT nullable: PENDING/PROCESSING/COMPLETED/FAILED) oraz
  `isVoiceNote` (BOOLEAN default false). Bez zmiany istniejących pól.
- [ ] **V1.2** Uzupełnić `NoteWorkspaceEntity` + builder pól Note o powyższe pola.
- [ ] **V1.3** Rozszerzyć walidację MIME o audio (`audio/webm`, `audio/mp4`/`m4a`,
  `audio/mpeg`, `audio/wav`) — `extract-file-info.utils.ts` / lista supportedMimeTypes.

### Frontend — nagrywanie i odtwarzanie
- [ ] **V1.4** Komponent `VoiceRecorder` (MediaRecorder API): start/stop/pauza, podgląd
  poziomu dźwięku, limit czasu, obsługa uprawnień mikrofonu, etykiety ARIA.
- [ ] **V1.5** Po nagraniu: `uploadFilesFieldFile` audio → utworzyć `Note`
  (isVoiceNote=true) + `Attachment` (targetNoteId) + `NoteTarget` z aktualnym rekordem
  (re-use `useUploadAttachmentFile` + hooków tworzenia notatki).
- [ ] **V1.6** Odtwarzacz audio na karcie notatki (wzór `call-recording/AudioPlayer`);
  body notatki jako miejsce na transkrypt.
- [ ] **V1.7** Przycisk nagrywania „wszędzie" przy notatkach: karty Person/Company/
  Opportunity/Task/Project + szybkie nagranie globalne; filtr/oznaczenie „voice note".

### Walidacja Fazy 1
- [ ] **V1.8** Typecheck + lint (front+server); smoke-test: nagraj → zapisz → odtwórz,
  notatka (głosowa) widoczna i przypięta do rekordu.

## Faza 2 — Transkrypcja

- [ ] **V2.1** Abstrakcja `TranscriptionProvider` (interfejs `transcribe()`), driver
  factory wzorowany na `file-storage-driver.factory.ts`.
- [ ] **V2.2** Driver OpenAI (Whisper / `gpt-4o-transcribe`) przez `SecureHttpClient`;
  konfiguracja przez env (`OPENAI_API_KEY` już istnieje).
- [ ] **V2.3** Job BullMQ `transcribeNote`: pobranie audio (Attachment) ze storage → STT →
  zapis transkryptu do `Note.bodyV2`, ustawienie `Note.transcriptionStatus`, retry + błędy.
- [ ] **V2.4** Trigger jobu po utworzeniu notatki głosowej / uploadzie audio (event/queue).
- [ ] **V2.5** UI: status transkrypcji (PENDING/PROCESSING/COMPLETED/FAILED),
  podgląd transkryptu w BlockNote, akcja „transkrybuj ponownie".
- [ ] **V2.6** Emit zdarzenia timeline „Voice note transcribed".
- [ ] **V2.7** Testy: mock providera STT, ścieżki sukcesu/błędu/retry.

## Faza 3 — Agent AI → propozycje akcji (human-in-the-loop)

> Decyzja: wyłącznie propozycje + akceptacja człowieka (brak auto-mutacji).

- [ ] **V3.1** Skill `voice-note-to-actions` (wzór `call-transcript-summarization.ts`):
  instrukcje generowania ustrukturyzowanych propozycji (taski/edycje/eventy).
- [ ] **V3.2** Definicja agenta (prompt, model, responseFormat JSON) korzystającego
  z narzędzi `database_crud` w trybie read/propose (find_*) — bez wywołań mutujących.
- [ ] **V3.3** (Decyzja) obiekt `AiSuggestedAction` **lub** przechowanie propozycji w
  `properties` zdarzenia timeline; model + metadane.
- [ ] **V3.4** Workflow trigger: `Note.transcriptionStatus = COMPLETED && isVoiceNote` →
  akcja `ai-agent.workflow-action` generująca propozycje (bez auto-zapisu).
- [ ] **V3.5** UI panel „Sugerowane akcje": lista propozycji, akceptuj/odrzuć/edytuj.
- [ ] **V3.6** Wykonanie zaakceptowanych propozycji przez istniejące akcje
  create/update-record / tool executor; ślad na timeline; status `APPLIED`.
- [ ] **V3.7** Testy: transkrypt → poprawne propozycje (bez mutacji), akceptacja → wykonanie.

## Faza 4 — Zaawansowane (opcjonalnie)
- [ ] **V4.1** Auto-dopasowanie notatki do właściwego klienta/rekordu (NER/embeddingi).
- [ ] **V4.2** Diaryzacja mówców + znaczniki czasu (driver AssemblyAI/Deepgram).
- [ ] **V4.3** Wielojęzyczność + automatyczne podsumowania.
- [ ] **V4.4** Komendy głosowe / nagrywanie z mobile.
- [ ] **V4.5** Konfigurowalna retencja audio + zgody na nagrywanie (compliance).
