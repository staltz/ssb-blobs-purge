# ssb-blobs-purge

SSB plugin to automatically remove old and large blobs.

Works by recursively traversing the blobs folder, and prioritizing deletes of blobs that are old (small creation timestamp) and big (large file size), or both. Does not delete blobs that your own feed posted (or mentioned first). Stops deletion once the `storageLimit` target is reached.

**Note!** This deletion process creates a bias against old content, and this may not be the best choice for every SSB app. For instance, in some cases it may make sense to favor blobs in a particular hashtag ("channel") over other hashtags, regardless of how old or large they are. The choices in this module also favor blobs that were mentioned by the local `ssb.id` feed, and does not consider 'same as' accounts. The heuristic used to pick the next deletable blob also favors implementation performance, instead of picking the best possible deletable blob (see section at the bottom of this readme).

## Usage

**Prerequisites:**

- **Node.js 6.5** or higher
- `secret-stack@^6.2.0`
- `ssb-blobs` installed
- Either of these two (preferably not both!):
  - `ssb-db2` (plus `ssb-db2/full-mentions`) installed
  - `ssb-backlinks` installed

```
npm install --save ssb-blobs-purge
```

Add this plugin to ssb-server like this:

```diff
 var createSsbServer = require('ssb-server')
     .use(require('ssb-onion'))
     .use(require('ssb-unix-socket'))
     .use(require('ssb-no-auth'))
     .use(require('ssb-plugins'))
     .use(require('ssb-master'))
     .use(require('ssb-db2'))
     .use(require('ssb-db2/full-mentions'))
     .use(require('ssb-conn'))
     .use(require('ssb-blobs'))
+    .use(require('ssb-blobs-purge'))
     .use(require('ssb-replicate'))
     .use(require('ssb-friends'))
     // ...
```

Now you should be able to access the following muxrpc APIs under `ssb.blobsPurge.*`:

| API | Type | Description |
|-----|------|-------------|
| **`start(opts?)`** | `sync` | Triggers the start of purge task. Optionally, you may pass an object with two fields: `cpuMax` and `storageLimit` |
| **`stop()`** | `sync` | Stops the purge task if it is currently active. |
| **`changes()`** | `source` | A pull-stream that emits events notifying of progress done by the purge task, these events are objects as described below. |

A `changes()` event object is one these three possibile types:

- `{ event: 'deleted', blobId }`
- `{ event: 'paused' }`
- `{ event: 'resumed' }`

## Setting the parameters

There are two parameters you can tweak with this module:

- `cpuMax`: **(default 50)** a number between `0` and `100` that represents a threshold for CPU usage; when your device's CPU usage is *below* this percentual number, the purge task will run, otherwise it will be paused
- `storageLimit`: **(default 10 gigabytes)** a number (in bytes) that represents the desired maximum storage for blobs; when your blobs folder is *larger* than this number, the purge task will run, otherwise it will be paused

There are two ways you can set these parameters:

**(1) In the SSB config object**:

```diff
 {
   path: ssbPath,
   keys: keys,
   port: 8008,
   blobs: {
     sympathy: 2
   },
+  blobsPurge: {
+    cpuMax: 30,
+    storageLimit: 2000000000
+  },
   conn: {
     autostart: false
   }
 }
```

**(2) When calling the `start` function**, note that these numbers are **ignored** in case you supply (1) above:

```javascript
ssb.blobsPurge.start({ cpuMax: 30, storageLimit: 2000000000 })
```

## How does this work?

For each blob, we calculate a heuristic:

```
heuristic = blobSizeInKb * ageInDays
```

Then, instead of sorting the blobs in descending heuristic order (that would be `O(n log n)`), we do at most 6 linear passes (hence `O(6 n) = O(n)`) over the blobs set:

- On the first pass, we delete any blob whose heuristic number is greater than `1e7`
- On the second pass, we delete any blob whose heuristic number is greater than `1e6`
- On the third pass, we delete any blob whose heuristic number is greater than `1e5`
- On the fourth pass, we delete any blob whose heuristic number is greater than `1e4`
- On the fifth pass, we delete any blob whose heuristic number is greater than `1e3`
- On the sixth pass, we delete any blob

As soon as the `storageLimit` is met, we immediately cancel the current pass, and begin waiting for new blobs to be added. Once there are sufficiently new blobs, we resume the 6 passes.

There is one exception: blobs that your own account has mentioned in posts are considered "yours", and are never deleted.

## License

MIT
