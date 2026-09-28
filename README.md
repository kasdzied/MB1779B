# MB1779B
# Comunicazione P2P FSK su B-WL5M-SUBG1

## Scopo

Questo repository documenta le attività svolte per replicare e caratterizzare una comunicazione radio punto-punto tra due board **B-WL5M-SUBG1** basate su **STM32WL5MOC**.

Le attività eseguite comprendono:

- collegamento UART tramite convertitori CP2102;
- verifica del firmware AT presente sulle board;
- configurazione del collegamento FSK a 868 MHz;
- test di trasmissione e ricezione in entrambe le direzioni;
- test esteso da 100 pacchetti;
- raccolta di PER, RSSI e CFO;
- identificazione della versione firmware;
- prima analisi del payload generato dal firmware di test;
- definizione delle attività successive per programmazione SWD e analisi SDR.

Il contenuto è organizzato come documentazione tecnica di avanzamento, in modo da poter essere riutilizzato per attività di laboratorio, presentazioni o materiale formativo interno.

---

## Hardware utilizzato

| Componente | Descrizione |
|---|---|
| Board radio | 2 × B-WL5M-SUBG1 / MB1779 |
| MCU / radio | STM32WL5MOC |
| Interfaccia seriale | 2 × CP2102 USB-UART |
| Terminale | PuTTY |
| Antenne | Collegate ai connettori SMA delle board |

### Porte COM osservate

Durante le prove Windows ha identificato i due convertitori CP2102 come:

- COM3
- COM7

---

## Collegamento UART

Per ciascuna board è stato utilizzato il seguente cablaggio:

```text
CP2102 TX   -> UART RX della B-WL5M-SUBG1
CP2102 RX   <- UART TX della B-WL5M-SUBG1
CP2102 GND  -> GND
CP2102 5 V  -> alimentazione 5 V della board
```

Parametri seriali utilizzati:

```text
Baud rate:    115200
Data bits:    8
Parity:       None
Stop bits:    1
Flow control: None
```

In PuTTY è stato necessario abilitare il local echo per visualizzare i caratteri digitati:

```text
Terminal -> Local echo -> Force on
```

---

## Verifica del firmware AT

Il primo comando utilizzato è stato:

```text
AT?
```

La board ha restituito l'elenco dei comandi disponibili e `OK`.

Tra i comandi rilevanti per le prove radio:

```text
AT+TCONF
AT+TTX
AT+TRX
AT+TOFF
AT+TTONE
AT+TRSSI
```

La versione firmware letta con:

```text
AT+VER=?
```

è risultata:

```text
APPLICATION_VERSION: V1.5.0
MW_RADIO_VERSION:    V1.4.0
```

---

## Configurazione radio utilizzata

Entrambe le board sono state configurate con:

```text
AT+TCONF=868000000:14:50000:50000:4/5:0:0:0:16:25000:2:3
```

Configurazione associata al test:

| Parametro | Valore |
|---|---:|
| Frequenza | 868 MHz |
| Potenza TX | 14 dBm |
| RX bandwidth | 50 kHz |
| Bitrate FSK | 50 kbit/s |
| Modulazione | FSK / GFSK |
| Payload | 16 byte |
| FSK deviation | 25 kHz |

Nota: sul firmware utilizzato il comando `AT+TCONF=?` non ha restituito la configurazione corrente, mentre la stringa completa di configurazione è stata accettata correttamente con risposta `OK`.

---

## Procedura di test P2P

La ricezione deve essere avviata prima della trasmissione.

### Ricevente

```text
AT+TRX=10
```

### Trasmittente

Subito dopo:

```text
AT+TTX=10
```

Il parametro numerico indica il numero di pacchetti e non il numero di byte.

Con payload da 16 byte:

```text
10 pacchetti  -> 160 byte di payload
100 pacchetti -> 1600 byte di payload
```

Sono esclusi preambolo, header, CRC e ulteriore overhead radio.

---

## Risultati delle prove

### Test bidirezionale da 10 pacchetti

La prova è stata eseguita in entrambe le direzioni invertendo i ruoli delle due board.

| Direzione | Risultato |
|---|---|
| Board A -> Board B | 10/10 ricevuti, PER = 0% |
| Board B -> Board A | 10/10 ricevuti, PER = 0% |

Esempio di output lato RX:

```text
OnRxDone
RssiValue=-28 dBm, cfo=0kHz
Rx 10 of 10 >>> PER= 0 %
```

Esempio di output lato TX:

```text
Tx 10 of 10
OnTxDone
```

---

## Test da 100 pacchetti

È stato eseguito un test esteso con:

```text
RX: AT+TRX=100
TX: AT+TTX=100
```

Risultato:

| Indicatore | Valore |
|---|---:|
| Pacchetti attesi | 100 |
| Pacchetti ricevuti | 100 |
| PER | 0% |
| RSSI medio | circa -18.65 dBm |
| RSSI minimo | -25 dBm |
| RSSI massimo | -17 dBm |
| CFO | 0 kHz nei log |
| Durata sequenza TX | circa 50 s |
| Intervallo tra pacchetti | circa 0.5 s |

L'intervallo tra pacchetti è introdotto dall'applicazione di test e non rappresenta il limite della modulazione a 50 kbit/s.

---

## Interpretazione dei principali messaggi

| Messaggio / parametro | Significato |
|---|---|
| `OnTxDone` | La trasmissione del pacchetto è terminata lato TX |
| `OnRxDone` | Il pacchetto è stato ricevuto correttamente |
| `OnRxTimeout` | La finestra RX è terminata senza ricevere un pacchetto valido |
| PER | Packet Error Rate |
| RSSI | Stima della potenza del segnale ricevuto |
| CFO | Carrier Frequency Offset |

`OnTxDone` non costituisce conferma di ricezione da parte dell'altra board.

I timestamp stampati dalle due board non devono essere confrontati direttamente tra loro, perché derivano da timer locali non sincronizzati.

---

## Problemi incontrati durante la replica

### Caratteri non visibili in PuTTY

Causa:

```text
Local echo disattivato
```

Soluzione:

```text
Terminal -> Local echo -> Force on
```

### Timeout in ricezione

Nei primi tentativi la board RX ha riportato:

```text
OnRxTimeout
Rx 1 of 1 >>> PER= 100 %
```

La causa osservata è stata l'avvio tardivo della trasmissione rispetto alla finestra di ricezione.

Procedura corretta:

```text
1. avviare AT+TRX=N sulla ricevente
2. avviare immediatamente AT+TTX=N sulla trasmittente
```

---

## Analisi del payload di test

L'analisi del firmware ST associato all'applicazione `SubGHz_Phy_AT_Slave` indica che il test TX utilizza un payload generato tramite una sequenza PRBS9.

Per un payload di 16 byte, la sequenza ricostruita dal codice è:

```text
88 21 A7 1A F6 B2 13 85 5A 7E 93 B4 9F AC CC 80
```

La PRBS9 è una sequenza pseudocasuale deterministica utilizzata per testare il livello fisico. Non costituisce cifratura.

Nel test corrente lo stesso buffer viene riutilizzato durante la sequenza di pacchetti.

Il payload non è stato ancora verificato direttamente over-the-air tramite SDR né stampato dalla callback RX del firmware in uso.

---

## Struttura FSK rilevante per le attività successive

Dall'analisi del firmware di test risultano i seguenti elementi:

```text
Preamble
Sync word
Length
Payload
CRC
```

Per il test analizzato:

```text
Preamble:  3 byte
Sync word: C1 94 C1
Payload:   16 byte
CRC:       CCITT, 2 byte
Whitening: disabilitato nel percorso FSK analizzato
```

Questi parametri saranno utilizzati come riferimento per la futura acquisizione e demodulazione SDR.

---

## Programmazione e debug

La B-WL5M-SUBG1 non integra un debugger ST-LINK. La programmazione avviene tramite il connettore STDC14 `CN3`, che espone l'interfaccia SWD.

Segnali principali:

```text
3V3
GND
SWDIO
SWCLK
NRST
```

È stata valutata la possibilità di utilizzare lo ST-LINK integrato in una STM32F3DISCOVERY come programmatore/debugger esterno.

Questa attività non è ancora stata eseguita sul setup corrente.

---

## Attività successive

Le attività previste sono:

1. eseguire misure a distanze crescenti e registrare RSSI, CFO e PER;
2. eseguire sweep della potenza TX;
3. modificare il firmware per trasmettere un payload definito dall'utente;
4. modificare la ricezione per stampare il payload ricevuto via UART;
5. utilizzare SWD per caricare il firmware modificato;
6. acquisire i burst a 868 MHz con RTL-SDR;
7. demodulare la FSK/GFSK e confrontare il payload OTA con quello atteso;
8. utilizzare la baseline PHY per successive prove controllate di replay, spoofing/injection e analisi delle contromisure.

---

## Comandi di riferimento

```text
AT?                     # elenco comandi
AT+VER=?                # versioni firmware
AT+TCONF=...            # configurazione radio
AT+TRX=N                # ricezione di N pacchetti
AT+TTX=N                # trasmissione di N pacchetti
AT+TOFF                 # arresto del test RF
```

Configurazione utilizzata nelle prove:

```text
AT+TCONF=868000000:14:50000:50000:4/5:0:0:0:16:25000:2:3
```

---

## Riferimenti

- STMicroelectronics, **UM3127 – B-WL5M-SUBG1 User Manual**.
- STMicroelectronics, **STM32CubeWL**.
- Applicazione ST **SubGHz_Phy_AT_Slave**.
- Procedura interna di riferimento per il test P2P FSK con due B-WL5M-SUBG1.
- Log sperimentali raccolti durante le prove descritte in questo documento.

---

## Stato attuale

La comunicazione P2P FSK tra le due B-WL5M-SUBG1 è stata verificata in entrambe le direzioni. Il test esteso da 100 pacchetti ha prodotto 100 pacchetti ricevuti su 100 con PER pari a 0%. Le attività successive riguardano la modifica del firmware, la programmazione via SWD e l'analisi del segnale tramite SDR.
