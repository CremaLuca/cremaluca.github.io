---
layout: post
title:  "Go OTel traces!"
tags: [golang, opentelemetry, grafana-tempo]
lang: it
excerpt_en: "My experience on how we introduced OpenTelemetry to trace requests in Golang services, the difficulties we encountered and what we learned."
excerpt_it: "La mia esperienza su come abbiamo introdotto OpenTelemetry per tracciare le richieste nei servizi scritti in Golang, le difficoltà incontrate e cosa abbiamo imparato."
---

# Go OTel traces!

Il framework [Open Telemetry](https://opentelemetry.io/docs/) definisce come **misurare**, **collezionare** e **trasportare** tre tipi di informazione:

* tracce
* metrice
* logs

Nello specifico in questo articolo si parlerà di come utilizziamo Open Telemetry per osservare le **tracce** generate dai servizi di back-end con cui interagiscono gli utenti in modo da:

* velocizzare la ricostruzione del quadro generale in caso di segnalazioni
* anticipare i problemi segnalando servizi particolarmente lenti o con più alte percentuali di errori del solito
* tenere un grafo delle dipendenze ed il tipo di messaggi scambiati per valutare gli impatti delle modifiche alle API interne

L'applicazione che utilizzeremo come esempio è composta da due servizi: l'`order-manager` riceve le richieste di ordine dagli utenti e, dopo averle validate, le inoltra al `db-service` che salva gli ordini nel database.


In questo articolo non si parlerà dei concetti di Open Telemetry, se ancora non lo conosci e sei interessato a saperne di più ti consiglio di andare a leggere i capitoli introduttivi sulla [documentazione ufficiale](https://opentelemetry.io/docs/), compresa la documentazione dell'SDK relativa al tuo linguaggio di programmazione preferito per avere un idea più chiara di come interagiscono i componenti.

## Lo stack tecnologico

I servizi sono scritti in go 1.24 e comunicano tra di loro utilizzando [NATS](https://github.com/nats-io). Le tracce sono raccolte utilizzando la libreria [go.opentelemetry.io/otel](https://pkg.go.dev/go.opentelemetry.io/otel) ed inviate direttamente a [Grafana Tempo](https://grafana.com/oss/tempo/) utilizzando l'exporter OTLP gRPC.

## OTel in Golang

La strumentazione/SDK di Open Telemetry in golang fornisce:

* **exporter**: il componente che si occupa di serializzare ed inviare le tracce al backend (Tempo), nel nostro caso è OTLP gRPC fornito da Open Telemetry. È wrappato con un **batcher** per effettuare l'invio ogni X secondi oppure ogni Y bytes raccolti.
* **provider**: contiene la configurazione dell'exporter e delle risorse, è la factory per i tracer e viene settato globalmente come default.
* **tracer**: produce le spans, è specifico per-modulo (ogni libreria utilizza un tracer diverso) ma ottenuto dallo stesso provider di default.

Nelle **resources** oltre al `service.name` includiamo:

*  `service.version` (passato con `go build -ldflags '-X main.Version=1.0.0'` a partire dal tag git della release)
* `host.name`, `host.id`, `process.pid`
* `deployment.environment` letto da `$ENV` (dev/test/prod)

Per la **propagazione** del contesto, e quindi del "parent trace id", utilizziamo il propagatore `TraceContext{}` di default. Per "iniettare" ed estrarre i campi propagati abbiamo dovuto scrivere un `NatsHeaderContext{}` poiché sebbene `nats.Header` sia un `map[string][]string` esattamente come `http.Header`, il `HeaderContext{}` fornito da open telemetry gestisce esclusivamente gli header HTTP.
Nella propagazione è possibile includere anche il **baggage**, ma non ho ancora ben capito la differenza che ha con gli attributi quindi se ne sai qualcosa di più scrivimi pure e *propaga* la tua conoscenza anche a me.

### Noop e unit tests

Ci sono dei casi in cui si vuole "spegnere" la raccolta delle tracce, magari da configurazione oppure durante l'esecuzione degli unit tests, in questi casi è sufficiente non chiamare `SetTracerProvider` siccome il default tracer provider è settato a `NoopTracerProvider{}`, e `trace.Tracer("")` restituirà un `NoopTracer{}`.

### Setup

In una libreria interna ho aggiunto le seguenti funzioni, in modo che i singoli servizi abbiano minore codice da scrivere e uniformità di configurazione.

```go
// NewExporter creates a new OTLP trace exporter with the given endpoint.
func NewExporter(ctx context.Context, endpoint string, opts ...otlptracegrpc.Option) (*otlptrace.Exporter, error) {
	var newOpts = []otlptracegrpc.Option{
		otlptracegrpc.WithEndpoint(endpoint),
		otlptracegrpc.WithInsecure(),
		otlptracegrpc.WithTimeout(30 * time.Second), // Default 10s
	}
	newOpts = append(newOpts, opts...)

	return otlptracegrpc.New(ctx, newOpts...)
}

// WithBatcher configures the tracer provider to use a batch span processor.
func WithBatcher(exporter *otlptrace.Exporter, batchTimeout time.Duration, opts ...trace.BatchSpanProcessorOption) trace.TracerProviderOption {
	var newOpts = []trace.BatchSpanProcessorOption{
		trace.WithBatchTimeout(batchTimeout),
	}
	newOpts = append(newOpts, opts...)

	return trace.WithBatcher(exporter, newOpts...)
}

// NewResource creates a new resource describing the service.
func NewResource(ctx context.Context, serviceName string, serviceVersion string, opts ...resource.Option) (*resource.Resource, error) {
	env, _ := os.LookupEnv("ENV")

	var newOpts = []resource.Option{
		resource.WithHost(),
		resource.WithHostID(),
		resource.WithProcessPID(),
		resource.WithAttributes(
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion(serviceVersion),
			semconv.DeploymentEnvironmentName(env),
		),
	}
	newOpts = append(newOpts, opts...)

	return resource.New(ctx, newOpts...)
}
```

Per lasciare comunque ai servizi la possibilità di passare configurazioni specifiche le funzioni hanno il parametro `opts…` che venendo aggiunte alla fine dell'elenco di opzioni possono eventualmente sovrascrivere le opzioni di default precedenti.

### main

I singoli servizi dovranno solamente chiamare le tre funzioni della libreria e gestirne gli errori.

```go
var tracer trace.Tracer

func main() {
    globalCtx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
	defer cancel()

    shutdownOtel, err := initTracer(globalCtx)
    if err != nil {
        panic(err)
    }
	defer func() {
        // Create a context with short timeout
		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()

		if err := tpShutdown(ctx); err != nil {
			panic(err)
		}
	}()
 
    tracer = otel.Tracer("main")
 
    <-globaCtx.Ctx()
    
}

// initTracer sets up OpenTelemetry tracing with an OTLP exporter.
// It returns a shutdown function to be called with defer.
func initTracer(ctx context.Context) (func(ctx context.Context) error, error) {
	// Re-define context with timeout for exporter and resource creation
	ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()

	// Create OTLP gRPC exporter
	exporter, err := NewExporter(ctx, "localhost:4317")
	if err != nil {
		return nil, fmt.Errorf("failed to create OTLP exporter: %w", err)
	}

	// Create resource
	res, err := NewResource(ctx, "db-service", Version)
	if err != nil {
		return nil, fmt.Errorf("failed to create resource: %w", err)
	}

	// Create tracer provider
	tp := trace.NewTracerProvider(
		WithBatcher(exporter, 5*time.Second),
		trace.WithResource(res),
	)
	otel.SetTracerProvider(tp)

	// Set up W3C Trace Context propagator for distributed tracing
	otel.SetTextMapPropagator(propagation.TraceContext{})
    
	return tp.Shutdown, nil
}
```

Fare attenzione al `ctx context.Context` passato alla funzione di `Shutdown` del `TracerProvider`, è importante che non sia un contesto già cancellato altrimenti non viene fatto nessun flush delle span in sospeso e potrebbero essere perse. Inoltre **non** è necessario gestire lo `Shutdown` dell'exporter, se ne occuperà il `TracerProvider`.

### NatsHeaderCarrier

Per poter iniettare ed estrarre il `TraceContext` all'interno di un `nats.Header` prima dell'invio è necessario implementare `TextMapCarrier`, l'unica funzione mancante è `Keys()`, che può essere scritta come segue.

```go
type NatsHeaderCarrier struct {
	nats.Header
}

func (c NatsHeaderCarrier) Keys() []string {
	keys := make([]string, 0, len(c.Header))
	for key := range c.Header {
		keys = append(keys, key)
	}
	return keys
}
```

[Si potrebbe ottimizzare](https://stackoverflow.com/questions/21362950/getting-a-slice-of-keys-from-a-map):

* inizializzando `keys` a `len(c.Header)` per poi assegnare gli elementi con `keys[i] = key` (ed un `i += 1`)
* utilizzando `slices.Collect(maps.Keys(c.Header))`

Ma a me piace di più così perché è più chiaro, e voi non ci potete fare niente.


Per inviare un messaggio tramite Nats ora è necessario costruire un `nats.Msg` esplicitamente.

```go
// Serialize request data
dbReqData, _ := json.Marshal(dbReq)

// Create message with trace context headers
msg := nats.NewMsg("DB.QUERY")
msg.Data = dbReqData

// Inject trace context into message headers
otel.GetTextMapPropagator().Inject(ctx, NatsHeaderCarrier{Header: msg.Header})
```

Il `msg.Header` ora dovrebbe contenere un campo `traceparent` che può essere estratto dal ricevitore del messaggio per creare spans aventi lo stesso `trace_id` del servizio che ha pubblicato il messaggio.

```go
func handleQueryRequest(msg *nats.Msg) {
	// Extract trace context from message headers
	ctx := otel.GetTextMapPropagator().Extract(context.Background(), NatsHeaderCarrier{Header: msg.Header})

	// Start span as child of the extracted context
	ctx, span := tracer.Start(ctx, "handle query")
	defer span.End()
 
    //...
}
```

Ora lo span "*handle query*" del servizio `db-service` utilizzerà lo stesso `trace_id` dell' `order-manager`.

### Attributi e Semconv

Di per se gli span non sarebbero propriamente utili a debuggare, sicuramente vanno associati ad un buon lavoro di logging ma la loro grande utilità deriva dalla possibilità di associarci degli attributi arbitrari, come ad esempio l'id dell'ordine che sta venendo processato, l'utente o il subject da cui è stato letto.

Gli attributi possono essere associati allo span alla creazione oppure durante lo svolgimento dello span, ad esempio l'id dell'ordine potrebbe essere associato allo span solo una volta che il messaggio viene parsato e validato:

```go
func (oh *OrderHandler) handleNewOrderRequest(msg *nats.Msg) {
    ctx, span := tracer.Start(
		context.Background(),
		"request_new",
		otrace.WithAttributes(
			attribute.String("messaging.system", "nats"),
			attribute.String("messaging.destination", msg.Subject),
		),
	)
	defer span.End()
    
    // De-serialize order
	var ord Order
	if err := json.Unmarshal(msg.Data, &ord); err != nil {
		span.RecordError(err)
		return
	}
    
    // Set attributes from parsed order
	span.SetAttributes(
		attribute.String("order.id", req.ID),
		attribute.String("user.id", req.User),
	)
 
    // ...
 }
```

NOTA: al momento non ho ancora avuto modo di utilizzare il campionamento degli span, ma mi sembra che il campionamento condizionato sugli attributi è possibile solo per gli attributi settati al momento della creazione dello span.


Può essere è difficile uniformare i nomi degli attributi tra i servizi, ad esempio `messaging.destination` potrebbe benissimo diventare `message.destination` o `messaging.subject` nel prossimo progetto. Fortunatamente open telemetry definisce i nomi degli attributi più comunemente utilizzati nello standard delle "semantic conventions" **semconv**. La libreria di convenzioni è molto ampia e richiede tempo per essere assorbita.

In generale esiste una chiave per la maggior parte dei dettagli che potrebbe aver senso registrare nelle operazioni comuni, forse però ne esistono anche troppe: la flessibilità che viene proposta quasi ne invalida l'utilità effettiva. Per questo consiglio di definire un sottoinsieme di chiavi più utili ed essere consistente nell'utilizzare quelle, oltre a quelle richieste dagli strumenti di analisi (vedi capitolo successivo).

Inoltre torna sempre utile definire alcune chiavi nel dominio dei propri servizi che possano aiutare nella ricerca degli spans relativi ad un entità su cui è probabile si voglia filtrare, come ad esempio l'id dell'utente, l'id dell'ordine, l'id del prodotto, il frontend utilizzato. Per definire i propri attributi nello stesso formato messo a disposizione da semconv è necessario:


1. Una stringa costante che definisce la chiave
2. Una variabile `attribute.KeyValue` per velocizzare la creazione dell'attributo
3. Una funzione per forzare il type del valore dell'attributo, o per definirne la conversione a stringa (nel caso non sia scontato, come un timestamp)

```go
import (
	"go.opentelemetry.io/otel/attribute"
)

const (
	UserIdAttributeKey  = "user.id"
	OrderIdAttributeKey = "order.id"
)

var (
	UserIdAttributeKeyAttr  = attribute.Key(UserIdAttributeKey)
	OrderIdAttributeKeyAttr = attribute.Key(OrderIdAttributeKey)
)

func UserIdAttribute(userId string) attribute.KeyValue {
	return UserIdAttributeKeyAttr.String(userId)
}

func OrderIdAttribute(orderId string) attribute.KeyValue {
	return OrderIdAttributeKeyAttr.String(orderId)
}
```

### Client/server e consumer/producer spans

Gli span che si occupano di comunicazione con l'esterno agiscono sincronamente o asincronamente. Nel primo caso lo span che fa la richiesta deve essere inizializzato con `otrace.WithSpanKind(otrace.SpanKindClient)` e lo span che la processa `otrace.SpanKindServer`, nel secondo caso gli span saranno di tipo `otrace.SpanKindProducer` e `otrace.SpanKindConsumer`.

La tipizzazione dello span non serve tanto al debug dove è (quasi) sempre chiaro chi fa la richiesta e chi risponde, serve a Grafana Tempo per costruire il grafo delle dipendenze dei servizi ed ad automatizzare la creazione delle dashboard sullo stato di salute del sistema: è importante sapere se il delay tra lo span di richiesta e lo span di risposta è importante (client/server) oppure è solo indicativo (producer/consumer).

Per maggiori informazioni consiglio una lettura approfondita della relativa [documentazione ufficiale](https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/) che contiene degli ulteriori esempi.

### Monitorare le librerie

Ogni libreria dovrebbe utilizzare un `tracer` con un nome diverso, così da capire più facilmente da quale codice proviene. 

I metodi delle librerie che si interfacciano con i sistemi esterni devono accettare un `ctx context.Context` come primo parametro, e cominciare a loro volta un nuovo span:

```go
// Performs a request-reply with the db-service
func (db *DbConnection) ExecuteStoredProcedure(ctx context.Context, storedProcedure string, args ...any) error {
    ctx, span := tracer.Start(
        ctx,
        "execute stored procedure",
        otrace.WithAttributes(
            semconv.DBQueryText(storedProcedure),
            semconv.DBOperationName("EXEC"),
            semconv.MessagingOperationName("produce"),
    		semconv.MessagingOperationTypeSend,
        ),
        otrace.WithSpanKind(otrace.SpanKindClient),
    )
    
    // ...
    
	dbReqData, err := json.Marshal(dbReq)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return err
	}
 
    // Create message with trace context headers
	msg := nats.NewMsg("DB.QUERY")
	msg.Data = dbReqData

	// Inject trace context into message headers
	otel.GetTextMapPropagator().Inject(ctx, NatsHeaderCarrier{Header: msg.Header})
    
    // Perform request
	resp, err := nc.RequestMsg(msg, 5*time.Second)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return err
	}
 
    // ...
    // Process response
    // ...
    
    span.SetStatus(codes.Ok, "")
    return nil
}
```


Si potrebbe utilizzare anche `debug.ReadBuildInfo()` per sapere la versione del modulo ed utilizzarlo come attributo; magari in futuro.

### Gli eventi

Non ho trovato un buon esempio di evento che valga la pena essere registrato, tutte le situazioni che vengono solitamente descritte riguardano eventi "bloccanti" e che quindi a cui si può risalire guardando al resto dello span:

* validazione: non hanno senso perché la span che contiene l'evento dovrebbe essere in stato di errore oppure avere un attributo `result: validation_failed`, ed il processamento della richiesta sarebbe interrotto senza le interazioni solite con gli altri sistemi.
* cache hit: la lettura da cache è da tracciare come span poiché contiene un interazione con un altro sistema se la cache è esterna, altrimenti può essere segnata come attributo `cache.hit: true`.

Se avete dei buoni esempi che non riesco a trovare o delle motivazioni per cui nei due esempi sopra è meglio utilizzare un evento fatemelo sapere.


UPDATE: la libreria otelhttptrace contiene un `Transport` per il `http.Client` e contiene un buon esempio di evento: alla ricezione del primo byte dal server segna un evento col timestamp, in questo modo è possibile capire quanto è durata la ricezione della risposta.

### Errori o non errori?

Riprendendo l'esempio degli attributi: nel caso il messaggio non fosse parsabile come JSON lo span non viene segnato come in stato di errore con `span.SetStatus(codes.Error, err)` ma viene registrato solamente l'error.

```go
var ord Order
if err := json.Unmarshal(msg.Data, &ord); err != nil {
	span.RecordError(err)
	return
}
```

La funzione `span.RecordError(err)` registra solamente il messaggio di errore in un campo standard dello span, ma non lo segnala come in stato di errore.

Non tutto quello che produce `err` è un errore, se l'errore è atteso o gestito non è necessario segnalare la span come anomala. Come regola interna cerchiamo di segnalare gli span in stato di errore solo se la comparsa di tale span dovesse generare una mail di massima urgenza (in funzione del servizio monitorato).

### Considerazioni

Leggere il codice sorgente Go di Open Telemetry ci ha affinato la conoscenza del linguaggio, aiutandoci a:

* gestire il lifecycle delle richieste introducendo il `ctx context.Context` in tutte le librerie interne (API di altri servizi), dando la possibilità di gestire i timeout e quindi unit-test più deterministici
* introdurre l'utilizzo dei parametri variabili di tipo `func WithX(...) Option` per poter controllare anche le configurazioni che erano hard-coded nelle nostre librerie
* utilizzare un contesto per catturare il segnale di SIGINT/SIGTERM e gestire il graceful shutdown delle connessioni, come consigliato dalla documentazione

## Per il futuro

### Cardinalità degli attributi

Mi piacerebbe capire se in Grafana Tempo troppi attributi contribuiscono linearmente allo spazio occupato dagli span (ma tutti sappiamo che nel 2026 la memoria su disco è gratis) oppure vengono anche indicizzati quindi vanno aggiunti con parsimonia ed utilizzati i log per reperire ulteriori informazioni.

## Vedi anche

* [OpenTelemetry: A Guide to Observability with Go - Luca Cavallin - 06/02/2025](https://www.lucavall.in/blog/opentelemetry-a-guide-to-observability-with-go)
