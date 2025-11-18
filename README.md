# home-assistant-tado-offset

![Tado Offset Manager](https://img.shields.io/badge/Home%20Assistant-Tado%20Offset-blue)  
Blueprint e automazioni per regolare l'offset delle teste termostatiche Tado in base a un sensore di temperatura esterno, con logica pensata per preservare batteria, evitare oscillazioni e applicare modifiche solo quando hanno effetto reale sul riscaldamento.

Indice
- Panoramica
- Caratteristiche principali
- Requisiti
- Come installare
- Configurazione (per ogni valvola)
- Esempi (wizard passo-passo)
- Tuning e parametri consigliati
- Test e troubleshooting
- FAQ
- Contribuire
- Licenza

----------------------------------------
Panoramica
----------------------------------------
Questo progetto fornisce un blueprint per Home Assistant che calcola e applica l'offset alle valvole Tado in modo intelligente.  
Principio matematico usato:
- Tado mostra una temperatura influenzata dall'offset: T_disp = T_reale + O_curr
- Per far sì che la testina mostri il valore del sensore esterno E vogliamo O_new che soddisfi: T_disp - O_curr + O_new = E
- Da cui: O_new = O_curr + (E - T_disp)

Il blueprint calcola O_new, la arrotonda a 0.1°C (precisione dell’API), applica clamp ai limiti accettati e chiama il servizio Tado solo quando:
- la differenza fra sensore esterno e Tado supera la tolleranza configurata per la modalità,
- è passato il tempo minimo dall’ultimo cambio (input_datetime dedicato),
- l’offset arrotondato differisce da quello corrente arrotondato,
- e l’applicazione provocherebbe un cambiamento nella richiesta di calore (passare da on → off o viceversa), per evitare cambi inutili che consumano batteria.

----------------------------------------
Caratteristiche principali
----------------------------------------
- Calcolo corretto dell’offset tenendo conto dell’offset corrente.
- Arrotondamento a 0.1°C (scelta A).
- Differenziazione tolleranze per modalità: Home heating / Home off-idle / Away.
- MinTime (in secondi) per limitare frequenza dei cambi in base alla modalità.
- Battery saver con back_off option e controllo di impatto su hvac_action.
- Persistence configurabile sui trigger per ridurre falsi positivi.
- Wait fino a 120s per conferma applicazione; aggiorna comunque l’input_datetime se non confermato (strategia per preservare batterie).
- Watcher automation per aggiornare input_datetime quando offset viene cambiato manualmente.
- Logging diagnostico e counter per monitoraggio.

----------------------------------------
Requisiti
----------------------------------------
- Home Assistant (versione che supporta blueprints e servizi Tado).
- Integrazione Tado funzionante in HA.
- Per ogni valvola:
  - una entity climate.* Tado;
  - un sensore esterno (sensor.*);
  - un sensore interno Tado (sensor.* o attribute current_temperature);
  - un input_datetime dedicato per tracciare l'ultimo cambio offset;
  - un counter (opzionale ma consigliato) per tracciare le applicazioni.
- Controllare che la tua integrazione esponga:
  - attributo offset_celsius (o adattare il blueprint se il nome differisce);
  - servizio tado.set_climate_temperature_offset (verifica Developer Tools → Services).

----------------------------------------
Come installare
----------------------------------------
1. Aggiungi `TadoOffset.yaml` (il blueprint) nella cartella `blueprints/automation/ilfede92/` nel repository del progetto o importalo via UI (Configuration → Blueprints → Import).
2. Crea le entità necessarie in Home Assistant (input_datetime, counter) per ogni valvola che vuoi gestire.
3. Importa e abilita la watcher automation (opzionale ma consigliata) che aggiorna l'input_datetime quando `offset_celsius` cambia manualmente.
4. Crea una nuova automazione da blueprint, assegnando i parametri per la singola valvola.

----------------------------------------
Configurazione (per ogni valvola)
----------------------------------------
Parametri principali del blueprint:
- ValvolaRiferimento: entità climate della valvola (es. climate.ingresso)
- TadoTemperature: entità sensor che riporta la temperatura rilevata dalla testa
- ExternalTemperaure: sensore esterno (es. sensor.temperatura_esterna)
- UltimoCambioOffset: input_datetime per l’ultimo cambio (es. input_datetime.last_offset_change_ingresso)
- Contatore: counter per monitorare applicazioni (es. counter.tado_ingresso)
- TolleranzaHomeHeating / TolleranzaHomeOffIdle / TolleranzaAway (°C)
- MinTime* (in secondi) per modalità
- PersistenceMinutesTarget / PersistenceMinutesSource (minutes) per i trigger
- battery_saver (true/false), back_off_secs, hysteresis_threshold, wait_epsilon

Esempi consigliati (default):
- TolleranzaHomeHeating: 0.2
- TolleranzaHomeOffIdle: 0.4
- TolleranzaAway: 0.5
- MinTimeHomeHeating: 300 (5 min)
- MinTimeHomeOffIdle: 900 (15 min)
- MinTimeAway: 1800 (30 min)
- PersistenceMinutes*: 0.17 (~10s)
- battery_saver: true
- back_off_secs: 900 (15 min)
- hysteresis_threshold: 0.1
- wait_epsilon: 0.05

----------------------------------------
Esempio rapido di setup (passo-passo)
----------------------------------------
1. Crea le entità per la valvola "ingresso":
   - input_datetime.last_offset_change_ingresso
   - counter.tado_ingresso
2. Importa il blueprint e crea una nuova automazione:
   - ValvolaRiferimento: climate.ingresso
   - TadoTemperature: sensor.tado_ingresso_temperature
   - ExternalTemperaure: sensor.temperatura_esterna
   - UltimoCambioOffset: input_datetime.last_offset_change_ingresso
   - Contatore: counter.tado_ingresso
   - Mantieni i valori default per i parametri inizialmente
3. Importa/attiva la watcher automation:
   - trigger: climate.ingresso attribute offset_celsius
   - azione: input_datetime.set_datetime → input_datetime.last_offset_change_ingresso
4. Abilita log debug (temporaneamente) per `blueprints.tado.offset` o controlla i logbook per vedere i messaggi.

----------------------------------------
Testing e troubleshooting
----------------------------------------
Checklist di test:
- Cambia manualmente l'offset dalla UI e verifica che la watcher aggiorni l'input_datetime.
- Modifica temporaneamente il valore del sensore esterno in Developer Tools → States e osserva se l’automazione decide di applicare il nuovo offset solo quando appropriato.
- Controlla:
  - servizio chiamato: tado.set_climate_temperature_offset
  - attributo aggiornato: offset_celsius
  - che la chiamata non si ripeta inutilmente (counter)
  - che set effettivi avvengano solo quando significant_state_change diventa true (se battery_saver on)

Problemi comuni e soluzioni:
- L’automazione ripete troppe chiamate:
  - aumenta i MinTime* o back_off_secs;
  - aumenta le tolerance;
  - aumenta la persistence dei trigger.
- L’attributo offset_celsius non esiste:
  - verifica il nome reale dell’attributo nella tua integrazione; sostituisci i riferimenti nel blueprint se necessario.
- Trigger “for” non funziona nel blueprint:
  - alcuni parser non supportano templating in `for`. Imposta il `for` a un valore fisso (es. "00:00:10") temporaneamente o crea istanza dell’automazione senza `for`.
- Offset che “accumulano”:
  - assicurati che il calcolo usi l’offset corrente come specificato e che il confronto sia su valori arrotondati (0.1°C).
- Cambio non applicato entro 120s:
  - controlla la connessione/integrità dell’integrazione Tado; alcuni device impiegano più tempo a confermare.

----------------------------------------
FAQ rapida
----------------------------------------
D: Perché arrotondo a 0.1°C?
R: L’API/valvole Tado lavorano a 0.1°C e l’arrotondamento evita micro-oscillazioni e chiamate inutili.

D: Cosa succede se l’offset non viene applicato entro 120s?
R: Per scelta il blueprint aggiorna comunque l’input_datetime per evitare retry aggressivi (preserva batteria). Il fallimento rimane loggato.

D: Posso usare una sola watcher per tutte le valvole?
R: Sì, è possibile implementare una watcher “multi‑valvola” con template nel trigger, ma per semplicità e robustezza è più chiaro associare un input_datetime e una watcher per ogni valvola.

----------------------------------------
Contribuire
----------------------------------------
Se vuoi contribuire:
- apri una issue con descrizione del caso d’uso;
- crea una PR con modifiche (README, miglioramenti, test);
- scegli nomi chiari e fornisci esempi di entity_id per i test.

----------------------------------------
Licenza
----------------------------------------
MIT — sentiti libero di usare/adattare con attribuzione.

----------------------------------------
Note finali
----------------------------------------
- Prima di distribuire su tutte le valvole, testa singolarmente con logging abilitato per qualche ciclo (1–2 giorni).
- Se preferisci, posso generare una branch e aprire PR con:
  - blueprint aggiornato,
  - watcher automation di esempio per più valvole,
  - file di esempio `examples/automations_example.yaml` e `examples/setup.md`.

Buon lavoro — se vuoi procedo a creare la PR con questi file e aggiungo il watcher multi-valvola ed esempi pronti all'uso.  
