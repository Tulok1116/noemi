# DocFlow AI: Skálázott, Eseményvezérelt Adatfeldolgozó Pipeline

Ez a dokumentum bemutatja a `docflow_ai_version.json` névre hallgató, n8n alapú eseményvezérelt munkafolyamatot, amely képes beérkező adatok kötegelt fogadására, biztonsági szanitizálására, mesterséges intelligencia alapú elemzésére, valamint robusztus hiba- és naplózási ágak kezelésére.

## 1. Rendszerarchitektúra és Adatáramlás

A munkafolyamat egy robusztus, többlépcsős pipeline mintát követ, amely elválasztja az érvényes, sikeres tranzakciókat a hibás vagy elutasított adatoktól.

```
[ Webhook Ingest ] 
       │
       ▼
[ Payload Validator ] ──(Hiba)──► [ Audit Logger ] ──► [ Discord / Table ]
       │
   (Érvényes)
       ▼
[ If (Conditional) ]
       │
   (Igaz ág)
       ▼
[ Data Mapper (Sanitize) ]
       │
       ▼
[ Basic LLM Chain (Google Gemini + Structured Parser) ]
       │
       ▼
[ AI Enrichment Validator ]
       │
       ├─(Sikeres)──► [ HTTP Request (Downstream API) ] ──► [ Database Audit ] ──► [ Respond to Webhook ]
       │
       └─(Hiba/Kivétel)─► [ Audit Logger ] ──► [ Discord / Database ] ──► [ Respond to Webhook1 ]

```

## 2. A Munkafolyamat Főbb Komponensei

### A. Adatbeolvasás és Szintaktikai Ellenőrzés (`Adatbeolvasás` & `Payload Validator`)

* **Trigger:** Egy biztonságos HTTP POST végpont (`/docflow-ingest`), amely fogadja a külső rendszerekből vagy webhookokból érkező kötegelt (batch) adatokat.

* **Validáció:** A *Payload Validator* node ellenőrzi, hogy a beérkező adatok tömböt alkotnak-e, nem üresek-e, valamint tartalmazzák-e a kötelező mezőket (`id` és `data`).

* **Biztonság:** Ha a payload érvénytelen, a rendszer azonnal elválasztja, és PII (személyes adatok) vagy titkok nélkül küldi tovább az audit logokba.

### B. Adatmegtisztítás és Elágazás (`Data Mapper` & `If`)

* **Biztonsági szanitizálás:** A *Data Mapper* rekurzívan bejárja a JSON struktúrát, és eltávolítja a potenciálisan veszélyes karaktereket (pl. HTML/XML tagek, XSS próbálkozások), miközben a hasznos adattartalmat sértetlenül hagyja.

* **Döntési logika:** Az *If* node szétválasztja az érvényes struktúrákat a hibásaktól, megakadályozva, hogy a rossz formátumú adatok elmenjenek az AI vagy a külső API felé.

### C. Mesterséges Intelligencia Alapú Besorolás (`Basic LLM Chain` & `Google Gemini`)

* **AI Integráció:** A munkafolyamat a **Google Gemini Chat Model** segítségével elemzi a szöveges vagy strukturált adatokat.

* **Strukturált kimenet (*Structured Output Parser*):** Biztosítja, hogy a Gemini válasza szigorú JSON sémát kövessen, amely tartalmazza:

  * **Kategória:** `order`, `support`, `invoice`, `complaint`, `spam`, `other`

  * **Prioritás:** `low`, `medium`, `high`

  * **Összefoglaló:** Rövid, legfeljebb 300 karakteres, személyes adatoktól mentes leírás.

  * **Confidence (Megbízhatóság):** 0 és 1 közötti érték.

### D. Minőségellenőrzés és Kézi Felülvizsgálat Jelölése (`AI Enrichment Validator`)

* Ellenőrzi az AI kimenetének érvényességét.

* Ha a modell bizonytalan (`confidence < 0.6`), a rendszer automatikusan megjelöli a tételt felülvizsgálatra (`needsReview: true`), ezzel növelve a vállalati folyamatok biztonságát.

### E. Többcélú Kézbesítés és Naplózás (`HTTP Request`, `Insert row`, `Discord`)

* **Sikeres ág:** A feldolgozott és gazdagított adatot elküldi egy külső célállomásra egy HTTP POST kéréssel, rögzíti az eseményt a belső `docflow_audit_log` adatbázis táblázatban, majd sikeres (200 OK) választ küld a webhook hívónak.

* **Hibaág / Riasztás:** Ha bármelyik fázisban hiba történik (pl. az AI elbukik vagy a külső API nem érhető el), az *Audit Logger* összegyűjti a kockázatokat, riasztást küld a **Discord** csatornára, naplózza a hibát az adatbázisban, és hibaüzenettel tér vissza a kliens felé.

## 3. Biztonsági Megfontolások (Sanitization & Compliance)

* **Titkok védelme:** A repóba feltöltött exportból minden valós API kulcs, webhook URL, instanciakód és credential azonosító ki lett tisztítva (helyőrzőkkel helyettesítve).

* **Adatvédelem (PII):** A naplózó kódok szándékosan üresen hagyják az `inputs` tömböt, így a nyers adatok vagy érzékeny információk soha nem kerülnek bele az audit logokba.
