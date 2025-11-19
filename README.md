# home-assistant-tado-offset

![Tado Offset Manager](https://img.shields.io/badge/Home%20Assistant-Tado%20Offset-blue)

Blueprint e automazioni per regolare l'offset delle teste termostatiche Tado in base a un sensore di temperatura esterno. Progettato per applicare cambi solo quando hanno effetto reale sul comportamento della valvola, riducendo chiamate inutili e preservando la batteria.

Indice
- Panoramica
- Caratteristiche principali
- Requisiti
- Installazione
- Configurazione (per ogni valvola)
- Logging CSV e shell_command
- Esempi (setup passo-passo)
- Tuning e parametri consigliati
- Testing e troubleshooting
- FAQ
- Contribuire
- Licenza

----------------------------------------
Panoramica
----------------------------------------
Questo progetto fornisce un blueprint Home Assistant che calcola e applica l'offset alle valvole Tado in modo intelligente:

- Rappresentazione fondamentale:
  - La valvola mostra una temperatura influenzata dall'offset: T_disp = T_reale + O_curr
  - Per far coincidere la temperatura mostrata con la temperatura del sensore esterno E si calcola:
    O_new = O_curr + (E - T_disp)

- Il blueprint:
  - calcola O_new e lo arrotonda a 0.1°C (precisione usata dall'API Tado);
  - applica clamp ai limiti (-9.9 … 10.0);
  - chiama il servizio Tado per impostare l'offset solo se:
    - la differenza tra sensore esterno e Tado supera la tolleranza definita per la modalità,
    - è passato il tempo minimo dall'ultimo cambio (input_datetime dedicato),
    - l'offset arrotondato differisce da quello corrente,
    - l'applicazione ha potenziale impatto sul funzionamento della valvola (evita cambi inutili che consumano batteria).

----------------------------------------
Caratteristiche principali
----------------------------------------
- Calcolo corretto dell'offset (tiene conto dell'offset corrente).
- Arrotondamento a 0.1°C per coerenza con la precisione API.
- Tolleranze separate per modalità: Home (heating / off-idle) e Away.
- MinTime (in secondi) per limitare la frequenza in funzione della modalità.
- Fallback conservativo quando la target temperature è null (tipico quando la valvola è "off"): applica cambi solo per differenze più ampie (diff >= 2×tollerance).
- Attesa fino a 120s per conferma dell'applicazione (wait_template); anche se non confermato, l'input_datetime viene aggiornato (strategia per preservare batteria).
- Logging CSV su file (comandi shell configurati in configuration.yaml) per facile importazione in Excel.
- Watcher automation esempio per aggiornare l'input_datetime quando l'offset viene cambiato manualmente.

----------------------------------------
Requisiti
----------------------------------------
- Home Assistant con supporto a Blueprints e accesso ai servizi Tado.
- Integrazione Tado attiva in HA che espone:
  - attributo offset_celsius
  - attributo temperature (se presente; gestiamo il caso null)
  - servizio tado.set_climate_temperature_offset
- Per ogni valvola gestita:
  - entità climate.<nome_valvola>
  - sensore Tado (sensore che riporta la temperatura della testina)
  - sensore esterno (sensor.<nome>)
  - input_datetime per tracciare l'ultimo cambio offset (es. input_datetime.last_offset_change_<room>)
  - counter opzionale per tracciare i tentativi/applicazioni

----------------------------------------
Installazione
----------------------------------------
1. Copia `TadoOffset.yaml` nella cartella `blueprints/automation/ilfede92/` (o importa il blueprint via UI).
2. Aggiungi le shell_command in `configuration.yaml` (esempio già presente nel repository — vedi sotto la sezione "Logging CSV and shell_command").
3. Crea per ogni valvola:
   - un `input_datetime` (es. input_datetime.last_offset_change_soggiorno)
   - un `counter` (opzionale; es. counter.tado_soggiorno)
4. Importa il blueprint in Home Assistant e crea una nuova automazione per ogni valvola impostando gli input corretti.

----------------------------------------
Configurazione (per ogni valvola)
----------------------------------------
Parametri principali:
- ValvolaRiferimento: entità climate della valvola (es. climate.soggiorno)
- TadoTemperature: sensore che riporta la temperatura della testa Tado
- ExternalTemperaure: sensore esterno di riferimento
- UltimoCambioOffset: input_datetime per tracciare l'ultimo cambio
- Contatore: counter (opzionale) per tracciare applicazioni
- TolleranzaHomeHeating / TolleranzaHomeOffIdle / TolleranzaAway (°C)
- MinTimeHomeHeating / MinTimeHomeOffIdle / MinTimeAway (secondi)
- wait_epsilon: soglia numerica per considerare l'offset applicato (es. 0.05)

Valori consigliati (default nel blueprint):
- TolleranzaHomeHeating: 0.2
- TolleranzaHomeOffIdle: 0.4
- TolleranzaAway: 0.5
- MinTimeHomeHeating: 300 (5 min)
- MinTimeHomeOffIdle: 900 (15 min)
- MinTimeAway: 1800 (30 min)
- wait_epsilon: 0.05

----------------------------------------
Logging CSV e shell_command
----------------------------------------
Il repository contiene un file `configuration.yaml` di esempio che definisce i seguenti shell_command usati dall'automazione:

```yaml
shell_command:
  tado_dir: /bin/bash -c "[ -d /config/log/offset/$(date +%Y-%m) ] || mkdir -p /config/log/offset/$(date +%Y-%m)"
  tado_fil: /bin/bash -c "[ -f /config/log/offset/$(date +%Y-%m)/day-$(date +%d).log ] || echo 'Data;Ora;Nome;Modalità;Preset_Mode;Hvac_Action;Tado_Temperature;Sensor_Temperature;Differenza_Temperature;Tolleranza;Offset_Impostato;Offset_Nuovo;NewTemp;TargetTemp;Ultima_Variazione;Now;SecLastadjust;min_time_between_adjust;C1_Sensore_isnumber;C2_Tado_isnumber;C3_OffsetImpostato != Offset_Nuovo;C4_Offset>Tolleranza;C5_Tempo;C6_preset!=Away;C7_hvac!=off;C8_NewTemp<target;TutteLeCondizioni;' > /config/log/offset/$(date +%Y-%m)/day-$(date +%d).log"
  tado_log: '/bin/bash -c "echo \"$(date +%Y-%m-%d); $(date +%H:%M:%S); {{text}}\" >> /config/log/offset/$(date +%Y-%m)/day-$(date +%d).log"'
```

- L'automazione esegue in sequenza: `tado_dir` (crea la directory mensile), `tado_fil` (crea il file giornaliero con intestazione se mancante) e `tado_log` (appende la riga CSV).
- Il file generato si trova in `/config/log/offset/YYYY-MM/day-DD.log`.
- La riga è in formato CSV separato da `;`. Intestazione e colonne corrispondono a quanto definito nel `tado_fil`.

Esempio di intestazione generata automaticamente:
Data;Ora;Nome;Modalità;Preset_Mode;Hvac_Action;Tado_Temperature;Sensor_Temperature;Differenza_Temperature;Tolleranza;Offset_Impostato;Offset_Nuovo;NewTemp;TargetTemp;Ultima_Variazione;Now;SecLastadjust;min_time_between_adjust;C1_Sensore_isnumber;C2_Tado_isnumber;C3_OffsetImpostato != Offset_Nuovo;C4_Offset>Tolleranza;C5_Tempo;C6_preset!=Away;C7_hvac!=off;C8_NewTemp<target;TutteLeCondizioni;

Nota: Se desideri un file `.csv` esplicito, puoi modificare `tado_fil` per scrivere `day-$(date +%d).csv`.

----------------------------------------
Esempio rapido di setup (passo-passo)
----------------------------------------
1. Creare le entità di supporto (per es. per stanza "soggiorno"):
   - input_datetime.last_offset_change_soggiorno
   - counter.tado_soggiorno
2. Importare e configurare il blueprint:
   - ValvolaRiferimento: climate.soggiorno
   - TadoTemperature: sensor.tado_soggiorno_temperature
   - ExternalTemperaure: sensor.temperatura_soggiorno
   - UltimoCambioOffset: input_datetime.last_offset_change_soggiorno
   - Contatore: counter.tado_soggiorno
3. (Opzionale ma consigliato) Aggiungere watcher automation che aggiorna l'input_datetime quando l'offset viene cambiato manualmente:
```yaml
alias: Aggiorna input_datetime su cambio offset Tado - soggiorno
trigger:
  - platform: state
    entity_id: climate.soggiorno
    attribute: offset_celsius
action:
  - service: input_datetime.set_datetime
    target:
      entity_id: input_datetime.last_offset_change_soggiorno
    data:
      date_time: "{{ now().strftime('%Y-%m-%d %H:%M:%S') }}"
mode: single
```
4. Abilitare logging (opzionale) per il debug:
   - Imposta il logger `blueprints.tado.offset` a livello `debug` per alcune ore.

----------------------------------------
Tuning e parametri consigliati
----------------------------------------
- Se vedi troppi set non necessari:
  - aumenta le Tolleranza* o
  - aumenta MinTime* o
  - aumenta la persistence dei trigger (nel blueprint sono impostati `for: 10s` sui trigger)
- Se i cambi non sono applicati entro 120s:
  - verifica stato integrazione Tado e connettività
  - aumenta il timeout `wait_template` se necessario
- Se vuoi variazioni più graduali, è possibile aggiungere smoothing (non implementato di default).

----------------------------------------
Testing e troubleshooting
----------------------------------------
- Test manuale:
  - Usa Developer Tools → States per simulare cambi nel sensore esterno e osservare la riga CSV.
  - Verifica che il counter incrementi solo quando viene effettivamente invocato il set.
- Problemi comuni:
  - errori template su `unavailable` o `None`: i template nel blueprint sono stati resi robusti usando `| float(0)`; assicurati comunque che le entità passate esistano.
  - trigger “for” non rispettato nel tuo environment: sostituisci il for con un valore fisso (es. "00:00:10") nell'istanza dell'automazione se necessario.
  - offset non aggiornato: verifica che l'attributo `offset_celsius` sia esposto dalla tua integrazione Tado.

----------------------------------------
FAQ rapida
----------------------------------------
Q: Perché l'automazione aggiorna l'input_datetime anche se il device non conferma entro 120s?  
A: È una scelta per evitare retry aggressivi che consumerebbero la batteria delle testine. Il fallimento resta loggato e puoi analizzarlo via CSV.

Q: Posso usare una sola watcher per tutte le valvole?  
A: Sì, però è più semplice e robusto avere un input_datetime e watcher per ciascuna valvola; è possibile però creare una watcher multi-valvola con un template nel trigger.

----------------------------------------
Contribuire
----------------------------------------
Se vuoi contribuire:
- apri una issue con il caso d'uso o bug;
- crea una PR con la modifica; includi esempi di entity_id concreti per i test.

----------------------------------------
Licenza
----------------------------------------
MIT — sentiti libero di usare/adattare con attribuzione.

