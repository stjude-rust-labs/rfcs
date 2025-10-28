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

To be able to cache the result of a [WDL][wdl] task's execution, we must define
what the _cache key_ of a task is.

A task's cache key is derived from a set of properties that, if altered, may
affect the task's execution and thus the outputs produced by the task.

This feature assumes that, for a given command and execution environment, a
task's execution will produce ***functionally equivalent*** outputs. This means
that while the outputs are not required to be byte-for-byte equivalent, they
are required to not alter the evaluation of any downstream tasks that are
dependent on those outputs. As this assumption does not hold true for every
task, individual tasks may need to [opt-out of call caching](#opting-out) via a
`volatile` hint to Sprocket.

In WDL, the following properties are used for deriving a task's cache key:

* The evaluated `command` section text.
* The evaluated `requirements` and `hints` (or `runtime` for WDL before 1.2)
  section values.
* The value of the task's "container" that is determined by:
  1. The `container` (i.e. `docker`) requirement,  if present.
  2. The value of the `task.container` setting provided to Sprocket, if present.
  3. Finally, the built-in default which is currently `ubuntu:latest`.
* The value of the task's "shell" that is determined by:
  1. `task.shell` setting provided to Sprocket, if present.
  2. The built-in default (`bash`).
* The _content_ of input files and directories provided to the task's execution
  environment (i.e. execution backend inputs).

Note that the following explicitly ***do not*** contribute to the identity of a
task for the purpose of call caching:

* The WDL identifier of the task (i.e. its "name")
* The individual `input` section values; an input value either contributes to
  the evaluated `command` section text, a requirement or hint value, or ends up
  as _content_ to the execution environment.
* The WDL document source itself, with the exception of any changes that might
  alter the above cache key properties after evaluation.

Note that the `container` used by the task may have a mutable tag on it (e.g.
`latest`) and therefore may change underneath call caching. As Sprocket will
not be resolving the referenced tag, a change to the tag or the image
repository used will not bust the cache.

For this implementation of call caching, the cache key will be a 32 byte
[Blake3][blake3] digest as a lowercase hexadecimal string, (e.g.
`295192ea1ec8566d563b1a7587e5f0198580cdbd043842f5090a4c197c20c67a`).

## Cache Key Calculation

The calculation of a task's cache key should occur just prior to where Sprocket
sends the task to the execution backend; after the key is calculated, the call
cache should be checked and the cached outputs reused if there was a [cache hit](#call-cache-hit).

To calculate the cache key, a new [Blake3][blake3] hasher will be created and
updated with the following:

1. The evaluated command text [string](#hashing-rust-strings).
2. The [sequence](#hashing-sequences) of ([key](#hashing-rust-strings), [value](#hashing-wdl-values))
   pairs from the `requirements` section, ordered lexicographically by key.
3. The [sequence](#hashing-sequences) of ([key](#hashing-rust-strings), [value](#hashing-wdl-values))
   pairs from the `hints` section, ordered lexicographically by key.
4. The calculated `container` [string](#hashing-rust-strings).
5. The calculated `shell` [string](#hashing-rust-strings).
6. The [sequence](#hashing-sequences) of ([location](#hashing-rust-strings), [content digest](#calculating-content-digests))
   pairs of the backend inputs, ordered lexicographically by input location
   (i.e. absolute path or URL).

The hasher will then be finalized to produce a Blake3 digest representing the
cache key of the task.

_Note: a failure to calculate a cache key will result in an automatic cache
miss and no cache entry created upon a successful task execution; the error
that caused the failure to calculate the cache key will be logged._

### Hashing Rust Strings

Hashing a Rust string will update the hasher with:

1. A four byte length value in little endian order.
2. The UTF-8 bytes representing the string.

### Hashing Sequences

Hashing a _sequence_ will update the hasher with:

1. A four byte length value in little endian order.
2. The hash of each element in the sequence.

### Hashing WDL Values

For hashing the values of the `requirements` and `hints` section, a [Blake3][blake3]
hasher will be updated as described in this section.

Compound values will recursively hash their contained values.

#### Hashing a `Boolean` value

A `Boolean` value will update the hasher with:

1. A byte with a value of `0` to indicate a `Boolean` variant.
2. A byte with a value of `1` if the value is `true` or `0` if the value is
   `false`.

#### Hashing an `Int` value

An `Int` value will update the hasher with:

1. A byte with a a value of `1` to indicate an `Int` variant.
2. An 8 byte value representing the signed integer in little endian order.

#### Hashing a `Float` value

A `Float` value will update the hasher with:

1. A byte with a value of `2` to indicate a `Float` variant.
2. An 8 byte value representing the float in little endian order.

#### Hashing a `String` value

A `String` value will update the hasher with:

1. A byte with a value of `3` to indicate a `String` variant.
2. The [string](#hashing-rust-strings) value of the `String`.

#### Hashing a `File` value

A `File` value will update the hasher with:

1. A byte with a value of `4` to indicate a `File` variant.
2. The [string](#hashing-rust-strings) value of the `File`.

#### Hashing a `Directory` value

A `Directory` value will update the hasher with:

1. A byte with a value of `5` to indicate a `Directory` variant.
2. The [string](#hashing-rust-strings) value of the `Directory`.

#### Hashing a `Pair` value

A `Pair` value will update the hasher with:

1. A byte with a value of `6` to indicate a `Pair` variant.
2. The recursive hash of the `left` [value](#hashing-wdl-values).
3. The recursive hash of the `right` [value](#hashing-wdl-values).

#### Hashing an `Array` value

An `Array` value will update the hasher with:

1. A byte with a value of `7` to indicate an `Array` variant.
2. The [sequence](#hashing-sequences) of [elements](#hashing-wdl-values)
   contained in the array, in insertion order.

#### Hashing a `Map` value

A `Map` value will update the hasher with:

1. A byte with a value of `8` to indicate a `Map` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-wdl-values), [value](#hashing-wdl-values))
   pairs, in insertion order.

#### Hashing an `Object` value

An `Object` value will update the hasher with:

1. A byte with a value of `9` to indicate an `Object` variant.
2. The [sequence](#hashing-sequences) of ([key](#hashing-rust-strings), [value](#hashing-wdl-values))
   pairs, ordered lexicographically by key.

#### Hashing a `Struct` value

A `Struct` value will update the hasher with:

1. A byte with a value of `10` to indicate a `Struct` variant.
2. The [sequence](#hashing-sequences) of ([field name](#hashing-rust-strings), [value](#hashing-wdl-values))
   pairs, ordered lexicographically by field name.

### Calculating Content Digests

As `wdl-engine` already calculates digests of files and directories for
uploading files to cloud storage, the call caching implementation will make use
of the existing content digest cache, with some improvements.

Keep in mind that `File` and `Directory` values may be either local file paths
(e.g. `/foo/bar.txt`) or remote URLs (e.g. `https://example.com/bar.txt`,
`s3://foo/bar.txt`, etc.).

#### Local File Digests

Calculating the content digest of a local file is as simple as feeding every
byte of the file's contents to a [Blake3][blake3] hasher; functions that `mmap`
large files to calculate the digest will also be utilized.

#### Remote File Digests

A `HEAD` request will be made for the remote file URL.

If a successful response contains a `x-digest-blake3` header, the header's
value will be used as the digest of the file's content.

If the request is unsuccessful and the error is considered to be a "transient"
failure (e.g. a 500 response), the `HEAD` request is retried internally up to
some configurable limit. If the request is unsuccessful after exhausting the
retries, the cache key will fail to calculate and result will be an automatic
cache miss.

If the response is missing the header, the empty Blake3 digest will be used for
the file's content; Sprocket will not attempt to download the file to calculate
its content digest.

As part of this implementation, `cloud-copy` will be enhanced to automatically
attach the `x-digest-blake3` header as metadata to the cloud storage objects it
creates.

That means that content digests will only be respected for:

* Files uploaded by Sprocket to cloud storage.
* Files uploaded by Planetary to cloud storage.
* Files manually uploaded by the `cloud-copy` tool to cloud storage.

Note that Sprocket will *not* verify that the `x-digest-blake3` header matches
the actual content digest of the file in cloud storage. However, as a `PUT` of
an object overwrites existing metadata, the header should only be inaccurate if
a user explicitly attaches an incorrect header to an object that was manually
uploaded.

#### Local Directory Digests

The content digest of a local directory is calculated by walking the directory
in a consistent order and updating a Blake3 hasher based on each entry of the
directory.

A directory's entry is hashed with:

1. The [basename](#hashing-rust-strings) of the directory entry.
2. The 32 byte Blake3 digest of the entry; if the entry is itself a directory, a
   recursion takes place based on the path to the sub-directory.

Finally, a four byte (little endian) entry count value is written to the hasher
before it is finalized to produce the 32 byte Blake3 content digest of the
directory.

***Note: symbolic links to directories within a directory input will not be
supported for call caching and be treated as an error.***

#### Remote Directory Digests

`cloud-copy` has the facility to walk a "directory" cloud storage URL; it uses
the specific cloud storage API to list all objects that start with the
directory's prefix.

The content digest of a remote directory is calculated by using `cloud-copy` to
walk the directory in a consistent order and then updating a Blake3 hasher
based on each entry of the directory.

A directory's entry is hashed with:

1. The [relative path](#hashing-rust-strings) of the entry from the base URL.
2. The 32 byte Blake3 digest of the entry, which is returned via the proposed
   `x-digest-blake3` header from a `HEAD` request for the file or the empty
   digest if the response header is missing.

Finally, a four byte (little endian) entry count value is written to the hasher
before it is finalized to produce the 32 byte Blake3 content digest of the
directory.

# Call Cache Directory

Once a cache key is calculated, it can be used to locate an entry within the
call cache directory.

The call cache directory may be configured via `sprocket.toml`:

```toml
[run.task]
call_cache = "<path_to_cache>"
```

The default call cache location will be the user's cache directory joined with
`./sprocket/calls`.

The call cache will have no eviction policy, meaning it will grow unbounded. A
future `sprocket cleanup` command might give statistics of current cache sizes
with the option to clean them, if desired.

The call cache directory will contain an empty `.lock` file that will be used
to acquire shared and exclusive file locks on the call cache; the lock file
serves to coordinate access between concurrent `sprocket` processes.

Each entry within the call cache will be a file with the same name of the
task's cache key.

The file will contain a JSON object with the following information:

```json
{
  "version": 1,                     // A monotonic version for the file format.
  "exit": <num>,                    // The last exit code of the task.
  "stdout": {
    "location": "<path-or-url>",    // The location of the last stdout output.
    "digest": "<content-digest>".   // The content digest of the last stdout output.
  },
  "stderr": {
    "location": "<path-or-url>",    // The location of the last stderr output.
    "digest": "<content-digest>"    // The content digest of the last stderr output.
  },
  "work": {
    "location": "<path-or-url>",    // The location of the last working directory.
    "digest": "<content-digest>"    // The content digest of the last working directory.
  }
}
```

## Call Cache Hit

If cache key calculation fails for any reason (e.g. a network error), the error
will be logged and the result will be an automatic cache miss. Task evaluation
will not fail due to an inability to calculate a cache key.

Checking for a cache hit acquires a shared lock on the call cache directory.

A call cache hit will occur if all of the following criteria are met:

* A file with the same name as the task's cache key is present in the call cache
  directory and the file can be deserialized to the expected JSON object.
* The calculated current digest of the location matches the digest value from
  the cache.

If any of the criteria above are not met, the failing criteria is logged (e.g.
"task not present in the cache", "output X no longer exists", "output X digest
mismatch", etc.) and it is treated like a cache miss.

Upon a call cache hit, a `TaskExecutionResult` will be created from the values
read from the cache and task execution will be skipped.

## Call Cache Miss

Upon a call cache miss, the task will be executed by passing the request to the
task execution backend.

After the task successfully executes (including after retries), the following
occurs:

* Content digests will be calculated for `stdout`, `stderr`, and `work` of the
  execution result returned by the execution backend.
* An exclusive lock is acquired for inserting the execution result into the
  cache.
* The execution result is JSON serialized into the call cache entry for the
  task.

If a task fails to execute after exhausting retries, the cache will not be
updated and the error returned as the result of task evaluation.

In the unlikely event that a scattered task's evaluation all use the same cache
key, the last successful execution result will be the one to be stored in the
cache.

***Note: a non-zero exit code of a task's execution is not inherently a failure
as the WDL task may specify permissible non-zero exit codes.***

# Opting Out

Users may desire to opt-out of call caching for individual tasks, a single
`sprocket run` invocation, or to disable call caching for every `sprocket run`
invocation.

## Task Opt Out

An individual task may opt out of call caching through the use of the `volatile`
hint:

```wdl
hints {
  "volatile": true  # Defaults to `false`
}
```

When `volatile` is true, the call cache is not checked prior to task execution
and the result of the task's execution is not cached.

## Run Opt Out

A single invocation of `sprocket run` may pass the `--no-call-cache` option.

Doing so disables the use of the call cache for that specific run, both in
terms of looking up results and storing results in the cache.

## Disabling Call Caching

A setting in `sprocket.toml` can control whether or not call caching is
disabled for every invocation of `sprocket run`:

```toml
[run.task]
use_call_cache = false   # Defaults to `true`
```

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
