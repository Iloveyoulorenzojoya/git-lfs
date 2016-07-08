# Adding Custom Transfer Agents to LFS

## Introduction

Git LFS supports multiple ways to transfer (upload and download) files. In the
core client, the basic way to do this is via a one-off HTTP request via the URL
returned from the LFS API for a given object. The core client also supports
extensions to allow resuming of downloads (via `Range` headers) and uploads (via
the [tus.io](http://tus.io) protocol).

Multiple transfer approaches are supported by the client including in the LFS
API request a list of transfer types it can support, in order of preference.
When replying, the API server will pick the first one of these it supports, and
make any necessary adjustments to the returned object actions so they will work
with that transfer type.

## Custom Transfer Types

Some people might want to be able to transfer content in other ways, however.
To enable this, git-lfs has an option to configure Custom Transfers, which are
simply processes which must adhere to the protocol defined later in this
document. git-lfs will invoke the process at the start of all transfers, 
and will communicate with the process via stdin/stdout for each transfer.

## Configuration

A custom transfer process is defined under a settings group called
`lfs.customtransfer.<name>`, where <name> is an identifier (see
[Naming](#naming) below).

* `lfs.customtransfer.<name>.path`

  `path` should point to the process you wish to invoke. This will be invoked at
  the start of all transfers (possibly many times, see the 'concurrent' option
  below) and the protocol over stdin/stdout is defined below in the
  [Protocol](#protocol) section.
  
* `lfs.customtransfer.<name>.args`

  If the custom transfer process requires any arguments, these can be provided
  here. Typically you would only need this if your process was multi-purpose or
  particularly flexible, most of the time you won't need it.

* `lfs.customtransfer.<name>.priority`

  Optional relative priority if there are multiple custom transfers defined.
  This merely affects the order they are listed in the call to the LFS API, 
  and the server will pick the first one it supports. A lower number is a higher
  priority (default 5).

* `lfs.customtransfer.<name>.concurrent`

  If true (the default), git-lfs will invoke the custom transfer process
  multiple times in parallel, according to `lfs.concurrenttransfers`, splitting
  the transfer workload between the processes.

  If you would prefer that only one instance of the transfer process is invoked,
  maybe because you want to do your own parallelism internally (e.g. slicing 
  files into parts), set this to false.

* `lfs.customtransfer.<name>.direction`

  Specifies which direction the custom transfer process supports, either 
  "download", "upload", or "both". The default if unspecified is "both".

## Naming

Each custom transfer must have a name which is unique to the underlying
mechanism, and the client and the server must agree on that name. The client
will advertise this name to the server as a supported transfer approach, and if
the server supports it, it will return relevant object action links. Because
these may be very different from standard HTTP URLs it's important that the
client and server agree on the name.

For example, let's say I've implemented a custom transfer process which uses
NFS. I could call this transfer type 'nfs' - although it's not specific to my
configuration exactly, it is specific to the way NFS works, and the server will
need to give me different URLs. Assuming I define my transfer like this, and the
server supports it, I might start getting object action links back like
`nfs://<host>/path/to/object`

## Protocol

The git-lfs client communicates with the custom transfer process via the stdin
and stdout streams. No file content is communicated on these streams, only
request / response metadata. The metadata exchanged is always in JSON format.
External files will be referenced when actual content is exchanged.

### Line Delimited JSON 
Because multiple JSON messages will be exchanged on the same stream it's useful
to delimit them explicitly rather than have the parser find the closing `}` in
an arbitrary stream, therefore each JSON structure will be sent and received on
a **single line** as per [Line Delimited
JSON](https://en.wikipedia.org/wiki/JSON_Streaming#Line_delimited_JSON_2).

In other words when git-lfs sends a JSON message to the custom transfer it will
be on a single line, with a line feed at the end. The transfer process must
respond the same way by writing a JSON structure back to stdout with a single
line feed at the end (and flush the output).

### Protocol Stages

The protocol consists of 3 stages:

#### Stage 1: Intiation

Immediately after invoking a custom transfer process, git-lfs sends initiation
data to the process over stdin. This tells the process useful information about
the configuration.

The message will look like this:
```json
{ "operation":"download", "concurrent": true, "concurrenttransfers": 3 }
```

* `operation`: will be "upload" or "download" depending on transfer direction
* `concurrent`: reflects the value of `lfs.customtransfer.<name>.concurrent`, in
  case the process needs to know
* `concurrenttransfers`: reflects the value of `lfs.concurrenttransfers`, for if
  the transfer process wants to implement its own concurrency and wants to
  respect this setting.

The transfer process should use the information it needs from the intiation
structure, and also perform any one-off setup tasks it needs to do. It should
then respond on stdout with a simple confirmation structure, as follows:

```json
{ "error": null }
```

Or if there was an error:

```json
{ "error": { "code": 32, "message": "Some init failure message" } }
```

#### Stage 2: 0..N Transfers

After the initiation exchange, git-lfs will send any number of transfer 
requests to the stdin of the transfer process. 

##### Uploads

For uploads the request sent from git-lfs to the transfer process will look
like this:

```json
{ "oid": "bf3e3e2af9366a3b704ae0c31de5afa64193ebabffde2091936ad2e7510bc03a", "size": 346232, "path": "/path/to/file.png" }
```

* `oid`: the identifier of the LFS object
* `size`: the size of the LFS object
* `path`: the file which the transfer process should read the upload data from

The transfer process should post one or more [progress messages](#progress) and 
then a final completion message as follows:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "error": null}
```

* `oid`: the identifier of the LFS object
* `error`: Should be null if there was no error

Or if there was an error in the transfer:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "error": { "code": 2, "message": "Explain what happened to this transfer" }}
```

##### Downloads

For downloads the request sent from git-lfs to the transfer process will look
like this:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "size": 21245 }
```

* `oid`: the identifier of the LFS object
* `size`: the size of the LFS object

Note there is no file path included in the download request; the transfer 
process should create a file itself and return the path in the final response
after completion (see below).

The transfer process should post one or more [progress messages](#progress) and 
then a final completion message as follows:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "path": "/path/to/file.png", "error": null}
```

* `oid`: the identifier of the LFS object
* `path`: the path to a file containing the downloaded data, which the transfer
  process reliquishes control of to git-lfs. git-lfs will move the file into LFS
  storage.
* `error`: Should be null if there was no error

Or, if there was a failure transferring this item:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "error": { "code": 2, "message": "Explain what happened to this transfer" }}
```

Errors for a single transfer request should not terminate the process. The error
should be returned in the response structure instead.

##### Progress

In order to support progress reporting while data is uploading / downloading, 
the transfer process should post messages to stdout as follows before sending
the final completion message:

```json
{ "oid": "22ab5f63670800cc7be06dbed816012b0dc411e774754c7579467d2536a9cf3e", "bytesSoFar": 1234, "bytesSinceLast": 64 }
```

* `oid`: the identifier of the LFS object
* `bytesSoFar`: the total number of bytes transferred so far
* `bytesSinceLast`: the number of bytes transferred since the last progress 
  message

The transfer process should post these messages such that the last one sent
has `bytesSoFar` equal to the file size on success.

#### Stage 3: Finish & Cleanup

When all transfers have been processed, git-lfs will send the following message
to the stdin of the transfer process:

```json
{ "complete": true }
```

On receiving this message the transfer process should clean up and terminate.
No response is expected.

## Error handling

Any unexpected fatal errors in the transfer process (not errors specific to a
transfer request) should set the exit code to non-zero and print information to 
stderr. Otherwise the exit code should be 0 even if some transfers failed.



