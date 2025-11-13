- Feature Name: call-caching
- Start Date: 2025-10

# Summary

The Sprocket call caching feature enables `sprocket run` to skip executing
tasks that have been previously executed successfully and instead reuse the
last known outputs of the task.

# Motivation

Sprocket currently cannot resume a failed or canceled workflow, meaning that it
must re-execute every task that completed successfully on a prior run of that
workflow.

As tasks can be very expensive to execute, in terms of both compute resources
and time, this can be a major barrier to using Sprocket in the bioinformatics
space.

The introduction of caching task outputs so that they can be reused in lieu of
re-executing a task will allow Sprocket to quickly resume a workflow run and
reduce redundant execution.

# Cache Key

A task's cache key is derived from the task's document URI and the name of the
task.

This implies that the task's cache key is sensitive to the **WDL document being
moved** or the **task being renamed**; either would cause a cache miss.

The cache key will be calculated as a [Blake3][blake3] hash of:

1. The WDL document URI [string](#hashing-internal-strings).
2. The task identifier as a [string](#hashing-internal-strings). The task
   identifier is based on the name of the task and its scatter index (if
   present).

The result is a 32 byte Blake3 digest that can be represented as a
lowercase hexadecimal string, (e.g.
`295192ea1ec8566d563b1a7587e5f0198580cdbd043842f5090a4c197c20c67a`) for the
purpose of cache entry file names.

# Call Cache Directory

Once a cache key is calculated, it can be used to locate an entry within the
call cache directory.

The call cache directory may be configured via `sprocket.toml`:

```toml
[run.task]
cache_dir = "<path_to_cache>"
```

The default call cache location will be the user's cache directory joined with
`./sprocket/calls`.

The call cache directory will contain an empty `.lock` file that will be used
to acquire shared and exclusive file locks on the _entire_ call cache; the lock
file serves to coordinate access between `sprocket run` and a future `sprocket clean`
command.

During the execution of `sprocket run`, only a single _shared lock_ will be
acquired on the `.lock` file and kept for the entirety of the run.

The call cache will have no eviction policy, meaning it will grow unbounded. A
future `sprocket clean` command might give statistics of current cache sizes
with the option to clean them, if desired. The `sprocket clean` command would
take an _exclusive lock_ on the `.lock` file to block any `sprocket run`
command from executing while it is operating.

Each entry within the call cache will be a file with the same name of the
task's cache key.

During a lookup of an entry in the cache, a _shared lock_ will be acquired on
the individual entry file. During the updating of an entry in the cache, an
_exclusive lock_ will be acquired on the individual entry file.

The file will contain a JSON object with the following information:

```json
{
  "version": 1,                          // A monotonic version for the entry format.
  "command: "<string-digest>",           // The digest of the task's evaluated command.
  "container": "<string>",               // The container used by the task.
  "shell": "<string>",                   // The shell used by the task.
  "requirements": [
    "<key>": "<value-digest>",           // The requirement key and value digest
    ...
  ],
  "hints": [
    "<key>": "<value-digest>",           // The hint key and value digest
    ...
  ],
  "inputs": [
    "<path-or-url>": "<content-digest>", // The previous backend input and its content digest
    ...
  ],
  "exit": <num>,                         // The last exit code of the task.
  "stdout": {
    "location": "<path-or-url>",         // The location of the last stdout output.
    "digest": "<content-digest>".        // The content digest of the last stdout output.
  },
  "stderr": {
    "location": "<path-or-url>",         // The location of the last stderr output.
    "digest": "<content-digest>"         // The content digest of the last stderr output.
  },
  "work": {
    "location": "<path-or-url>",         // The location of the last working directory.
    "digest": "<content-digest>"         // The content digest of the last working directory.
  }
}
```

See the [section on cache entry digests](#cache-entry-digests) for information
on how the digests in the cache entry file are calculated.

# Call Cache Hit

Checking for a cache hit acquires a _shared lock_ on the call cache entry file.

A cache entry lookup only occurs for the _first_ execution of the task; the
call cache is skipped for subsequent retries of the task.

A call cache hit will occur if all of the following criteria are met:

* A file with the same name as the task's cache key is present in the call cache
  directory and the file can be deserialized to the expected JSON object.
* The cache entry's `version` field matches the cache version expected by
  Sprocket.
* The digest of the currently executing task's evaluated command matches the
  cache entry's `command` field.
* The container used by the task matches the cache entry's `container` field.
* The shell used by the task matches the cache entry's `shell` field.
* The digests of the task's requirements exactly match those in the cache
  entry's `requirements` field.
* The digests of the task's hints exactly match those in the cache entry's
  `hints` field.
* The digests of the task's backend inputs exactly match those in the cache
  entry's `inputs` field.
* The digest of the cache entry's `stdout` field matches the current digest of
  its location.
* The digest of the cache entry's `stdout` field matches the current digest of
  its location.
* The digest of the cache entry's `work` field matches the current digest of
  its location.

If any of the criteria above are not met, the failing criteria is logged (e.g.
"entry not present in the cache", "command was modified", "input was modified",
"stdout file was modified", etc.) and it is treated as a cache miss.

Upon a call cache hit, a `TaskExecutionResult` will be created from the
`stdout`, `stderr`, and `work` fields of the cache entry and task execution
will be skipped.

# Call Cache Miss

Upon a call cache miss, the task will be executed by passing the request to the
task execution backend.

After the task successfully executes _on its first attempt only_, the following
occurs:

* Content digests will be calculated for `stdout`, `stderr`, and `work` of the
  execution result returned by the execution backend.
* An _exclusive lock_ is acquired on the cache entry file.
* The new cache entry is JSON serialized into the cache entry file.

If a task fails to execute on its first attempt, the task's cache entry will
not be updated regardless of a successful retry.

***Note: a non-zero exit code of a task's execution is not inherently a failure
as the WDL task may specify permissible non-zero exit codes.***

# Cache Entry Digests

A cache entry may contain three different types of digests as lowercase
hexadecimal strings:

* A digest produced by [hashing an internal string](#hashing-a-string-value).
* A digest produced by [hashing a WDL value](#hashing-wdl-values).
* A [content digest](#content-digests) of a backend input.

[Blake3][blake3] will be used as the hash algorithm for producing the digests.

## Hashing Internal Strings

Hashing an internal string (i.e. a string used internally by the engine) will
update the hasher with:

1. A four byte length value in little endian order.
2. The UTF-8 bytes representing the string.

## Hashing WDL Values

For hashing the values of the `requirements` and `hints` section, a [Blake3][blake3]
hasher will be updated as described in this section.

Compound values will recursively hash their contained values.

### Hashing a `None` value

A `None` value will update the hasher with:

1. A byte with a value of `0` to indicate a `None` variant.

### Hashing a `Boolean` value

A `Boolean` value will update the hasher with:

1. A byte with a value of `1` to indicate a `Boolean` variant.
2. A byte with a value of `1` if the value is `true` or `0` if the value is
   `false`.

### Hashing an `Int` value

An `Int` value will update the hasher with:

1. A byte with a a value of `2` to indicate an `Int` variant.
2. An 8 byte value representing the signed integer in little endian order.

### Hashing a `Float` value

A `Float` value will update the hasher with:

1. A byte with a value of `3` to indicate a `Float` variant.
2. An 8 byte value representing the float in little endian order.

### Hashing a `String` value

A `String` value will update the hasher with:

1. A byte with a value of `4` to indicate a `String` variant.
2. The [internal string](#hashing-internal-strings) value of the `String`.

### Hashing a `File` value

A `File` value will update the hasher with:

1. A byte with a value of `5` to indicate a `File` variant.
2. The [internal string](#hashing-internal-strings) value of the `File`.

For the purpose of hashing a `File` value, the contents of the file specified
by the value are _not_ considered.

If the `File` is a backend input, the contents will be taken into consideration
when backend input content digests are produced.

### Hashing a `Directory` value

A `Directory` value will update the hasher with:

1. A byte with a value of `6` to indicate a `Directory` variant.
2. The [internal string](#hashing-internal-strings) value of the `Directory`.

For the purpose of hashing a `Directory` value, the contents of the directory
specified by the value are _not_ considered.

If the `Directory` is a backend input, the contents will be taken into
consideration when backend input content digests are produced.

### Hashing a `Pair` value

A `Pair` value will update the hasher with:

1. A byte with a value of `7` to indicate a `Pair` variant.
2. The recursive hash of the `left` [value](#hashing-wdl-values).
3. The recursive hash of the `right` [value](#hashing-wdl-values).

### Hashing an `Array` value

An `Array` value will update the hasher with:

1. A byte with a value of `8` to indicate an `Array` variant.
2. The [sequence](#hashing-sequences) of [elements](#hashing-wdl-values)
   contained in the array, in insertion order.

#### Hashing a `Map` value

A `Map` value will update the hasher with:

1. A byte with a value of `9` to indicate a `Map` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-wdl-values), [value](#hashing-wdl-values))
   pairs, in insertion order.

#### Hashing an `Object` value

An `Object` value will update the hasher with:

1. A byte with a value of `10` to indicate an `Object` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-internal-strings), [value](#hashing-wdl-values))
   pairs, in insertion order.

### Hashing a `Struct` value

A `Struct` value will update the hasher with:

1. A byte with a value of `11` to indicate a `Struct` variant.
2. The [sequence](#hashing-sequences) of ([field name](#hashing-internal-strings), [value](#hashing-wdl-values))
   pairs, in field declaration order.

### Hashing a `hints` value (WDL 1.2+)

A `hints` value will update the hasher with:

1. A byte with a value of `12` to indicate a `hints` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-internal-strings), [value](#hashing-wdl-values))
   pairs, in insertion order.

### Hashing an `input` value (WDL 1.2+)

An `input` value will update the hasher with:

1. A byte with a value of `13` to indicate an `input` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-internal-strings), [value](#hashing-wdl-values))
   pairs, in insertion order.

### Hashing an `output` value (WDL 1.2+)

An `output` value will update the hasher with:

1. A byte with a value of `14` to indicate an `output` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-internal-strings), [value](#hashing-wdl-values))
   pairs, in insertion order.

### Hashing Sequences

Hashing a _sequence_ will update the hasher with:

1. A four byte length value in little endian order.
2. The hash of each element in the sequence.

## Content Digests

As `wdl-engine` already calculates digests of files and directories for
uploading files to cloud storage, the call caching implementation will make use
of the existing content digest cache, with some improvements.

Keep in mind that `File` and `Directory` values may be either local file paths
(e.g. `/foo/bar.txt`) or remote URLs (e.g. `https://example.com/bar.txt`,
`s3://foo/bar.txt`, etc.).

### Local File Digests

Calculating the content digest of a local file is as simple as feeding every
byte of the file's contents to a [Blake3][blake3] hasher; functions that `mmap`
large files to calculate the digest will also be utilized.

### Remote File Digests

A `HEAD` request will be made for the remote file URL.

If the remote URL is for a supported cloud storage service, the response is
checked for the appropriate metadata header (e.g. `x-ms-meta-content_digest`,
`x-amz-meta-content-digest`, or `x-goog-meta-content-digest`) and the header is
treated like a [`Content-Digest`][content-digest] header.

Otherwise, the response must have either a [`Content-Digest`][content-digest]
header or a [strong `ETag`][etag] header. If the response does not have the
required header or if the header's value is invalid, it is treated as a failure
to calculate the content digest.

If the `HEAD` request is unsuccessful and the error is considered to be a
"transient" failure (e.g. a 500 response), the `HEAD` request is retried
internally up to some configurable limit. If the request is unsuccessful after
exhausting the retries, it is treated as a failure to calculate the content
digest.

If a `Content-Digest` header was returned, the hasher is updated with:

* A `0` byte to indicate the header was `Content-Digest`.
* The [algorithm string](#hashing-internal-strings) of the header.
* The [sequence of digest bytes](#hashing-sequences) of the header.

If an `ETag` header was returned, the hasher is updated with:

* A `1` byte to indicate the header was `ETag`.
* The [strong ETag header value string](#hashing-internal-strings).


Note that Sprocket will *not* verify that the content digest reported by the
header matches the actual content digest of the file as that requires
downloading the file's entire contents.

### Local Directory Digests

The content digest of a local directory is calculated by recursively walking
the directory in a consistent order and updating a Blake3 hasher based on each
entry of the directory.

A directory's entry is hashed with:

1. The [relative path](#hashing-internal-strings) of the directory entry.
2. A `0` byte if the entry is a file or `1` if it is a directory.
3. If the entry is a file, the hasher is updated with the contents of the file.

Finally, a four byte (little endian) entry count value is written to the hasher
before it is finalized to produce the 32 byte Blake3 content digest of the
directory.

***Note: it is an error if the directory contains a symbolic link to a
directory that creates a cycle (i.e. to an ancestor of the directory being
hashed).***

### Remote Directory Digests

`cloud-copy` has the facility to walk a "directory" cloud storage URL; it uses
the specific cloud storage API to list all objects that start with the
directory's prefix.

The content digest of a remote directory is calculated by using `cloud-copy` to
walk the directory in a consistent order and then updating a Blake3 hasher
based on each entry of the directory.

A directory's entry is hashed with:

1. The [relative path](#hashing-internal-strings) of the entry from the base
   URL.
2. The 32 byte Blake3 digest of the [remote file entry](#remote-file-digests).

Finally, a four byte (little endian) entry count value is written to the hasher
before it is finalized to produce the 32 byte Blake3 content digest of the
directory.

# Enabling Call Caching

A setting in `sprocket.toml` can control whether or not call caching is
enabled for every invocation of `sprocket run`:

```toml
[run.task]
cache = "off|on|explicit" # defaults to `off`
```

The supported values for the `cache` setting are:

* `off` - do not check the call cache or write new cache entries at all.
* `on` - check the call cache and write new cache entries for all tasks except
  those that have a `cacheable: false` hint.
* `explicit` - check the call cache and write new cache entries _only_ for
  tasks that have a `cacheable: true` hint.

Sprocket will default the setting to `off` as it safer to let users consciously
opt-in than potentially serve stale results from the cache without the user's
knowledge that call caching is occurring.

## Opting Out

When call caching has been enabled, users may desire to opt-out of call caching
for individual tasks or a single `sprocket run` invocation.

### Task Opt Out

An individual task may opt out of call caching through the use of the
`cacheable` hint:

```wdl
hints {
  "cacheable": false
}
```

The `cacheable` hint defaults to `false` if the `task.cache` setting is
`explicit`; otherwise, the hint defaults to `true`.

When `cacheable` is `false`, the call cache is not checked prior to task
execution and the result of the task's execution is not cached.

### Run Opt Out

A single invocation of `sprocket run` may pass the `--no-call-cache` option.

Doing so disables the use of the call cache for that specific run, both in
terms of looking up results and storing results in the cache.

# Failure Modes for Sprocket

Sprocket currently uses a _fail fast_ failure mode where Sprocket immediately
attempts to cancel any currently executing tasks and return the error it
encountered. This is also the behavior of the user invoking `Ctrl-C` to
interrupt evaluation.

Failing this way may cancel long-running tasks that would otherwise have
succeeded and subsequently prevent caching results for those tasks.

To better support call caching, Sprocket should be enhanced to support a _fail
slow_ failure mode (the new default), with users able to configure Sprocket to
use the previous _fail fast_ behavior when desired.

With a _fail slow_ failure mode, currently executing tasks are awaited to
completion, and their successful results are cached before attempting to abort
the run.

This also changes how Sprocket handles `Ctrl-C`. Sprocket should now support
multiple `Ctrl-C` invocations depending on its configured failure mode:

1. If the configured failure mode is _fail slow_, the user invokes `Ctrl-C` and
   Sprocket prints a message informing the user that it is waiting on
   outstanding task executions to complete and to hit `Ctrl-C` again to cancel
   tasks instead. It then proceeds to wait for executing tasks to complete to
   cache successful results.
2. The user invokes `Ctrl-C` and Sprocket prints a message informing the user
   that it is now canceling the executing tasks and to hit `Ctrl-C` again to
   immediately terminate Sprocket. It then proceeds to cancel the executing
   tasks and wait for the cancellations to occur.
3. The user invokes `Ctrl-C` and Sprocket immediately errors with a "evaluation
   aborted" error message.

The failure mode can be configured via `sprocket.toml`:

```toml
[run]
fail = "slow|fast"   # Defaults to `slow`
```

[wdl]: https://openwdl.org/
[blake3]: https://github.com/BLAKE3-team/BLAKE3
[hardlink]: https://en.wikipedia.org/wiki/Hard_link
[symboliclink]: https://en.wikipedia.org/wiki/Symbolic_link
[content-digest]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Digest
[etag]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/ETag
