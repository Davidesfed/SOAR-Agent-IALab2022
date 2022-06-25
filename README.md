# Progetto ESCAPE
Intelligenza Artificiale e Laboratorio, a.a. 21/22

Davide Mauri, Giordano Scarso

## Esecuzione del programma

Il programma è diviso in diversi file, ciascuno dei quali contiene un insieme ben preciso di produzioni, la cui funzione generica è specificata dal nome del file stesso. In particolare:
- __init.soar__ contiene i settings del Reinforcement Learning e il caricamento degli altri file;
- __ambient.soar__ contiene le produzioni che specificano il comportamento dell'ambiente, in particolare la ``propose-space`` iniziale e le produzioniche regolano la terminazione di un singolo episodio
- __rlagent.soar__ contiene invece le produzioni propose/apply delle azioni che può compiere l'agente
- __rlrules.soar__ contiene le regole RL che calcolano i Q-value relativi alle azioni dell'agente
- __rewards.soar__ contiene le produzioni di elaborazione che assegnano ricompense alle azioni compiute dall'agente

Per caricare il programma nella memoria di SOAR è sufficiente dare il seguente comando nel SoarJavaDebugger.
```
load file init.soar
```
A questo punto un comando ``run`` permette l'esecuzione di un singolo episodio. Se si vuole monitorare l'evoluzione dei Q-value assegnati ad ogni regola RL, è consigliato, in uno dei terminali a lato, il comando ``print --all``.


## Ipotesi di lavoro
L'ambiente è immaginato come una stanza dove non c'è una concezione dello spazio fisico vero e proprio. L'unica eccezione a tale regola la si ha nell'altezza. L'ambiente ha, infatti, un attributo _^height_ che tiene traccia dell'altezza massima raggiungibile dall'agente in ogni momento.

Tutti gli elementi (pietre, elastico, rametto, tronchi e finestra) sono considerati oggetti e si trovano dispersi nell'ambiente. Proprio in virtù del fatto che, come detto poc'anzi, non c'è una concezione vera e propria dello spazio fisico, i movimenti dell'agente sono unicamente da e verso gli oggetti presenti nel mondo. Inoltre egli ha a disposizione una lista di oggetti nelle proprie vicinanze, che verrà modificata dai movimenti. 

Supponiamo inoltre che 
- l'agente all'inizio di ogni episodio si trova vicino all'elastico;
- l'agente possiede due mani, ciascuna delle quali è in grado di tenere un oggetto;
- una volta che l'agente combina due oggetti, non è più in grado di scomporli. Pertanto se dovesse combinare l'elastico con le pietre o il rametto con le pietre, non avrebbe più alcun modo per fuggire. Ciò comporta, chiaramente, la terminazione dell'episodio.
- la presenza/assenza di un oggetto nel mondo è determinata dall'attributo _taken_. In particolare lo stato $s$ avrà un attributo ``<s> ^taken <o>`` per ogni oggetto $o$ che è stato preso in mano una volta. L'azione _letgo_ elimina questo attributo taken riposizionando l'oggetto nel mondo.

## Descrizione delle azioni

Le azioni che l'agente, in generale, può compiere sono le seguenti: _move_, _take_, _combine_, _utilize_, _letgo_, _escape_. Nel caso della _utilize_ è necessario dividere in _utilize*fionda_ e _utilize*tronco_.
L'azione _letgo_, lasciare un oggetto, è attualmente permessa per i tre oggetti componibili (rametto, pietre ed elastico).
E' possibile estendere tale azione anche per i tronchi e la fionda. Per farlo è necessario modificare le LHS delle produzioni `propose*letgo` del file _rlagent.soar_ e `generate*rl-rule*letgo` del file _rlrules.soar_. Effettuare questo cambiamento comporta, però un processo di addestramento molto più lungo. 

Muoversi verso un oggetto: _move_
  - _Condizioni:_ l'oggetto desiderato è lontano.
  - _Azioni:_ modifica della lista degli oggetti vicini

Prendere un oggetto: _take_
- _Condizioni:_ avere almeno una mano libera  e un oggetto vicino
- _Azioni:_ l'oggetto viene tolto dal mondo e posizionato nella mano libera
- _Ipotesi fatte:_ La finestra non potrà mai essere presa in mano.

Lasciare un oggetto: _letgo_
- _Condizioni:_ avere un oggetto in mano
- _Azioni:_ l'oggetto viene tolto dalla mano (che diventa libera) e posizionato nel mondo
- _Ipotesi fatte:_ l'azione letgo può essere compiuta solo con gli oggetti componibili. Le motivazioni di tale scelta sono riportate sopra.

Usare la fionda: _utilize*fionda_
- _Condizioni:_ avere una fionda in una mano e delle pietre nell'altra
- _Azioni:_ si prova a rompere la finestra, si tolgono le pietre dalla mano per riposizionarle nel mondo
- _Ipotesi fatte:_ Quando si utilizza la fionda c'è circa un 20% di probabilità che la finestra si rompa. Questa probabilità viene gestita tramite generazione di un numero pseudo-casuale tra 1 e 5, che se risulta uguale a 1 causa la rottura della finestra.

Usare un tronco: _utilize*tronco_
- _Condizioni:_ avere un tronco in una mano ed essere vicino alla finestra
- _Azioni:_ si posiziona il tronco vicino alla finestra, eventualmente sopra un altro tronco nel caso in cui questo sia presente
- _Ipotesi fatte:_ I tronchi hanno un attributo numerico _^height_ che specifica, appunto, quanto sono alti. L'applicazione di _utilize*tronco_ si limita, molto semplicemente, a sommare i valori di _^height_ del tronco attualmente in possesso dell'agente con l'attributo _^height_ dell'ambiente.

Comporre due oggetti: _combine_
- _Condizioni:_ avere due oggetti combinabili in mano
- _Azioni:_ eliminare dal mondo i due oggetti semplici togliendoli dalle mani e posizionare in una mano quello composto
- _Ipotesi fatte:_ Le combinazioni possibili sono solamente tre: fionda (elastico-rametto), pietre-rametto, pietre-elastico. Le informazioni su queste combinazioni non sono possedute direttamente dall'agente, ma sono insite nell'ambiente. Lo stato S, infatti, possiede un attributo _^combinations_ che tiene traccia delle combinazioni possibili. Queste sono considerate come oggetti ma hanno due attributi _^ingredient_ che specificano quali oggetti combinabili servano per costruirle. 


Fuggire: _escape_
  - _Condizioni:_ la finestra è rotta, e l'agente può raggiungerla, cioè ha impilato i due tronchi uno sopra l'altro.
  - _Azioni:_ fuga dalla stanza
  - _Ipotesi fatte:_ la rottura della finestra è spiegata nelle ipotesi dell'azione _utilize*fionda_, mentre l'impilamento dei tronchi nell'azione _utilize*tronchi_. Dopo che l'agente è fuggito, l'episodio termina.


## Ricompense

Le ricompense vengono fornite all'agente dopo le seguenti azioni:
- movimento: -0.5
- lasciare un oggetto: -0.5
- combinazione di due oggetti: 1 se fionda, -1 altrimenti
- posizionamento di un tronco: 1
- colpo di fionda sulla finestra: 1
- fuga: 2
- fallimento: -2
