# Entropy Key Format — da entropia grezza a materiale per chiavi

> ⚠ **v1.0.0-beta2 — BETA non ancora pubblicata su GitHub.** Due giri di
> audit esterno indipendente: il primo ha trovato solo rilievi documentali;
> il secondo, più approfondito (Montgomery ladder X25519 eseguita
> end-to-end, tabelle FIPS 186-4 citate con numeri esatti), ha confermato
> nessun difetto critico/alto e portato a una whitelist esplicita sulle
> dimensioni RSA consentite e a un tetto sulla dimensione dell'IKM. Dettaglio
> completo in `CHANGELOG.md`. Vedi `SECURITY-NOTES.md` per i principi di
> design.

Quinto tool della stessa serie (EntropyPipeline / entropy-extractor /
entropy-extractor-raw-2photo / entropy-crosscheck). Colma il gap più
concreto lasciato aperto dagli altri quattro: producono ottima entropia
grezza verificata statisticamente, ma si fermano prima che diventi materiale
effettivamente usabile da una libreria crittografica reale. Questo tool fa
esattamente quel passo, e nient'altro.

## Cosa fa

Prende l'output finale di uno degli altri tre tool (bit sequenziali o
esadecimale, minimo 256 bit) e lo passa da **HKDF** (RFC 5869, HMAC-SHA256)
per derivare, con separazione di dominio esplicita, materiale grezzo per:

- **Chiavi simmetriche**: AES-256, HMAC-SHA256 (32 byte ciascuna)
- **Seed per curve ellittiche**: X25519 (con clamping RFC 7748 già applicato),
  Ed25519 (seed grezzo — il clamping è interno al keygen standard, non va
  applicato qui)
- **Primi per RSA**: p e q, generati con crivello di divisione + Miller-Rabin
  a 40 round, per RSA-2048/3072/4096

## Cosa NON fa — di proposito

- **Non produce mai un file di chiave finito** (niente PEM, DER, PKCS#8).
  Quella codifica è ad alto rischio di errori sottili con conseguenze enormi:
  il tool consegna byte/scalari/numeri primi grezzi, con istruzioni su come
  importarli in OpenSSL, libsodium, age o GPG per il passo finale.
- **Non calcola mai la chiave pubblica su curva ellittica.** Implementare
  l'aritmetica di campo di Curve25519 da zero è terreno per librerie
  controllate (libsodium), non per un tool scritto in un pomeriggio — un
  bug sottile lì può introdurre vulnerabilità a canale laterale.
- **Non completa mai una chiave RSA.** Nessun calcolo di φ(n), nessuna
  scelta dell'esponente e, nessun calcolo di d. Solo p e q.

## Uso

Apri `entropy-key-format-beta.html` in un browser moderno. Incolla l'output
finale di uno degli altri tre tool, premi "Estrai PRK", poi scegli quale
materiale derivare. Ogni derivazione usa un'etichetta (`info`) diversa dalla
stessa PRK — mai la stessa chiave riusata per due scopi diversi.

## Perché fidarsi della separazione di dominio invece che di più sorgenti

Un solo blocco di entropia (256+ bit da uno degli altri tool) è sufficiente
per derivare in sicurezza tutte le chiavi di cui sopra: è esattamente lo
scopo per cui HKDF esiste. Non serve — anzi è controproducente — generare
entropia separata per ogni chiave.

## Verifica eseguita

- **HKDF**: implementazione incrociata contro `crypto.hkdfSync` di Node.js
  (indipendente) e contro i 3 vettori ufficiali RFC 5869 (Test Case 1, 2, 3)
  — coincidenza esatta, byte per byte.
- **Clamping X25519**: verificato contro il testo ufficiale della RFC 7748
  (scaricato da rfc-editor.org) in due modi — corrispondenza strutturale con
  lo pseudocodice `decodeScalar25519` della specifica, e coincidenza byte per
  byte sul vettore Diffie-Hellman ufficiale di §6.1 — oltre alla verifica per
  proprietà su centinaia di array casuali (bit 0-2 del primo byte sempre
  azzerati, bit più alto dell'ultimo byte sempre azzerato, bit penultimo
  sempre impostato, tutti gli altri bit intatti).
- **Miller-Rabin**: tutti i numeri di Carmichael fino a 8911 correttamente
  rilevati come compositi (il caso che fa fallire un test di Fermat
  semplice), primi noti riconosciuti, compositi ovvi rilevati, generazione
  reale di primi con CSPRNG verificata end-to-end.

Dettaglio numerico completo in `CHANGELOG.md`.
