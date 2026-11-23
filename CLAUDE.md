# FileMaker + Claude – referens för utvecklingssessioner

Senast uppdaterad: 2026-11-23. Läs denna fil i början av varje FileMaker-session.
Verifiera mot Claris dokumentation om något verkar ha ändrats.

---

## 1. Läget just nu

### FileMaker 2026 (version 26)
- 26.0.1 släpptes juni 2026. 26.1 finns och har nyare AI-modeller (se §5).
- Nytt: annoteringar på fält och tabeller (ger AI-modeller kontext om schemat),
  stöd för Google Gemini, villkorligt redigerbara fält, ny Layout Inspector,
  Standby Server och Remote Backup för FileMaker Server.
- Ingen inbyggd kodagent i själva releasen.

### Agentic Development for FileMaker (developer preview)
- Användaren är anmäld till förhandsprogrammet (nov 2026).
- Claris öppnade det bredare i oktober 2026. Riktar sig mot Claude Code, Cursor och Codex.
- Agenten arbetar direkt i FileMaker-filer med vanliga script steps, fält,
  beräkningar och layouter. Allt går att granska och redigera.
- Pris har inte offentliggjorts. Officiell sida: https://www.claris.com/filemaker/agentic-development/

### Claris MCP-server (tillgänglig nu)
- Låter Claude (Desktop, claude.ai eller Claude Code) söka, skapa och uppdatera poster samt köra skript.
- Konfiguration utan kod: kopiera "Proxy Configuration (Command-based)"-JSON från
  serverfilens inställningar till Claudes MCP-konfiguration.
- Kräver FileMaker Server 22.0.2.204 eller senare. Följer FileMaker-kontots behörigheter.
  AI-åtgärder loggas i FileMaker Server.
- Tredjepartsalternativ: MCP-servrar via Data API (kräver behörigheten `fmrest`), FM Dojo.

---

## 2. AI-skriptsteg (översikt)

Källa: https://help.claris.com/en/pro-help/content/artificial-intelligence-script-steps.html

| Skriptsteg | Funktion |
|---|---|
| Configure AI Account | Namngivet AI-konto: leverantör (eller Custom-endpoint) + API-nyckel |
| Configure Machine Learning Model | Laddar en Core ML-modell |
| Configure Prompt Template | Namngivna promptmallar |
| Configure RAG Account | Namngivet RAG-konto (endpoint + nyckel) |
| Configure Regression Model | Tränar, sparar och laddar en regressionsmodell på embeddings |
| Fine-Tune Model | Finjusterar en modell med träningsdata |
| Generate Response from Model | Prompt → svar, med verktyg och agentiskt läge (se §4) |
| Insert Embedding / …in Found Set | Lagrar vektorer i fält eller variabler |
| Insert Image Caption / …in Found Set | Bildtexter via Claris AI Model Server (nytt i 2026) |
| Perform Find by Natural Language | Naturligt språk → FileMaker-sökning |
| Perform RAG Action | Frågar, lägger till eller tar bort data i ett RAG-utrymme |
| Perform Semantic Find | Semantisk sökning på text, bild eller vektor |
| Perform SQL Query by Natural Language | Naturligt språk + schema → SQL → svar |
| Set AI Call Logging | Slår på eller av loggning av AI-anrop till fil |

---

## 3. Configure AI Account

Källa: https://help.claris.com/en/pro-help/content/configure-ai-account.html

- Parametrar: `Account Name`, `Model Provider` (fördefinierad eller `Custom`), `API key`.
  Endast för Custom: `Endpoint` och `Verify SSL Certificates`.
- Kontot gäller **bara i aktuell fil** och nollställs när filen stängs.
  Kör steget i ett skript vid filöppning (OnFirstWindowOpen).
- Custom-endpoint måste vara kompatibel med OpenAI:s API och URL:en ska sluta med `/`.
- Claris AI Model Server: välj `Custom`, nyckel från Admin Console, endpoint från sidan AI Services.
- Hårdkoda inte API-nycklar. Använd ett globalt fält eller liknande:

```
Configure AI Account [ Account Name: "claude" ; Model Provider: Anthropic ; API key: Global::API_Key ]
```

- Fungerar i Pro, Go, WebDirect, Server, Cloud och Data API. Kom i version 21.0.

---

## 4. Generate Response from Model

Källa: https://help.claris.com/en/pro-help/content/generate-response-from-model.html (kom i 22.0)

### Parametrar
- `Account Name`, `Model`, `User Prompt`, `Instructions` (systeminstruktion).
- `Agentic mode`
  - **På:** modellen anropar verktyg upprepade gånger tills den har ett slutsvar.
  - **Av:** steget stannar vid första verktygsanropet och lämnar det i `Tool Calls from Model`
    utan att köra det. Används för felsökning, tester och manuell kontroll.
- `Response`: fält eller variabel. Med `Stream` måste fältet finnas på aktuell layout.
- `Messages`: historik som JSON-array med `{"role":"user"|"assistant","content":...}`.
- `Save Message History To` + `Message History Count`: använd en `$$`-variabel (sessionen)
  eller spara den i ett fält så att den finns kvar mellan sessioner.
- `Include tool calls and tool results`: tar med verktygsanropen i historiken (felsökning).
- `Tool Definitions`: JSON-array i **leverantörens format** (se nedan).
- `Tool Calls from Model`: var verktygsanropen loggas (felsökning och revision).
- `Temperature`.
- `Parameters`: JSON. `CURLOPT_TIMEOUT` (sekunder) hanteras av FileMaker själv och ger fel 1629
  vid timeout. Övriga nycklar skickas vidare till leverantören.
- `Perform JavaScript in Web Viewer`: strömmar svaret till en JS-funktion i en webbvisare.

### Verktyg som modellen kan anropa
- `execute_sql` (fördefinierat): kör SQL. Lägg schemat sist i parameterbeskrivningen,
  i DDL-form från `GetTableDDL`. Använd Claris beskrivningstext ordagrant som bas
  (regler: citerade namn, `FETCH FIRST n ROW ONLY`, ingen semikolon, inga nästlade frågor,
  `DATE 'YYYY-MM-DD'`, ROWID utan citattecken).
- `retrieve_image` (fördefinierat): parametrarna `sql_query` (ger ROWID), `table` och `field`.
  Returnerar `[{"data":"<base64>"}]`.
- **Custom functions:** `name` = exakt funktionsnamn. Samma antal och ordning på parametrarna
  som i Edit Custom Function. Alla parametrar är `"type": "string"`.
- Vanliga fel: "No tool call exists for the prompt", "Function not found".
- Säkerhet: modellen kommer aldrig åt databasen direkt. Verktygen körs i FileMaker
  under kontots behörigheter.

### Verktygsformat för Anthropic (Claude)
Claris exempel använder OpenAI:s format (`{"type":"function","function":{name,description,parameters}}`).
För Claude används Anthropics format. Den här omskrivningen är **ej verifierad i FileMaker**:

```json
[
  {
    "name": "CityHasSalesOffice",
    "description": "Check whether our company has a sales office in a specified city. Returns 'yes' or 'no'.",
    "input_schema": {
      "type": "object",
      "properties": { "city": { "type": "string", "description": "The name of a city" } },
      "required": ["city"]
    }
  }
]
```

---

## 5. AI-modeller som stöds (tekniska specifikationer)

Källa: https://community.claris.com/en/s/article/Claris-FileMaker-2026-Technical-Specifications
(sidan kunde inte renderas från molnmiljön; uppgifterna kommer från sökutdrag, så verifiera dem).

| Leverantör | FileMaker 26.1 | FileMaker 26.x (tidigare) |
|---|---|---|
| Anthropic | `claude-sonnet-5-5`, `claude-haiku-5-5` | `claude-sonnet-4-6`, `claude-haiku-4-5` |
| OpenAI | `gpt-6.1-sol`, `gpt-5.5-2026-04-23` | `gpt-5.1`, `gpt-4.1` (RAG) |
| Cohere | `command-a-03-2025`, `command-r7b-12-2024`, `command-r-08-2024`, `command-r-plus-08-2024` | `command-r-plus-08-2024`, `command-a-03-2025` |
| Google | `gemini-2.5-flash` | `gemini-2.5-pro` |
| Claris AI Model Server | ja | ja |

- `claude-sonnet-5-5`: agentiskt läge med verktyg och mer komplexa uppgifter.
- `claude-haiku-5-5`: snabba och billiga anrop (sökning med naturligt språk, klassificering, bildtexter).
- Opus finns inte med i listan och är otestat av Claris.

---

## 6. Automatiska tester

FileMaker saknar inbyggt testramverk. Rekommenderade sätt:

1. **Testskript i filen:** varje test är ett skript som anropar skriptet eller funktionen
   som testas och jämför med förväntat resultat. Ett huvudskript kör allt och sammanställer.
   Ramverk: FM-Test (https://github.com/jbante/FM-Test, TAP-inspirerat).
2. **Externa tester via Data API eller OData** (rekommenderas för CI): t.ex. pytest mot en
   **testkopia** på FileMaker Server, aldrig mot produktionsfilen. Skapa testdata, kör skript,
   kontrollera resultatet och städa efteråt. Kräver behörigheten `fmrest`.
3. **AI-funktioner:** kör med `Agentic mode` av och kontrollera verktygsanropen (namn och
   argument) i stället för fritext. Sätt `{"temperature": 0}` i `Parameters`. Testa egenskaper
   ("nämner 4 kontakter") snarare än exakt text. Logga med `Set AI Call Logging`.

UI (layouter, script triggers) är svårt att testa automatiskt. Fokusera på affärslogik.

---

## 7. Arbetsregler för Claude i FileMaker-sessioner

- Slå upp exakt syntax för AI-skriptstegen i Claris hjälp. Gissa inte, eftersom stegen är nya.
- Nätverk i molnsessioner: `*.claris.com` är tillåtet. Salesforce-sidorna (support.claris.com,
  community.claris.com) renderas inte i headless-webbläsaren. Be användaren klistra in innehållet.
  help.claris.com fungerar med curl.
- Lägg aldrig API-nycklar i skript, filer eller commits.
- Ändra inte artifacts utan uttrycklig instruktion (användarens preferens).

---

## Källor
- Claris – Agentic development: https://www.claris.com/filemaker/agentic-development/
- Claris – FileMaker 2026 is now available: https://www.claris.com/blog/2026/claris-filemaker-2026-is-now-available
- Claris – Why MCP matters: https://www.claris.com/blog/2025/why-mcp-matters-for-claris-filemaker-developers
- Claris Help – AI script steps: https://help.claris.com/en/pro-help/content/artificial-intelligence-script-steps.html
- IT Brief – wider preview: https://itbrief.co.uk/story/claris-opens-wider-preview-of-agentic-filemaker-development
- Soliant – FileMaker 2026 Executive Summary: https://www.soliantconsulting.com/blog/filemaker-2026-executive-summary/
- Soliant – Regression testing: https://www.soliantconsulting.com/blog/regression-testing-filemaker/
- DB Services – Claris MCP: https://dbservices.com/blog/claris-mcp-for-filemaker
- FM-Test: https://github.com/jbante/FM-Test
