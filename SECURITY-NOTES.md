# Principi di sicurezza e di design — Entropy Key Format

## Principio 1 — Fermarsi dove finisce la matematica ben nota e comincia il rischio di implementazione

HKDF, il clamping X25519, e Miller-Rabin sono operazioni ben specificate,
verificabili contro vettori pubblici o proprietà matematiche semplici. La
moltiplicazione scalare su Curve25519 per ottenere una chiave pubblica, la
costruzione di un file PEM/DER, il calcolo di φ(n)/e/d per RSA — sono
operazioni dove un bug sottile (spesso invisibile ai test funzionali
ordinari) può introdurre una vulnerabilità reale, in alcuni casi a canale
laterale. Questo tool si ferma deliberatamente prima di quella linea. Il
criterio pratico: se un errore nell'implementazione produce un output che
"sembra giusto" ma è comunque insicuro (es. una chiave pubblica calcolata
con un'aritmetica di campo leggermente sbagliata, che spesso continua a
funzionare ma con proprietà di sicurezza compromesse), quella funzionalità
non appartiene a questo tool.

## Principio 2 — Un solo pool di entropia, separazione di dominio esplicita

Non c'è bisogno di generare entropia diversa per ogni chiave: HKDF con
etichette (`info`) distinte dalla stessa PRK è esattamente lo strumento
progettato per questo, ed è quello che il tool usa ovunque (chiavi
simmetriche, seed di curve, candidati RSA). Riusare la stessa PRK con la
stessa etichetta per scopi diversi, invece, romperebbe la separazione — ogni
nuovo tipo di output aggiunto in futuro deve avere la propria etichetta
univoca, mai riusata altrove nel codice.

## Principio 3 — Mai bit riusati fra candidati RSA scartati

Ogni candidato primo testato e scartato consuma la sua porzione di
keystream derivato (etichettato con un contatore univoco) e non viene mai
riletto. Riusare gli stessi bit per un secondo tentativo equivarrebbe a
non consumare entropia fresca, vanificando lo scopo della ricerca.

## Principio 4 — Verificare per proprietà quando un vettore memorizzato è a rischio di errore di trascrizione

Durante lo sviluppo, un tentativo di verificare il clamping X25519 contro un
vettore RFC 7748 §5.2 scritto a mano dalla memoria è fallito per un byte
mancante — scoperto subito dal controllo di lunghezza, non silenziosamente.
Anziché correggere il vettore a memoria una seconda volta (stesso rischio),
la verifica è stata sostituita con un controllo delle proprietà matematiche
richieste dalla specifica su centinaia di input casuali. Regola generale:
un vettore di test memorizzato va sempre preferito quando è disponibile con
certezza (come i 3 vettori RFC 5869, cross-verificati contro `crypto.hkdfSync`
di Node prima di essere usati), ma quando la certezza sul valore esatto
manca, verificare le proprietà matematiche della specifica è più robusto di
un vettore potenzialmente sbagliato.

## Principio 5 — Mai materiale crittografico reale in una sessione condivisa

Come per `entropy-crosscheck`: questo tool gira interamente in RAM lato
client (nessuna rete, nessuna scrittura su disco), ma questo non elimina i
rischi ordinari di un ambiente browser non controllato. Le chiavi derivate
qui, se destinate a un uso reale, andrebbero generate su una macchina isolata
(vedi le note di processo discusse con l'utente: sessione Live, rete
fisicamente disconnessa) — mai in una sessione di test o su una macchina
condivisa.

## Principio 6 (ereditato) — Mai un "PASS" come prova di sicurezza

Come per gli altri quattro tool della serie: superare l'autotest interno è
necessario ma mai sufficiente. In particolare, la generazione di primi RSA
qui non ha ricevuto alcun audit crittografico esterno. Le basi di
Miller-Rabin sono scelte casualmente a ogni round (CSPRNG), non da un
elenco fisso — corretto prima della consegna proprio per eliminare il
rischio, discusso in letteratura (FIPS 186-4/SP 800-89), che un composito
possa in teoria essere costruito ad arte contro un set di basi note in
anticipo. A 40 round con basi casuali il tasso di falso "primo" è ≤4⁻⁴⁰,
trascurabile per uso pratico — ma resta una garanzia probabilistica, non
una dimostrazione matematica assoluta, e nessuna terza parte l'ha ancora
rivista.

## Principio 7 (nuovo, v1.0.0-beta2) — Mai fidarsi di un elemento `<select>` come unica validazione

Un menu a tendina HTML con 3 opzioni limita cosa può scegliere un utente
che interagisce normalmente con la pagina — non limita cosa può arrivare
al codice JavaScript. Un DOM manomesso (DevTools, un'estensione, una copia
modificata della pagina) può assegnare qualunque valore a `.value` prima
che il codice lo legga. Il tool aveva `rsaBitsSelect` con questo esatto
problema (trovato da un audit esterno, secondo giro): nessuna validazione
lato codice, solo l'apparente sicurezza delle opzioni HTML. Regola per il
futuro: qualunque parametro che influenza la scala di un calcolo (dimensione
di un modulo, numero di round, lunghezza di un output) va sempre validato
esplicitamente contro una whitelist nel codice, indipendentemente da quali
opzioni offre l'interfaccia — la stessa logica già applicata ai tetti
dimensionali (`MAX_IKM_BYTES` qui, `MAX_PAIRWISE_BYTES`/`MAX_PER_SOURCE_BYTES`
in `entropy-crosscheck`), estesa esplicitamente anche ai parametri "a scelta
multipla", non solo alle dimensioni libere dell'input.
