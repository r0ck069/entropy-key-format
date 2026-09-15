# Changelog

## v1.0.0-beta2 (2026-09-15) — risposta al secondo giro di audit

SHA-256 di `entropy-key-format-beta.html`:
`a4ba706e21218439001993ebc8220c94a2c57dc3484661808f3b82b82af4d9d7`.

Un secondo giro di audit, più approfondito del primo (Montgomery ladder
X25519 implementata ed eseguita end-to-end contro l'intero vettore
Diffie-Hellman ufficiale RFC 7748 §6.1, tabelle FIPS 186-4 citate con
numeri esatti, ricerca attiva di input patologici, verifica sistematica
dell'assenza di `innerHTML`), ha confermato **nessun difetto critico o
alto**. Tre rilievi minori, tutti verificati e corretti:

1. **[Bassa, difesa in profondità] Nessuna validazione di `rsaBitsSelect`.**
   Un DOM manomesso (DevTools) poteva forzare dimensioni RSA fuori dalle 3
   opzioni previste — 512 bit (chiave debole) o valori enormi (calcolo
   impraticabile, rischio di bloccare la scheda). **Corretto**: whitelist
   esplicita `{2048, 3072, 4096}`, rifiutato con messaggio chiaro qualunque
   altro valore. Verificato con 2 casi (512 e 100000) correttamente
   rifiutati, e il caso valido (2048) ancora funzionante.
2. **[Bassa, documentale] Contraddizione interna nel CHANGELOG.** La
   sezione che descrive la verifica di clamping "solo per proprietà" non
   era stata aggiornata per riflettere l'addendum post-audit (che aggiunge
   il vettore ufficiale) — un lettore che saltava direttamente a quella
   sezione avrebbe letto un'informazione superata. **Corretto**: la sezione
   ora rimanda esplicitamente all'addendum.
3. **[Informativa] `randomBigIntInRange` con `max<=min` non documentato.**
   Comportamento già corretto (ritorna `min`, mai raggiunto dalle chiamate
   reali del tool), solo non commentato esplicitamente. **Corretto**:
   aggiunto un commento che descrive il caso.

Aggiunto inoltre, per coerenza con un principio già applicato nel tool
gemello `entropy-crosscheck` (mai un input che scala senza un tetto
dichiarato): un limite `MAX_IKM_BYTES = 5.000.000` sul materiale in
ingresso, anche se l'audit non l'ha classificato come rilievo formale —
un IKM di decine di MB incollato in una textarea resterebbe comunque un
problema di reattività del browser indipendentemente dal costo
trascurabile dell'HMAC-SHA256 sottostante. Verificato: IKM oltre il tetto
correttamente rifiutato con messaggio chiaro.

Verifica indipendente dei claim principali del secondo audit prima di
accettarli: i tre valori del vettore Diffie-Hellman RFC 7748 §6.1 che
l'audit dichiara di aver ottenuto dalla propria Montgomery ladder (chiave
pubblica di Alice, di Bob, segreto condiviso) coincidono esattamente con
il testo ufficiale della RFC scaricato indipendentemente in questa stessa
serie di conversazioni; i numeri FIPS 186-4 citati (4 round per modulo
2048 bit, 3 per ≥3072 bit a errore 2^-100) sono confermati in sostanza dal
codice sorgente pubblico di OpenSSL che implementa quelle stesse tabelle
(`bn_rsa_fips186_4_prime_MR_min_checks`).

---

## v1.0.0-beta1 (2026-09-15) — BETA, non ancora pubblicata su GitHub

**Stato: da verificare e auditare prima del rilascio pubblico.** SHA-256 del
file `entropy-key-format-beta.html` di questa build (dopo la correzione
delle basi di Miller-Rabin descritta sotto):
`c8f5f16e58782777c3d404dbda79a526e4e2da942392365fe443088e4dd191cc`.

### Addendum post-audit (solo documentazione, hash del codice invariato)

Un audit esterno indipendente ha confermato la correttezza del codice
(nessun difetto critico/alto) ma ha trovato 3 problemi documentali reali:
testo residuo che descriveva ancora le basi di Miller-Rabin come fisse
(§ Componenti implementati sotto — corretto), e numeri di verifica
presentati senza distinguere fra script di sviluppo (più estesi) e autotest
imbarcato nell'app (più snello) — corretto separando esplicitamente i due
in tutta questa sezione.

Ho inoltre risolto io stesso, dopo l'audit, un punto su cui non ero
d'accordo con la sua chiusura troppo rapida ("le proprietà bastano, il
codice sembra corrispondere alla specifica"): ho scaricato il testo
ufficiale della RFC 7748 da rfc-editor.org e confrontato `clampX25519` sia
contro lo pseudocodice ufficiale `decodeScalar25519` (identico, operazione
per operazione) sia contro il vettore Diffie-Hellman ufficiale di §6.1
(scalare di Alice) — coincidenza byte per byte. Dettaglio in
`AUDIT-NOTES.md` §2.

Prima versione. Nato per colmare il gap identificato nella serie di 4 tool
di estrazione entropia: producono bit verificati statisticamente, ma nessuno
li trasforma in materiale usabile da una libreria crittografica reale.

### Corretto prima della consegna: basi di Miller-Rabin da fisse a casuali

La prima stesura usava un elenco fisso di 25 basi (2, 3, 5, ..., 97),
scelto per rendere il test riproducibile. Riconsiderato prima di consegnare:
per la GENERAZIONE DI MATERIALE CRITTOGRAFICO REALE (non per un autotest
che verifica solo fatti noti), FIPS 186-4/SP 800-89 raccomandano basi
**casuali** scelte a ogni round, non fisse — con basi note in anticipo esiste
in teoria la possibilità di costruire ad arte un composito che risulti
pseudo-primo proprio per quel set specifico. **Corretto**: le basi sono ora
generate con CSPRNG (`crypto.getRandomValues`, campionamento uniforme in
[2, n-2] per rifiuto, senza bias di modulo) a ogni round. Riverificato dopo
la correzione: nessuna intermittenza su 30 esecuzioni ripetute del controllo
sui numeri di Carmichael (40 round ciascuna) — teoricamente atteso, dato che
Miller-Rabin garantisce ≤1/4 di falsi "primo" per round qualunque sia la
base scelta, quindi basi casuali non riducono l'affidabilità del test.

### Componenti implementati

1. **HKDF (RFC 5869)** — HKDF-Extract e HKDF-Expand, basati su HMAC-SHA256
   nativo del browser (`crypto.subtle.sign('HMAC', …)`), non reimplementato
   a mano. Stessa filosofia già usata per SHA-256 negli altri 4 tool: fidarsi
   della primitiva nativa quando esiste, riscrivere solo quello che manca
   (come SHAKE256 in EntropyPipeline).
2. **Chiavi simmetriche**: AES-256 e HMAC-SHA256, derivate dalla stessa PRK
   con etichette (`info`) distinte.
3. **Seed per curve ellittiche**: X25519 con clamping RFC 7748 §5 applicato;
   Ed25519 con seed grezzo, non clampato (il clamping è interno al keygen
   standard, applicarlo qui sarebbe un errore).
4. **Primi per RSA**: crivello di divisione sui primi <128 seguito da
   Miller-Rabin a 40 round (basi scelte casualmente via CSPRNG a ogni round —
   vedi correzione sotto), su candidati derivati progressivamente dalla PRK
   (`info` = `RSA-{p|q}-candidate-{n}`, mai bit riusati fra un candidato
   scartato e il successivo).

### Verifica eseguita

**HKDF — verificato in fase di sviluppo su tutti e 3 i vettori ufficiali,
imbarcati nell'autotest solo in 2.** Lo script di sviluppo (non incluso in
questa consegna) ha confrontato l'implementazione contro `crypto.hkdfSync` di
Node.js (implementazione indipendente, non scritta da me) sui 3 vettori
ufficiali RFC 5869:

- Test Case 1 (IKM 22 byte, salt 13 byte, info 10 byte, L=42): coincidenza
  esatta, OKM = `3cb25f25faacd57a90434f64d0362f2a2d2d0a90cf1a5a4c5db02d56ecc4c5bf34007208d5b887185865`
- Test Case 2 (IKM/salt/info da 80 byte, L=82): coincidenza esatta
- Test Case 3 (senza salt né info, L=42): coincidenza esatta, OKM =
  `8da4e775a563c18f715f802a063c5a31b8a11f5c5ee1879ec3454e5f3c738d2d9d201395faa4b61a96c8`

Nell'autotest che gira DENTRO l'app a ogni caricamento sono imbarcati solo
Test Case 1 e Test Case 3 (il caso "con salt e info" e il caso "senza
nessuno dei due") — il Test Case 2 usa input/salt/info da 80+ byte ciascuno
e non aggiunge una proprietà distinta da verificare a runtime, solo peso al
codice.

**Clamping X25519 — verifica per proprietà (stato originale di questa
build, poi esteso — vedi l'addendum post-audit in cima a questo file).** Un
tentativo iniziale di verificare contro un vettore RFC 7748 §5.2 memorizzato
a mano è fallito per un probabile errore di trascrizione (byte mancante) —
scoperto subito dal controllo di lunghezza della funzione stessa. Invece di
correggere il vettore a memoria (rischiando di introdurre un secondo errore
di trascrizione invisibile), la verifica è stata sostituita con un controllo
per proprietà: bit 0-2 del primo byte sempre azzerati, bit 255 (più alto
dell'ultimo byte) sempre azzerato, bit 254 sempre impostato, tutti gli altri
253 bit invariati rispetto all'input. Eseguito su **500 array casuali** nello
script di sviluppo, su **100** nell'autotest imbarcato nell'app (stessa
proprietà, campione più piccolo per restare leggero a ogni caricamento
pagina — la correttezza della proprietà non dipende dalla numerosità, solo
la fiducia statistica che nessun caso limite sia stato perso per puro caso,
già alta a 100 prove). **Questo era il livello di verifica più debole di
tutta la build** (nessun vettore ufficiale, solo proprietà) — risolto
nell'addendum post-audit descritto in cima al file, che aggiunge sia il
confronto con lo pseudocodice ufficiale sia un vettore RFC 7748 §6.1 reale.

**Miller-Rabin — Carmichael, primi noti, generazione reale.** Numeri
verificati nello script di sviluppo (non incluso in questa consegna):
- **10** numeri di Carmichael (561 fino a 29341) tutti correttamente
  rilevati come compositi — il caso che serve DAVVERO a Miller-Rabin
  rispetto a un test di Fermat più semplice, che li classificherebbe
  erroneamente primi.
- 8 primi noti (da 2 a un primo di Mersenne 2⁶¹-1) tutti riconosciuti.
- 7 compositi ovvi tutti rilevati.
- Generazione reale con CSPRNG: primo a 512 bit trovato in 29 tentativi
  (24ms); primo a 1024 bit in 118 tentativi (150ms); primo a 1536 bit in
  685 tentativi (1,46s); primo a 2048 bit in 2178 tentativi (10,4s) — tempi
  coerenti con la densità attesa dei primi (~1 ogni ln(2^bit) candidati
  dispari).
- Coppia p,q completa per RSA-2048 (due primi da 1024 bit): 169ms totali nel
  benchmark standalone, 841ms nel test end-to-end della UI (include overhead
  di derivazione HKDF e I/O del harness di test, non solo Miller-Rabin).

Nell'autotest imbarcato nell'app, per restare leggero, sono presenti solo
**7 dei 10** numeri di Carmichael (561–8911, non 10585/15841/29341) e i
round di Miller-Rabin usati per questi controlli sono 20 (non i 40 usati
per la generazione reale delle chiavi) — sufficienti a verificare la
correttezza logica del test, non a rappresentare il livello di sicurezza
usato in produzione.

**Test end-to-end della UI**: flusso completo con IKM reale (32 byte CSPRNG)
→ estrazione PRK → derivazione simmetrica, X25519, Ed25519, RSA-2048 — tutti
i passaggi completati senza errori, output nei formati attesi.

**Autotest interno (imbarcato nell'app, gira a ogni caricamento pagina)**:
9 controlli — HKDF Test Case 1 e 3, proprietà del clamping su 100 prove,
vettore ufficiale RFC 7748 §6.1 (aggiunto nell'addendum post-audit sopra),
7 numeri di Carmichael, primi noti, compositi ovvi, generazione reale di un
primo a 256 bit, separazione di dominio — tutti superati. Numeri
volutamente più piccoli di quelli usati nello script di sviluppo una tantum
(sopra): l'autotest esiste per rilevare REGRESSIONI a ogni avvio in modo
economico, non per ristabilire da zero la fiducia già acquisita una volta
in fase di sviluppo.



### Non incluso in questa prima versione

- Nessuna interfaccia per costruire il file di chiave finale (PEM/DER) — per
  design, vedi `SECURITY-NOTES.md`.
- Nessun calcolo della chiave pubblica per X25519/Ed25519 — per design.
- Nessun completamento della chiave RSA oltre p, q — per design.
- Nessun audit crittografico esterno indipendente.
- Nessun benchmark su un browser reale (solo Node.js in questo ambiente di
  sviluppo) — i tempi di generazione RSA-4096 (~20s per i due primi) non
  sono stati misurati su hardware mobile o più lento.
