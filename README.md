<!--
SPDX-FileCopyrightText: 2021 Anders Rune Jensen

SPDX-License-Identifier: CC0-1.0
-->

# SSB-DB2

SSB-DB2 is a new database for secure-scuttlebutt, it is meant as a
replacement for [ssb-db]. The main reason for creating a new database
is to be able to rework some of the existing decisions without having
to be 100% backwards compatible. The main reasons are:

 - Performance, the database stores data in [bipf]
 - Replace flume with [jitdb] and specialized indexes
 - Run in the browser via [ssb-browser-core](https://github.com/arj03/ssb-browser-core)
 - Work well with partial replication

SSB-DB2 is a secret-stack plugin that registers itself in the db
namespace.

By default SSB-DB2 only loads a base index (indexes/base), this index
includes the basic functionality for getting messages from the log and
for doing EBT.

By default the database is stored in `~/.ssb/db2/log.bipf` and indexes
are stored in `~/.ssb/db2/indexes/`.

🎥 [Watch a presentation about this new database](https://www.youtube.com/watch?v=efzJheWQey8).

[Read the developer guide](https://dev.scuttlebutt.nz/#/javascript/?id=ssb-db2)

## Usage

To get the post messages of a specific author, you can do:

```js
const SecretStack = require('secret-stack')
const caps = require('ssb-caps')
const {where, and, type, author, toCallback} = require('ssb-db2/operators')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .call(null, { path: './' })

sbot.db.query(
  where(
    and(
      type('post'),
      author('@6CAxOI3f+LUOVrbAl0IemqiS7ATpQvr9Mdw9LC4+Uv0=.ed25519')
    ),
  ),
  toCallback((err, msgs) => {
    console.log('There are ' + msgs.length + ' messages of type "post" from arj')
    sbot.close()
  })
)
```

To get post messages that mention Alice, you can do:

```js
const SecretStack = require('secret-stack')
const caps = require('ssb-caps')
const {where, and, type, mentions, toCallback} = require('ssb-db2/operators')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .call(null, { path: './' })

sbot.db.query(
  where(and(type('post'), mentions(alice.id)))),
  toCallback((err, msgs) => {
    console.log('There are ' + msgs.length + ' messages')
    sbot.close()
  })
)
```

### Leveldb plugins

The queries you've seen above use JITDB, but there are some queries
that cannot rely on JITDB alone, and we need to depend on
Leveldb. This section shows some example leveldb indexes, explains
when you need leveldb, and how to make your own leveldb plugin in
ssb-db2.

#### Full-mentions

An extra index plugin that is commonly needed in SSB communities is
the **full-mentions** index. It has one method: getMessagesByMention.

Although this accomplishes the same as the previous `mentions()`
example, this plugin is meant as an example for application developers
to write their own plugins if the functionality of JITDB is not
enough. JITDB is good for indexing specific values, like
`mentions(alice.id)` which gets its own dedidated JITDB index for
`alice.id`. But when querying mentions of several feeds or several
messages, this creates many indexes, so a specialized index makes more
sense.

What `full-mentions` does is index all possible mentionable items at
once, using Leveldb instead of JITDB. You can include it and use it
like this:

```js
const SecretStack = require('secret-stack')
const caps = require('ssb-caps')
const {where, and, type, toCallback} = require('ssb-db2/operators')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .use(require('ssb-db2/full-mentions')) // include index
  .call(null, { path: './' })

const {fullMentions} = sbot.db.operators

sbot.db.query(
  where(and(type('post'), fullMentions(alice.id)))),
  toCallback((err, msgs) => {
    console.log('There are ' + msgs.length + ' messages')
    sbot.close()
  })
)
```

#### About-self

Another extra index plugin that is commonly needed in SSB communities
is the **about-self** index. This indexes only self-assigned about
messages in contrast to [ssb-social-index] that indexes all about
messages.

Example usage:

```js
const SecretStack = require('secret-stack')
const caps = require('ssb-caps')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .use(require('ssb-db2/about-self')) // include index
  .call(null, { path: './' })

sbot.db.onDrain('aboutSelf', () => {
  const profile = sbot.db.getIndex('aboutSelf').getProfile(alice.id)
  console.log('Alice has name:' + profile.name)
})
```

#### Your own leveldb index plugin

It's wise to use JITDB when:

1. You want the query output to be the msg itself, not state derived
   from msgs
2. You want the query output ordered by timestamp (either descending
   or ascending)

There are some cases where the assumptions above are not met. For
instance, with abouts, we often want to aggregate all `type: "about"`
msgs and return all recent values for each field (`name`, `image`,
`description`, etc). So assumption number 1 does not apply.

In that case, you can make a leveldb index for ssb-db2, by creating a
class that extends the class at `require('ssb-db2/indexes/plugin')`,
like this:

```js
const Plugin = require('ssb-db2/indexes/plugin')

// This is a secret-stack plugin
exports.init = function (sbot, config) {
  class MyIndex extends Plugin {
    constructor(log, dir) {
      //    log, dir, name, version, keyEncoding, valueEncoding
      super(log, dir, 'myindex', 1, 'utf8', 'json')
    }

    processRecord(record, seq) {
      const buf = record.value // this is a BIPF buffer, directly from the log
      // ...
      // Use BIPF seeking functions to decode some fields
      // ...
      this.batch.push({
        type: 'put',
        key: key, // some utf8 string here (see keyEncoding in the constructor)
        value: value, // some object here (see valueEncoding in the constructor)
      })
    }

    myOwnMethodToGetStuff(key, cb) {
      this.level.get(key, cb)
    }
  }

  sbot.db.registerIndex(MyIndex)
}
```

There are three parts you'll always need:

- `constructor`: here you set configurations for the Leveldb index
  - `log` and `dir` you probably don't need to fiddle with, but you
    can use `this.log` methods if you know how to use
    async-append-only-log
  - `name` is a string that you'll use in `getIndex(name)`, it's also
    used as a directory name
  - `version`, upon changing, will cause a full rebuild of this index
  - `keyEncoding` and `valueEncoding` must be strings from
    [level-codec]
- `processRecord`: here you handle a msg (in [bipf]) and potentially
  write something to the index using
  `this.batch.push(leveldbOperation)`
- **custom method**: this is an API of your own choosing, that allows
  you to read data from the index

To call your custom methods, you'll have to pick them like this:

```js
sbot.db.getIndex('myindex').myOwnMethodToGetStuff()
```

Or you can wrap that in a secret-stack plugin (in the example above,
`exports.init` should return an object with the API functions).

There are other special methods you can implement in order to add
"hooks" in the `Plugin` subclass:

- `onLoaded(cb)`: called once, at startup, when the index is
  successfully loaded from disk and is ready to receive queries
- `onFlush(cb)`: called when the leveldb index is about to be saved to
  disk
- `indexesContent()`: method used when reindexing private group
  messages to determine if the leveldb index needs to be updated for
  decrypted messages. The default method returns true.

### Compatibility plugins

SSB DB2 includes a couple of plugins for backwards compatibility,
including legacy replication, ebt and publish. They can be loaded as:

```js
const SecretStack = require('secret-stack')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .use(require('ssb-db2/compat')) // include all compatibility plugins
  .call(null, {})
```

or specifically:

```js
const SecretStack = require('secret-stack')

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .use(require('ssb-db2/compat/db')) // basic db compatibility
  .use(require('ssb-db2/compat/log-stream')) // legacy replication
  .use(require('ssb-db2/compat/history-stream')) // legacy replication
  .use(require('ssb-db2/compat/ebt')) // ebt db helpers
  .call(null, {})
```

## Secret-stack modules using ssb-db2

The following is a list of modules that works well with ssb-db2:

 - [ssb-threads] for working with post messages as threads
 - [ssb-suggest-lite] for fetching profiles of authors 
 - [ssb-friends] for working with the social graph
 - [ssb-search2] for full-text searching
 - [ssb-crut] for working with records that can be modified

## Migrating from ssb-db

The log used underneath ssb-db2 is different than that one in ssb-db,
this means we need to scan over the old log and copy all messages onto
the new log, if you wish to use ssb-db2 to make queries.

**⚠️ Warning: please read the following instructions** about using two
logs and carefully apply them to avoid forking feeds into an
irrecoverable state.

### Preventing forking feeds

The log is the source of truth in SSB, and now with ssb-db2, we
introduce a new log alongside the previous one. **One of them, not
both** has to be considered the source of truth.

While the old log exists, it will be continously migrated to the new
log, and ssb-db2 forbids you to use its database-writing APIs such as
`add()`, `publish()`, `del()` and so forth, to prevent the two logs
from diverging into inconsistent states. The old log will remain the
source of truth and the new log will just mirror it.

If you want to switch the source of truth to be the new log, we must
delete the old log, after it has been fully migrated. Only then can
you use database-writing APIs such as `publish()`. To delete the old
log, one method is to use the [config
`dangerouslyKillFlumeWhenMigrated`](#configuration). Set it to `true`
only when you are **absolutely sure** that no other app will attempt
to read/write to `~/.ssb/flume/log.offset` or wherever the old log
lives. It will delete the entire flume folder once migration has
completed writing the messages to the new log. From that point
onwards, using APIs such as `publish()` will succeed to append
messages to the new log.

### Triggering migration

ssb-db2 comes with migration methods built-in, you can enable them
(they are off by default!) in your config file (or object):

```js
const path = require('path')
const SecretStack = require('secret-stack')
const ssbKeys = require('ssb-keys')
const keys = ssbKeys.loadOrCreateSync(path.join(__dirname, 'secret'))

const config = {
  keys: keys,
  db2: {
    automigrate: true
  }
}

const sbot = SecretStack({ caps })
  .use(require('ssb-db2'))
  .use(require('ssb-db2/compat'))
  .call(null, config)
```

The above script will initiate migration as soon as the plugins are
loaded. If you wish the manually dictate when the migration starts,
don't use the `automigrate` config above, instead, call the
`migrate.start()` method yourself:

```js
sbot.db.migrate.start()
```

Note, it is acceptable to load both ssb-db and ssb-db2 plugins, the
system will still function correctly and migrate correctly:

```js
const sbot = SecretStack({ caps })
  .use(require('ssb-db'))
  .use(require('ssb-db2'))
  .use(require('ssb-db2/compat'))
  .call(null, config)
```

### Migrating without including ssb-db2

Because ssb-db2 also begins indexing basic metadata once it's included
as a plugin, this may cost more (precious) CPU time. **If you are not
yet using db2 APIs** but would like to migrate the log anyway, in
preparation for later activating db2, then you can include only the
migration plugin, like this:

```js
const sbot = SecretStack({appKey: caps.shs})
  .use(require('ssb-db2/migrate'))
  .call(null, config)
```

Note that the `start` behavior is the same: you can either start it
automatically using `config.db2.automigrate` or manually like this:

```js
sbot.db2migrate.start()
```

## Methods

### get(msgId, cb)

Get a particular message value by message id.

### getMsg(msgId, cb)

Get a particular message including key by message id.

### del(msgId, cb)

Delete a specific message given the message id from the
database. Please note that this will break replication for anything
trying to get that message, like createHistoryStream for the author or
EBT. Because of this, it is not recommended to delete message with
this method unless you know exactly what you are doing.

### deleteFeed(feedId, cb)

Delete all messages of a specific feedId. Compared to `del` this
method is safe to use.

### publish(msgContent, cb)

Convenience method for validating and adding a classic SSB message to
the database written by the feed running the secret-stack. If message
`msgContent` contains recps, the message will automatically be encrypted.

### publishAs(feedKeys, msgContent, cb)

Convenience method for validating and adding a classic SSB message to
the database written by a different feed than running the secret-stack.
If message `msgContent` contains recps, the message will automatically be
encrypted.

### add(msgValue, cb)

Validate and add a message value (without id and timestamp) to the
database. In the callback will be the stored message (id, timestamp,
value = `msgValue`) or err. Supports `msgValue` in SSB classic feeds
as well as [Bendy Butt] messages

### addOOO(msgValue, cb)

Validate without checking the previous link and add to db. Useful for
partial replication.

### addOOOBatch(msgValues, cb)

Similar to `addOOO`, but you can pass an array of many message
values. If the author is not yet known, the message is validated
without checking if the previous link is correct, otherwise normal
validation. This makes it possible to use for partial replication to
add all contact messages from a feed.

### addTransaction(msgValues, oooMsgValues, cb)

Similar to `addOOOBatch`, except you pass in an array of `msgValues`
that will be validated in order and an array of `oooMsgValues` that
will be validated similar to `addOOOBatch`. Finally all the messages
are added to the database in such a way that either all of them are
written to disc or none of them are.

### post(cb)

Subscribe to any data added to the database. The `cb` will only
receive one argument, the message added. `post` is an [observable] so
the latest message added to the database can also be read using
`ssb.db.post.value`.

### getStatus

Gets the current db status, same functionality as
[db.status](https://github.com/ssbc/ssb-db#dbstatus) in ssb-db.

### reindexEncrypted(cb)

This function is useful in [ssb-db2-box2] where box2 keys can be added
at runtime and that changes what messages can be decrypted. Calling
this function is needed after adding a new key. The function can be
called multiple times safely.

### onDrain(indexName?, cb)

Waits for the index with name `indexName` to be in sync with the main
log and then call `cb` with no arguments. If `indexName` is not
provided, the base index will be used. 

The reason we do it this way is that indexes are updated
asynchronously in order to not block message writing.

## Configuration

You can use ssb-config parameters to configure some aspects of ssb-db2:

```js
const config = {
  keys: keys,
  db2: {
    /**
     * Start the migration plugin automatically as soon as possible.
     * Default: false
     */
    automigrate: true,

    /**
     * If the migration plugin is used, then when migration has completed, we
     * will remove the entire `~/.ssb/flume` directory, including the log.
     *
     * As the name indicates, this is dangerous, because if there are other apps
     * that still use `~/.ssb/flume`, they will see an empty log and progress to
     * write on that empty log using the `~/.ssb/secret` and this will very
     * likely fork the feed in comparison to new posts on the new log. Only use
     * this when you know the risks and you know that only the new log will be
     * written.
     * Default: false
     */
    dangerouslyKillFlumeWhenMigrated: false,

    /**
     * A debouncing interval (measured in milliseconds) to control how often
     * should messages given to `sbot.add` be flushed in batches.
     * Default: 250
     */
    addBatchDebounce: 250,

    /**
     * An upper limit on the CPU load that ssb-db2 can use while indexing
     * and scanning. `85` means "ssb-db2 will only index when CPU load is at
     * 85% or lower".
     * Default: Infinity
     */
    maxCpu: 85,

    /** This applies only if `maxCpu` is defined.
     * See `maxPause` in the module `too-hot`, for its definition.
     * Default: 300
     */
    maxCpuMaxPause: 180,

    /** This applies only if `maxCpu` is defined.
     * See `wait` in the module `too-hot`, for its definition.
     * Default: 90
     */
    maxCpuWait: 90,
  }
}
```

## Operators

The following operators are included by default, see
[operators/index.js] for how they are implemented. Also exposed are
all [JITDB operators]

* type
* author
* channel
* key
* votesFor
* contact
* mentions
* about
* hasRoot
* hasFork
* hasBranch
* authorIsBendyButtV1
* isRoot
* isPrivate
* isPublic

[ssb-db]: https://github.com/ssbc/ssb-db/
[bipf]: https://github.com/ssbc/bipf/
[jitdb]: https://github.com/ssb-ngi-pointer/jitdb/
[Bendy Butt]: https://github.com/ssb-ngi-pointer/ssb-bendy-butt
[ssb-social-index]: https://github.com/ssbc/ssb-social-index
[ssb-db2-box2]: https://github.com/ssb-ngi-pointer/ssb-db2-box2
[level-codec]: https://github.com/Level/codec#builtin-encodings
[ssb-threads]: https://github.com/ssbc/ssb-threads
[ssb-suggest-lite]: https://github.com/ssb-ngi-pointer/ssb-suggest-lite
[ssb-friends]: https://github.com/ssbc/ssb-friends
[ssb-search2]: https://github.com/staltz/ssb-search2
[ssb-crut]: https://gitlab.com/ahau/lib/ssb-crut
[operators/index.js]: https://github.com/ssb-ngi-pointer/ssb-db2/blob/master/operators/index.js
[JITDB operators]: https://github.com/ssb-ngi-pointer/jitdb#operators
