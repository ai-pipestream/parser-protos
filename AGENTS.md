# Agent rules for parser-protos

This repo is the wire contract for the PDF parser fleet: package
`ai.protomolt.parse.pdf.v1`, service `PdfBackendService` (Probe, Parse,
Render). It stands alone.

- **Not the pipestream platform.** This project has nothing to do with
  `/work/main/pipestream-ai` or the `pipestream-protos` repo. Never add this
  contract there, never import their protos here, never touch that checkout
  from this project. (History: the contract briefly lived in
  `pipestream-protos` as a `pdf-backend` module and was reverted on
  2026-09-04. Do not repeat that.)
- **Additive only, forever.** Never renumber, retype, rename, or remove a
  field, enum value, message, or RPC. New fields, families, and messages are
  always safe.
- **Twin copies.** gRParse `backends/pdf_backend_{types,service}.proto` must
  stay identical to the files here. A contract change lands in both, same
  bytes, then the three backend repos advance `PDF_PROTOS_COMMIT` and the
  per-file sha256 pins in their `CMakeLists.txt`.
- **Consumers:** gRParse (vendored copies), grpc-pdfium, grpc-qparse,
  grpc-poppler (pinned raw download from this repo at a commit).
- Remotes: `git.rokkon.com/ai-pipestream/parser-protos` (origin) and a
  GitHub mirror at `github.com/ai-pipestream/parser-protos`. Push to both;
  never force-push.
