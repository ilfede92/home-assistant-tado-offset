# home-assistant-tado-offset

![Tado Offset Manager](https://img.shields.io/badge/Home%20Assistant-Tado%20Offset-blue)

Blueprint e automazioni per regolare l'offset delle teste termostatiche Tado usando un sensore di temperatura esterno. Progettato per applicare cambi solo quando hanno effetto reale sul comportamento della valvola, riducendo chiamate inutili e preservando batteria.

Indice
- Panoramica
- Funzionamento e logica
- Requisiti
- Installazione
- Configurazione (per ogni valvola)
- Logging CSV e shell_command
- Esempi di setup
- Tuning e parametri consigliati
- Contribuire
- Licenza

----------------------------------------
Panoramica
----------------------------------------
Questo blueprint calcola e applica in modo conservativo l'offset di una testina Tado per allineare la temperatura mostrata dalla testina con quella di un sensore ambiente di riferimento.

Principio:
- La testina Tado mostra una temperatura influenzata dall'offset: T_disp = T_reale + O_curr
- Per far coincidere T_disp con il sensore esterno E si calcola:
  O_new = O_curr + (E - T_disp)

O_new → Offset nuovo (valore calcolato da applicare)
O_curr → Offset corrente (valore attualmente impostato sul device)
T_disp → Temperatura mostrata dalla testina Tado (display, già comprensiva dell'offset)
T_reale → Temperatura reale dell'ambiente (valore “vero” che vogliamo ottenere)
E → Temperatura rilevata dal sensore esterno di riferimento

Il blueprint:
- arrotonda O_new a 0.1 °C (precisione API Tado),
- applica clamp tra -9.9 e +10.0 °C,
- applica l'offset solo se una serie di controlli (vedi sotto) sono soddisfatti.

----------------------------------------
Funzionamento e logica
----------------------------------------
Condizioni che devono essere vere per applicare l'offset:
1. preset_mode della valvola = 'home' (opera solo in modalità home)
2. Entrambi i sensori (Tado e sensore esterno) sono numerici e disponibili
3. L'offset calcolato differisce dall'offset corrente (arrotondato)
4. La differenza |External - Tado_disp| supera la Tolleranza configurata
5. È trascorso almeno MinTime dall'ultimo cambio (input_datetime)
6. significant_state_change == True (valutato solo se la valvola ha una temperatura target): significa che l'applicazione dell'offset provocherebbe un cambio nella richiesta di calore (ON↔OFF)
- Se la temperatura target non è presente (ad es. valvola in off senza target), significant_state_change = False e l'automazione non applica l'offset.

Tutte le condizioni applicate servono per ridurre al minimo le impostazioni di offset per preservare al massimo la batteria e la vita dei dispositivi, considerando che ogni cambio di offset provoca la ricalibrazione della valvola.

----------------------------------------
Requisiti
----------------------------------------
- Home Assistant con supporto Blueprints.
- Integrazione Tado che espone:
  - attributo offset_celsius
  - (opzionale) attribute temperature (target)
  - servizio tado.set_climate_temperature_offset
- Per ciascuna valvola:
  - entità climate.<valvola>
  - sensore Tado che riporta temperatura della testa
  - sensore esterno di riferimento (sensor.<nome>)
  - input_datetime per tracciare l'ultimo cambio offset
  - counter opzionale per contare i tentativi/applicazioni

----------------------------------------
Installazione
----------------------------------------
1. Copia `TadoOffset.yaml` nella cartella `blueprints/automation/<tuo_utente>/` oppure importa il blueprint via UI.
2. Aggiungi le shell_command nel tuo `configuration.yaml` (vedi sezione sotto).
3. Per ogni valvola crea:
   - input_datetime.<nome> (es. input_datetime.last_offset_change_soggiorno)
   - counter.<nome> (opzionale)
4. Importa il blueprint in HA e crea un'istanza per ogni valvola, impostando i parametri.

----------------------------------------
Configurazione (parametri principali)
- ValvolaRiferimento: entità climate della valvola (es. climate.soggiorno)
- TadoTemperature: sensore che riporta la temperatura della testina Tado
- ExternalSensor: sensore esterno di riferimento
- UltimoCambioOffset: input_datetime che registra l'ultimo set (creato in precedenza)
- Contatore: counter (opzionale)
- Tolleranza: soglia (°C) per la differenza tra sensori (default 0.2)
- MinTime: tempo minimo (secondi) tra cambi (default 300)
- wait_epsilon: soglia (°C) usata per considerare confermato l'offset sul device (default 0.05)

Valori consigliati (default del blueprint):
- Tolleranza: 0.2 °C
- MinTime: 300 s (5 min)
- wait_epsilon: 0.05 °C
- wait timeout nel blueprint: 240 s (configurabile nel file se vuoi ridurlo)

----------------------------------------
Logging CSV e shell_command
----------------------------------------
L'automazione registra una riga CSV per ogni evento (anche non eseguito, per diagnostica). Il repository contiene un esempio di configurazione `shell_command` da inserire in `configuration.yaml`.

----------------------------------------
Esempio rapido di setup
1. Crea:
   - input_datetime.last_offset_change_soggiorno
   - counter.tado_soggiorno (opzionale)
2. Importa il blueprint e imposta gli input:
   - ValvolaRiferimento: climate.soggiorno
   - TadoTemperature: sensor.tado_soggiorno_temperature
   - ExternalSensor: sensor.temperatura_soggiorno
   - UltimoCambioOffset: input_datetime.last_offset_change_soggiorno
   - Contatore: counter.tado_soggiorno

----------------------------------------
Tuning e parametri consigliati
- Se vedi troppi set: aumenta Tolleranza o MinTime.
- Se vedi troppi blocchi per significant_state_change (target assente): verifica le modalità/preset della valvola; se vuoi agire anche senza target, richiedi modifica della logica (non consigliato di default).
- Se l'offset non viene confermato: verifica integrazione Tado e connessione delle testine; puoi aumentare il timeout del wait_template.


----------------------------------------
Contribuire
- Apri una issue con bug o richiesta.
- Per PR: aggiornare README e includere esempi concreti di entity_id.

----------------------------------------
Licenza
MIT
