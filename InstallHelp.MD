# Paperclip — Guida all'installazione e controllo con Claude Code

## Cos'è Paperclip

Paperclip è una piattaforma **multi-agente AI self-hosted**: un sistema in cui agenti AI (Claude, Codex, modelli locali via Ollama/LM Studio, ecc.) lavorano insieme all'interno di una struttura organizzativa virtuale — con company, progetti, task, approvazioni e flussi di lavoro.

In pratica è un "ufficio virtuale" dove:
- il **board** (tu, o Claude che ti rappresenta) assegna compiti
- gli **agenti** li eseguono autonomamente
- tutto è tracciato, approvabile, versionato

Paperclip gira interamente in locale sul tuo Mac — nessun dato esce dalla macchina.

---

## Installazione

### Prerequisiti

- Node.js 18+ (verifica con `node --version`)
- npm o npx disponibile nel terminale

### Installazione e primo avvio

```bash
# Esegui l'onboarding (installa e configura tutto in automatico)
npx paperclipai onboard --yes
```

Il flag `--yes` accetta tutti i default: modalità `local_trusted`, accessibile solo da `localhost`, nessuna autenticazione richiesta.

Al termine, avvia il server:

```bash
npx paperclipai run
```

Il server gira su **`http://localhost:3100`** — la UI è accessibile da browser, ma per questa guida lo controlliamo da Claude Code.

### Modalità di deployment

| Modalità | Descrizione | Auth richiesta |
|----------|-------------|----------------|
| `local_trusted` | Default. Solo localhost, nessuna auth | No |
| `lan` | Accessibile sulla rete locale | Sì |
| `tailnet` | Accessibile via Tailscale | Sì |

Per questa guida usiamo `local_trusted`.

---

## Controllare Paperclip con Claude Code

Ci sono tre modi per far gestire Paperclip a Claude Code. Ognuno ha un ruolo diverso, limiti diversi e un'architettura diversa. Capire la differenza è fondamentale.

---

### Modo 1 — CLI (`npx paperclipai`)

**Come funziona:** Claude esegue comandi terminale tramite il tool `Bash`.

**Setup:** nessuno. In `local_trusted` basta avere il server attivo.

```bash
# Imposta il context una volta sola
npx paperclipai context set --api-url http://localhost:3100

# Esempi di comandi
npx paperclipai company list
npx paperclipai agent list --company-id <id>
npx paperclipai issue create --company-id <id> --title "Implementa feature X"
npx paperclipai invite create --company-id <id> --payload-json '{"email":"...","role":"member"}'
```

**Cosa può fare:** gestione company, agenti, issue, inviti, approvazioni, skill, backup, worktree.

**Limiti:**
- Alcuni comandi mancano rispetto al REST API (es. `company create` non è nel CLI)
- Richiede che il server sia attivo
- Output testuale — meno strutturato da parsare rispetto a JSON puro

**Quando usarlo:** operazioni amministrative quotidiane, automazioni semplici.

---

### Modo 2 — REST API diretta (`curl` / fetch)

**Come funziona:** Claude chiama direttamente l'API HTTP del server Paperclip tramite `curl`.

**Setup:** nessuno in `local_trusted`. Nessun header di autenticazione necessario.

```bash
# Creare una company
curl -s -X POST http://localhost:3100/api/companies \
  -H "Content-Type: application/json" \
  -d '{"name":"La mia org","shortname":"org"}'

# Creare un agente
curl -s -X POST http://localhost:3100/api/companies/<id>/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"builder","adapterType":"codex_local"}'

# Creare un task
curl -s -X POST http://localhost:3100/api/companies/<id>/issues \
  -H "Content-Type: application/json" \
  -d '{"title":"Analizza il codice","description":"..."}'
```

**Cosa può fare:** tutto — è l'interfaccia completa. 41 endpoint che coprono company, agenti, issue, documenti, approvazioni, costi, attività, plugin, segreti, routine.

**Limiti:**
- Più verboso da scrivere rispetto al CLI
- Richiede di gestire gli ID manualmente
- In modalità autenticata (`lan`/`tailnet`) serve un token Bearer

**Quando usarlo:** operazioni non disponibili nel CLI, automazioni complesse, integrazione con altri sistemi.

---

### Modo 3 — MCP Server (`@paperclipai/mcp-server`)

**Come funziona:** un processo separato che espone i tool Paperclip come strumenti nativi di Claude Code, senza passare per il terminale.

**Setup:**
```json
// ~/.claude/mcp.json
{
  "mcpServers": {
    "paperclip": {
      "command": "npx",
      "args": ["-y", "@paperclipai/mcp-server"],
      "env": {
        "PAPERCLIP_API_URL": "http://localhost:3100",
        "PAPERCLIP_API_KEY": "<api-key-di-un-agente>",
        "PAPERCLIP_COMPANY_ID": "<company-id-default>"
      }
    }
  }
}
```

**Perché serve una API key (e perché appartiene a un agente):**

Paperclip non emette API key generiche — le emette solo per entità "agente". Questo perché il server MCP è stato progettato per uno scenario specifico: un agente AI worker che usa il server per interagire con i propri task (fare checkout, aggiornare stato, aggiungere commenti, segnalare completamento). L'agente si autentica con la sua key per dire "sono io, ho finito questo task".

Quando usi il server MCP per *comandare* la company invece di *lavorarci dentro*, stai piegando il meccanismo al contrario: crei un agente "fittizio" (es. `claude-mcp`) solo per ottenere la chiave, ma quell'agente non esegue task — è solo un'identità di autenticazione.

```
Uso previsto dal design:
  Paperclip board → assegna task → Agente usa MCP server → esegue task → aggiorna Paperclip

Uso come orchestratore (non ideale):
  Claude usa MCP server → comanda company, agenti, issue → ...ma è il tool sbagliato per questo ruolo
```

**Cosa può fare:** 23 tool di lettura + 16 tool di scrittura — issue, agenti, approvazioni, commenti, documenti, workspace.

**Limiti:**
- Progettato per worker, non per admin
- Richiede un agente registrato nella company solo per avere la chiave
- `PAPERCLIP_COMPANY_ID` è opzionale (default), i tool accettano `companyId` per richiesta
- Richiede riavvio di Claude Code per caricare i nuovi server

**Quando usarlo:** quando vuoi che Claude operi *come* un agente worker all'interno di un flusso Paperclip — non per amministrare la piattaforma.

---

## Riepilogo comparativo

| | CLI | REST API | MCP Server |
|--|-----|----------|------------|
| Setup | Nessuno | Nessuno | Agente + API key + mcp.json |
| Auth (local_trusted) | Non serve | Non serve | Serve API key |
| Copertura funzionale | ~80% | 100% | ~60% (focus worker) |
| Ruolo ideale | Admin quotidiano | Automazione completa | Agente worker |
| Come lo usa Claude | Tool `Bash` | Tool `Bash` (curl) | Tool nativi MCP |

**Raccomandazione:** per pilotare Paperclip con Claude Code come orchestratore/admin, usa **CLI + REST API**. Il server MCP è utile solo se vuoi che Claude si comporti come un agente worker all'interno di un flusso di lavoro Paperclip.

---

## Esempio pratico — Team di sviluppo software autonomo

Questo esempio mostra come creare una company Paperclip strutturata come un team di sviluppo software, con 8 agenti specializzati coordinati da un Tech Lead.

> Il file `software_development_company_claudecode_fixed.zip` allegato contiene il dump esportabile di questa company, importabile con `npx paperclipai company import`.

### Architettura del team

Il principio è avere un **unico punto di contatto** (Tech Lead) a cui dare istruzioni in linguaggio naturale. Il Tech Lead scompone il lavoro e coordina gli specialisti.

| Agente | Ruolo |
|--------|-------|
| **tech-lead** | Unico punto di contatto. Riceve brief, li scompone in task tecnici, coordina tutti gli altri agenti, riporta lo stato |
| **architect** | Progetta la struttura del software — componenti, interfacce, scelte tecnologiche |
| **backend-developer** | Implementa API, logica di business, database |
| **frontend-developer** | Implementa UI e integrazione con le API backend |
| **test-engineer** | Scrive test (unit, integration, e2e) in parallelo allo sviluppo |
| **security-reviewer** | Analizza il codice per vulnerabilità prima del merge — può bloccare con approvazione esplicita |
| **devops** | CI/CD, deploy, infrastruttura, monitoring |
| **tech-writer** | Documentazione API, README, changelog |

### Operazioni comuni operabili da Claude Code (se usato per gestire Paperclip)

Puoi saltare questa sezione se importi il file, è già pronto

#### Creazione della company via REST API

Il comando `company create` non è disponibile nel CLI, quindi si usa il REST API direttamente:

```bash
# 1. Crea la company
curl -s -X POST http://localhost:3100/api/companies \
  -H "Content-Type: application/json" \
  -d '{"name":"AutoDev Team","shortname":"autodev"}'
# Salva l'id restituito → COMPANY_ID

# 2. Crea gli agenti (ripeti per ognuno)
curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"tech-lead","title":"Tech Lead","adapterType":"codex_local"}'

curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/agents \
  -H "Content-Type: application/json" \
  -d '{"name":"architect","title":"Software Architect","adapterType":"codex_local"}'

# ... (ripeti per backend-developer, frontend-developer,
#      test-engineer, security-reviewer, devops, tech-writer)
```

#### Assegnare un task al Tech Lead

Una volta creata la struttura, si parla solo col Tech Lead:

```bash
TECHLEAD_ID="<id-agente-tech-lead>"

curl -s -X POST http://localhost:3100/api/companies/$COMPANY_ID/issues \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Sviluppa web app di monitoraggio processi di sistema",
    "description": "Crea un'\''applicazione web cross-platform (Windows, macOS, Linux) che visualizzi in tempo reale i processi attivi sul dispositivo.\n\nRequisiti:\n- Backend Python con Flask\n- Lettura processi tramite psutil (cross-platform)\n- Frontend con auto-refresh ogni 2-3 secondi\n- Tabella con PID, nome, CPU%, memoria%, stato, utente\n- Ordinamento per colonna, filtro per nome\n- Indicatori CPU e RAM totale del sistema\n\nCoordina gli specialisti del team per architettura, sviluppo, test, security review e documentazione.",
    "assigneeId": "$TECHLEAD_ID",
    "priority": "high"
  }'
```

### Configurare i modelli locali dopo l'import

> **Nota:** a causa di un bug nell'import di Paperclip, quando si importa una company con gli agenti che hanno un adapter come OpenCode e un modello locale, vengono _(o possono venire)_ reimpostati su Claude Code indipendentemente da come erano configurati nel file. Dopo ogni import occorre riassegnare manualmente l'adapter.

L'esempio seguente usa **gemma4:e4b via Ollama**.

> **Importante — context length:** prima di assegnare task agli agenti, assicurati che il context window del modello sia impostato a 64k o più (Ho testato e 64k funziona, altri valori minori potrebbero non funzionare - in particolare se si usa un modello con pochi miliadi di parametri). Un context corto causa `step failed` anche a run "succeeded". Vedi la sezione [**Context length — impostazione critica**](#context-length--impostazione-critica) per Ollama e LM Studio.

#### Prerequisiti

1. Ollama in esecuzione con il modello scaricato:
   ```bash
   ollama pull gemma4:e4b
   ollama serve   # già avviato se l'icona è nella barra menu su macOS
   ```

2. Se i modelli Ollama non appaiono in OpenCode (vedi sezione dedicata più avanti), crea `~/.config/opencode/config.json` con il modello (la necessità di esplicitare la ricerca di modelli ollama dipende dalla versione di OpenCode):
   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "provider": {
       "ollama": {
         "npm": "@ai-sdk/openai-compatible",
         "name": "Ollama (local)",
         "options": { "baseURL": "http://127.0.0.1:11434/v1" },
         "models": {
           "gemma4:e4b": { "name": "Gemma 4 E4B" }
         }
       }
     }
   }
   ```

#### Via UI (per ogni agente)

1. Apri Paperclip → Company → agente → **Settings**
2. Sezione **Adapter** → seleziona **OpenCode**
3. Nel selettore **Model**, scegli il modello sotto la voce `ollama` (es. `ollama / Gemma 4 E4B`)
4. Se è presente un **modello secondario** (fallback), disabilitalo o lascialo vuoto — il modello di default potrebbe non essere disponibile e causare errori silenziosi
5. Salva e ripeti per tutti e 8 gli agenti

#### Via REST API (tutti gli agenti in una volta)

Prima recupera gli ID degli agenti:

```bash
COMPANY_ID="<id-della-company>"

curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "
import sys, json
agents = json.load(sys.stdin)
for a in agents:
    print(a['id'], a['name'])
"
```

Poi aggiorna ogni agente con un singolo comando (sostituisci `<AGENT_ID>`):

```bash
curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/<AGENT_ID> \
  -H "Content-Type: application/json" \
  -d '{"adapterType":"opencode_local","adapterConfig":{"model":"ollama/gemma4:e4b"}}'
```

Oppure aggiornali tutti in un loop:

```bash
COMPANY_ID="<id-della-company>"

# Recupera tutti gli ID in un array
AGENT_IDS=$(curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "import sys,json; [print(a['id']) for a in json.load(sys.stdin)]")

# Aggiorna tutti in sequenza
for ID in $AGENT_IDS; do
  curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/$ID \
    -H "Content-Type: application/json" \
    -d '{"adapterType":"opencode_local","adapterConfig":{"model":"ollama/gemma4:e4b"}}' \
    | python3 -c "import sys,json; a=json.load(sys.stdin); print(a.get('name'), '→', a.get('adapterType'))"
done
```

Per usare un modello diverso (es. LM Studio), sostituisci `ollama/gemma4:e4b` con `lmstudio/<model-id>` e assicurati che la sezione `lmstudio` sia presente nel `config.json` di OpenCode (vedi sezione LM Studio più avanti).

---

### Perché questo approccio funziona

**Parallelismo reale:** mentre il Backend Developer scrive le API Flask, il Frontend Developer lavora sull'interfaccia e il Test Engineer prepara i test. Non si aspetta nessuno.

**Gate di qualità:** il Security Reviewer deve approvare esplicitamente prima che il codice venga mergiato. Se trova un problema (es. esposizione di dati sensibili di sistema), blocca il flusso.

**Scalabilità:** la stessa struttura gestisce un progetto piccolo (la web app sopra) e uno grande. Aggiungi task al Tech Lead, lui smista.

**Autonomia graduabile:** puoi lasciare che il team lavori completamente da solo su task ben definiti, oppure impostare approvazioni umane ai checkpoint critici (architettura, deploy in produzione).

---

## Modelli consigliati per Apple Silicon

Per agenti Paperclip che scrivono codice autonomamente, la famiglia migliore disponibile su Ollama è **Qwen2.5-Coder** (Alibaba): segue istruzioni complesse, produce codice strutturato, gestisce contesti lunghi.

### MacBook Pro M5 — 24GB RAM

```bash
ollama pull qwen2.5-coder:14b
```

~8.5GB in memoria — gira comodamente, velocità ottima su M5. È il punto dolce per 24GB: abbastanza capace per agenti autonomi, abbastanza leggero da non stressare la RAM.

### Mac Studio M2/M4 Max — 64-96GB RAM

```bash
ollama pull qwen2.5-coder:32b
```

~19GB in memoria — qualità molto alta, adatto a task complessi multi-file.

### Mac Studio M2/M4 Ultra — 128-192GB RAM

```bash
ollama pull qwen2.5-coder:72b
```

~43GB in memoria — qualità eccellente, vicino ai modelli cloud per task di coding.

---

## Modelli locali — Ollama e LM Studio

Gli agenti Paperclip non sono vincolati a Claude. Puoi farli girare su modelli locali tramite **Ollama** o **LM Studio**, usando **OpenCode** come adapter. Questa sezione descrive come configurare entrambi.

__Il progetto con gli agenti impostati con OpenCode, nonostante tutte le procedure siano spiegate a seguito, è disponibile qui: software_development_company_claudecode_fixed_local.zip__

---

### Adapter: OpenCode

In Paperclip, quando crei o modifichi un agente, puoi scegliere `opencode_local` come `adapterType`. OpenCode è il processo che fa da ponte tra Paperclip e i modelli AI — sia cloud che locali. Supporta nativamente Ollama e, con configurazione manuale, LM Studio.

---

### Ollama (consigliato e ha setup più semplice rispetto a LM Studio)

#### Installazione e modelli

```bash
# Installa Ollama da https://ollama.com
# Avvia il server (si avvia automaticamente al login su macOS)
ollama serve

# Scarica modelli (esempi)
ollama pull llama3.2          # 3B parametri — leggero, poco preciso
ollama pull llama3.2:8b       # 8B — buon bilanciamento
ollama pull gemma4:26b        # 26B — qualità alta, richiede più RAM
ollama pull qwen3.5           # ottimo per coding

# Verifica che un modello risponda
ollama run llama3.2 "ciao"
```

#### Context length — impostazione critica

Paperclip invia agli agenti un system prompt molto lungo (istruzioni del protocollo, stato della company, contesto dell'issue, ecc.). Se il context window del modello è troppo corto, il prompt arriva troncato e il modello non capisce cosa fare — restituisce `step failed` o risposte vuote pur riportando il run come "succeeded".

**Imposta il context length a 64k o più** nell'app Ollama:

1. Apri l'app Ollama (icona nella barra menu) → **Settings**
2. Cerca il campo **Context Length** (o `num_ctx`) e impostalo a `65536` (64k)
3. Riavvia Ollama

In alternativa, puoi impostarlo per singolo modello creando un Modelfile:

```bash
# Crea un Modelfile che estende un modello esistente con context 64k
cat > /tmp/Modelfile <<'EOF'
FROM gemma4:e4b
PARAMETER num_ctx 65536
EOF
ollama create gemma4-e4b-64k -f /tmp/Modelfile
```

Un context da 4k (default di alcuni modelli) causa `step failed`. Con 64k gli agenti Paperclip funzionano correttamente.

**In LM Studio** il context length si imposta per ogni modello al momento del caricamento: nella scheda **Developer → Local Server**, quando clicchi **Load model**, a destra del nome del modello c'è una **freccia (▶)** — cliccandola si apre un pannello dove puoi specificare la dimensione della context window prima di avviare il server.

#### Endpoint Ollama

Ollama espone un server OpenAI-compatible su:

```
http://127.0.0.1:11434
```

Puoi listare i modelli disponibili con:

```bash
curl http://127.0.0.1:11434/v1/models
```

#### Configurazione in Paperclip

1. Modifica l'agente → `adapterType: opencode_local`
2. Seleziona il modello dal dropdown (es. `ollama/llama3.2`)

In alcune versioni di OpenCode, Ollama viene scoperto automaticamente senza nessuna configurazione. Se però aprendo OpenCode i modelli Ollama non appaiono nel selettore (Ctrl+P → "Change model"), occorre dichiararli esplicitamente in `~/.config/opencode/config.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": { "baseURL": "http://127.0.0.1:11434/v1" },
      "models": {
        "llama3.2:3b": { "name": "Llama 3.2 3B" },
        "qwen2.5-coder:14b": { "name": "Qwen 2.5 Coder 14B" }
      }
    }
  }
}
```

Sostituisci i modelli con quelli restituiti da `curl http://127.0.0.1:11434/v1/models`.

#### Testare i modelli in OpenCode

Per verificare quali modelli OpenCode vede prima di assegnarli a un agente:

```bash
opencode
# poi: Ctrl+P → "Change model"
# apparirà la lista di tutti i modelli disponibili
```

---

### LM Studio

#### Abilitare il server locale

1. Apri LM Studio
2. Barra laterale destra → **Developer** → **Local Server**
3. Imposta **Status: Enabled**
4. Clicca **Load model** e seleziona il modello desiderato

#### Trovare l'ID del modello

LM Studio espone il suo server su `http://127.0.0.1:1234`. Per conoscere l'ID esatto del modello caricato:

```bash
curl http://127.0.0.1:1234/v1/models
```

Il JSON di risposta contiene il campo `id` — ad esempio `gemma-4-e4b-it`. Questo è il nome che userai nella configurazione.

#### Configurare OpenCode per LM Studio

A differenza di Ollama, LM Studio **non viene scoperto automaticamente** da OpenCode. Devi registrarlo manualmente nel file di configurazione di OpenCode:

**`~/.config/opencode/config.json`**

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LM Studio (local)",
      "options": { "baseURL": "http://127.0.0.1:1234/v1" },
      "models": {
        "gemma-4-e4b": { "name": "Gemma 4 E4B (local)" }
      }
    }
  },
  "model": "lmstudio/gemma-4-e4b"
}
```

Sostituisci `gemma-4-e4b` con l'`id` ottenuto dal curl sopra.

#### Configurazione in Paperclip

Dopo aver salvato il `config.json`, in Paperclip l'agente con adapter `opencode_local` mostrerà nel dropdown il modello `lmstudio/<model-id>`.

---

### Utilizzare adapter generci o porte non predefinite (non il nostro caso): Quando servono `OPENAI_API_KEY` e `OPENAI_BASE_URL`

Con l'adapter **OpenCode** queste variabili d'ambiente **non sono necessarie** — né per Ollama né per LM Studio. OpenCode gestisce la connessione internamente.

Servono invece in questi casi:

| Caso | Variabili necessarie |
|------|---------------------|
| Adapter generico OpenAI-compatible (non OpenCode) | `OPENAI_BASE_URL=http://127.0.0.1:11434/v1` (Ollama) o `http://127.0.0.1:1234/v1` (LM Studio) + `OPENAI_API_KEY=ollama` (valore fittizio ma richiesto) |
| Ollama su porta non standard o host remoto | `OPENAI_BASE_URL=http://<host>:<porta>/v1` |
| Forzare OpenCode a usare un endpoint specifico sovrascrivendo l'auto-discovery | `OPENAI_BASE_URL` esplicito |

---

### OpenRouter — GLM 5.2 (modello cloud)

**OpenRouter** è un gateway cloud che aggrega centinaia di modelli AI (inclusi quelli di Zhipu AI come GLM) tramite un'unica API OpenAI-compatibile. Permette di usare GLM da cloud senza registrarsi direttamente su bigmodel.cn.

> **Prerequisiti:** account su [openrouter.ai](https://openrouter.ai) con una API key generata. Per la procedura, vedi la sezione [Creare un account OpenRouter e ottenere la API key](#creare-un-account-openrouter-e-ottenere-la-api-key) più avanti.

#### 1. Trovare l'ID del modello GLM su OpenRouter

I modelli GLM su OpenRouter sono pubblicati dal provider **Z.ai** (non "zhipuai"). Il model ID di GLM 5.2 è:

```
z-ai/glm-5.2
```

Verificalo sulla pagina [openrouter.ai/z-ai/glm-5.2](https://openrouter.ai/z-ai/glm-5.2).

> **GLM 5.2 è un modello a pagamento** ($0.95/M token input, $3/M token output). Per usarlo occorre aggiungere crediti su OpenRouter (carta di credito necessaria). OpenRouter assegna un piccolo credito gratuito alla registrazione, sufficiente per qualche test iniziale.
>
> **Alternativa gratuita:** se vuoi verificare che l'integrazione funzioni prima di aggiungere la carta, usa `z-ai/glm-4.5-air:free` — è completamente gratuita e non richiede crediti. Una volta confermato che tutto gira, sostituisci il model ID con `z-ai/glm-5.2`.

#### 2. Configurare OpenCode per OpenRouter

Apri (o crea) `~/.config/opencode/config.json` e aggiungi il provider `openrouter` nella sezione `provider`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openrouter": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OpenRouter",
      "options": {
        "baseURL": "https://openrouter.ai/api/v1",
        "apiKey": "TUA_API_KEY_QUI"
      },
      "models": {
        "z-ai/glm-5.2": { "name": "GLM 5.2" }
      }
    }
  }
}
```

Se hai già altri provider nel file (es. Ollama), aggiungi `openrouter` affiancandolo agli altri:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": { ... },
    "openrouter": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "OpenRouter",
      "options": {
        "baseURL": "https://openrouter.ai/api/v1",
        "apiKey": "TUA_API_KEY_QUI"
      },
      "models": {
        "z-ai/glm-5.2": { "name": "GLM 5.2" }
      }
    }
  }
}
```

> Per usare la versione gratuita invece, sostituisci `z-ai/glm-5.2` con `z-ai/glm-4.5-air:free` in entrambi i blocchi `models`.

Dopo aver salvato, verifica che OpenCode veda il nuovo modello:

```bash
opencode
# Ctrl+P → "Change model" — deve comparire "OpenRouter / GLM 5.2"
```

#### 3. Configurare gli agenti Paperclip per usare GLM 5.2

> Esempio pratico: importa `software_development_company_claudecode_fixed.zip` e dopo l'import configura gli agenti su GLM 5.2 tramite REST API, senza modificare il file ZIP.

**Via UI (per ogni agente):**

1. Apri Paperclip → Company → agente → **Settings**
2. Sezione **Adapter** → seleziona **OpenCode**
3. Nel selettore **Model**, scegli il modello sotto la voce `openrouter` (es. `openrouter / GLM 5.2`)
4. Se compare un **modello secondario** (fallback), disabilitalo o lascialo vuoto
5. Salva e ripeti per tutti gli agenti

**Via REST API (tutti gli agenti in una volta):**

```bash
COMPANY_ID="<id-della-company>"
MODEL="openrouter/z-ai/glm-5.2"   # oppure "openrouter/z-ai/glm-4.5-air:free" per la versione gratuita

# Recupera tutti gli ID degli agenti
AGENT_IDS=$(curl -s http://localhost:3100/api/companies/$COMPANY_ID/agents \
  | python3 -c "import sys,json; [print(a['id']) for a in json.load(sys.stdin)]")

# Aggiorna tutti gli agenti
for ID in $AGENT_IDS; do
  curl -s -X PATCH http://localhost:3100/api/companies/$COMPANY_ID/agents/$ID \
    -H "Content-Type: application/json" \
    -d "{\"adapterType\":\"opencode_local\",\"adapterConfig\":{\"model\":\"$MODEL\"}}" \
    | python3 -c "import sys,json; a=json.load(sys.stdin); print(a.get('name'), '→', a.get('adapterType'), a.get('adapterConfig',{}).get('model',''))"
done
```

#### Context length

I modelli cloud via OpenRouter supportano context window da 128k o più per default — non è necessaria nessuna configurazione aggiuntiva. Il problema `step failed` per context corto riguarda solo i modelli locali (Ollama/LM Studio con default a 4k).

---

### Creare un account OpenRouter e ottenere la API key

1. Vai su [openrouter.ai](https://openrouter.ai) e clicca **Sign In** → registrati con Google o email
2. Una volta dentro, vai su **Keys** (menu in alto a destra o sidebar)
3. Clicca **Create Key**, dai un nome e copia la chiave generata — non verrà mostrata di nuovo
4. Incolla la chiave nel campo `apiKey` del `config.json` di OpenCode come indicato sopra

> **Carta di credito:** non è obbligatoria per usare i modelli **gratuiti** (contrassegnati con `:free` o badge Free su openrouter.ai/models). Per i modelli a pagamento occorre aggiungere crediti (minimo $5). Verifica sempre il badge del modello prima di usarlo.

> **Sicurezza:** non committare mai la API key nel codice o in file versionati. Tienila solo in `~/.config/opencode/config.json` che è fuori dal repository.

---

### Quale modello scegliere

| Modello | Parametri | Adatto per |
|---------|-----------|------------|
| `llama3.2:3b` | 3B | Test rapidi, non affidabile per coding autonomo |
| `llama3.2:8b` | 8B | Task semplici, istruzioni base |
| `qwen3.5` | variabile | Ottimo per coding, segue istruzioni bene |
| `gemma4:26b` | 26B | Alta qualità, richiede ~20GB RAM |

**Regola pratica:** per agenti che scrivono codice autonomamente in Paperclip servono almeno 7-14B parametri e preferibilmente un modello ottimizzato per coding (Qwen, DeepSeek Coder, CodeLlama). I modelli 3B tendono a non seguire le istruzioni in modo preciso e falliscono il probe di verifica di Paperclip.
