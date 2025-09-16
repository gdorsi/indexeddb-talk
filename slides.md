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
# open graph
seoMeta:
  # By default, Slidev will use ./og-image.png if it exists,
  # or generate one from the first slide if not found.
  ogImage: auto
  # ogImage: https://cover.sli.dev
---
# IndexedDB

## Il Database che non volevamo ma impariamo ad amare

---

# Cosa è IndexedDB?

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
- API più complessa ma più potente
- Standard W3C ufficiale
- Implementato su Firefox usando SQLite ha!

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

## Il Problema
- Le transazioni si **chiudono automaticamente** dopo un microtask se non ci sono "operazioni pending"

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

# Pain Point #3: State deleted after 7 days of inactivity (Safari)

Venduto come "Intelligent Tracking Prevention (ITP)"

Rende IndexedDB usabile solo come "disposable storage"

---

# Alternative Moderne: OPFS

## Origin Private File System

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

- Ci permette di avere Database **veri** al costo però di doverli caricare
- Devtools praticamente inesistenti

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
- [MDN IndexedDB Guide](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [idb Library](https://github.com/jakearchibald/idb)
- [The pain and anguish of using IndexedDB](http://gist.github.com/pesterhazy/4de96193af89a6dd5ce682ce2adff49a) outdated, ma interessante
- [OPFS Spec](https://fs.spec.whatwg.org/)

**Contatti:**
- GitHub: [@gdorsi]
- Twitter: [@gdorsi]
