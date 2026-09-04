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
  (server stream).

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
