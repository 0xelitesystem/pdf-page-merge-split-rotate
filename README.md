# PDF Merge and Split

Merge, split, reorder and rotate PDF pages entirely in your browser. Parses the real cross reference chain, names every file it refuses, and lists per file the structures it does not carry across before you download.

**Live demo:** https://0xelitesystem.github.io/pdf-merge-and-split/

A sibling repo, `watermarks-remover`, strips provenance metadata from several formats including PDF. It never resolves a page tree, an xref chain or an object stream, and this tool never touches metadata, so the two are not overlapping entrances. Say this plainly and once: **qpdf and pdftk are strictly more capable than this page.** They handle encryption, damaged cross reference reconstruction and linearization, none of which this tool attempts. The reason to use this instead is that the bytes never leave the machine, not that the feature set is larger.

## Use

1. Open the page and add one or more PDF files, by the file picker or by dropping them on the box.
2. Each file immediately prints a structure line: byte size, page count, cross reference kind (classic table, cross reference stream, or hybrid table plus `/XRefStm`), whether object streams were used, and how many incremental updates the chain carries. A file that cannot be handled prints a named refusal instead, and is never silently skipped.
3. Pick pages per file with a range such as `all`, `1-3,7,12-` or `-4`. Files stack in the order you add them, so merging is selecting all pages of two or more files and splitting is selecting a subset of one.
4. Reorder, rotate in 90 degree steps, or remove individual pages in the output list. Every one of those controls is a real button with an accessible name that includes the page identity, so the whole flow works from the keyboard.
5. Press Build. The result line reports the output page count, byte size and object count, then names every source structure that was not carried across. A file whose damage only shows while its pages are copied is named and dropped at that point, and the remaining files still build. The file arrives through a `Blob` object URL, which is not a network call.

Nothing is uploaded at any point. The page makes no network request of any kind.

## Why this exists

Most browser PDF tools either ship a large parser from a package host or send your file to a server. This one does neither. It is a single HTML file with no dependencies, and it parses PDF structure directly: the `startxref` pointer, the cross reference chain with `/Prev`, classic tables, cross reference streams with PNG predictors, object streams, and the page tree with its inheritable attributes.

The part that is easy to get wrong, and that quietly corrupts output when you do, is a **hybrid reference file** (ISO 32000-1 section 7.5.8.4). In such a revision the trailer of a classic table carries an `/XRefStm` key pointing at a cross reference stream. The stream is the authoritative view; the classic table is a deliberately stale legacy view that points contested objects at superseded copies. Merging is first wins, so the stream must be merged **before** the table. Reverse those two steps and the tool emits a byte valid PDF carrying a superseded revision of the page content, with no error raised anywhere and no independent parser complaining.

That ordering is enforced in the code with the two merges kept visibly separate, and it is guarded by a positive control fixture built into the page. If the `/XRefStm` cannot be read at all, because the offset is out of range or the stream will not inflate, the file is refused by name rather than resolved from the classic table: quietly falling back to a view that is stale by construction is the same corruption reached by a second route.

## Accuracy

Everything below was measured on this machine with the exact engine that ships in `index.html`. Nothing here is copied from elsewhere.

**Corpus.** 302 real PDF files collected from one working machine, deduplicated by SHA-256, and filtered to exclude every file this tool had written itself, so no output of this engine is counted as an input to it. Every file was parsed, its full page set was rebuilt in reverse page order, and the output was written out.

| Outcome | Files |
| --- | --- |
| Built | 276 |
| Refused, encrypted (`/Encrypt` in the trailer) | 23 |
| Refused, not a PDF | 2 |
| Refused, stream could not be inflated | 1 |
| Refused, damaged or absent cross reference data | 0 |
| Refused, damaged page tree | 0 |

That is 91.4 percent of the whole corpus, and 276 of the 279 files that were not encrypted, which is 98.9 percent. This is one person's corpus on one machine, not a claim about PDFs in general.

**The refusal classes are five, not one.** Encrypted, damaged or absent cross reference data, not a PDF, damaged page tree, and stream could not be inflated. Each one prints its own named line attached to the offending file. Two of the five did not occur in this corpus, and they are listed above with a count of zero rather than omitted. Damaged cross reference data is **refused, not reconstructed**: this tool never scans a file for `obj` patterns to guess a cross reference table back into existence.

**Independent re-parse.** All 276 outputs were read back with pypdf 6.12.2, which did not write any of them. Page counts matched on 276 of 276, and pypdf emitted no warning on any output. pypdf is a recovery tolerant reader and is therefore more forgiving than a strict structural checker, so treat this as evidence that the outputs parse, not as a conformance certificate. qpdf, pdftk, mutool and Ghostscript are not installed on this machine and none of them was run, so nothing here was validated with them.

**Text fidelity.** All 276 built files were compared page by page against their sources. Because every output was built with its pages reversed, output page *i* must carry the same extracted text as source page *n minus 1 minus i*. Extracted text was identical on every page of all 276, with no source and no output that pypdf could not read.

**Cross reference kinds in that corpus.** 177 classic table only, 64 hybrid table plus `/XRefStm`, 58 cross reference stream only. The other 3 were refused before a kind could be determined. 121 of the 302 use object streams, which the per file structure line reports from the cross reference map rather than from whatever the parser happened to decode.

**The hybrid precedence control.** A 930 byte hand built fixture is embedded in the page as a base64 string. Its update revision writes a new object 4 whose content stream draws `FRESH`, points the `/XRefStm` entry at that new object, and points its own classic table entry for object 4 at the superseded offset whose stream draws `STALE`. The self test parses it and requires object 4 to decode to `FRESH`. The same engine run with the two merge steps swapped resolves the same fixture to `STALE`, so the control is capable of failing and the self test reports that reversed run as a second check. For reference, pypdf 6.12.2 reads this fixture as `STALE`. A fixture whose update marks object 4 free instead is not usable here: a free entry does not shadow, so both orderings resolve to `FRESH` and the control could never fail.

**Why the fixture is necessary.** Of the 64 hybrid files in the corpus, 59 built. Both merge orders produced identical output on all 59, compared up to the trailer, which carries a randomly generated `/ID` and therefore differs on every build. The corpus alone would therefore not have caught a reversed precedence. That is the whole argument for shipping a positive control rather than trusting a pass rate.

**Worked merge.** Two hybrid reference PDFs from the author's own disk, 2 pages each, 182,662 and 180,424 bytes, each carrying one incremental update and using object streams. Merged by this engine with no library and no network call. Output: 4 pages, 348,884 bytes, 58 objects. Read back with pypdf 6.12.2: 4 pages, extracted text identical to the source pages on 4 of 4, page 1 and page 3 both begin with the same first line, which is where the join should land, `MediaBox [0, 0, 612, 792]` preserved on all four pages and `/Rotate` absent as in the sources. Both sources carry no `/Outlines` and no `/AcroForm`, so no bookmark or form field warning was due and none was printed.

**Warnings actually raised.** Of the 276 built files, 37 sources carried bookmarks and 5 carried an `/AcroForm`. Every one of them produced a named warning line before any build.

**What a page level edit does not preserve.** This tool copies the transitive closure of the page objects you select and builds a fresh catalog and page tree. Anything hanging off the source catalog rather than off a page is not carried, and each one present in your file is named in that file's entry before you build: bookmarks (`/Outlines`), form fields (`/AcroForm`), named destinations (`/Names`), page labels (`/PageLabels`), tagged structure (`/StructTreeRoot`, so reading order for assistive technology is lost), optional content groups (`/OCProperties`), and document metadata (`/Info` and `/Metadata`). Page level `/Annots` are carried, and the self test asserts they survive a rebuild. A link annotation whose destination points at a page you did not include has that destination replaced with `null`, and the build result counts how many were dropped.

**Self test.** A button on the page runs 17 checks against fixtures built into the file, and prints pass or fail per check. Two are positive controls that must be reported as correctly rejected: the reversed merge order resolving the fixture to `STALE`, and the false claim that `/Rotate 450` normalises to 180. All 17 pass on the shipped file.

## Privacy

Everything runs in your browser. Nothing is uploaded, stored or transmitted. There is no `fetch`, no `XMLHttpRequest`, no `WebSocket`, no `EventSource`, no beacon, no analytics, no external font and no external script anywhere in the file. Your PDFs are read with `File.arrayBuffer` into memory in the tab, and the output is handed back through a `Blob` object URL, which is a browser local mechanism and not a network call. Nothing is written to `localStorage` or `sessionStorage`. Close the tab and nothing remains.

## Run locally

```
git clone https://github.com/0xelitesystem/pdf-merge-and-split.git
cd pdf-merge-and-split
```

Open `index.html` in a browser. That is the whole thing, and it works from the local file system with no server. If you prefer to serve it:

```
python -m http.server 8000
```

Then open `http://localhost:8000/`.

## Build

There is no build. One self contained `index.html` with inline CSS and JavaScript, no dependencies, no package manager, no bundler, no build step.

## Accessibility

Verified against WCAG 2.1 AA. The palette was checked by arithmetic, every text token against every background token, in both the light and the dark theme, and not by looking at a rendered page, because the semantic colours only paint once a refusal or a warning exists. The worst text pair is 5.06:1 in light and 7.77:1 in dark, both above the 4.5:1 minimum. Control and input boundaries clear the 3:1 non text minimum, with a worst case of 3.46:1.

- `prefers-color-scheme` is honoured, and both themes are defined explicitly.
- The file input is a real labelled `<input type="file">` reachable by Tab and operable by Enter or Space. Drag and drop is an addition, never the only route.
- Reordering, rotating and removing pages are real buttons with accessible names that include the page identity, for example "Move page 3 of report.pdf up". There is no drag only interaction.
- Results, refusals and self test output sit in `role="status" aria-live="polite"` regions, so a screen reader hears "REFUSED, encrypted" without hunting for it.
- No result is conveyed by colour alone. Every refusal line carries the word REFUSED, every warning line carries WARNING, and every passing line carries PASS or OK.
- Focus is visible on every control through a `:focus-visible` outline.
- The file list and the output page list are real `<ul>` and `<li>` elements with a visible count, and every table header cell carries `scope`.

## Third-party notices

No third-party code is reproduced in this repository. There is no vendored library, no bundled dependency and no copied source. The PDF behaviour was reimplemented from the published specification, ISO 32000-1, in particular sections 7.5.4 (cross reference table), 7.5.7 (object streams), 7.5.8 (cross reference streams), 7.5.8.4 (hybrid reference files) and 7.7.3.2 (page tree). Compression uses the browser's own `DecompressionStream`.

This project is independent and unaffiliated with Adobe, with the ISO, with pypdf, with qpdf, with pdftk, or with any other named party. Those names appear only to describe what was measured and what this tool deliberately does not attempt. No endorsement is implied.

## Related

- https://github.com/0xelitesystem/image-compressor
- https://github.com/0xelitesystem/image-to-data-uri
- https://github.com/0xelitesystem/hash-generator

## License

MIT. Copyright (c) 2026 RBL Industries LLC.
