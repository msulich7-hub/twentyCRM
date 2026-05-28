# Tasks: Notatki głosowe + transkrypcja + agent AI

Statusy: `[ ]` todo, `[~]` w toku, `[x]` zrobione.
Powiązany spec: `./SPEC.md`.

## Faza 1 — Nagrywanie + przechowywanie (bez AI)

### Backend — obiekt VoiceNote
- [ ] **V1.1** Wpis `voiceNote` (+ `voiceNoteTarget`) w `STANDARD_OBJECTS`
  (`twenty-shared/.../standard-object.constant.ts`): universalIdentifiers obiektów,
  pól, indeksów, widoków.
- [ ] **V1.2** `VoiceNoteWorkspaceEntity`
  (`packages/twenty-server/src/modules/voice-note/standard-objects/voice-note.workspace-entity.ts`)
  + `SEARCH_FIELDS_FOR_VOICE_NOTE` (title, transcript).
- [ ] **V1.3** `VoiceNoteTargetWorkspaceEntity` (mostek; wzór `TaskTarget`).
- [ ] **V1.4** Buildery metadanych pól/obiektu/widoków + rejestracja w mapach
  (`compute-voice-note-*`, `build-standard-flat-*-metadata-maps`).
- [ ] **V1.5** Rozszerzyć walidację MIME o audio (`audio/webm`, `audio/mp4`/`m4a`,
  `audio/mpeg`, `audio/wav`) — `extract-file-info.utils.ts` / lista supportedMimeTypes.

### Frontend — nagrywanie i odtwarzanie
- [ ] **V1.6** Komponent `VoiceRecorder` (MediaRecorder API): start/stop/pauza, podgląd
  poziomu dźwięku, limit czasu, obsługa uprawnień mikrofonu, etykiety ARIA.
- [ ] **V1.7** Upload nagrania (`uploadFilesFieldFile`) + utworzenie `VoiceNote`
  i powiązanie z aktualnym rekordem (re-use wzorca `useUploadAttachmentFile`).
- [ ] **V1.8** Karta „Voice notes" na rekordzie (wzór `NotesCard`/`FilesCard`) +
  odtwarzacz audio (wzór `call-recording/AudioPlayer`).
- [ ] **V1.9** Wpięcie przycisku nagrywania „wszędzie": karty Person/Company/Opportunity/
  Task/Project + szybkie nagranie globalne.

### Walidacja Fazy 1
- [ ] **V1.10** Typecheck + lint (front+server); smoke-test: nagraj → zapisz → odtwórz,
  notatka widoczna i przypięta do rekordu.

## Faza 2 — Transkrypcja

- [ ] **V2.1** Abstrakcja `TranscriptionProvider` (interfejs `transcribe()`), driver
  factory wzorowany na `file-storage-driver.factory.ts`.
- [ ] **V2.2** Driver OpenAI (Whisper / `gpt-4o-transcribe`) przez `SecureHttpClient`;
  konfiguracja przez env (`OPENAI_API_KEY` już istnieje).
- [ ] **V2.3** Job BullMQ `transcribeVoiceNote`: pobranie audio ze storage → STT →
  zapis `transcript`, ustawienie `transcriptionStatus`, retry + obsługa błędów.
- [ ] **V2.4** Trigger jobu po utworzeniu/uploadzie VoiceNote (event/queue).
- [ ] **V2.5** UI: status transkrypcji (PENDING/PROCESSING/COMPLETED/FAILED),
  podgląd transkryptu w BlockNote, akcja „transkrybuj ponownie".
- [ ] **V2.6** Emit zdarzenia timeline „Voice note transcribed".
- [ ] **V2.7** Testy: mock providera STT, ścieżki sukcesu/błędu/retry.

## Faza 3 — Agent AI → propozycje akcji (human-in-the-loop)

- [ ] **V3.1** Skill `voice-note-to-actions` (wzór `call-transcript-summarization.ts`):
  instrukcje generowania ustrukturyzowanych propozycji (taski/edycje/eventy).
- [ ] **V3.2** Definicja agenta (prompt, model, responseFormat JSON) korzystającego
  z narzędzi `database_crud` (`find_*`/`create_*`/`update_*`).
- [ ] **V3.3** (Decyzja) obiekt `AiSuggestedAction` **lub** przechowanie propozycji w
  `properties` zdarzenia timeline; model + metadane.
- [ ] **V3.4** Workflow trigger: `VoiceNote.transcriptionStatus = COMPLETED` →
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
