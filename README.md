# Sistema di Distribuzione Podcast Decentralizzato
**Distopia – Documento di Design e Architettura v1.0**

---

## 1. Introduzione e Obiettivi

### 1.1 Descrizione del Progetto

Distopia è una piattaforma di distribuzione podcast decentralizzata basata su .NET 9. L'obiettivo principale è eliminare la dipendenza da un unico punto centrale di archiviazione, permettendo ai creator di mantenere i propri file audio nel proprio computer locale, mentre una rete di server federati garantisce la distribuzione agli utenti finali.

### 1.2 Obiettivi Chiave

- **Decentralizzazione:** i file audio rimangono sui computer dei creator
- **Federazione:** i server si sincronizzano automaticamente tra di loro
- **Resilienza:** nessun single point of failure nell'infrastruttura
- **Semplicità d'uso:** il creator usa un'app desktop, l'utente naviga sul web
- **Cache intelligente:** i server memorizzano i file più richiesti per ridurre il carico sui creator

### 1.3 Principi Architetturali

> Il sistema segue il principio **"origin pulls on demand"**: i server scaricano i file dai creator solo quando un utente li richiede per la prima volta, poi li memorizzano in cache locale.

---

## 2. Architettura di Sistema

### 2.1 Componenti Principali

Il sistema è composto da **due soli progetti**, non tre: il Web Frontend è integrato nel Distribution Server come parte dello stesso processo ASP.NET Core. Un unico progetto espone sia le API REST (per client programmatici e federazione) che le pagine Blazor (per gli utenti finali). Questo semplifica il deployment, il certificato TLS e la gestione della cache.

| Componente | Tecnologia | Ruolo |
|---|---|---|
| Creator Agent | C# .NET 9 / Avalonia UI | App desktop sul PC del creator |
| Distribution Server + Web UI | C# .NET 9 / ASP.NET Core + Blazor | API REST, pagine web e cache — tutto in un unico processo |

### 2.2 Flusso di Richiesta

Quando un utente richiede un file audio, il server segue questa sequenza di risoluzione in ordine di priorità:

1. **Cache locale** — il server ha già il file in cache disco → lo serve direttamente
2. **Rete federata** — il server interroga i peer in ordine casuale (randomizzato ad ogni richiesta) per sapere chi ha già il file in cache → lo scarica dal primo che risponde positivamente. L'ordine casuale garantisce che il carico si distribuisca uniformemente tra i peer e che nessun server venga sempre interpellato per primo
3. **Home Server** — nessun server ce l'ha → il server contatta l'home server del creator (Server A)
4. **Creator Agent** — l'home server non ha il file in cache → lo recupera dal Creator Agent e lo passa al server richiedente

> Il Creator Agent viene contattato **solo dall'home server** e **solo come ultima risorsa**. Una volta che un file è stato scaricato una volta, la rete lo distribuisce autonomamente senza più coinvolgere il creator.

### 2.3 Diagramma Logico

```
[ Utente Web ]
      ↓
[ Server B ] ──1. cache locale? ──→ serve direttamente
             ──2. qualche peer ha il file? ──→ [ Server C / D / ... ] → serve da lì
             ──3. nessuno ce l'ha → chiede ad A (home server del creator)
                        ↓
             [ Server A (Home Server) ] ──4. ha in cache? ──→ passa a B
                        ↓ (se non in cache)
             [ Server A ] ──SignalR──→ "ho bisogno di EP123"
                        ↓
             [ Creator Agent ] ──HTTP upload streaming──→ [ Server A ]
                        ↓
             [ Server A mette in cache e serve il file ]
```

La connessione tra Creator Agent e Home Server avviene tramite **SignalR** — una connessione WebSocket persistente aperta dal Creator Agent all'avvio. Nessuna dipendenza da servizi esterni, nessuna porta da aprire sul router del creator.

---

## 3. Creator Agent (App Desktop)

### 3.1 Funzionalità

Il Creator Agent si collega a **un solo server**, chiamato "home server". È il server di fiducia del creator — l'unico autorizzato a contattare il suo computer. La distribuzione verso gli altri server della rete avviene tramite federazione, senza coinvolgere il creator.

Il Creator Agent gestisce una gerarchia a due livelli: **Podcast** (il contenitore) ed **Episodi** (i singoli file audio). Un creator può avere più podcast distinti, ognuno con la propria copertina, descrizione e lista di episodi.

```
Creator Agent
└── Podcast A  (es. "Tech Talk")
│   ├── Episodio 1
│   ├── Episodio 2
│   └── Episodio 3
└── Podcast B  (es. "Cucina con Mario")
    ├── Episodio 1
    └── Episodio 2
```

Ogni podcast corrisponde a una **cartella** nella directory configurata. Il Creator Agent la monitora automaticamente: se viene aggiunta una nuova cartella rileva un nuovo podcast, se viene aggiunto un file audio rileva un nuovo episodio.

```
C:/Podcasts/
├── TechTalk/
│   ├── podcast.json          ← metadati del canale (titolo, copertina, descrizione...)
│   ├── cover.jpg
│   ├── ep01-introduzione.mp3
│   └── ep02-secondo-episodio.mp3
└── CucinaConMario/
    ├── podcast.json
    ├── cover.jpg
    └── ep01-pasta.mp3
```

**Altre funzionalità:**

- Esposizione di un mini-server HTTP locale (Kestrel embedded)
- Connessione a **un solo home server** (configurato dal creator)
- Streaming dei file audio con supporto **HTTP Range Requests**
- Registrazione automatica all'home server all'avvio
- Hot-reload: nuovi podcast ed episodi rilevati automaticamente (`FileSystemWatcher`)
- Dashboard UI per monitorare download attivi, statistiche per podcast e per episodio

### 3.2 Struttura del Progetto

```
Distopia.CreatorAgent/
├── Program.cs                   // Host .NET 9 + Kestrel
├── appsettings.json
├── Services/
│   ├── FileStreamingService.cs  // Lettura e streaming file audio
│   ├── HomeServerService.cs     // Connessione e registrazione all'home server
│   ├── PodcastWatcherService.cs // FileSystemWatcher — rileva nuovi podcast ed episodi
│   └── FeedBuilderService.cs    // Genera feed JSON per ogni podcast
├── Controllers/
│   ├── PodcastsController.cs    // GET /podcasts — lista tutti i podcast del creator
│   ├── EpisodesController.cs    // GET /podcasts/{id}/episodes/{epId}/stream
│   └── FeedController.cs        // GET /podcasts/{id}/feed
├── Models/
│   ├── Podcast.cs
│   └── Episode.cs
└── UI/
    └── MainWindow.axaml         // Avalonia UI (cross-platform)
```

### 3.3 Endpoint Esposti

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/podcasts` | Lista di tutti i podcast del creator |
| GET | `/podcasts/{id}/feed` | Metadati del podcast + lista episodi |
| GET | `/podcasts/{id}/episodes` | Lista episodi di un podcast |
| GET | `/podcasts/{id}/episodes/{epId}/stream` | Streaming file audio (Range supportato) |
| GET | `/health` | Health check per l'home server |

### 3.4 Sicurezza

> ⚠️ Il Creator Agent **NON deve essere esposto su internet**. Deve essere raggiungibile solo dall'home server tramite tunnel (Cloudflare Tunnel, Tailscale) o whitelist IP.

- Il Creator Agent accetta richieste **solo dall'home server** tramite un Bearer Token univoco
- Se arriva una richiesta da un IP o token non riconosciuto, viene rifiutata immediatamente
- Il creator genera il token dalla UI e lo copia manualmente nella configurazione dell'home server
- Gli altri server della rete federata non contattano mai direttamente il Creator Agent — passano sempre per l'home server

---

## 4. Distribution Server + Web UI

Il Distribution Server e il Web Frontend vivono nello stesso progetto ASP.NET Core 9. Le pagine Blazor condividono direttamente i service e la cache del server — nessuna chiamata API interna, nessun layer aggiuntivo.

### 4.1 Struttura del Progetto

```
Distopia.Server/
├── Program.cs
├── appsettings.json
├── Controllers/                  // API REST (JSON) per client esterni e federazione
│   ├── PodcastsController.cs
│   ├── FederationController.cs
│   └── AdminController.cs
├── Components/                   // Blazor Server — pagine per utenti web
│   ├── App.razor
│   ├── Pages/
│   │   ├── Home.razor            // Homepage e ricerca podcast
│   │   ├── Creator.razor         // Pagina del creator con lista episodi
│   │   └── Player.razor          // Player audio con streaming
│   └── Layout/
│       └── MainLayout.razor
├── Services/
│   ├── CacheService.cs           // Cache locale (disco + memoria)
│   ├── CreatorProxyService.cs    // Proxy verso Creator Agent
│   ├── FederationService.cs      // Sync con altri Distribution Server
│   └── MetadataService.cs        // DB metadati (SQLite / PostgreSQL)
├── BackgroundServices/
│   ├── FeedSyncJob.cs            // Sincronizza feed dai creator
│   ├── PeerSyncJob.cs            // Sincronizza metadati tra server
│   └── CacheEvictionJob.cs       // Elimina file con LastAccessedAt scaduto
├── Models/
│   ├── CachedEpisode.cs
│   └── FederatedServer.cs
└── wwwroot/                      // Asset statici (CSS, JS player audio)
```

### 4.2 Cache Strategy — 3 Livelli

| Livello | Tecnologia | TTL | Contenuto |
|---|---|---|---|
| L1 – Memoria | IMemoryCache (.NET) | 1 ora | Metadati, feed JSON |
| L2 – Disco | File System locale | Sliding TTL (configurabile, default 30 giorni) | File audio .mp3/.m4a |
| L3 – Origin | Creator Agent HTTP | N/A | File sorgente sul PC del creator |

### 4.3 Cache Sliding TTL

I file audio in cache disco usano un **TTL scorrevole (sliding)**: ogni volta che un utente richiede un file, il contatore viene azzerato e riparte da zero. Un file viene eliminato automaticamente solo se non viene richiesto per un periodo pari al TTL configurato.

Questo significa che:
- File molto popolari rimangono in cache indefinitamente finché continuano ad essere ascoltati
- File che non vengono più richiesti vengono eliminati automaticamente dopo il periodo configurato
- Non è necessario nessun intervento manuale per la gestione della cache

Il TTL è configurabile in giorni a livello di server. Il valore di default è 30 giorni.

```
Esempio con TTL = 7 giorni:

Giorno 0  → file scaricato, TTL = 7 giorni
Giorno 3  → utente ascolta il file → TTL azzerato, riparte da 7 giorni
Giorno 8  → utente ascolta il file → TTL azzerato, riparte da 7 giorni
Giorno 16 → nessuna richiesta → file eliminato dalla cache
```

Il processo di pulizia gira come **Background Service** una volta al giorno, controlla la colonna `LastAccessedAt` di ogni file e rimuove quelli scaduti.

Il modello dati della cache viene aggiornato di conseguenza:

```csharp
public class CachedEpisode {
    public string EpisodeId { get; set; }
    public string CreatorId { get; set; }
    public string Title { get; set; }
    public string HomeServerUrl { get; set; }
    public bool IsLocallyCached { get; set; }
    public DateTime CachedAt { get; set; }
    public DateTime LastAccessedAt { get; set; }  // aggiornato ad ogni richiesta
    public long FileSizeBytes { get; set; }
}
```

### 4.3 Logica di Risoluzione del File

```csharp
// CacheService.cs
public async Task<Stream> ResolveEpisodeAsync(string episodeId)
{
    // 1. Cache locale
    if (await _diskCache.ExistsAsync(episodeId))
        return await _diskCache.ReadAsync(episodeId);

    // 2. Cerca nella rete federata in ordine casuale
    var peers = await _federationService.GetPeersAsync();
    var shuffledPeers = peers.OrderBy(_ => Guid.NewGuid()); // ordine random ad ogni richiesta
    foreach (var peer in shuffledPeers)
    {
        if (!await _federationService.PeerHasFileAsync(peer, episodeId)) continue;
        var stream = await _federationService.FetchFromPeerAsync(peer, episodeId);
        await _diskCache.WriteAsync(episodeId, stream);
        return stream;
    }

    // 3. Nessun peer ce l'ha → chiede all'home server del creator
    var homeServer = await _metadataService.GetHomeServerAsync(episodeId);
    var homeStream = await _federationService.FetchFromPeerAsync(homeServer, episodeId);
    // L'home server a sua volta contatterà il Creator Agent se necessario (step 4)
    await _diskCache.WriteAsync(episodeId, homeStream);
    return homeStream;
}
```

### 4.4 API Pubblica

| Metodo | Path | Descrizione |
|---|---|---|
| GET | `/api/podcasts` | Lista tutti i podcast |
| GET | `/api/podcasts/{creatorId}` | Feed di un creator specifico |
| GET | `/api/episodes/{id}` | Metadati di un episodio |
| GET | `/api/episodes/{id}/stream` | Stream audio |
| GET | `/api/search?q={query}` | Ricerca full-text |
| POST | `/federation/announce` | Un server si annuncia ai peer |
| GET | `/federation/feed/{creatorId}` | Feed condiviso tra server |
| GET | `/federation/has/{episodeId}` | Indica se il file è disponibile in cache locale |

---

## 5. Federazione tra Server

### 5.1 Protocollo

- Ogni server mantiene una lista di **server peer** nel DB
- Ogni 5 minuti (configurabile) ogni server chiede ai peer i metadati aggiornati
- Vengono sincronizzati solo i **metadati** (titolo, durata, `HomeServerUrl`), non i file audio
- I file audio vengono scaricati in cache solo quando un utente li richiede
- Ogni episodio nei metadati include `HomeServerUrl` — indica qual è l'home server del creator, usato come ultima risorsa se nessun peer ha il file in cache
- I server espongono un endpoint `/federation/has/{episodeId}` che risponde se hanno il file in cache, permettendo agli altri di trovare rapidamente la fonte più vicina

### 5.2 Onboarding di un Nuovo Server

**Prerequisito unico:** chi installa un nuovo server deve conoscere l'URL di almeno un server già nella federazione. Può essere un server "ufficiale" pubblicato nella documentazione del progetto, oppure uno qualsiasi che l'amministratore conosce. È lo stesso principio dei seed node di BitTorrent o Bitcoin.

Questo URL va inserito nell'`appsettings.json` prima del primo avvio:

```json
"FederationBootstrap": "https://server-noto.example.com"
```

**Al primo avvio, in automatico:**

**Fase 1 — Annuncio al bootstrap**
Il nuovo server contatta il server noto con `POST /federation/announce`, presentando il proprio URL pubblico e la propria firma. Il server di bootstrap verifica la firma, aggiunge il nuovo server alla propria lista peer e risponde con la lista completa di tutti i peer che conosce.

**Fase 2 — Propagazione a cascata (gossip)**
Il nuovo server contatta uno per uno tutti i peer ricevuti, ripetendo l'announce. Ogni server che riceve l'annuncio aggiunge il nuovo peer alla propria lista e risponde con i propri peer. Il processo si propaga a cascata finché tutta la rete conosce il nuovo server. In una rete di dimensioni normali questo richiede pochi secondi.

**Fase 3 — Download dei metadati**
Una volta dentro la rete, il nuovo server chiede a tutti i peer i metadati di ogni creator che conoscono (`GET /federation/feed/{creatorId}`). Popola il proprio DB con titoli, durate e `HomeServerUrl` di tutti gli episodi disponibili. Nessun file audio viene scaricato in questa fase.

Da questo momento il server è **pienamente operativo**: risponde alle ricerche degli utenti, partecipa alla distribuzione dei file e fa da bootstrap per eventuali futuri nuovi server.

> I file audio non vengono scaricati durante l'onboarding. La cache si costruisce progressivamente man mano che gli utenti richiedono i file, seguendo la logica di risoluzione descritta nella sezione 2.2.

**Rete aperta o permissioned?**

Per default Distopia usa una **rete aperta**: chiunque può unirsi annunciandosi a un server esistente. Se si vuole controllare chi entra nella rete, è sufficiente abilitare la modalità permissioned nell'`appsettings.json`:

```json
"FederationMode": "open"        // chiunque può unirsi
"FederationMode": "permissioned" // serve un token di invito
```

In modalità permissioned, l'announce deve includere un token generato dall'amministratore del server di bootstrap. Senza token valido, la richiesta viene rifiutata.

### 5.3 Eliminazione di un Episodio o Podcast

L'eliminazione segue lo stesso meccanismo dell'aggiornamento — propagazione tramite il ciclo di sync periodico dei 5 minuti.

**Eliminazione di un episodio:**

```
1. Creator elimina il file audio dal disco
2. Creator Agent rileva la rimozione tramite FileSystemWatcher
   e marca l'episodio come eliminato nel DB locale (DeletedAt != null)
3. Al ciclo di sync successivo l'home server rileva l'episodio
   marcato come eliminato e rimuove il file dalla propria cache
4. Al ciclo di sync successivo ogni peer riceve l'aggiornamento,
   rimuove il file dalla cache e rimuove l'episodio dai metadati
```

**Eliminazione di un podcast intero:**

Stesso flusso — il Creator Agent marca tutti gli episodi del podcast come eliminati e il podcast stesso come eliminato. La propagazione avviene nello stesso modo.

Il modello `Episode` include un campo di cancellazione soft per permettere la propagazione:

```csharp
public DateTime? DeletedAt { get; set; }  // null = attivo, valorizzato = eliminato
```

I server federati che ricevono un episodio con `DeletedAt` valorizzato eliminano il file dalla cache e rimuovono i metadati dal proprio DB. I peer temporaneamente offline si sincronizzeranno al primo sync dopo essere tornati online.

### 5.4 Aggiornamento e Invalidazione della Cache

Quando un creator modifica un episodio — che sia i metadati (titolo, descrizione, immagine) o il file audio — la propagazione avviene automaticamente attraverso il normale ciclo di sync periodico dei 5 minuti. Non esiste un meccanismo di notifica immediata.

**Modifica dei metadati:**

```
1. Creator modifica titolo/descrizione/immagine dal Creator Agent
2. Creator Agent aggiorna il proprio DB locale
3. Al ciclo di sync successivo (max 5 minuti) l'home server
   rileva la versione aggiornata e aggiorna il proprio DB
4. Al ciclo di sync successivo ogni peer riceve i metadati
   aggiornati tramite GET /federation/feed/{creatorId}
   e aggiorna il proprio DB
```

**Sostituzione del file audio:**

```
1. Creator sostituisce il file audio sul disco
2. Creator Agent aggiorna il checksum del file nel DB locale
3. Al ciclo di sync successivo l'home server rileva
   il checksum cambiato e invalida la propria cache del file
4. Al ciclo di sync successivo ogni peer riceve il nuovo checksum
   nei metadati e invalida la propria cache del file
5. La prossima richiesta di quell'episodio seguirà il normale
   flusso di risoluzione scaricando il file aggiornato
```

Il **checksum** (SHA-256 del file audio) è il meccanismo che permette ai server di capire se un file è cambiato senza doverlo scaricare. Viene incluso nei metadati dell'episodio e aggiunto al modello `Episode`:

```csharp
public string AudioChecksum { get; set; }  // SHA-256 del file audio
public DateTime UpdatedAt { get; set; }    // data ultima modifica
```

I peer temporaneamente offline si aggiorneranno automaticamente al primo sync dopo essere tornati online — nessuna logica di retry necessaria.

### 5.5 Modello Dati (EF Core)

```csharp
public class FederatedServer {
    public Guid Id { get; set; }
    public string BaseUrl { get; set; }
    public DateTime LastSeen { get; set; }
    public bool IsActive { get; set; }
}
```

Il modello `CachedEpisode` è definito nella sezione 4.3 e include `LastAccessedAt` per la gestione del TTL scorrevole.

I metadati sono divisi in due modelli distinti: il **Podcast** (il canale del creator) e il singolo **Episodio**.

```csharp
// Il canale del creator — uno per creator
public class Podcast {
    public string CreatorId { get; set; }        // ID univoco del creator
    public string HomeServerUrl { get; set; }    // Home server del creator

    public string Title { get; set; }            // Nome del podcast
    public string Description { get; set; }      // Descrizione del canale
    public string CoverImageUrl { get; set; }    // Immagine di copertina del canale
    public string Author { get; set; }           // Nome dell'autore
    public List<string> Categories { get; set; } // Categorie (es. "Tecnologia", "Sport")
    public string Language { get; set; }         // Lingua (es. "it", "en")
    public DateTime CreatedAt { get; set; }      // Data di creazione del canale
    public DateTime UpdatedAt { get; set; }      // Ultimo aggiornamento
}

// Il singolo episodio — molti per creator
public class Episode {
    // Identificazione
    public string EpisodeId { get; set; }        // ID univoco dell'episodio
    public string CreatorId { get; set; }        // ID del creator (FK → Podcast)

    // Metadati dell'episodio
    public string Title { get; set; }            // Titolo dell'episodio
    public string Description { get; set; }      // Descrizione / note dell'episodio
    public string CoverImageUrl { get; set; }    // Copertina specifica (se diversa dal canale)
    public DateTime PublishedAt { get; set; }    // Data di pubblicazione
    public TimeSpan Duration { get; set; }       // Durata

    // Organizzazione
    public int? Season { get; set; }             // Stagione (opzionale)
    public int? EpisodeNumber { get; set; }      // Numero episodio (opzionale)
    public List<string> Tags { get; set; }       // Tag per la ricerca

    // File audio
    public string AudioUrl { get; set; }         // Path relativo sull'home server
    public string AudioFormat { get; set; }      // es. "mp3", "m4a", "ogg"
    public long FileSizeBytes { get; set; }      // Dimensione file in byte
    public string AudioChecksum { get; set; }    // SHA-256 del file — usato per rilevare modifiche
    public DateTime UpdatedAt { get; set; }      // Data ultima modifica (metadati o file)

    // Cache (solo sui server, non sul Creator Agent)
    public bool IsLocallyCached { get; set; }
    public DateTime? CachedAt { get; set; }
    public DateTime? LastAccessedAt { get; set; }
}
```

---

## 6. Sicurezza

### 6.1 Autenticazione dei Canali

| Canale | Autenticazione | Crittografia |
|---|---|---|
| Utente → Server | Nessuna (pubblica) | HTTPS/TLS |
| Server → Creator Agent | Bearer Token (JWT) | HTTPS/TLS |
| Server → Server (Fed.) | HMAC-SHA256 su nonce | HTTPS/TLS |
| Admin → Server | API Key + OAuth2 | HTTPS/TLS |

> 💡 Se si usa **Cloudflare Tunnel**, i server federati non conoscono l'IP reale del creator. L'indirizzo esposto è quello del tunnel provider.

### 6.2 Firma dei Messaggi

Ogni richiesta tra componenti del sistema deve essere firmata, in modo che il destinatario possa verificare che il messaggio arrivi effettivamente dal mittente dichiarato e non sia stato alterato in transito.

**Creator Agent → Home Server e Home Server → Creator Agent**

Ogni richiesta HTTP include un header di firma calcolato sull'intero body e su un timestamp:

```
X-Distopia-Signature: HMAC-SHA256(secret, timestamp + method + path + body)
X-Distopia-Timestamp: 1718000000
```

Il ricevente ricalcola la firma e la confronta. Se non corrisponde, la richiesta viene rifiutata. Il timestamp previene i replay attack: richieste con timestamp più vecchio di 60 secondi vengono scartate.

**Server → Server (Federazione)**

I server federati usano lo stesso meccanismo HMAC-SHA256. Ogni coppia di server condivide un segreto generato al momento del primo handshake di federazione. Il segreto viene scambiato una sola volta in modo sicuro (durante l'onboarding, vedi sezione 5.2) e poi usato per firmare tutte le comunicazioni successive.

### 6.3 Considerazioni sulla Fiducia

- Il Creator Agent accetta richieste **solo dall'home server** — un token non riconosciuto causa rifiuto immediato
- Un server federato che presenta una firma non valida viene marcato come `IsActive = false` nel DB e non viene più contattato finché un admin non lo riabilita
- Tutti i segreti HMAC sono ruotabili dalla UI senza downtime

---

## 7. Piano di Implementazione

| Fase | Componente | Attività | Priorità |
|---|---|---|---|
| 1 | Creator Agent | Kestrel + streaming file + feed JSON | Alta |
| 2 | Distribution Server | Cache disco + proxy creator + API pubblica | Alta |
| 3 | Web UI (integrata) | Blazor: ricerca, player audio, liste episodi | Alta |
| 4 | Federazione | Sync metadati tra server + gossip protocol | Media |
| 5 | Sicurezza | JWT, HMAC, rate limiting, audit log | Media |
| 6 | DevOps | Docker, CI/CD, Helm chart per Kubernetes | Bassa |
| 7 | Dashboard Creator | Statistiche ascolti, gestione accessi | Bassa |

### Stack Tecnologico

| Layer | Tecnologia |
|---|---|
| Runtime | .NET 9 (C#) |
| Web Framework | ASP.NET Core 9 Minimal APIs |
| Web Server | Kestrel (embedded) |
| Realtime | SignalR (connessione Creator Agent → Home Server) |
| Cache Memoria | IMemoryCache / Redis (opzionale) |
| Database | **SQL Server** (Distribution Server) / **SQLite** (Creator Agent) via EF Core 9 |
| Desktop UI | Avalonia UI 11 (Win/Mac/Linux) |
| Feed Format | RSS 2.0 + JSON Feed |
| Auth | Microsoft.AspNetCore.Authentication.JwtBearer |
| Containerizzazione | Docker + docker-compose |
| Logging | Serilog + OpenTelemetry |

---

## 8. Configurazione e Deployment

### Creator Agent — appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=distopia-agent.db"
  },
  "CreatorAgent": {
    "ListenPort": 5100,
    "PodcastsDirectory": "C:/Podcasts",
    "HomeServer": {
      "Url": "https://pod1.example.com",
      "Token": "..."
    },
    "CreatorId": "mario-rossi",
    "MaxConcurrentStreams": 5
  }
}
```

### Distribution Server — appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=Distopia;Trusted_Connection=True;"
  },
  "DistributionServer": {
    "CacheDirectory": "/var/podcast-cache",
    "MaxCacheSizeGB": 50,
    "CacheTTLDays": 30,
    "CacheCleanupHour": 3,
    "FederationMode": "open",
    "FederationBootstrap": "https://server-noto.example.com",
    "FederationPeers": [
      "https://pod2.example.com",
      "https://pod3.example.com"
    ],
    "SyncIntervalMinutes": 5
  }
}
```

`CacheTTLDays` definisce il numero di giorni di inattività dopo cui un file viene eliminato dalla cache. Il contatore viene azzerato ad ogni richiesta. `CacheCleanupHour` definisce l'ora notturna in cui il Background Service esegue la pulizia (default: 3:00).

### Docker Compose

```yaml
services:
  distopia-server:
    image: distopia/server:latest
    ports: ["443:8080"]
    volumes:
      - podcast-cache:/var/podcast-cache
      - ./appsettings.json:/app/appsettings.json
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
```

### Connessione Creator Agent — Home Server (SignalR)

Distopia non dipende da nessun servizio esterno per la connessione tra Creator Agent e Home Server. La comunicazione avviene tramite **SignalR**, già incluso in ASP.NET Core 9 — zero librerie aggiuntive, zero dipendenze esterne, zero registrazioni su servizi di terzi.

**Come funziona:**

All'avvio il Creator Agent apre una connessione WebSocket persistente verso il suo Home Server tramite SignalR. La connessione è **sempre uscente dal Creator Agent** — non serve aprire porte sul router, non serve un IP fisso, non serve nessuna configurazione di rete. Funziona dietro qualsiasi NAT o firewall domestico esattamente come un browser che apre una pagina web.

```
1. Creator Agent si avvia
2. Apre connessione SignalR → Home Server (connessione uscente)
3. La connessione rimane aperta — è il canale permanente
4. Server ha bisogno di un file:
      Server ──SignalR──→ Creator Agent: "ho bisogno di EP123"
5. Creator Agent riceve il messaggio e risponde con un HTTP upload:
      Creator Agent ──HTTP streaming──→ Server
6. Server riceve il file, lo mette in cache, lo serve all'utente
```

**Perché SignalR per la segnalazione e HTTP per il file:**

SignalR è ottimo per messaggi leggeri e in tempo reale, ma non è adatto al trasferimento di file audio da centinaia di MB. Il canale è quindi diviso in due ruoli distinti:

- **SignalR** — segnalazione: il server avvisa il Creator Agent di quale file ha bisogno
- **HTTP streaming** — trasferimento: il Creator Agent fa un upload in streaming del file direttamente verso il server

In entrambi i casi è sempre il Creator Agent ad aprire le connessioni. Il server non deve mai "chiamare" il client in senso tradizionale.

**Gestione della connessione:**

SignalR gestisce automaticamente la riconnessione se la connessione cade (es. PC in sleep, cambio rete). Il Creator Agent tenta la riconnessione con backoff esponenziale. Se il creator è offline, il server serve il file dalla propria cache; se non è in cache, restituisce un errore temporaneo all'utente.

### Installazione del Creator Agent

Il creator deve solo scaricare ed eseguire un singolo file — `Distopia-CreatorAgent-Setup.exe`. Il wizard di installazione gestisce tutto in automatico, senza nessuna registrazione su servizi esterni.

**Flusso di installazione dal punto di vista del creator:**

```
1. Scarica Distopia-CreatorAgent-Setup.exe
2. Avvia il wizard di installazione
3. Sceglie la cartella dove terrà i suoi podcast
4. Inserisce l'URL del suo home server e il token di accesso
5. Finito — il Creator Agent è attivo e connesso
```

**Cosa fa l'installer in automatico:**

**Step 1 — Configurazione**
L'installer scrive l'`appsettings.json` con i dati inseriti dal creator: cartella podcast, URL home server, token.

**Step 2 — Registrazione all'home server**
Il Creator Agent contatta l'home server con `POST /creators/register` e si registra. Il server salva le informazioni del creator nel DB.

**Step 3 — Apertura connessione SignalR**
Il Creator Agent apre la connessione WebSocket persistente verso l'home server. Da questo momento è operativo e raggiungibile.

**Step 4 — Registrazione come servizio di sistema**
Il Creator Agent viene registrato come servizio di sistema (Windows Service su Windows, launchd su macOS, systemd su Linux). Si avvia automaticamente con il PC — il creator non deve ricordarsi di avviare nulla manualmente.

---

---

## 9. Statistiche

### 9.1 Cosa viene tracciato

Ogni server tiene traccia degli ascolti degli episodi che serve. Per ogni riproduzione vengono registrati:

- **EpisodeId** — quale episodio è stato ascoltato
- **CreatorId** — a quale creator appartiene
- **Date** — giorno dell'ascolto (senza ora per privacy)
- **IsUniqueUser** — se l'utente è registrato, viene contato una sola volta per episodio per giorno; se anonimo, viene contato tramite hash anonimizzato dell'IP

```csharp
public class PlayEvent {
    public Guid Id { get; set; }
    public string EpisodeId { get; set; }        // FK → Episode
    public string CreatorId { get; set; }         // FK → Podcast
    public DateTime Date { get; set; }            // giorno dell'ascolto
    public bool IsRegisteredUser { get; set; }    // utente registrato o anonimo
    public Guid? UserId { get; set; }             // null se anonimo
}
```

### 9.2 Aggregazione e Invio al Creator

Le statistiche non vengono inviate in tempo reale — viaggiano nel ciclo di sync periodico dei 5 minuti. Ogni server aggrega i dati del periodo e li invia all'home server del creator tramite il canale federato:

```
1. Ogni server conta gli ascolti per episodio nel periodo
2. Al ciclo di sync l'home server raccoglie i dati da tutti i peer
3. L'home server aggrega i totali e li rende disponibili al Creator Agent
4. Il Creator Agent mostra le statistiche nella dashboard
```

### 9.3 Dashboard del Creator

Il creator vede nella UI del Creator Agent:

| Metrica | Dettaglio |
|---|---|
| Ascolti totali per episodio | Numero totale di riproduzioni |
| Utenti unici per episodio | Utenti distinti che hanno ascoltato |
| Andamento nel tempo | Grafico giornaliero e mensile degli ascolti |

Le statistiche sono aggregate a livello di episodio e di podcast. Non sono disponibili dati geografici o demografici — solo conteggi anonimi.

---

## 11. Registrazione del Creator sull'Home Server

### 9.1 Flusso di Registrazione

Il creator deve inserire **solo l'URL dell'home server** durante l'installazione del Creator Agent. Non è richiesta nessuna registrazione preventiva sul server, nessun account, nessun token generato manualmente.

Il Creator Agent gestisce tutto in autonomia:

```
1. Creator installa il Creator Agent e inserisce l'URL dell'home server
2. Il Creator Agent genera autonomamente un token univoco (GUID + firma HMAC)
3. Il Creator Agent contatta l'home server con POST /creators/register
   inviando: token, CreatorId, nome, lista podcast disponibili
4. L'home server riceve la registrazione e la mette in stato "in attesa"
5a. Se il server è in modalità approvazione manuale →
    l'amministratore approva dal pannello admin
5b. Se il server è in modalità approvazione automatica →
    il creator viene approvato immediatamente
6. Il Creator Agent riceve la conferma e apre la connessione SignalR
7. Da questo momento il creator è operativo sulla rete
```

### 9.2 Modalità di Approvazione

Esattamente come per la federazione, ogni server può scegliere la propria politica:

```json
"CreatorRegistration": "manual"    // l'admin approva ogni creator manualmente
"CreatorRegistration": "automatic" // ogni creator viene approvato in automatico
```

In modalità **manuale** il creator vede nella UI del Creator Agent lo stato "in attesa di approvazione" finché l'amministratore non interviene. La connessione SignalR non viene aperta fino all'approvazione.

In modalità **automatica** la registrazione viene confermata immediatamente e il creator è operativo in pochi secondi dall'installazione.

### 9.3 Sicurezza

Il token generato dal Creator Agent viene usato per firmare tutte le comunicazioni successive con l'home server (vedi sezione 6.2). Se un amministratore vuole revocare l'accesso a un creator, gli basta disattivarlo dal pannello admin — il token viene invalidato e la connessione SignalR viene chiusa.

---

## 12. Gestione degli Utenti Web

### 12.1 Accesso Pubblico vs Registrato

La piattaforma è accessibile a tutti senza registrazione per le funzionalità base. La registrazione sblocca funzionalità personalizzate che richiedono persistenza dei dati sul server.

| Funzionalità | Non registrato | Registrato |
|---|---|---|
| Ricerca podcast ed episodi | ✓ | ✓ |
| Ascolto episodi | ✓ | ✓ |
| Navigazione per categorie/tag | ✓ | ✓ |
| Profilo creator | ✓ | ✓ |
| Playlist | ✗ | ✓ |
| Preferiti | ✗ | ✓ |
| Storico ascolti + posizione di ripresa | ✗ | ✓ |
| Abbonamenti a creator + feed personale | ✗ | ✓ |

### 12.2 Registrazione

La registrazione avviene tramite **email e password** oppure tramite **provider esterni opzionali** (Google, Apple). In entrambi i casi viene creato un account univoco sul server.

L'autenticazione è gestita tramite **ASP.NET Core Identity** con JWT per le sessioni. I provider esterni sono integrati tramite OAuth2.

### 12.3 Funzionalità per Utenti Registrati

**Playlist**
Un utente può creare una o più playlist contenenti episodi dello **stesso podcast**. Gli episodi possono essere ordinati liberamente. Una playlist appartiene a un singolo utente e non è condivisibile.

**Preferiti**
Un utente può salvare episodi o interi podcast tra i preferiti per ritrovarli rapidamente. I preferiti sono una lista piatta senza ordinamento specifico.

**Storico ascolti e posizione di ripresa**
Ogni episodio ascoltato viene registrato nel DB con la posizione raggiunta (in secondi). Quando l'utente riapre un episodio già ascoltato, il player riprende automaticamente dall'ultimo punto. Lo storico è visibile nella propria area personale. Nessun dato viene salvato localmente nel browser — tutto risiede sul server.

**Abbonamenti e feed personale**
Un utente può seguire uno o più creator. L'abbonamento genera un **feed personale** che aggrega in ordine cronologico tutti i nuovi episodi dei creator seguiti. Il feed è la prima cosa che vede l'utente loggato quando accede alla piattaforma.

### 12.4 Modello Dati Utenti

```csharp
public class User {
    public Guid Id { get; set; }
    public string Email { get; set; }
    public string? PasswordHash { get; set; }    // null se login con provider esterno
    public string? ExternalProvider { get; set; } // es. "Google", "Apple"
    public string? ExternalProviderId { get; set; }
    public DateTime RegisteredAt { get; set; }
}

public class Playlist {
    public Guid Id { get; set; }
    public Guid UserId { get; set; }             // FK → User
    public string CreatorId { get; set; }         // playlist legata a un solo podcast
    public string PodcastId { get; set; }         // FK → Podcast
    public string Title { get; set; }
    public List<PlaylistItem> Items { get; set; }
}

public class PlaylistItem {
    public Guid Id { get; set; }
    public Guid PlaylistId { get; set; }          // FK → Playlist
    public string EpisodeId { get; set; }         // FK → Episode
    public int Order { get; set; }                // posizione nella playlist
}

public class Favorite {
    public Guid Id { get; set; }
    public Guid UserId { get; set; }              // FK → User
    public string? EpisodeId { get; set; }        // episodio preferito (opzionale)
    public string? PodcastId { get; set; }        // podcast preferito (opzionale)
    public DateTime SavedAt { get; set; }
}

public class ListeningHistory {
    public Guid Id { get; set; }
    public Guid UserId { get; set; }              // FK → User
    public string EpisodeId { get; set; }         // FK → Episode
    public int PositionSeconds { get; set; }      // posizione di ripresa
    public DateTime LastListenedAt { get; set; }
}

public class Subscription {
    public Guid Id { get; set; }
    public Guid UserId { get; set; }              // FK → User
    public string CreatorId { get; set; }         // creator seguito
    public DateTime SubscribedAt { get; set; }
}
```

## 13. Considerazioni Future

- **Notifiche** — push nel browser o via email quando un creator seguito pubblica un nuovo episodio (non implementato nella versione iniziale)
- **Condivisione playlist** — rendere una playlist pubblica e condivisibile tramite link

- **ActivityPub / Fediverse** — compatibilità con Mastodon e sistemi federati
- **IPFS** — distribuzione P2P dei file come alternativa ai Creator Agent
- **Lightning Network** — micropagamenti per podcast premium
- **Transcoding** — conversione automatica in più formati/bitrate
- **Analytics privacy-first** — statistiche aggregate tra server federati
- **App mobile** — client iOS/Android che può fare anche da Creator Agent

> La scalabilità orizzontale è nativa: si possono aggiungere Distribution Server senza modifiche architetturali. I Creator Agent sono il solo bottleneck, mitigabile con `MaxConcurrentStreams` e un CDN come layer L0 per i file più popolari.
