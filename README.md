# firestore-batcher

The easier way to do batched write operations for Cloud Firestore.

```bash
npm install firestore-batcher
```

```typescript
import * as firebase from 'firestore-admin'
import firestoreBatcher, { deleteOperation } from 'firestore-batcher'

const app = firebase.initializeApp()
const db = firebase.firestore()

const batcher = firestoreBatcher(db)

async function bulkDelete() {
    const finishedTodosQuery = db
        .collection('todos')
        .where('finished', '==', true)

    await batcher.all(finishedTodosQuery, (todoRef) => deleteOperation(todoRef))
}

bulkDelete()
```

## Description

Using _batches_ is the recommended way of performing Firestore write operations in bulk, like [deleting a collection](https://firebase.google.com/docs/firestore/manage-data/delete-data#collections).

There are some considerations you need to make when using `batch`, that this library will handle for you:

-   A batch has an upper limit of 500 operations. If you need more operations, you need to make more batches.
-   Even if you stick to less than 500 ops/batch, a batch might fail due to "Transaction is too big", depending of the size of the data.

This tool will run batches recursively on your operations. The batch size is changed dynamically to handle "Transaction too big" errors.

## API

The default export is a function that takes a `firestore` instance and returns a `Batcher` instance.

```typescript
import * as firebase from 'firestore-admin'
import firestoreBatcher, { deleteOperation } from 'firestore-batcher'

const app = firebase.initializeApp()
const db = firebase.firestore()

const batcher = firestoreBatcher(db)
```

### Batcher

An object with various methods:

#### `add` function

`<T>(operation: Operation<T>) => void`

Adds an operation on a single document to the batch queue. Make sure you call `batcher.commit()` at least once after using this.

Example

```typescript
const lotr = db.collection('books').doc('lord-of-the-rings')
const batcher.add(updateOperation(lotr, { read: true }))
const harry = db.collection('books').doc('harry-potter')
batcher.add(deleteOperation(harry))
await batcher.commit()
```

#### `all` function

`(query: FirebaseFirestore.Query<DocumentData>, operationBuilder: <T>(doc: DocumentReference<T>) => Operation<T>) => Promise<void>`

This function takes a query and a callback. The query will be limited according to `batchSize`, and fetched and committed recursively.

Example:

```typescript
const finishedTodosQuery = db.collection('todos').where('finished', '==', true)

await batcher.all(finishedTodosQuery, (todoRef) => deleteOperation(todoRef))
```

If we imagine there are 1337 finished todos, the above code will first fetch 500, delete them, fetch 500 more, delete them, fetch 337 and delete them.

This method will commit each batch, so there is no need to call `batcher.commit()` yourself.

#### `commit` function

`() => Promise<void>`

Commits all queued operations recursively, using multiple batches if necessary.

### Operations

```
type Operation<T> = CreateOperation<T> | DeleteOperation<T> | SetOperation<T> | UpdateOperation<T>
```

An _operation_ is an object that represents a batch write operation (create, set, update or delete) on a specific document.

Four helper methods are exported to create these operation objects:

#### createOperation

```
<T>(documentRef: firestore.DocumentReference<T>, data: T) => CreateOperation<T>
```

Creates a new document.

#### deleteOperation

```
<T>(documentRef: firestore.DocumentReference<T>, precondition?: FirebaseFirestore.Precondition) => DeleteOperation<T>
```

Deletes a document.

#### setOperation

```
<T>(documentRef: firestore.DocumentReference<T>, data: T) => SetOperation<T>
```

Sets the data for a document.

#### updateOperation

```
<T>(documentRef: firestore.DocumentReference<T>, data: T, precondition?: FirebaseFirestore.Precondition) => UpdateOperation<T>
```

Updates parts of a document.
