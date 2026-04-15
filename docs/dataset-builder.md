# Dataset Builder — łączenie danych z integracji, Google Sheets i regułami

## Context

Potrzebujemy modułu w Sunbay, który pozwoli **łączyć dane z wielu źródeł** (systemy księgowe: Fakturownia/inFakt/wFirma/SaldeoSmart/Xero; CRM: Zoho; Google Sheets) z **warunkową logiką, joinami i transformacjami**, produkując jeden wynikowy dataset. Finalnie ma zasilać procesy typu CollectionFlow/InvoicingFlow, ale **MVP = tylko export CSV / push do Google Sheets** (bez bezpośredniego podpięcia pod procesy).

Dostarczamy w dwóch iteracjach:

- **v1 (MVP, szybko)** — edytor SQL dla power-usera/admina. Wiele źródeł danych (każde dostaje alias = nazwę tabeli w DuckDB) + jeden edytor SQL gdzie je łączysz dowolnie (JOIN/UNION/CASE WHEN/podzapytania) + run + preview + export. Dostarczamy wartość natychmiast dla osób technicznych; produkt można użyć w wewnętrznych procesach jeszcze zanim powstanie wizualny UI.
- **v2 (dla nietechnicznych)** — wizualny pipeline-builder (Join / Filter / If-Then / Formula / Select / Sort). **Kluczowe**: v2 rozszerza v1, nie przepisuje go. Ta sama baza danych, ten sam runtime, ten sam compiler. Dochodzą tylko nowe typy kroków i nowe karty UI.

Kontekst techniczny Sunbay:
- Stack: .NET 10, EF Core 10 (MSSQL), MediatR 12 CQRS, FluentValidation, AutoMapper, Azure Service Bus + Azure Functions (Async/Durable), Serilog, SignalR. Clean Architecture + feature slices.
- Istniejące konektory: `Integrations.Fakturownia|inFakt|wFirma|SaldeoSmart|Xero|Zoho|Insert` + `Integrations.Shared`.
- Google Sheets: `Application/Features/GoogleSheets/{Commands,Queries}` (m.in. `GetGoogleSeetsCsv`, push do sheetów).
- Selektory strategii per-integracja: pattern `IntegrationProductStrategySelector` / `IntegrationTaxRateStrategySelector` w `Sunbay.Integration/Strategies/`.
- Brak istniejącego rule engine — greenfield.
- Frontend: React 19, TS strict, Vite, Monaco (da się dołożyć), SignalR client, Zod + RHF, SCSS.

## Kluczowa decyzja architektoniczna: "Step pipeline" od v1

Żeby v2 dołożyć bez przepisywania, **v1 już używa modelu "pipeline of steps"**, tylko z jednym zdefiniowanym typem kroku: `RawSqlStep`. Compiler, runtime, storage, endpointy — wszystko operuje na kolekcji kroków od pierwszego dnia. W v2 dokładamy nowe typy kroków (`JoinStep`, `FilterStep`, `IfThenStep`, `FormulaStep`, `SelectStep`, `SortStep`) do compilera i nowe komponenty do FE. **Zero migracji danych. Zero branchu "stara vs nowa ścieżka" w runtime.**

`RawSqlStep` żyje dalej w v2 jako "advanced step" — dev może wrzucić ręczny SQL na dowolnej pozycji pipeline'u.

## Silnik: DuckDB embedded (in-process)

Niezależnie od wersji, wykonaniem zajmuje się **DuckDB** (`DuckDB.NET.Data` NuGet):
- embedded, bez dodatkowego serwera,
- columnar/OLAP, skala 10k–10M wierszy in-process bez problemu,
- natywnie CSV/Parquet/Arrow,
- izolowany — operuje tylko na załadowanych tabelach, brak dostępu do MSSQL hosta/plików.

### Architektura (high-level, wspólna dla v1 i v2)

```
[ Source Adapters ]      [ Staging ]       [ Engine ]                [ Output ]
Fakturownia  ─┐                                                       ┌─ CSV
inFakt       ─┤   pull rows   ┌─────────┐   DefinitionCompiler      │
Xero         ─┤──────────────▶│ DuckDB  │── (JSON steps → SQL) ────▶├─ Google Sheets
Google Sheets─┤               │ in-proc │        + execute          │
Zoho (CRM)   ─┘               └─────────┘                           └─ (v3: CollectionFlow)
```

## Model domenowy — jednolity dla v1 i v2

Nowy projekt `Sunbay.Datasets`, feature-slice jak inne moduły (Commands/Queries/Services/Models).

### Entities (Domain/Sunbay/)

- **`Dataset`** — `Id`, `Name`, `Description`, `OwnerUserId`, `TenantId`, timestamps.
- **`DatasetSource`** — `Id`, `DatasetId`, `SourceType` (enum: `Fakturownia|InFakt|WFirma|SaldeoSmart|Xero|Zoho|Insert|GoogleSheets`), `Alias` (nazwa tabeli w SQL, np. `invoices`), `ConfigJson` (parametry źródła per-typ: zakres dat / sheetId + tab / filtr).
- **`DatasetDefinition`** — `Id`, `DatasetId`, `DefinitionJson` (struktura kroków pipeline'u, discriminated union), `CompiledSql` (cache wygenerowanego SQL odświeżany przy zapisie), `Version` (inkrementowany przy każdym update). Dataset ma jedną aktywną definicję w MVP; historia wersji to phase 2.
- **`DatasetRun`** — `Id`, `DatasetId`, `DefinitionVersion` (snapshot), `StartedAt`, `FinishedAt`, `Status` (`Running|Success|Failed|Cancelled`), `RowCount`, `ErrorMessage`, `OutputBlobUri`, `TriggeredByUserId`.

### Format `DefinitionJson` (discriminated union)

```json
{
  "steps": [
    { "type": "rawSql", "id": "s1", "sql": "SELECT ... FROM invoices i LEFT JOIN sheet g ON ..." }
  ]
}
```

W v1 akceptujemy tylko `type: "rawSql"`. Kompilator ma switch na typ kroku; w v1 obsługuje jeden case, w v2 — siedem.

Migracja EF: **nie twórz ręcznie** — instrukcja dla usera po mergu:
`dotnet ef migrations add AddDatasets` w odpowiednim katalogu, a potem `dotnet ef database update`.

## Komponenty backendowe (wspólne v1 + v2)

### `ISourceAdapter`

```csharp
public interface ISourceAdapter {
    SourceType Type { get; }
    Task<SourceFetchResult> FetchAsync(DatasetSource src, int? sampleLimit, CancellationToken ct);
    Task<SourceSchema> DescribeAsync(DatasetSource src, CancellationToken ct);
}

public record SourceFetchResult(IReadOnlyList<IDictionary<string, object?>> Rows, SourceSchema Schema);
public record SourceSchema(IReadOnlyList<ColumnDef> Columns);
public record ColumnDef(string Name, DuckDbType Type, bool Nullable);
```

Implementacje — po jednej per typ źródła — **delegują do istniejących serwisów** z `Integrations.*` (nie duplikują logiki pobierania):
- `FakturowniaSourceAdapter`, `InFaktSourceAdapter`, `WFirmaSourceAdapter`, `SaldeoSmartSourceAdapter`, `XeroSourceAdapter`, `ZohoSourceAdapter`, `InsertSourceAdapter`
- `GoogleSheetsSourceAdapter` — deleguje do `Application/Features/GoogleSheets/Queries/GetGoogleSeetsCsv` albo odpowiednika.

Selektor: `SourceAdapterSelector` w stylu `IntegrationProductStrategySelector` — `Select(SourceType) → ISourceAdapter`.

### `DefinitionCompiler`

Kompiluje `DefinitionJson` do jednego stringa SQL (CTE per step dla izolacji):

```sql
WITH s1 AS ( ... ),
     s2 AS ( SELECT * FROM s1 WHERE ... ),
     s3 AS ( SELECT s2.*, CASE WHEN ... THEN ... ELSE ... END AS new_col FROM s2 )
SELECT * FROM s3
```

W v1 compiler obsługuje tylko `rawSql` (pierwszy krok jest po prostu osadzany). W v2 obsługuje wszystkie typy. Wygenerowany SQL jest cache'owany w `CompiledSql` przy zapisie definicji.

### `DatasetExecutionService`

Flow (identyczny v1 i v2):
1. Załaduj `Dataset` + `DatasetSource[]` + `DatasetDefinition`.
2. Dla każdego `DatasetSource` → wywołaj adapter → `CREATE TABLE {alias}` w sesji DuckDB (Appender API lub Arrow).
3. Wykonaj `CompiledSql` (lub recompile jeśli cache stale).
4. Zapisz wynik do Blob Storage jako CSV/Parquet.
5. Update `DatasetRun` (status + row count + blob uri).
6. Push SignalR event `datasetRunCompleted` na kanał per-user.

**Limity (konfigurowalne):**
- Timeout egzekucji SQL: 60s w MVP.
- Max rows per run: 500k.
- Max memory DuckDB: 2GB.
- Przy przekroczeniu — failure z czytelnym błędem. Scale-out do `Sunbay.FunctionApp.Durable` jest przygotowany architektonicznie (wzorzec w repo), ale nie w MVP.

### `SqlSafetyValidator`

Walidacja wygenerowanego SQL przed wykonaniem (chroni też `RawSqlStep` pisany przez usera w v1):
- Whitelist: `SELECT`, `WITH` (CTE).
- Blacklist: `ATTACH`, `COPY`, `INSTALL`, `LOAD`, `PRAGMA` (poza `PRAGMA table_info`), `EXPORT`, `IMPORT`, zapisy do filesystem.
- Użycie parsera DuckDB (sparsuj i sprawdź root node'y) zamiast regexów.

Dodatkowa izolacja w DuckDB: `SET disabled_filesystems='LocalFileSystem'`, bez `httpfs`, sesja czysta per-run.

## Struktura plików

### Backend — nowy projekt `Sunbay.Datasets`

```
Sunbay.Datasets/
├─ Sunbay.Datasets.csproj                    # referencje: Integrations.*, DuckDB.NET.Data
├─ Commands/
│  ├─ CreateDataset/                         # v1
│  ├─ UpdateDatasetDefinition/               # v1 (przyjmuje DefinitionJson)
│  ├─ AddDatasetSource/                      # v1
│  ├─ DeleteDatasetSource/                   # v1
│  ├─ RunDataset/                            # v1
│  ├─ ExportDatasetRun/                      # v1 (CSV + push Sheets)
│  └─ DeleteDataset/                         # v1
├─ Queries/
│  ├─ GetDataset/                            # v1
│  ├─ ListDatasets/                          # v1
│  ├─ GetDatasetRun/                         # v1
│  ├─ GetDatasetRunPreview/                  # v1 (pierwsze N wierszy z Blob)
│  ├─ GetSourceSchemaPreview/                # v1 (próbka + schema źródła)
│  ├─ GetCompiledSql/                        # v1 (podgląd SQL — advanced)
│  └─ GetStepPreview/                        # v2 (mini-preview do danego kroku — w v1 stub zwraca full run dla jedynego kroku)
├─ Services/
│  ├─ DatasetExecutionService.cs             # v1
│  ├─ DuckDbSessionFactory.cs                # v1 (izolowane sesje, limity)
│  ├─ DefinitionCompiler.cs                  # v1 (case rawSql), v2 (wszystkie typy)
│  └─ SqlSafetyValidator.cs                  # v1
├─ SourceAdapters/
│  ├─ ISourceAdapter.cs
│  ├─ SourceAdapterSelector.cs
│  ├─ FakturowniaSourceAdapter.cs
│  ├─ InFaktSourceAdapter.cs
│  ├─ WFirmaSourceAdapter.cs
│  ├─ SaldeoSmartSourceAdapter.cs
│  ├─ XeroSourceAdapter.cs
│  ├─ ZohoSourceAdapter.cs
│  ├─ InsertSourceAdapter.cs
│  └─ GoogleSheetsSourceAdapter.cs
└─ Models/
   └─ Steps/
      ├─ Step.cs                              # abstract base, discriminated union root
      ├─ RawSqlStep.cs                        # v1
      ├─ JoinStep.cs                          # v2
      ├─ FilterStep.cs                        # v2
      ├─ IfThenStep.cs                        # v2
      ├─ FormulaStep.cs                       # v2
      ├─ SelectStep.cs                        # v2
      └─ SortStep.cs                          # v2
```

Zmiany w istniejących projektach:
- `Domain/Sunbay/Dataset.cs`, `DatasetSource.cs`, `DatasetDefinition.cs`, `DatasetRun.cs` + enumy.
- `DataAccess/Configuration/Dataset*Configuration.cs` — EF Core mappings.
- `Sunbay/Controllers/Platform/DatasetsController.cs` — REST.
- Rejestracja DI w `Sunbay/Program.cs` (lub Extensions).
- `Sunbay.sln` — dodaj projekt.

### NuGet
- `DuckDB.NET.Data` (dla `Sunbay.Datasets`).

### Frontend — `src/components/datasets/`

Wspólne dla v1 i v2:
- `src/api/datasets.ts` — klient API (endpointy poniżej).
- `src/services/datasetsService.ts` — warstwa domeny (mapowanie API ↔ model).
- `src/models/dataset.ts` — typy TS dla `Dataset`, `DatasetSource`, `DatasetDefinition`, `Step` (discriminated union).
- `src/components/datasets/DatasetsList/` — ekran 1 (lista).
- `src/components/datasets/DatasetEditor/` — ekran 2, 3 panele: `SourcesPanel`, `PipelinePanel`, `ResultsPanel`.
- `SourcesPanel/` — lista źródeł + `AddSourceModal` (wybór typu, alias, config, live schema preview).
- `ResultsPanel/` — grid preview (reuse pattern z `csvImporterModal`), download CSV, `PushToSheetsModal`, zakładka `RunsHistoryTab`.
- SignalR listener na `datasetRunCompleted` (wzorzec z `src/components/processes/`).
- Route: `src/routes/user.routes.tsx` + `src/constants/routes.constants.ts`.
- i18n: `i18n/en.json`, `i18n/pl.json`.

Różnica v1 vs v2 dotyczy wyłącznie `PipelinePanel`:

**v1 — `PipelinePanel` w trybie single-step SQL:**
- Jeden komponent: `steps/RawSqlStepCard.tsx` — Monaco Editor (DuckDB dialect syntax highlight), nad edytorem accordion "Available tables" (aliasy + kolumny klikalne, wstawiają do edytora), skrót `Ctrl+Enter` = Run.
- `PipelinePanel` trzyma listę kroków z dokładnie jednym `RawSqlStep`; button "Run pipeline" pod edytorem.
- UI nie ujawnia mechanizmu "steps" usera — wygląda jak klasyczny SQL editor. Pod spodem leci już `{ steps: [{ type: "rawSql", sql: "..." }] }`.

**v2 — `PipelinePanel` w trybie multi-step wizualnym:**
- Lista kart kroków z drag-drop reorder (`dnd-kit` lub podobne).
- Button `+ Add step` → menu z typami: Join, Filter, If-Then-Else, Formula, Select, Sort, (Advanced) Raw SQL.
- Komponenty kart: `steps/JoinStepCard.tsx`, `FilterStepCard.tsx`, `IfThenStepCard.tsx`, `FormulaStepCard.tsx`, `SelectStepCard.tsx`, `SortStepCard.tsx`, `RawSqlStepCard.tsx` (ten sam z v1, przeniesiony bez zmian).
- Każda karta: mini-preview (fetch `POST /datasets/:id/steps/:idx/preview` → pierwsze 20 wierszy po wykonaniu pipeline'u do tego kroku).
- `AdvancedSqlView.tsx` — read-only podgląd wygenerowanego SQL z `GET /datasets/:id/compiled-sql` (toggle; działa też w v1 dla edukacji).

## Kontrakt REST — stabilny między v1 a v2

```
POST   /api/datasets                               # create
GET    /api/datasets                               # list
GET    /api/datasets/:id                           # detail (sources + definition + lastRun)
PUT    /api/datasets/:id                           # update (name/description)
DELETE /api/datasets/:id

POST   /api/datasets/:id/sources                   # add source
DELETE /api/datasets/:id/sources/:sid
POST   /api/datasets/:id/sources/:sid/preview      # sample rows + schema (podgląd w UI)

PUT    /api/datasets/:id/definition                # body: { steps: Step[] } — v1 akceptuje wyłącznie rawSql; v2 wszystkie typy
GET    /api/datasets/:id/compiled-sql              # advanced view

POST   /api/datasets/:id/run                       # trigger run → { runId, status }
GET    /api/datasets/:id/runs                      # list runs
GET    /api/datasets/:id/runs/:rid                 # run detail (status, meta)
GET    /api/datasets/:id/runs/:rid/preview?limit=N # pierwsze N wierszy wyniku (JSON)
GET    /api/datasets/:id/runs/:rid/export.csv      # stream CSV z Blob
POST   /api/datasets/:id/runs/:rid/push-to-sheets  # { spreadsheetId, tab } → { sheetUrl }

POST   /api/datasets/:id/steps/:stepIdx/preview    # mini-preview do kroku
```

W v1 endpoint `/steps/:stepIdx/preview` fizycznie istnieje ale przy pipeline z jednym krokiem zachowuje się tak samo jak `/run`. Żadnej zmiany kontraktu w v2 — FE po prostu zaczyna go wołać per-karta.

SignalR event: `datasetRunCompleted { runId, datasetId, status, rowCount, error? }` — per-user channel (reuse istniejącego huba).

## UX — co user widzi

### Ekran 1: Lista datasetów (`/datasets`) — v1 i v2
Tabela: nazwa, liczba źródeł, ostatni run (status + data + row count), akcje (Edit, Run, Duplicate, Delete). Button "+ New dataset".

### Ekran 2: Dataset Editor (`/datasets/:id`) — 3 panele

**Panel lewy — Sources (v1 i v2 identycznie):**
- Lista źródeł z ikoną typu, aliasem, podsumowaniem configu.
- `+ Add source` → modal: typ integracji (dropdown z listą zarejestrowanych integracji usera, GET `/integrations`), alias, parametry specyficzne dla typu, po zapisie `/sources/:sid/preview` pokazuje kolumny z typami — **user od razu wie jakimi polami dysponuje**.

**Panel środkowy — Pipeline:**
- **v1:** Monaco SQL editor, accordion "Available tables", button Run.
- **v2:** lista kart kroków, `+ Add step` menu, mini-preview per karta, drag-drop reorder. Raw SQL wciąż dostępny przez `+ Add step → Advanced: Raw SQL`.

**Panel prawy — Results (v1 i v2 identycznie):**
- Stany: empty / running (spinner + timer + progres: "pobieranie źródła X/Y") / success (grid pierwszych 500 wierszy + "Showing 500 of N · Run took Xs") / error.
- Zakładki nad gridem: **Preview** | **Runs history**.
- Akcje: **Download CSV**, **Push to Google Sheets** (modal z wyborem spreadsheet + tab).

### Ekran 3: Runs history (zakładka w edytorze) — v1 i v2
Tabela: when, status, rows, duration, triggered by. Klik → pokazuje snapshot definicji z tego runu (w v1: SQL; w v2: widok pipeline'u read-only) + link do wyniku (jeśli w retencji) + akcje download/push.

### Przepływ FE↔BE (user klika Run — identyczny v1 i v2)

```
FE                                   BE
──                                   ──
POST /datasets/:id/run          ─▶   RunDatasetCommand
                                     ├─ load DatasetDefinition (JSON steps)
                                     ├─ DefinitionCompiler → SQL (CTE per step)
                                     ├─ SqlSafetyValidator
                                     ├─ create DatasetRun (status=Running)
                                     └─ kick off async execution
                              ◀──   202 { runId, status: 'Running' }

[FE pokazuje spinner]

                                     [BE async:]
                                     for each DatasetSource:
                                       adapter.FetchAsync() → rows
                                       CREATE TABLE {alias} in DuckDB
                                     execute compiled SQL
                                     write result → Blob (CSV)
                                     update DatasetRun (Success/Failed)
                                     push SignalR event

SignalR 'datasetRunCompleted' ◀──
GET /datasets/:id/runs/:rid/preview?limit=500
                              ◀──   first 500 rows (JSON)
[FE renderuje grid]

[Download CSV]
GET /runs/:rid/export.csv     ◀──   stream from Blob

[Push to Sheets]
POST /runs/:rid/push-to-sheets ─▶   deleguje do Application/Features/GoogleSheets
                              ◀──   { sheetUrl }
```

## Reuse (nie piszemy od zera)

- Każdy `*SourceAdapter` deleguje do istniejących serwisów z `Integrations.*` — tylko opakowanie + mapping do wierszy + opis schemy.
- Push do Google Sheets: `Application/Features/GoogleSheets/Commands/*` (już istnieje).
- Pattern selektora: wzoruj na `Sunbay.Integration/Strategies/IntegrationProductStrategySelector.cs`.
- Grid preview FE: wzorzec z `src/components/csvImporterModal/`.
- SignalR: istniejący hub używany w `src/components/processes/`.
- Durable orchestration (gdy scale-out będzie potrzebny): `Sunbay.FunctionApp.Durable`.

## Zakres v1 vs v2 — checklist

### v1 (MVP)

Backend:
- [ ] Projekt `Sunbay.Datasets` + dodanie do solution.
- [ ] Entities + EF configs + instrukcja migracji dla usera.
- [ ] `ISourceAdapter` + `SourceAdapterSelector` + wszystkie 8 implementacji (delegują do istniejących integracji).
- [ ] `DuckDbSessionFactory` z limitami i izolacją.
- [ ] `SqlSafetyValidator` (parser-based).
- [ ] `DefinitionCompiler` z obsługą `RawSqlStep` (case na jeden typ + placeholder na pozostałe z NotSupported exception).
- [ ] `DatasetExecutionService` + async execution + SignalR push.
- [ ] Commands/Queries pełny zestaw z listy wyżej.
- [ ] `DatasetsController` z pełnym kontraktem REST (w tym `/steps/:idx/preview` i `/compiled-sql`, które w v1 będą miały trywialną implementację).
- [ ] Unit testy: `DefinitionCompiler` (rawSql case), `SqlSafetyValidator`, `DatasetExecutionService` z mockami adapterów, każdy `*SourceAdapter`, wszystkie handlery.
- [ ] Integration test: pełen flow Fakturownia + Sheets + SQL z `CASE WHEN`+`JOIN` → run → assert.
- [ ] **Uruchomić `dotnet test` po każdej zmianie** (per memory).

Frontend:
- [ ] API klient + service + modele TS.
- [ ] `DatasetsList` (ekran 1).
- [ ] `DatasetEditor` (ekran 2) z `SourcesPanel` + `PipelinePanel` (tylko `RawSqlStepCard` z Monaco) + `ResultsPanel`.
- [ ] `AddSourceModal` z live preview schemy.
- [ ] `PushToSheetsModal`, download CSV.
- [ ] `RunsHistoryTab`.
- [ ] SignalR listener.
- [ ] Routing + i18n EN/PL.
- [ ] **`npm run lint -- --fix` + `npx tsc -b --noEmit` przed pushem** (per memory).

### v2 (rozszerzenie — bez przepisywania)

Backend (tylko `DefinitionCompiler` + walidatory):
- [ ] `DefinitionCompiler` dodaje case'y: `JoinStep`, `FilterStep`, `IfThenStep`, `FormulaStep`, `SelectStep`, `SortStep`. Każdy produkuje swoje CTE w kompozycji.
- [ ] Walidacja semantyczna step'ów (np. że referowany alias istnieje, kolumna istnieje w schemie poprzedniego kroku).
- [ ] `GetStepPreview` — implementacja realna (compile do pozycji `stepIdx` i execute tylko do tego miejsca).
- [ ] Testy: `DefinitionCompiler` dla każdego nowego typu + snapshot testy na kompozycjach.
- [ ] Controller i pozostały runtime — **zero zmian**.

Frontend:
- [ ] `PipelinePanel` przechodzi na tryb multi-step: lista kart + `+ Add step` menu + drag-drop reorder.
- [ ] Nowe karty: `JoinStepCard`, `FilterStepCard`, `IfThenStepCard`, `FormulaStepCard`, `SelectStepCard`, `SortStepCard`. Condition builder jako shared component (reuse w Filter i IfThen).
- [ ] Mini-preview per karta (fetch step preview).
- [ ] `AdvancedSqlView` (toggle pokazujący compiled SQL).
- [ ] `RawSqlStepCard` z v1 — przeniesiony jako "Advanced: Raw SQL" w menu dodawania kroku. **Zero zmian w komponencie.**
- [ ] Istniejące datasety z v1 otwierają się poprawnie w v2 (są definicją z jednym rawSql step — UI renderuje jedną kartę RawSql, user może dodać kolejne kroki).

**Co nie zmienia się między v1 a v2:** DB schema, entities, EF configs, source adapters, `DuckDbSessionFactory`, `SqlSafetyValidator`, `DatasetExecutionService`, cały zestaw endpointów, format SignalR eventów, `DatasetsList`, `SourcesPanel`, `ResultsPanel`, `AddSourceModal`, `PushToSheetsModal`, `RunsHistoryTab`, routing, SignalR integration.

## Ograniczenia / decyzje

**v1:**
- Tylko power user (rozumie SQL).
- Jeden aktywny `DatasetDefinition` per dataset; historia wersji nie jest wymagana (ale `Version` w modelu już jest).
- Free-text SQL — walidacja whitelisty w `SqlSafetyValidator` to linia obrony.

**v2:**
- Condition builder: AND/OR grupy, max głębokość zagnieżdżeń 3 (readability).
- IfThenStep: max 3 poziomy zagnieżdżonego else-if (po tym push na raw SQL).
- FormulaStep: tylko predefiniowane szablony (concat, date diff, days overdue, multiply, format currency). **Nie** dopuszczamy free-text formuł — inaczej de facto wracamy do SQL i tracimy sens v2.
- Group By / agregaty: **poza MVP v2**, phase 3.

**Wspólne:**
- Wynik runu: Blob Storage jako CSV; metadane w MSSQL (`DatasetRun`). Retention 30 dni (domyślne, konfigurowalne).
- Limity: timeout 60s, max 500k wierszy, max 2GB memory DuckDB.
- Brak integracji z CollectionFlow/InvoicingFlow w MVP — output to wyłącznie CSV/Sheets. Bezpośrednie zasilanie procesów to phase 3.

## Roadmap (dla kontekstu, poza scope tego planu)

- **Phase 3:** Dataset jako input dla `AddCollectionFlowCommand` + `InvoicingFlow` (mapping kolumn → pola procesu).
- **Phase 3:** Scheduled runs (Azure Functions timer trigger) + event-driven runs (trigger przy nowej fakturze).
- **Phase 4:** Wersjonowanie definicji + diff view.
- **Phase 4:** Parametryzacja datasetów (runtime params: zakres dat, ID klienta, wybór integracji w momencie runu).
- **Phase 5:** Group By + agregaty w wizualnym builderze.

## Verification

### v1

1. **Unit tests** (`Sunbay.Datasets.Tests`):
   - `DefinitionCompiler.RawSqlStep_ProducesPassthrough`.
   - `SqlSafetyValidator` — odrzuca `ATTACH`/`COPY`/`PRAGMA`/`INSTALL`/`LOAD`, akceptuje `SELECT`/`WITH`.
   - `DatasetExecutionService` — mock adapterów, weryfikacja że rows ładują się do DuckDB i SQL daje oczekiwany wynik (test na `CASE WHEN`, `JOIN`).
   - Każdy `*SourceAdapter` — weryfikacja delegacji do istniejącego serwisu integracji (mock serwisu, assert parametrów).
   - Handlery: `CreateDataset`, `UpdateDatasetDefinition`, `AddDatasetSource`, `RunDataset`, `ExportDatasetRun`.
   - **`dotnet test` po każdej zmianie.**

2. **Integration test**: create dataset → add 2 sources (Fakturownia stub + Sheets stub) → definition z jednym `rawSql` step (`SELECT i.nr, i.nip, g.segment, CASE WHEN i.overdue_days > 30 THEN 'Y' ELSE 'N' END AS should_collect FROM invoices i LEFT JOIN sheet g ON i.nip = g.nip`) → run → assert output rows.

3. **Manual E2E**:
   - Start API + FE (`npm run dev`).
   - Login → New Dataset → Add Fakturownia source (`alias=invoices`) → Add Google Sheets source (`alias=sheet`) → napisz SQL w edytorze → Run → preview → Download CSV → Push to Sheets.
   - Przed pushem FE: `npm run lint -- --fix` + `npx tsc -b --noEmit`.

4. **Smoke**: syntetyczny dataset ~50k wierszy, weryfikacja timeout/memory w limitach.

### v2 (oprócz regresji v1)

1. **Unit tests**:
   - `DefinitionCompiler` — osobny test per typ kroku (Join/Filter/IfThen/Formula/Select/Sort) + snapshot tests na kompozycji 5–7 kroków.
   - Walidacja semantyczna (reference do nieistniejącego aliasu → clear error).
   - `GetStepPreview` — compile do pozycji `stepIdx`, assert że kolejne kroki nie są wykonywane.

2. **Integration test**: ten sam scenariusz co v1, ale zbudowany wizualnie (Join + IfThen + Select) zamiast rawSql — assert identyczny wynik.

3. **Migration test**: dataset utworzony w v1 (rawSql) otwiera się poprawnie w v2 UI jako pipeline z jedną kartą RawSql, można dodać kolejny step.

4. **Manual E2E v2**:
   - Istniejący v1 dataset → otwórz w v2 → widzisz rawSql jako kartę → dodaj Filter step nad nim → dodaj IfThen step → Run → assert że wynik jest poprawny i compiled SQL (toggle) pokazuje sensowne CTE.
   - Nowy dataset czysto wizualny: Join (invoices + sheet) → Filter (overdue_days > 30) → IfThen (create `should_collect`) → Select (zostaw 4 kolumny) → Sort → Run → Export.

## Otwarte decyzje (nie blokują implementacji)

- Storage wyniku: Blob Storage vs tabela MSSQL (rekomendacja: Blob jako CSV, tabela trzyma metadane).
- Retention runów: domyślnie 30 dni, konfigurowalne w appsettings.
- Hosting DuckDB przy wysokim obciążeniu: in-process w API vs wydzielony worker w Durable Functions (na razie in-process; wzorzec Durable gotowy do scale-out).
- Czy `RawSqlStep` w v2 wymaga dodatkowego permission gatingu (np. tylko dla roli Admin) — do ustalenia przy v2.
