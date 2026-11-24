# parser-protos

The wire contract for the interchangeable PDF backend services.

Package `ai.protomolt.parse.pdf.v1`:

- `proto/ai/protomolt/parse/pdf/v1/pdf_backend_types.proto` -- typed messages
  for every data family the engines emit (text cells, fonts, images, links,
  outline, annotations, forms, attachments, signatures, structure tree,
  vector shapes, metadata, encryption, graphics resources), plus per-document
  capabilities so absence is stated instead of inferred from an empty stream.
- `proto/ai/protomolt/parse/pdf/v1/pdf_backend_service.proto` --
  `PdfBackendService`: `Probe`, `Parse` (server stream), `Render`
  (server stream), `GetServiceInfo`, and `ParseStream` (bidirectional:
  chunked upload, then parse and render requests on the same stream).

## Content-addressed document handshake

Callers like a consensus-mode orchestrator hit every backend once per page,
which would re-upload the full document bytes inside `PdfDocument.data` on
every call. The handshake avoids that: the client sends
`PdfDocument.sha256` (lowercase hex SHA-256 of `data`) and the bytes only
when the server does not already have them.

Flow:

1. First call for a document: `data` populated, `sha256` set. The server
   verifies the hash, may cache the bytes under it, and proceeds.
2. Later calls (`Probe`, `Parse`, `Render` alike): `data` empty, `sha256`
   set. The server serves the request from its cache; a server that has
   never seen those bytes answers `LOAD_STATUS_BYTES_REQUIRED` in the typed
   load status (`BackendCapabilities.load_status` for `Probe` and the
   `Parse` header, `RenderResponse.head.load_status` for `Render`).
3. `LOAD_STATUS_BYTES_REQUIRED` is a cache miss verdict, not a document
   defect. The client retries exactly once with `data` populated before
   treating it as an error.
4. `LOAD_STATUS_HASH_MISMATCH` means `data` and `sha256` were both present
   and the bytes do not hash to the given value; that is a client bug, not
   a retry case.

`data` empty with `sha256` absent is invalid (`INVALID_ARGUMENT`). Cache
bounds, eviction, and lifetime are server-private: the contract promises
only the verdict, never that bytes are retained.

## ParseStream

`ParseStream` serves a whole document session on one bidirectional stream.
The bytes travel as `PdfChunk` messages, so a PDF is not capped by the
message size limit, and a client can send them while it still reads its
responses.

1. The client sends `ParseStreamOpen` (families, page range and options for
   the open's own parse, password, `doc_id`, optional `sha256` and
   `declared_size`, and `header_only` for a session that asks for pages
   later), then `PdfChunk` messages in order (`offset` is the bytes sent
   before the chunk) until one has `last` set.
2. Once the upload is complete the server answers the open as request 0:
   a `ParseResponse` stream exactly as `Parse` would give it, header first
   (header alone with `header_only`). That header is the first message of
   the stream; a `sha256` the bytes do not match is
   `LOAD_STATUS_HASH_MISMATCH` there.
3. `ParsePages` and `RenderPages` carry a client-chosen nonzero
   `request_id` and may be sent at any time after the open, mid-upload
   included. They are answered like `Parse` and `Render` on the stream's
   bytes, every response message tagged with the `request_id`. A server
   may serve several at once, so messages of different requests can
   interleave; within one request they keep the unary RPC's order.
4. Every request ends with a `RequestDone` whose `RequestOutcome` says how
   (`OK`, `CANCELLED`, `INVALID_ARGUMENT` where the unary RPC would refuse
   the request, `NOT_LOADED` after a failed open, ...). A bad page request
   fails that request, not the stream. Load failures stay typed in the
   header or render head, as on the unary RPCs.
5. `Cancel` with a `request_id` stops that request; `Cancel` without one
   stops the stream. The stream ends with one `ParseStreamDone`
   (`bytes_received`, the server's `sha256` of them, `cancelled`) after the
   client half-closes, or right after a stream-wide `Cancel`.

Protocol violations (no open first, a second open, chunks out of order or
after the last, a half-close before the last chunk, a zero or reused
`request_id`, an upload past `declared_size` or the server's size limit)
fail the stream with a gRPC status. `Probe`, `Parse` and `Render` stay; a
server that predates `ParseStream` answers `UNIMPLEMENTED`, and a client
falls back to them.

## Warnings

An engine often reads a damaged document as far as it can: it rebuilds a
cross-reference table, skips a stream that does not decode, draws a page
without an image it could not read. The result is not the whole document,
and the contract has one message for saying so, `ParseWarning` (the page
it is about, the family it is about, the engine's message):

- `Parse` carries them in `ParseTrailer.warnings`.
- `Render` carries them in `RenderResponse.warnings`, on the message of
  the raster they are about. A page the server cannot draw at all still
  fails the call with a gRPC status.

A server that predates `RenderResponse.warnings` sends none, so an empty
list from an older build does not say the page was clean.

## Service info and the demo shell

`GetServiceInfo` reports the backend's identity (`backend_name`,
`engine_version`, `build_version`) without loading a document, so a caller
can label a backend before ever probing one. The response also carries the
`UiInfo` block (`title`, `path`, `description`) every service in the
ai-pipestream grpc-services family advertises, letting the demo shell
(`gRParse/examples/web-demo`) mount the service's web UI under a tab. The
block has the same shape in every repo; keep it that way.

## Consumers

- [gRParse](https://git.rokkon.com/ai-pipestream/gRParse) builds from its
  vendored copies in `backends/`; those files and this repo must stay
  byte-identical.
- [grpc-pdfium](https://git.rokkon.com/ai-pipestream/grpc-pdfium),
  [grpc-qparse](https://git.rokkon.com/ai-pipestream/grpc-qparse), and
  [grpc-poppler](https://git.rokkon.com/ai-pipestream/grpc-poppler) download
  the two files from this repo at a commit pinned in each `CMakeLists.txt`
  (`PDF_PROTOS_COMMIT`), verified by sha256.

## Rules

The contract is additive only: never renumber, retype, or remove a field,
value, or RPC. New fields, families, and messages are always safe to add.
When the contract changes, land the same bytes here and in gRParse
`backends/`, then advance `PDF_PROTOS_COMMIT` (and the two file hashes) in
the three backend repos.
