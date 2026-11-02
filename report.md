# Assignment 01 - Ascensore: modello FSP e le 4 proprietà

Concurrent Systems a.a. 2026-2027 - Cesena

File con il codice: `as1_completo.lts`.
In questo report ci sono: il modello FSP (cap. 1), il modello con le reti di Petri (cap. 2), le 4 proprietà in FLTL (cap. 3-6) e i risultati dell'analisi con LTSA (cap. 7).
Il codice della rete di Petri è in `lift_petri_completo.py` (genera `lift_completo.pnml`).

## 0. Come ho diviso i 4 punti

| # | Proprietà | Fluenti | Cosa ho scritto | Dove si controlla in LTSA |
|---|---|---|---|---|
| 1 | Un solo utente alla volta usa l'ascensore | `A_IN_LIFT`, `B_IN_LIFT` | `MUTEX` (FLTL) e anche `property ONE_AT_A_TIME` | `LIFT_SYSTEM` (Check > LTL property) e `CHECK_ONE_AT_A_TIME` (Check > Safety) |
| 2 | Se chiamo l'ascensore prima o poi arriva (tranne allarme) | `A_CALLED`, `B_CALLED` | `A_LIFT_ARRIVES`, `B_LIFT_ARRIVES` | `LIFT_SYSTEM` (Check > LTL property) |
| 3 | Se scelgo un piano prima o poi ci arrivo (tranne allarme) | `A_SELECTED`, `B_SELECTED` | `A_SELECT_ARRIVES`, `B_SELECT_ARRIVES` | `LIFT_SYSTEM` (Check > LTL property) |
| 4 | Chi aspetta l'ascensore prima o poi lo ottiene (tranne allarme) | `A_WAITING`, `B_WAITING` | `A_GETS_LIFT`, `B_GETS_LIFT` | `LIFT_SYSTEM` (Check > LTL property) |

Il fluente `ALARM_ON` serve ai punti 2, 3 e 4, perché in tutti c'è l'eccezione "tranne in caso di allarme".
Il punto 1 è una proprietà di safety (una cosa brutta non deve mai succedere), i punti 2, 3 e 4 sono di liveness (una cosa buona deve prima o poi succedere).

---

## 1. Il modello FSP

Ho messo due utenti (`a` e `b`) e `N = 3` piani (il piano 0 è il piano terra).
Il sistema è:

```
||LIFT_SYSTEM = (USERS || QUEUE || LIFT || ALARM)
```

**Utente.** L'utente chiama l'ascensore al piano `f` (`call[f]`), aspetta che arrivi (`arrive[f]`), entra (`enter`), sceglie un piano `d` (`select[d]`) ed esce a quel piano (`exit[d]`).
Se suona l'allarme la chiamata viene annullata (`cancel`), oppure, se è già dentro, viene fatto uscire (`evacuate`).

```
USER       = (call[f:Floors] -> WAITING[f]),
WAITING[f:Floors] = (arrive[f] -> AT_DOOR | cancel -> USER),
AT_DOOR    = (enter -> INSIDE | cancel -> USER),
INSIDE     = (select[d:Floors] -> RIDING[d] | evacuate -> USER),
RIDING[d:Floors]  = (exit[d] -> USER | evacuate -> USER).

||USERS = (a:USER || b:USER).
```

**Coda FIFO.** La coda si ricorda in che ordine sono arrivate le chiamate. Per esempio `QAB` vuol dire "ha chiamato prima `a` e poi `b`". Con `serve` si prende sempre il più vecchio, quindi nessuno passa avanti agli altri.

```
QUEUE = EMPTY,
EMPTY = (a.call[Floors] -> QA | b.call[Floors] -> QB),
QA    = (a.serve -> EMPTY | b.call[Floors] -> QAB),
QB    = (b.serve -> EMPTY | a.call[Floors] -> QBA),
QAB   = (a.serve -> QB),
QBA   = (b.serve -> QA).
```

**Allarme e ascensore.** L'ascensore serve un utente alla volta e torna in `IDLE` solo quando quell'utente è uscito.
Se scatta l'allarme interrompe tutto: l'utente (se c'è) viene fatto scendere e l'ascensore aspetta il segnale `solved`.
Non ho modellato il movimento tra i piani passo per passo: `arrive[f]` vuol dire solo "l'ascensore è arrivato al piano f".

```
ALARM = (alarm -> solved -> ALARM).

LIFT = IDLE,
IDLE = (a.serve -> A_GO | b.serve -> B_GO | alarm -> STOPPED),
A_GO   = (a.arrive[f:Floors] -> A_DOOR | alarm -> a.cancel -> STOPPED),
A_DOOR = (a.enter -> A_IN | alarm -> a.cancel -> STOPPED),
A_IN   = (a.select[d:Floors] -> A_RIDE[d] | alarm -> a.evacuate -> STOPPED),
A_RIDE[d:Floors] = (a.exit[d] -> IDLE | alarm -> a.evacuate -> STOPPED),
(uguale per b)
STOPPED = (solved -> IDLE).
```

In LTSA il sistema `LIFT_SYSTEM` ha 178 stati e 426 transizioni e **non ha deadlock**.

---

## 2. Il modello con le reti di Petri

Ho fatto una rete posti/transizioni (pesi tutti 1) dello stesso sistema, per 2 utenti e 3 piani. La genera lo script `lift_petri_completo.py` e la esporta in `lift_completo.pnml`: **39 posti e 54 transizioni**. Ogni transizione della rete corrisponde a un'azione del modello FSP.

**Posti** (`u` = `a` oppure `b`, `f` = piano):

| Posti | Cosa vuol dire |
|---|---|
| `idle_u` | l'utente è fermo (stato `USER`) |
| `waiting_u_f` | l'utente ha chiamato al piano `f` e aspetta |
| `atdoor_u`, `inside_u`, `riding_u_f` | l'utente è alla porta / dentro / sta andando al piano `f` |
| `queue_E`, `queue_A`, `queue_B`, `queue_AB`, `queue_BA` | la coda FIFO: ha sempre un solo token, che dice l'ordine delle chiamate |
| `lift_idle` | l'ascensore è libero |
| `go_u`, `door_u`, `in_u`, `ride_u_f` | l'ascensore sta servendo `u`: va da lui / porta aperta / `u` dentro / porta `u` al piano `f` |
| `stopped` | l'ascensore è fermo per l'allarme |
| `no_alarm`, `alarm_on` | stato dell'allarme |

**Transizioni** (stesso nome delle azioni FSP):
- `call_u_f@Q`: consuma `idle_u` e lo stato della coda `Q`, produce `waiting_u_f` e il nuovo stato della coda.
- `serve_u@Q`: consuma lo stato della coda in cui `u` è il primo e `lift_idle`, produce `go_u`. È l'unico modo di iniziare un servizio, quindi l'ascensore serve **un utente alla volta e in ordine di arrivo**.
- `arrive_u_f`: consuma `go_u` e `waiting_u_f`, produce `door_u` e `atdoor_u`.
- `enter_u`, `select_u_f`, `exit_u_f`: fanno avanzare insieme il token dell'ascensore e quello dell'utente. `exit` rimette il token in `lift_idle`.
- allarme: ho una transizione `alarm_...` per ogni posto in cui può trovarsi l'ascensore. Consuma `no_alarm`, produce `alarm_on` e `stopped`; se c'era un utente lo rimette in `idle_u` (è la `cancel` o la `evacuate` del modello FSP). Per semplificare ho unito `alarm` e `cancel`/`evacuate` in una sola transizione.
- `solved`: consuma `stopped` e `alarm_on`, produce `lift_idle` e `no_alarm`.

**Analisi** (`python3 lift_petri_completo.py 3` esplora tutte le marcature):

```
posti=39 transizioni=54
marcature raggiungibili=114, max token in un posto=1
marcature di deadlock=0, transizioni mai abilitate=0
al piu' un utente dentro l'ascensore in ogni marcatura: True
reversibile (m0 raggiungibile da ogni marcatura): True
```

Quindi la rete è 1-safe (mai più di un token per posto), non va mai in deadlock, tutte le transizioni possono scattare, e in ogni marcatura c'è al massimo un utente nell'ascensore (stesso risultato del punto 1). Si può tornare sempre alla marcatura iniziale.

---

## 3. Punto 1 - un solo utente alla volta

In FLTL (come chiede la consegna):

```
fluent A_IN_LIFT = <a.enter, {a.exit[Floors], a.evacuate}>
fluent B_IN_LIFT = <b.enter, {b.exit[Floors], b.evacuate}>

assert MUTEX = []!(A_IN_LIFT && B_IN_LIFT)
```

`A_IN_LIFT` è vero da quando `a` entra fino a quando esce (o viene evacuato). L'assert dice che in ogni stato non possono essere veri insieme `A_IN_LIFT` e `B_IN_LIFT`.

Come `property` (si controlla direttamente con Check > Safety):

```
property ONE_AT_A_TIME = (a.enter -> A_IN | b.enter -> B_IN),
A_IN = (a.exit[Floors] -> ONE_AT_A_TIME | a.evacuate -> ONE_AT_A_TIME),
B_IN = (b.exit[Floors] -> ONE_AT_A_TIME | b.evacuate -> ONE_AT_A_TIME).

||CHECK_ONE_AT_A_TIME = (LIFT_SYSTEM || ONE_AT_A_TIME).
```

Dopo un `enter` non può esserci un altro `enter` finché quell'utente non è uscito (`exit`, oppure `evacuate` se c'è l'allarme).
Se il sistema potesse fare un'azione vietata, LTSA arriverebbe allo stato `ERROR`.

**Perché vale:** dopo `serve` l'ascensore resta occupato con quell'utente e torna in `IDLE` (dove può servirne un altro) solo dopo `exit` o `evacuate`.

---

## 4. Punto 2 - se chiamo l'ascensore prima o poi arriva (tranne allarme)

```
fluent A_CALLED = <a.call[Floors], {a.arrive[Floors], a.cancel}>
fluent B_CALLED = <b.call[Floors], {b.arrive[Floors], b.cancel}>

assert A_LIFT_ARRIVES = [](A_CALLED -> <>(!A_CALLED || ALARM_ON))
assert B_LIFT_ARRIVES = [](B_CALLED -> <>(!B_CALLED || ALARM_ON))
```

`A_CALLED` è vero da quando `a` chiama fino a quando l'ascensore arriva (o la chiamata viene annullata dall'allarme).
L'assert dice: sempre (`[]`), se la chiamata è in attesa, prima o poi (`<>`) non è più in attesa (l'ascensore è arrivato) oppure c'è l'allarme.

**Perché vale:** la coda è FIFO, quindi ogni chiamata prima o poi diventa la più vecchia e l'ascensore la serve.

---

## 5. Punto 3 - se scelgo un piano prima o poi ci arrivo (tranne allarme)

```
fluent A_SELECTED = <a.select[Floors], {a.exit[Floors], a.evacuate}>
fluent B_SELECTED = <b.select[Floors], {b.exit[Floors], b.evacuate}>

assert A_SELECT_ARRIVES = [](A_SELECTED -> <>(!A_SELECTED || ALARM_ON))
assert B_SELECT_ARRIVES = [](B_SELECTED -> <>(!B_SELECTED || ALARM_ON))
```

`A_SELECTED` è vero da quando l'utente sceglie il piano fino a quando esce a quel piano (o viene evacuato dall'allarme).

**Perché vale:** dopo `select[d]` l'ascensore può fare solo `exit[d]` oppure, se scatta l'allarme, `evacuate`.

---

## 6. Punto 4 - chi aspetta l'ascensore prima o poi lo ottiene (tranne allarme)

```
fluent A_WAITING = <a.call[Floors], {a.enter, a.cancel}>
fluent B_WAITING = <b.call[Floors], {b.enter, b.cancel}>

assert A_GETS_LIFT = [](A_WAITING -> <>(a.enter || ALARM_ON))
assert B_GETS_LIFT = [](B_WAITING -> <>(b.enter || ALARM_ON))
```

`A_WAITING` è vero da quando `a` chiama fino a quando entra (o la chiamata viene annullata).
L'assert dice che un utente che aspetta prima o poi entra, oppure c'è l'allarme.

**Perché vale:** per lo stesso motivo del punto 2, la coda è FIFO e nessuno resta indietro per sempre.

**Perché serve l'eccezione dell'allarme:** senza `|| ALARM_ON` la proprietà non vale su tutte le esecuzioni, perché l'allarme annulla la chiamata. La traccia che non rispetta la proprietà c'è, e la consegna dice espressamente di escludere il caso dell'allarme.

---

## 7. Risultati

| Controllo in LTSA | Target | Risultato |
|---|---|---|
| Safety (deadlock) | `LIFT_SYSTEM` | Nessun deadlock/errore (178 stati, 426 transizioni) |
| Safety, punto 1 (property) | `CHECK_ONE_AT_A_TIME` | Nessun deadlock/errore |
| LTL, punto 1 | `MUTEX` | **DA COMPILARE** dopo averlo lanciato in LTSA |
| LTL, punto 2 | `A_LIFT_ARRIVES`, `B_LIFT_ARRIVES` | **DA COMPILARE** dopo averlo lanciato in LTSA |
| LTL, punto 3 | `A_SELECT_ARRIVES`, `B_SELECT_ARRIVES` | **DA COMPILARE** dopo averlo lanciato in LTSA |
| LTL, punto 4 | `A_GETS_LIFT`, `B_GETS_LIFT` | **DA COMPILARE** dopo averlo lanciato in LTSA |
