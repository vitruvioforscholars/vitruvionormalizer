# Normalizza App (MVP)

Web app mobile-friendly (PWA-ready) per:
1) caricare un articolo `.docx`
2) selezionare le norme redazionali (dataset)
3) scaricare un `.docx` normalizzato

## Avvio rapido (con Docker)
Prerequisiti: Docker + Docker Compose

```bash
cd normalizza-app
docker compose up --build
```

Apri: http://localhost:3000

## Avvio manuale (senza Docker)
### Backend
```bash
cd backend
python -m venv .venv
source .venv/bin/activate  # (Windows: .venv\Scripts\activate)
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

### Frontend
```bash
cd frontend
npm install
NEXT_PUBLIC_API_BASE=http://localhost:8000 npm run dev
```

Apri: http://localhost:3000

## Dataset norme
Backend: `backend/app/standards/*.yaml`

Per aggiungere una norma:
- crea un file `id.yaml` con i campi minimi `id` e `name`
- apparirà automaticamente nel menu.

## Cosa fa questo MVP
- titolo: MAIUSCOLO + centrato (primo paragrafo non vuoto)
- corpo: 12 pt + giustificato (best-effort)
- note: 10 pt (best-effort, via modifica XML `word/footnotes.xml`)
- correzioni testo:
  - `, cit.` -> ` cit.`
  - normalizzazione semplice dei trattini: ` -- ` e ` - ` -> ` – `

> Prossimo step: implementare tutte le regole Atti MOD (citazioni lunghe 11, gestione autore/affiliazione, bibliografia più profonda, ecc.)


## Citazioni lunghe
- Se una citazione (tra virgolette “ ” / " " / « ») ha **>=400 battute**, viene trasformata in **blocco** a **corpo 11** e **senza virgolette** (senza rientri aggiunti).
- Nella fascia **400±200** battute, l'app può **derogare** e lasciare la citazione inline se il blocco indebolirebbe il paragrafo (es. resta un frammento troppo corto prima/dopo).


- Le citazioni a blocco applicano anche una spaziatura sopra/sotto (default 6pt, configurabile in YAML: `long_quote_space_before_pt`, `long_quote_space_after_pt`).


- Le citazioni a blocco NON hanno spaziatura prima/dopo (0 pt), ma viene inserita una **riga bianca** prima e dopo, nello stesso corpo del testo principale.


## Bibliografia / note (MVP)
- Normalizzazione meccanica nelle note: `, cit.` → ` cit.` (senza virgola prima), `CIT.` → `cit.`, `et al` → `et al.`, `ivi/ibidem` → `Ivi/Ibidem`, `p 12` → `p. 12`, `pp 12` → `pp. 12`.
- Le regole strutturali più avanzate (corsivo titoli, maiuscoletto cognomi, ricostruzione prima/successiva citazione) sono il prossimo step.

- Applicazione corsivo ai titoli nelle note (heuristic: testo tra prima e seconda virgola in citazioni tipo 'Autore, Titolo, Editore').


## Verifica riviste (ANVUR)
- Per evitare refusi, la normalizzazione di `«...»` come *rivista* usa un **registro locale** (nessun testo del manoscritto viene inviato fuori).
- Scarica i PDF ANVUR degli elenchi di riviste (area/e di interesse) e mettili in: `backend/app/journals/pdfs/`
- Poi genera il registro:

```bash
python tools/build_registry_from_anvur.py
```

- Il file generato è: `backend/app/journals/registry.json`

## Ivi / Ibidem (conservativo)
- Best-effort su note *testuali* del tipo `... , p. 12`.
- Se due note consecutive hanno la stessa opera e stessa pagina → `Ibidem`.
- Se stessa opera ma pagina diversa → `Ivi, p. X`.


## Ivi / Ibidem (chain-aware)
- L'algoritmo segue la catena: se una nota è già `Ivi`/`Ibidem`, mantiene come opera di riferimento quella della nota precedente efficace.
- Le citazioni con `cit.` vengono trattate come **stessa opera** della prima citazione (canon: `Autore, Titolo`).
- Interviene solo quando trova un pattern chiaro di pagina (`p.`/`pp.`).


## Segnala problemi di normalizzazione (note)
- Quando una nota è ambigua (es. manca `p.`/`pp.`, oppure contiene più riferimenti), l'app **non interviene** e evidenzia la nota in **giallo**, aggiungendo in coda un tag: `[AMBIGUITÀ: ...]`.

## Pagine nelle note
- Per i numeri di pagina nelle note si usa il **trattino corto `-`** (anche se l'originale aveva `–`).


## Segnala problemi di normalizzazione (note)
- Quando una nota è ambigua, l'app **non interviene** e la evidenzia in **giallo**.
- Inserisce inoltre un **commento Word** (autore: `Normalizer`) con la motivazione.

## Modalità rigorosa
- Toggle in UI.
- In modalità rigorosa l'algoritmo interviene di più (es. non esclude automaticamente note multi-riferimento), ma segnala comunque con commento quando non è certo.
