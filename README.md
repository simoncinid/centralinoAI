# IDRAULICO-AI

Centralino multi-impresa per artigiani italiani. Backend Node 24 LTS / TypeScript / Fastify, PostgreSQL 17 e Drizzle, Telnyx Call Control v2, OpenAI Realtime WebSocket. Nessuna app mobile; SIM1 rimane pubblica e viene deviata dall’operatore al DID dedicato. Il backend può chiamare esclusivamente SIM2, un cellulare italiano distinto da SIM1 e DID.

## Stato verificabile

Prototipo locale implementato: database e migrazioni, pannello autenticato, whitelist CRUD, instradamento, macchina a stati, trasferimento A/B, AMD, Bridge, richiamate, coda email, stime e scenari simulati. I provider reali usano REST ufficiale e WebSocket GA, senza SDK telefonici con metodi presunti.

**Telefonate reali, qualità audio, disponibilità del modello nell’account, firma proveniente da Telnyx, caller ID dopo deviazione, squillo SIM2, Bridge e consegna SMTP reale: DA COLLAUDARE.** I test locali usano firme Ed25519 generate dal test e server WebSocket simulati. Non dimostrano un collaudo con i provider. Nessun account acquistato, nessun credito consumato, nessuna telefonata reale eseguita.

## Avvio locale

Prerequisiti: Docker Desktop avviato e Node.js 24 LTS con npm (con nvm: `nvm install && nvm use`).

```sh
npm ci
npm run setup:local
docker compose up -d
npm run db:migrate
npm run db:seed
npm run dev
```

Apri http://localhost:3000. Utente e password generata sono nel file locale `.local/access.txt`; `.env` e `.local/` sono esclusi dal repository. `setup:local` non sovrascrive una configurazione già presente. Non usare la password d’esempio o chiavi API in file pubblici.

Su questa macchina è stato installato anche un runtime Node 24 isolato. Per usarlo nella shell corrente, dalla radice del progetto:

```sh
export PATH="$PWD/.local/runtime/node_modules/node/bin:$PATH"
node --version
```

Per ricrearlo se necessario con il Node già presente: `npm install --prefix .local/runtime node@24`. Non è necessario con un Node 24 installato normalmente.

Esecuzione compilata:

```sh
npm run build
npm start
```

L’app è disponibile solo su loopback per impostazione predefinita. PostgreSQL usa localhost:54329; credenziali locali nel compose, da cambiare prima di qualsiasi pubblicazione. `docker compose up -d` avvia il solo database necessario. Per avviare anche l’app containerizzata: `docker compose --profile app up -d --build`. Il container migra e applica il seed idempotente; richiede `.env` già creato.

## Prova senza crediti

Dal pannello scegli uno scenario. Oppure, con il server avviato:

```sh
npm run mock:call -- whitelist
npm run mock:call -- unknown
npm run mock:call -- anonymous
npm run mock:call -- human
npm run mock:call -- answered
npm run mock:call -- busy
npm run mock:call -- no-answer
npm run mock:call -- machine
npm run mock:call -- duplicate
npm run mock:call -- error
npm run mock:call -- ended
```

La CLI si autentica al server locale ed esegue lo scenario tramite la stessa istanza che gestisce il pannello. I numeri demo sono fixture e non devono essere chiamati: whitelist `+393471234567`, sconosciuto `+393481234567`. Il mock non contatta OpenAI, Telnyx o SMTP; l’audio è un echo nei test, non un modello di comprensione. Gli scenari AI restano attivi fino a fine chiamata o timeout; usa “Termina” prima di modificare la configurazione. `human` lascia B in squillo; `answered` simula risposta, AMD umana e Bridge.

## Verifica

```sh
npm run lint
npm run build
npm test
npm run test:db
npm audit
curl http://localhost:3000/health
curl http://localhost:3000/ready
```

`lint` è il controllo statico TypeScript strict. `npm test` salta intenzionalmente il test PostgreSQL se non richiesto; `test:db` lo esegue sul database configurato, creando e rimuovendo due tenant temporanei senza modificare i dati demo. Risultati effettivi in [TEST_RESULTS.md](TEST_RESULTS.md).

## Routing e sicurezza

1. Verifica firma Ed25519 sui byte HTTP originali, finestra temporale e ID evento; inserimento durevole nella inbox prima di rispondere 200.
2. Individuazione tenant dal DID; normalizzazione E.164; un caller anonimo o un caller ID non diagnosticato non riceve privilegi whitelist.
3. Whitelist verificata o AI disattivata: risposta A e Dial B verso la sola SIM2. Nessuna sessione AI per whitelist.
4. Altrimenti: risposta A, streaming PCMU 8 kHz mono e una sessione OpenAI con strumenti backend validati.
5. Richiesta persona: trasferimento immediato senza qualificazione; interruzione e chiusura AI, messaggio di attesa Telnyx, Dial. Un trasferimento per chiamata.
6. B risponde e AMD conferma un umano: Bridge. Segreteria, risultato incerto, busy, errore o timeout: B chiusa e fallback configurato. Con AMD disabilitata il Bridge avviene alla risposta; la segreteria potrebbe quindi ricevere la chiamata.
7. Fallback richiamata: lead persistito, notifica in coda, messaggio di conferma e chiusura dopo 15 secondi. Se numero assente la notifica segnala impossibilità di richiamare. Nessun audio grezzo salvato.

Il caller ID è una regola di instradamento, mai un’autorizzazione sensibile. Il pannello richiede conferma “Caller ID originale verificato nel collaudo” per abilitare il confronto whitelist. Il seed demo la abilita; per numeri reali disattivarla finché non si verifica la deviazione.

Media WebSocket: `stream_auth_token` casuale per chiamata, consumo singolo all’upgrade, scadenza 60 s, correlazione control/session ID nel frame start e verifica PCMU. Non si assume che il media upgrade abbia una firma Ed25519 come il webhook HTTP. Chunk in ingresso deduplicati e riordinati con finestra limitata; uscita in pacchetti di 20 ms con massimo 5 secondi di coda, clear alla presa di parola e truncate della cronologia audio.

## Configurazione reale

Segui [SETUP_ACCOUNTS.md](SETUP_ACCOUNTS.md) e [REAL_CALL_TEST.md](REAL_CALL_TEST.md). Completare almeno:

- `TELNYX_API_KEY`, `TELNYX_PUBLIC_KEY`, `TELNYX_CONNECTION_ID`, `TELNYX_DID`.
- `OPENAI_API_KEY`; verificare accesso a `gpt-realtime-2.1-mini` e voce `marin`.
- `SIM1_PUBLIC_NUMBER`, `SIM2_PRIVATE_NUMBER` e configurazione corrispondente nel pannello/database.
- `PUBLIC_BASE_URL=https://…`, `PUBLIC_WSS_BASE_URL=wss://…`.
- `MOCK_MODE=false`; `ALLOW_REAL_OUTBOUND_CALLS=true` solo quando la SIM2 è stata autorizzata al test. Lasciare false blocca Dial e porta al fallback.
- SMTP, email destinatario per tenant e tariffe effettive per notifiche e contabilità indicativa.

Le variabili dei numeri alimentano solo il seed iniziale. Se il database contiene già il tenant demo, modificare esplicitamente i numeri e la connection ID nel pannello prima di passare in reale; non vengono sovrascritti al riavvio. Il flag AI del singolo tenant si gestisce nel pannello. Disabilitarlo non annulla la deviazione dell’operatore.

## Architettura e limiti

`src/config`, `db`, `telephony`, `voice`, `calls`, `whitelist`, `notifications`, `tenants`, `admin`, `security`, `testing` separano le responsabilità. Migrazioni SQL versionate, query parametrizzate Drizzle e vincoli compositi tenant/call impediscono collegamenti di lead/eventi a chiamate di altre imprese. Le API amministrative sono per un **unico operatore super-admin autorizzato a tutte le imprese**; non sono un portale self-service per singoli idraulici. Non distribuire quelle credenziali ai tenant. Non ci sono ruoli utente per tenant, reset password o MFA in questa versione.

Una sola istanza orchestratrice per database è imposta da advisory lock PostgreSQL; la inbox viene elaborata in serie. Gli eventi non correlabili attendono fino a 31 passaggi, poi risultano `unmatched`. Gli errori arrivano in `failed` dopo tre tentativi. I comandi hanno UUID persistiti e riutilizzati nei retry. In caso di crash fra effetto esterno e commit non si può garantire exactly-once a tempo illimitato: intervengono idempotenza Telnyx e recupero conservativo. Al riavvio le chiamate non già collegate vengono terminate e registrate; non si tenta di riaprire una conversazione audio persa. I Bridge già attivi rimangono gestiti da Telnyx. Questa versione non offre alta disponibilità né failover telefonico autonomo.

La coda SMTP riprova 5 volte con backoff e chiave univoca lead/canale. Un `Message-ID` stabile riduce duplicati, ma un crash dopo consegna SMTP e prima del commit può produrre un doppio invio: SMTP non fornisce exactly-once. In caso di errore configurazione o consegna consultare le notifiche nel pannello.

Minuti AI: prenotazione conservativa di una sessione intera per chiamata attiva e limite globale di concorrenza. Costi token registrati per risposta; lo stop sul costo avviene all’arrivo dell’uso, quindi una risposta può superare la soglia impostata. Un timeout indipendente e `max_output_tokens` limitano l’esposizione, non sostituiscono i limiti del provider. Cache conteggiata al prezzo pieno per una stima prudente. Minuti telefonici basati sui tempi osservati, non sul CDR fatturato. Mancano riconciliazione automatica fatture, scheduling appuntamenti, registrazione audio e trascrizione integrale. Vedi [COST_MODEL.md](COST_MODEL.md).

La conservazione si applica eseguendo `npm run retention`, da pianificare in produzione; non parte automaticamente. [PRIVACY_CHECKLIST.md](PRIVACY_CHECKLIST.md) e [PRODUCTION_DEPLOYMENT.md](PRODUCTION_DEPLOYMENT.md) descrivono le verifiche residue.

Se usi l’app Docker già avviata da questa sessione, per riavviarla basta `docker compose --profile app up -d --build`. Prima di passare a `npm run dev` sul computer esegui `docker compose stop app` per liberare la porta 3000, lasciando PostgreSQL attivo.
# centralinoAI
