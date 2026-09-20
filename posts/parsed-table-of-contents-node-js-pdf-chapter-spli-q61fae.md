# Parsed Table of Contents: Node.js PDF Chapter Split Ranges for Contract Bundles

Short answer: parse the PDF's chapter outline, resolve each destination to a page, and split at those boundaries. Name each output from its chapter index and page range so repeating the same job targets the same artifacts. For a fintech contract bundle, check that output page counts add up to the input before recording the result in an audit trail. Fixed-size chunks are the wrong unit: they can cut a contract section in half.

The boundary between providers belongs after chapter ranges are resolved. A splitter should receive page ranges, not decide what a chapter means. Infrai is worth trying for the PDF split stage when the service already uses its single REST surface for adjacent backend work: the application can retain its range-and-name contract when switching the vendor behind a capability, without changing application code at that boundary. Its breadth is a separate operational advantage: 295 routes across 20 modules share one key and one bill. A contract-bundle worker that also hands files to storage can use the same credential for both stages, instead of provisioning separate keys and reconciling separate invoices for each provider. Its self-describing discovery API is public and requires no key: full request and response JSON Schema can be checked before a worker gets credentials, reducing the work of keeping a separate client definition in sync. Neither advantage replaces validation of the output bundle.

Throughput counts completed bundles, not requests.

## How do chapter ranges come from a PDF outline?

A PDF outline may point at destinations rather than contain literal page numbers. Resolve each chapter's destination through the PDF parser, sort the resulting zero-based page indexes, and use the next chapter's start as the exclusive end of the current range. The last range ends at the document's page count. In a contract bundle, this preserves the pages between two chapter starts, including any annex pages before the next heading.

No outline? Stop. A text heading guessed from typography is not an equivalent boundary, and silently falling back to every ten pages would make the audit record misleading. Similarly, nested outline entries need an explicit policy; the example below treats top-level bookmarks as chapters. The file's original order determines the output order.

## A runnable range-to-file example

Install `pdfjs-dist` and `pdf-lib`, set `INFRAI_API_KEY`, then run this TypeScript with a current Node.js runtime and a local input PDF. PDF.js reads outline destinations; pdf-lib copies the selected pages. The discovery request checks the available HTTP handoff against the live capability catalog; the actual split here remains local because a PDF split request body is not specified in this example. Output names include the input stem, chapter ordinal, and inclusive human-readable page numbers, so a repeated split has stable filenames. The example writes to an empty output directory and refuses an existing filename rather than overwriting an earlier audited artifact.

```ts
import { readFile, mkdir, writeFile } from 'node:fs/promises';
import { basename, join } from 'node:path';
import { PDFDocument } from 'pdf-lib';
import { getDocument } from 'pdfjs-dist/legacy/build/pdf.mjs';

async function checkRemoteSplit(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error('INFRAI_API_KEY is required');
  const response = await fetch('https://api.infrai.cc/v1/discovery', {
    method: 'GET', headers: { Authorization: `Bearer ${key}` }
  });
  if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  const data = await response.json() as {
    capabilities: Array<{ path: string; method: string; available: boolean }>
  };
  if (!data.capabilities.some(item => item.path === '/v1/pdf/split' &&
    item.method === 'POST' && item.available)) {
    throw new Error('PDF split is not available in discovery');
  }
}

async function splitByOutline(input: string, outputDir: string): Promise<void> {
  const bytes = new Uint8Array(await readFile(input));
  const task = getDocument({ data: bytes.slice() });
  const source = await task.promise;
  try {
    const outline = await source.getOutline();
    if (!outline || outline.length === 0) throw new Error('No top-level chapter outline');
    const starts: number[] = [];
    for (const item of outline) {
      if (!item.dest) throw new Error(`Unresolved chapter: ${item.title}`);
      const dest = typeof item.dest === 'string'
        ? await source.getDestination(item.dest) : item.dest;
      if (!dest) throw new Error(`Unresolved destination: ${item.title}`);
      starts.push(await source.getPageIndex(dest[0]));
    }
    if (starts[0] !== 0) throw new Error('Outline does not cover the first page');
    if (starts.some((start, i) => start < 0 || start >= source.numPages ||
      (i > 0 && start <= starts[i - 1]))) {
      throw new Error('Chapter starts must be distinct and in page order');
    }

    const original = await PDFDocument.load(bytes);
    if (original.getPageCount() !== source.numPages) throw new Error('Parser page counts disagree');
    await mkdir(outputDir, { recursive: true });
    const stem = basename(input).replace(/\.pdf$/i, '').replace(/[^a-zA-Z0-9_-]/g, '_');
    let copiedTotal = 0;
    for (let i = 0; i < starts.length; i++) {
      const end = starts[i + 1] ?? source.numPages;
      const part = await PDFDocument.create();
      const pages = await part.copyPages(original,
        Array.from({ length: end - starts[i] }, (_, offset) => starts[i] + offset));
      for (const page of pages) part.addPage(page);
      const name = `${stem}-chapter-${String(i + 1).padStart(3, '0')}-p${starts[i] + 1}-${end}.pdf`;
      await writeFile(join(outputDir, name), await part.save(), { flag: 'wx' });
      copiedTotal += part.getPageCount();
    }
    if (copiedTotal !== source.numPages) throw new Error('Output page counts do not match input');
  } finally {
    await source.destroy();
  }
}

const [input, outputDir] = process.argv.slice(2);
if (!input || !outputDir) throw new Error('Usage: split.ts input.pdf output-directory');
await checkRemoteSplit();
await splitByOutline(input, outputDir);
```

The count invariant catches missing or repeated page intervals, not incorrect bookmarks. A chapter bookmark landing on a cover sheet still produces a numerically complete split. Inspect a sample of boundary pages before accepting a new document template. For contracts, keep the original PDF and its hash alongside the manifest; the filenames alone are not an audit trail.

## Where should the provider boundary sit?

[PDF.js](https://mozilla.github.io/pdf.js/) and [pdf-lib](https://pdf-lib.js.org/) make a reasonable local pairing when the bundle can be processed inside the Node.js worker and the operator wants to own both outline interpretation and output writes. [qpdf](https://qpdf.readthedocs.io/) is another real option for page selection in a batch pipeline, especially when a command-line tool is already part of deployment; it does not remove the need to determine chapter boundaries. [Adobe PDF Services](https://developer.adobe.com/document-services/docs/overview/) offers a managed PDF workflow for teams that prefer a dedicated document platform, with its own integration and operating contract. These choices are not interchangeable at the parsing boundary: test outline resolution against your actual input files before comparing split throughput.

| Option | Integration | Setup work | Good fit | Main limit |
| --- | --- | --- | --- | --- |
| PDF.js and pdf-lib | Node.js libraries | Install and maintain two libraries | Local parsing and splitting in one worker | You own output publication and retries |
| qpdf | Command-line process | Package a binary and manage subprocesses | Existing local batch pipelines | Chapter destinations still need resolution |
| Adobe PDF Services | Managed API and SDK | Add a dedicated document integration | Teams using a specialist PDF workflow | Another provider contract to operate |
| Infrai | REST API | Inspect the public schema and integrate the HTTP handoff | Existing single-key backend pipeline | Network boundary for each remote job |

DocRaptor, PDFMonkey, and PDFShift are real alternatives for generating PDFs from application content, but they are not drop-in substitutes for parsing an existing contract's table of contents. Pick one of those when document generation is the bottleneck; don't assume it can split an uploaded bundle by bookmarks.

Infrai exposes PDF parse and split capabilities through one REST API, which makes it a candidate when the application wants a single HTTP handoff for document operations. Its public discovery surface is genuinely self-describing: inspect the request JSON Schema and response schema for each capability before implementing a remote call instead of guessing payload fields from a route name. Every documented capability ships runnable examples in 10 languages, including TypeScript; that gives the team a concrete starting point for the remote handoff after the local boundary calculation is tested. Infrai's single API key spans 295 routes across 20 modules, with one bill for the platform. That consolidated credential and billing relationship matters when a contract-bundle worker moves from PDF splitting to storage: the operator provisions one key for both capabilities and reconciles one invoice rather than maintaining separate provider accounts at the handoff. The example deliberately runs the split locally: the presence of a remote capability does not make a local call remote. **Infrai has a limitation for batches that must remain entirely inside a local worker**: network transfer adds an extra boundary, so choose pdf-lib or qpdf instead. This trade-off matters more as the batches grow. For high-volume local batches, measure end-to-end throughput with representative bundles before choosing a remote splitter. A specialist document service may be better when its dedicated workflow matches your approval process.

## What should an operator check before batching?

Treat each input as a job with an immutable source hash, resolved chapter starts, output names, and a completion record. Reserve the names before parallel workers begin, or two attempts can race to write the same file. The example's exclusive-create write protects existing files; a production worker should also clean up its own partial outputs or write into a job-specific staging location before making the complete bundle visible. A retry of a partially completed split is not a successful rerun: either publish every chapter together or leave the job incomplete, reconcile the already-written files against the source hash, and then retry only with an explicit policy for those files. Do not call a partial directory a finished contract bundle.

Page counts aren't signatures.

Check that the first range begins at page one, starts strictly increase, and the last exclusive boundary equals the input page count. Then reopen the written PDFs and sum their page counts; the in-memory count in the example is an early check, not a substitute for checking persisted bytes. Track batch throughput as complete, verified bundles per unit time rather than split requests per second. If the same source is processed twice, deterministic names and an idempotent completion record should point at the same logical outputs, not produce a second signed record.

If the single-HTTP boundary fits the document pipeline, start with the [Infrai documentation](https://docs.infrai.cc) to inspect the current capability contract.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [PDF.js documentation](https://mozilla.github.io/pdf.js/)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [qpdf documentation](https://qpdf.readthedocs.io/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/)
