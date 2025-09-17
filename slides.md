---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://plus.unsplash.com/premium_photo-1733309638257-d0aea805dba6?q=80&w=1374&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
# some information about your slides (markdown enabled)
title: IndexedDB
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---
# IndexedDB

## Il Database che non volevamo ma abbiamo imparato ad amare

---

# Cosa è IndexedDB?

<style>
li { font-size: 1.8rem; }
</style>

- **Database NoSQL** per il browser
- Memorizzazione lato client di **grandi quantità di dati strutturati**
- Supporta file, blob e dati binari e perfino chiavi crittografiche!
- **Persistente** (sopravvive al riavvio del browser, a volte...)

---

# La Storia di IndexedDB

## Web SQL Database (2009-2010)
- Basato su SQLite 3.6.19
- Sintassi SQL familiare
- **Firefox disse NO** ❌ sta roba non è standardizzabile!
- W3C abbandonò la specifica

## IndexedDB (2010-2012)
- Nikunj Mehta da Oracle propose IndexedDB (quelli che hanno il trademark su JS!)
- **Accettato rapidamente** da tutti i browser
- Brutto, meno usabile ma standardizzabile
- Implementato su Firefox usando SQLite...

---

# Come Iniziare con IndexedDB

```javascript
import { openDB } from 'idb';

const db = await openDB('my-db', 1, {
  upgrade(db) {
    db.createObjectStore('users');
  }
});
```

vs

```javascript
function openDB(done) {
  const request = indexedDB.open("my-db");
  request.onupgradeneeded = (event) => {
    const db = event.target.result;
    db.createObjectStore('users')
  };
  request.onsuccess = (event) => {
    const db = event.target.result;
    done(db)
  };
}
```

---

# Object Stores


<style>
li { font-size: 1.6rem; }
</style>

## Cosa sono?
- **Contenitori** per i dati (simili alle tabelle)
- Ogni store ha una **chiave primaria**

## Tipi di chiavi:
- **Incrementale**: `{ keyPath: 'id', autoIncrement: true }`
- **Chiave specifica**: `{ keyPath: 'email' }`
- **Out-of-line**: la chiave è esterna ed è inserita in fase di creazione

---

# Esempio: Creazione Object Store

```javascript
const db = await openDB('ecommerce', 1, {
  upgrade(db) {
    // Store per prodotti
    const productStore = db.createObjectStore('products', {
      keyPath: 'id',
      autoIncrement: true
    });
    
    // Store per ordini
    const orderStore = db.createObjectStore('orders', {
      keyPath: 'orderId'
    });
    
    // Indici per ricerche
    productStore.createIndex('category', 'category');
    orderStore.createIndex('customer', 'customerId');
  }
});
```

---

# Filtrare e Reperire Dati

## Operazioni CRUD di base

```javascript
// CREATE
await db.add('products', {
  name: 'iPhone 15',
  category: 'smartphone',
  price: 999
});

// READ
const product = await db.get('products', 1);
const allProducts = await db.getAll('products');

// UPDATE
await db.put('products', { id: 1, name: 'iPhone 15 Pro' });

// DELETE
await db.delete('products', 1);
```

---

# Filtrare con Indici

```javascript
// Filtro per categoria
const smartphones = await db.getAllFromIndex('products', 'category', 'smartphone');

// Range queries
const expensiveProducts = await db.getAllFromIndex(
  'products', 
  'price', 
  IDBKeyRange.lowerBound(500)
);

const productsInMyRange = await db.getAllFromIndex(
  'products', 
  'price', 
  IDBKeyRange.bound(500, 1000)
);
```

---

# Pain Point #1: Transaction Isolation


<style>
li { font-size: 1.6rem; }
</style>

## IndexedDB è un database transazionale

- Tutto avviene tramite transaction
- Ci sono tre modalità: readwrite, readonly e versionchange
- readwrite crea un lock su objectstore
- readonly transactions possono essere eseguite in modo concorrente

## Il Problema
- Una readonly eseguita in modo concorrente a una readwrite non è isolata
- Questo vale anche quando il DB è caricato su più tab

---

# Pain Point #2: Autocommit delle Transazioni
.

<style>
p { font-size: 1.6rem; }
</style>

Le transazioni si **chiudono automaticamente** dopo un microtask se non ci sono "operazioni pending"

```javascript
// ❌ SBAGLIATO
const tx = db.transaction('keyval', 'readwrite');
const store = tx.objectStore('keyval');
const val = (await store.get('counter')) || 0;
// Questo non va fatto:
const newVal = await fetch('/increment?val=' + val);
// La transazione è chiusa, quindi questo darà errore
await store.put(newVal, 'counter');
await tx.done;
```

---

# Pain Point #3: Safari
I dati vengono cancellati dopo 7 giorni di inattività

<style>
p { font-size: 1.6rem; }
</style>

Venduto come "Intelligent Tracking Prevention"

Rende IndexedDB usabile solo come "disposable storage"

---

# Pain Point #4: Performance
Molto complicato da ottimizzare

<style>
p { font-size: 1.6rem; }
</style>

- Data la natura transazionale, tante piccole query sono lente
- Ma non ci sono API per aggregare dati (e.g semplici operatori OR)
- Fun: absurd-sql ha dimostrato che SQLite sopra IndexedDB è più veloce

---

# Origin Private File System
Un filesystem privato con API sincrone e asincrone

```javascript
// Accesso al file system privato
const root = await navigator.storage.getDirectory();

// Creazione file
const fileHandle = await root.getFileHandle('data.json', { create: true });
const writable = await fileHandle.createWritable();
await writable.write(JSON.stringify(data));
await writable.close();

// Lettura file
const file = await fileHandle.getFile();
const content = await file.text();
```

---

# Origin Private File System


<style>
li { font-size: 1.6rem; }
</style>

- Ci permette di avere Database **veri** al costo però di doverli caricare
- Devtools praticamente inesistenti
- Non soggetto al "Intelligent" Tracking Prevention di Safari
- Usabile solo in secure contexts

---

# Conclusioni

## IndexedDB è orribile ma...
- Funziona
- Se qualcuno te lo spiega è abbastanza semplice
- è un API nativa, quindi non ci costa nulla in termini di caricamento
- per tutto il resto c'è OPFS e WASM

---

# Grazie! 🙏

## Domande?

**Risorse:**
- [idb Library](https://github.com/jakearchibald/idb)
- [The pain and anguish of using IndexedDB](http://gist.github.com/pesterhazy/4de96193af89a6dd5ce682ce2adff49a) outdated, ma interessante
- [MDN OPFS Guide](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system)
- [OPFS docs di Safari](https://webkit.org/blog/12257/the-file-system-access-api-with-origin-private-file-system/)

**Contatti:**
- GitHub: [@gdorsi]
- Discord di RomaJS
