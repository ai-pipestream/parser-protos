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
  (server stream), `GetServiceInfo`.

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
