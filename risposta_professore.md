# Bozza risposta al professore

**Oggetto:** Re: Feedback sul progetto Madrid Air Quality — risposte ai Task 3, 8 e 9

---

Gentile Professore,

la ringraziamo per il feedback dettagliato sui diversi task — sia per il Task 7 che per le osservazioni sui Task 3, 8 e 9. Di seguito proviamo a rispondere punto per punto, e accenniamo alle modifiche che abbiamo già riportato nel notebook sul dataset completo.

---

## Task 3 — Imputation

**1) Quale modello usa l'`IterativeImputer`?**

L'`IterativeImputer` di `sklearn` usa come stimatore di default `BayesianRidge`, cioè una regressione lineare bayesiana con regolarizzazione ridge (10 iterazioni di default). Nel nostro codice non abbiamo sovrascritto l'argomento `estimator`, quindi l'imputazione regressiva che riportiamo è interamente *lineare*.

Condividiamo la sua osservazione: alcune relazioni nel dataset (tipicamente NO2 vs. traffico e NO2 vs. umidità) non sono lineari e un `RandomForestRegressor` o un `GradientBoostingRegressor` come `estimator` potrebbero in linea di principio catturarle meglio. Abbiamo valutato il trade-off e scelto di rimanere sul default lineare per tre motivi:

- i segnali principali nel blocco degli inquinanti (NO2 ↔ NO ↔ NOX) sono molto vicini a essere lineari;
- un estimator non lineare farebbe esplodere il runtime di uno o due ordini di grandezza sui 64M di righe;
- sul campione, le metriche MAD e KS dell'imputazione regressiva con `BayesianRidge` sono già competitive rispetto agli altri metodi, quindi il guadagno atteso da un modello non lineare non ci sembra sufficiente a giustificarne il costo.

Abbiamo aggiunto questa spiegazione esplicitamente nella sezione 3.5 del notebook sul dataset completo.

**2) Cosa stiamo confrontando nei KDE (sezione 3.7)?**

Qui ci rendiamo conto che il grafico poteva essere ambiguo, e abbiamo riscritto la spiegazione nel notebook. Il confronto KDE **non** è tra "tutti i dati osservati della variabile (MetraQ)" da una parte e "tutti i dati dopo imputazione" dall'altra. Entrambe le curve (la METRAQ di riferimento e quella di ciascun metodo) sono calcolate sullo **stesso sottoinsieme di righe**: quelle con `is_interpolated == True`, cioè le posizioni che erano originariamente mancanti nel dato grezzo e che METRAQ ha riempito con la propria interpolazione spaziale.

Concretamente, per ogni metodo *m* (mean / median / KNN / regression) prendiamo il valore imputato da *m* in quelle posizioni e ne stimiamo la KDE; per METRAQ prendiamo il suo valore imputato nelle *stesse* posizioni e ne stimiamo la KDE. Le curve sono quindi confrontabili posizione per posizione.

L'intuizione rimane: se la distribuzione dei valori che il nostro metodo produce in quelle posizioni è simile alla distribuzione di METRAQ, il metodo sta riempiendo i gap in modo coerente con il riferimento.

**3) Stessa domanda per la sezione 3.8 (MAD e KS)**

Identica risposta: anche MAD e KS sono calcolati solo sulle righe originariamente mancanti, non sull'intera distribuzione osservata. Il MAD è la media, posizione per posizione, di `|valore_imputato_nostro - valore_METRAQ|` su quelle righe. Il KS compara le due distribuzioni empiriche su quello stesso sottoinsieme.

Abbiamo riscritto i testi sia in 3.7 che in 3.8 perché fosse esplicito quale insieme di punti entra nel confronto.

**4) Idea di visualizzazione su una singola variabile**

Abbiamo seguito il suo suggerimento: nel notebook sul dataset completo abbiamo aggiunto una nuova sezione **3.9 — Time-Series Illustration**. Scegliamo il sensore con più ore osservate di NO2, isoliamo una finestra di 30 giorni interamente non-interpolata, introduciamo artificialmente tre gap di 6 ore, e sovrapponiamo sullo stesso grafico la ricostruzione di Mean, Median, KNN-proxy (ffill/bfill) e Regression-proxy (time-interpolation). Le differenze tra i metodi diventano immediatamente visibili: Mean/Median appiattiscono il segnale su un valore costante, mentre KNN e regressione seguono l'andamento locale.

Sulla sua intuizione che sul dataset completo useremo probabilmente solo la mediana: sì, era l'idea di partenza per contenere il costo; ma dal confronto integrato sul campione la differenza di costo tra mediana e ffill/bfill è trascurabile, mentre ffill/bfill ricostruisce meglio la dinamica locale. Quindi nell'eseguire il notebook sul full dataset manteniamo tutti e quattro i metodi in parallelo (mean, median, KNN-proxy via ffill/bfill, regression-proxy via ffill/bfill + clip), e documentiamo MAD/KS per ciascuno.

---

## Task 7 — Propagation Modeling

Grazie per la validazione. Abbiamo aggiunto una nota conclusiva alla sezione 7 che (i) riporta la sua conferma che il modello è valido così come implementato e (ii) motiva la scelta di non esplorare ulteriori configurazioni (altri grafi, altre equazioni di diffusione, altri inquinanti) data la natura combinatoria del problema e il fatto che il task è opzionale. Come unica analisi di robustezza manteniamo la sensitivity al parametro α che era già presente.

---

## Task 8 — Parallelization

La sua spiegazione sul comportamento di `multiprocessing.Pool` con `fork` era corretta e ci ha aiutato molto. Abbiamo aggiunto due cose nel notebook:

1. **Sezione 8.2 — Memory Sanity Check**: usiamo `psutil` per misurare la RSS del processo padre prima / durante / dopo il passo parallelo, e la somma delle RSS dei processi figli. Sul campione confermiamo quello che lei prevedeva: con `fork()` i figli ereditano `df_par` via copy-on-write e, dato che il nostro worker non muta il DataFrame padre, la memoria totale non esplode linearmente col numero di core. Il test è ripetibile anche sul dataset completo per misurare l'andamento reale.

2. **Sezione 8.3 — Pre-partition by Year (Parquet)**: abbiamo implementato esattamente la strategia che avevamo proposto. Dal `df_par` viene scritto una volta un file Parquet per anno (`data/partitioned_by_year/year=<Y>.parquet`). Il worker parallelo ora riceve solo la chiave `(year, sensor)`, apre **solo** il file dell'anno pertinente con filtro push-down `sensor_name == <sensor>`, e restituisce la matrice di correlazione oraria. Usiamo `multiprocessing.get_context("spawn")` in questa variante, così ogni worker è un interprete fresco senza ereditare alcun DataFrame dal padre. Nel notebook confrontiamo poi il runtime delle due strategie (fork+COW vs. Parquet+spawn) e includiamo una tabella di trade-off.

La raccomandazione che ne traiamo, e che lasciamo scritta nel notebook, è: sul dataset completo useremo il percorso Parquet, perché il costo one-off della partizione è trascurabile rispetto al rischio di OOM se la RAM non basta a tenere `df_par` in ogni worker.

---

## Task 9 — Forecasting Model

Sull'errore che le avevamo accennato: sul campione il modello allena senza problemi perché il campione è concentrato su anni in cui i sensori di traffico sono già attivi. Sul dataset completo, però, il dataset copre anche anni antecedenti al ~2018, in cui le variabili di traffico (`TI_*`, `SP_*`, `OC_*`) sono interamente NaN per quei mesi. La nostra pipeline fa pivot mensile del pannello `(sensor × month) → features`, poi applica un `.dropna()` per avere righe complete. Nei mesi pre-2018 la combinazione è:

- target `NO2` presente,
- variabili weather presenti,
- variabili traffic tutte NaN.

Il risultato è che `.dropna()` rimuove ogni riga pre-2018 e, se il pivot include solo anni pre-2018 (o se per qualche motivo un filtro li seleziona), `model_df` diventa vuota. Il problema non è un'eccezione esplicita ma un **errore silenzioso**: se `model_df` è vuota, `cross_val_score` solleva un `ValueError: n_splits=5 cannot be greater than the number of samples`, oppure (peggio) un run parziale restituisce punteggi R² e MAE calcolati su una manciata di righe con varianza enorme senza che sia evidente che la feature matrix era quasi vuota. Abbiamo riprodotto il caso simulando un subset pre-2018 del campione: confermiamo il fallimento silenzioso.

Abbiamo quindi aggiunto un **guard esplicito** nel notebook subito dopo il `.dropna()`: se `model_df` è vuota viene sollevato un `RuntimeError` con un messaggio che spiega la causa più probabile (assenza di traffico pre-2018) e suggerisce come intervenire (restringere il range temporale, oppure escludere le feature di traffico se si vuole modellare anche anni precedenti). Aggiungiamo anche un log della coverage effettiva delle feature di traffico dopo il `.dropna()`, così se la pipeline parte con pochissime righe ce ne accorgiamo subito.

Se vuole possiamo prevedere una variante del modello che drop-a le feature di traffico e allena sul range completo usando solo meteorologia — ma concettualmente ci sembra un modello diverso, quindi per ora lo lasciamo come alternativa documentata nella discussione e non come default.

---

Grazie ancora per il tempo e il dettaglio del suo riscontro: i punti che ha sollevato ci hanno davvero aiutato a rendere la pipeline più robusta prima del run finale sul dataset completo. Rimaniamo disponibili per una call se volesse chiarire qualcuno di questi aspetti a voce.

Un cordiale saluto,
Anna, Giulia e Maria
