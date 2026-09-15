# Note d'audit — Entropy Key Format

Versione v1.0.0-beta2. Ha ricevuto due giri di audit esterno indipendente:
il primo ha trovato solo rilievi documentali; il secondo, più approfondito
(Montgomery ladder X25519 eseguita end-to-end, tabelle FIPS 186-4 citate
con numeri esatti, ricerca attiva di input patologici), ha confermato
nessun difetto critico/alto e trovato 2 rilievi bassi + 1 informativo, tutti
corretti in questa build (whitelist su `rsaBitsSelect`, contraddizione
documentale sul clamping risolta, caso degenere di `randomBigIntInRange`
documentato) — vedi `CHANGELOG.md` per il dettaglio completo di entrambi i
giri. **Nota**: i numeri citati qui sotto (500 prove di clamping, 10
Carmichael, i 3 vettori HKDF) sono quelli usati negli script di sviluppo
una tantum, non nell'autotest imbarcato nell'app — quello, per restare
leggero a ogni caricamento pagina, usa campioni più piccoli (100 prove, 7
Carmichael, 2 vettori HKDF): esiste per rilevare regressioni in modo
economico, non per ripetere da zero la verifica di sviluppo.

## 1. HKDF (RFC 5869)

Cross-verificato contro `crypto.hkdfSync` di Node.js — un'implementazione
indipendente, non scritta da questo processo — sui 3 vettori di test
ufficiali della RFC (SHA-256): coincidenza esatta, byte per byte, in tutti
e tre i casi (con salt+info, con salt+info più lunghi, senza salt né info).
Questo è il livello di verifica più solido disponibile per questo
componente: non solo un vettore noto, ma un confronto diretto con un'altra
implementazione indipendente.

## 2. Clamping X25519 (RFC 7748 §5)

**Aggiornato dopo un rilievo dell'audit esterno.** Un primo tentativo di
verifica contro un vettore ufficiale RFC 7748 §5.2 scritto a memoria era
fallito per un errore di trascrizione (byte mancante), rilevato subito dal
controllo di lunghezza della funzione stessa — non un fallimento silenzioso.
Era stato sostituito con una sola verifica per proprietà, e un audit esterno
aveva giustamente osservato che l'assenza di un vettore numerico ufficiale
restava un limite reale, non chiuso dal solo fatto che il codice "sembra"
corrispondere alla specifica.

**Risolto**: scaricato il testo ufficiale della RFC 7748 direttamente da
`rfc-editor.org` (non dalla memoria) e confrontato in due modi:

1. **Confronto strutturale con lo pseudocodice ufficiale.** La RFC (§5)
   definisce `decodeScalar25519` come: `k[0] &= 248; k[31] &= 127; k[31] |= 64`
   — identico, operazione per operazione, alla funzione `clampX25519` del
   tool. Non un'approssimazione: è la stessa definizione, testuale.
2. **Vettore numerico ufficiale reale.** Preso lo scalare "Alice's private
   key, a" dal vettore Diffie-Hellman ufficiale (§6.1):
   `77076d0a7318a57d3c16c17251b26645df4c2f87ebc0992ab177fba51db92c2a`.
   Applicando `clampX25519` del tool si ottiene
   `70076d0a7318a57d3c16c17251b26645df4c2f87ebc0992ab177fba51db92c6a`,
   identico byte per byte al risultato dell'applicazione diretta dello
   pseudocodice `decodeScalar25519` della RFC allo stesso input.

Verifica ora equivalente per solidità a quella già fatta per HKDF: non solo
proprietà generiche, ma corrispondenza diretta con la specifica ufficiale
e con un suo vettore di test reale.

## 3. Miller-Rabin

Verificato contro: 10 numeri di Carmichael noti (561–29341, il caso che
serve davvero a Miller-Rabin rispetto a un test di Fermat più debole), 8
primi noti (incluso un primo di Mersenne), 7 compositi ovvi, e generazione
reale di primi con CSPRNG a più taglie di bit (512/1024/1536/2048),
verificando che il numero di tentativi necessari sia coerente con la
densità attesa dei primi.

**Corretto durante lo sviluppo, prima della consegna**: le basi di test
erano inizialmente fisse (elenco deterministico) — cambiato a basi casuali
(CSPRNG a ogni round) dopo aver riconsiderato le raccomandazioni FIPS
186-4/SP 800-89 per la generazione di materiale crittografico reale (a
differenza di un autotest che verifica solo fatti noti, dove basi fisse
restano adeguate). Riverificato dopo la correzione: 30 esecuzioni ripetute
del controllo Carmichael, nessuna intermittenza.

**Non verificato**: nessun confronto con un'implementazione di riferimento
esterna (es. la funzione `isProbablePrime` di una libreria BigInt matura, o
OpenSSL) sugli stessi identici candidati — la verifica si basa su casi noti
e proprietà statistiche attese, non su un secondo algoritmo indipendente
eseguito sugli stessi numeri.

## 4. Cosa NON è stato verificato in questo giro

- Nessun test su un browser reale (solo Node.js in questo ambiente di
  sviluppo) — in particolare `crypto.subtle.sign('HMAC', …)` non è stato
  testato su Firefox/Chrome reali, solo tramite l'implementazione
  Web-Crypto-compatibile di Node.
- Nessun benchmark di RSA-4096 su hardware più lento di questo ambiente di
  sviluppo (il tempo stimato ~20s per i due primi è dedotto dal benchmark a
  2048 bit per primo, non misurato specificamente per il caso 4096 completo).
- Nessun audit crittografico esterno indipendente su nessuno dei quattro
  componenti (HKDF, clamping, Miller-Rabin, l'integrazione fra i tre nella UI).
- Nessuna verifica dell'interazione con le librerie di destinazione
  (OpenSSL, libsodium) — cioè non è stato verificato che il materiale
  prodotto qui venga effettivamente importato correttamente da quelle
  librerie con il formato/endianness atteso, solo che rispetti le
  specifiche RFC sulla carta.
