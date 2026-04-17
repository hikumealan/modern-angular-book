# Chapter 34: File Handling

Transactions rarely live inside the app alone. Users drag a year of bank activity in from a CSV export, snap a photo of a gas-station receipt, download a quarterly statement as a PDF, or ship a consolidated ledger to their accountant. Every one of those flows is a file crossing the boundary between the browser and the rest of the world, and every one of them has to be fast, accurate, and accessible.

FinancialApp treats files as first-class domain objects. The transaction ledger accepts CSV imports from ten different bank export formats. Receipts upload from phones and shrink client-side before they ever leave the device. Account statements render as PDFs without a server round-trip. When an advisor exports a client's portfolio history, the download is authenticated, chunked, and resumable. This chapter covers the patterns that make all of that work -- from the humble `<input type="file">` through chunked uploads with progress, CSV round-trips validated with Zod, and client-side PDF generation.

> **Companion code:** `financial-app/libs/shared/file-utils/` with `FileUploadService`, `CsvExportService`, `PdfStatementService`; integrated into the existing transaction-import feature.

---

## File Input Basics

The foundation is a native `<input type="file">`. Angular adds nothing on top of the browser primitive, but understanding what the browser gives you prevents a class of bugs.

```typescript
// libs/shared/file-utils/src/lib/file-picker/fin-file-picker.component.ts
@Component({
  selector: 'fin-file-picker',
  template: `
    <input type="file"
           [accept]="accept()"
           [multiple]="multiple()"
           (change)="onChange($event)" />`,
})
export class FinFilePickerComponent {
  readonly accept = input<string>('*/*');
  readonly multiple = input<boolean>(false);
  readonly filesPicked = output<File[]>();

  protected onChange(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.filesPicked.emit(Array.from(input.files ?? []));
    input.value = '';
  }
}
```

Three details are load-bearing. The `accept` attribute is a hint, not a guarantee: it filters the system picker to the listed types (`accept=".csv,text/csv"`), but users can still drag in anything, and the iOS camera picker ignores it entirely. Treat `accept` as UX polish and do real validation in JavaScript. The `files` property is a `FileList`, not an array -- live and indexed but without `.map` or `.filter`, so convert it with `Array.from` immediately. Resetting `input.value = ''` after emitting lets the same file be selected twice in a row; without it, no second `change` event fires and users file bugs that read "the upload button is broken".

Each `File` exposes `name`, `size` (bytes), `type` (MIME string, possibly empty), and `lastModified`. A `File` *is* a `Blob` with a filename, so every `Blob` method -- `slice`, `stream`, `arrayBuffer`, `text` -- works on a `File` too. We will lean on this for chunked uploads.

Modern Chromium browsers expose `window.showOpenFilePicker()` and `showSaveFilePicker()` for persistent file handles, but Firefox and Safari do not implement them. Keep `<input type="file">` as the universal primitive and reach for the File System Access API only as an opt-in power feature.

---

## Drag and Drop Upload

A drop zone feels modern, but the native events are un-ergonomic: `dragenter`, `dragover`, `dragleave`, and `drop` all fire, `preventDefault()` on `dragover` is mandatory to opt into accepting drops, and `dragleave` fires whenever the pointer crosses a child element, so naive visual feedback flickers constantly.

```typescript
// libs/shared/file-utils/src/lib/drop-zone/fin-drop-zone.component.ts
@Component({
  selector: 'fin-drop-zone',
  host: {
    '[class.is-dragging]': 'isDragging()',
    '(dragenter)': 'onEnter($event)',
    '(dragover)': '$event.preventDefault()',
    '(dragleave)': 'depth.update(d => Math.max(0, d - 1))',
    '(drop)': 'onDrop($event)',
  },
  template: `<ng-content />`,
})
export class FinDropZoneComponent {
  readonly accept = input<string>('*/*');
  readonly maxSize = input<number>(10 * 1024 * 1024);
  readonly filesAccepted = output<File[]>();
  readonly filesRejected = output<{ file: File; reason: string }[]>();

  protected depth = signal(0);
  protected isDragging = computed(() => this.depth() > 0);

  protected onEnter(e: DragEvent): void {
    e.preventDefault();
    this.depth.update(d => d + 1);
  }

  protected onDrop(e: DragEvent): void {
    e.preventDefault();
    this.depth.set(0);
    const accepted: File[] = [], rejected: { file: File; reason: string }[] = [];
    const patterns = this.accept().split(',').map(s => s.trim()).filter(Boolean);

    for (const file of Array.from(e.dataTransfer?.files ?? [])) {
      if (file.size > this.maxSize()) {
        rejected.push({ file, reason: `Exceeds ${this.maxSize() / 1024 / 1024} MB limit` });
      } else if (patterns.length && !this.matches(file, patterns)) {
        rejected.push({ file, reason: `Unsupported type: ${file.type || 'unknown'}` });
      } else {
        accepted.push(file);
      }
    }

    if (accepted.length) this.filesAccepted.emit(accepted);
    if (rejected.length) this.filesRejected.emit(rejected);
  }

  private matches(file: File, patterns: string[]): boolean {
    return patterns.some(p =>
      p.startsWith('.') ? file.name.toLowerCase().endsWith(p.toLowerCase())
      : p.endsWith('/*') ? file.type.startsWith(p.slice(0, -1))
      : file.type === p);
  }
}
```

The `depth` counter is the trick that tames `dragleave`: every `dragenter` increments, every `dragleave` decrements, and the zone is only truly "not being dragged over" when the counter hits zero -- which correctly handles children, pseudo-elements, and text nodes.

MIME validation is deliberately defensive. The browser's `file.type` is advisory and sometimes empty -- Safari often reports `""` for CSVs while Chrome reports `"text/csv"`. Accept both by allowing extension matches (`.csv`) alongside MIME matches (`text/csv`), and do authoritative content validation later during parsing.

---

## Accessibility for File Inputs

Native `<input type="file">` is accessible by default, but ugly and hard to style. The standard accessible pattern wraps a visually hidden input inside a `<label>` so the label becomes the styleable surface while screen readers still announce "file upload" properly -- the same hidden-control technique established in [Chapter 22](ch22-accessibility-aria.md).

```html
<!-- fin-file-picker.component.html -->
<label class="file-picker">
  <span class="file-picker__label">{{ label() }}</span>
  <input class="visually-hidden" type="file"
         [accept]="accept()" [multiple]="multiple()"
         [attr.aria-describedby]="hintId()"
         (change)="onChange($event)" />
  @if (hint()) { <span [id]="hintId()" class="hint">{{ hint() }}</span> }
</label>
```

```scss
.visually-hidden {
  position: absolute; width: 1px; height: 1px;
  padding: 0; margin: -1px; overflow: hidden;
  clip: rect(0, 0, 0, 0); white-space: nowrap; border: 0;
}
```

Do not use `display: none` or `visibility: hidden`: both remove the input from the accessibility tree, and keyboard users can no longer activate it.

Selection must also be announced. Without it, a screen reader user activates the label, the native picker opens and closes, and they get no feedback on whether a file was chosen. Pipe selection and rejection through the `LiveAnnouncer` from [Chapter 22](ch22-accessibility-aria.md):

```typescript
private announcer = inject(LiveAnnouncer);

protected onChange(event: Event): void {
  const input = event.target as HTMLInputElement;
  const files = Array.from(input.files ?? []);
  if (files.length) {
    this.announcer.announce(
      files.length === 1
        ? `Selected ${files[0].name}, ${this.formatSize(files[0].size)}`
        : `Selected ${files.length} files`,
      'polite',
    );
  }
  this.filesPicked.emit(files);
  input.value = '';
}
```

Rejections use `'assertive'` -- the user needs to know *right now* that their 50 MB PNG was too large.

---

## Chunked Uploads

A 200 MB brokerage CSV does not belong in a single POST. A dropped connection at 180 MB wastes everyone's time, proxies enforce per-request size limits, and there is no way to show meaningful progress for a request that hasn't hit the wire yet. The solution is chunked uploads: slice the file, upload each chunk sequentially, and let the server reassemble.

`Blob.slice(start, end)` returns a new `Blob` view over the original bytes without copying -- slicing a 200 MB file into 5 MB chunks is instant and memory-efficient.

```typescript
// libs/shared/file-utils/src/lib/upload/chunked-upload.service.ts
const CHUNK_SIZE = 5 * 1024 * 1024;
type Status = 'pending' | 'uploading' | 'complete' | 'error';

@Injectable({ providedIn: 'root' })
export class ChunkedUploadService {
  private http = inject(HttpClient);

  upload(file: File, uploadId: string) {
    const progress = signal({ uploaded: 0, total: file.size, percent: 0, status: 'pending' as Status });

    (async () => {
      progress.update(p => ({ ...p, status: 'uploading' }));
      const totalChunks = Math.ceil(file.size / CHUNK_SIZE);

      for (let i = 0; i < totalChunks; i++) {
        const end = Math.min((i + 1) * CHUNK_SIZE, file.size);
        const form = new FormData();
        form.append('chunk', file.slice(i * CHUNK_SIZE, end), file.name);
        form.append('uploadId', uploadId);
        form.append('chunkIndex', String(i));

        await this.uploadChunk(form);
        progress.update(p => ({ ...p, uploaded: end, percent: Math.round((end / file.size) * 100) }));
      }

      await firstValueFrom(this.http.post(`/api/uploads/${uploadId}/complete`, {}));
      progress.update(p => ({ ...p, status: 'complete' }));
    })().catch(() => progress.update(p => ({ ...p, status: 'error' })));

    return progress.asReadonly();
  }

  private async uploadChunk(form: FormData, attempt = 0): Promise<void> {
    try { await firstValueFrom(this.http.post('/api/uploads/chunk', form)); }
    catch (err) {
      if (attempt >= 2) throw err;
      await new Promise(r => setTimeout(r, 500 * (attempt + 1)));
      return this.uploadChunk(form, attempt + 1);
    }
  }
}
```

Resumable uploads extend this with a manifest call before the first chunk: `GET /api/uploads/:uploadId/status` returns the chunk indices the server already has, and the loop skips those. With content-hashed `uploadId`s, a dropped connection resumes from the last acknowledged chunk with zero bytes re-sent.

---

## Upload Progress with HttpClient

For single-request uploads, `HttpClient` exposes fine-grained progress events. Opt in with `reportProgress: true`:

```typescript
// libs/shared/file-utils/src/lib/upload/file-upload.service.ts
@Injectable({ providedIn: 'root' })
export class FileUploadService {
  private http = inject(HttpClient);

  upload(url: string, file: File) {
    const state = signal({ percent: 0, status: 'uploading' as 'uploading' | 'success' | 'error' });
    const form = new FormData();
    form.append('file', file, file.name);

    const req = new HttpRequest('POST', url, form, { reportProgress: true });

    this.http.request(req).subscribe({
      next: event => {
        if (event.type === HttpEventType.UploadProgress && event.total) {
          state.update(s => ({ ...s, percent: Math.round((event.loaded / event.total!) * 100) }));
        } else if (event.type === HttpEventType.Response) {
          state.set({ percent: 100, status: 'success' });
        }
      },
      error: () => state.set({ percent: 0, status: 'error' }),
    });

    return state.asReadonly();
  }
}
```

`reportProgress: true` tells `HttpClient` to emit intermediate events; `HttpRequest` implicitly observes events, so the observable yields every `UploadProgress` tick plus the final `Response`. Use the `HttpEventType` enum rather than string comparisons -- the compiler catches missing cases in strict mode. Feeding the resulting signal into the progress bar from [Chapter 22](ch22-accessibility-aria.md) gets accessible progress announcements for free:

```html
@if (state().status === 'uploading') {
  <fin-progress-bar [value]="state().percent" [label]="'Uploading ' + fileName()" />
}
```

---

## Transaction CSV Import

Bank CSVs are the messy reality of personal finance: every issuer has its own column order, date format, and credit/debit convention. FinancialApp's `features/transactions/import/` scaffold walks the user through picking a file, previewing parsed rows, resolving errors, and committing the clean batch. The building blocks are `papaparse` for parsing and Zod schemas from [Chapter 31](ch31-advanced-typescript-openapi.md) for validation.

```typescript
// libs/shared/file-utils/src/lib/csv/csv-import.service.ts
export const TransactionImportRow = z.object({
  date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Date must be YYYY-MM-DD'),
  description: z.string().min(1).max(200),
  amount: z.coerce.number().refine(n => !Number.isNaN(n), 'Amount must be numeric'),
  type: z.enum(['credit', 'debit']),
  category: z.string().optional(),
});
export type TransactionImportRow = z.infer<typeof TransactionImportRow>;
export interface ImportResult {
  valid: TransactionImportRow[];
  invalid: { row: number; errors: string[] }[];
}

@Injectable({ providedIn: 'root' })
export class CsvImportService {
  parse(file: File): Promise<ImportResult> {
    return new Promise((resolve, reject) => {
      Papa.parse<Record<string, string>>(file, {
        header: true,
        skipEmptyLines: true,
        transformHeader: h => h.trim().toLowerCase(),
        complete: results => {
          const valid: TransactionImportRow[] = [];
          const invalid: ImportResult['invalid'] = [];
          results.data.forEach((raw, i) => {
            const parsed = TransactionImportRow.safeParse(raw);
            if (parsed.success) valid.push(parsed.data);
            else invalid.push({ row: i + 2, errors: parsed.error.issues.map(x => `${x.path.join('.')}: ${x.message}`) });
          });
          resolve({ valid, invalid });
        },
        error: reject,
      });
    });
  }
}
```

`papaparse` handles the details you do not want to handle yourself: quoted fields with commas, CRLF vs LF line endings, BOM characters on Windows exports, and streaming for files larger than memory. `transformHeader` normalizes column names so `"Date"`, `"date"`, and `"DATE"` land in the same field.

Row-level error UIs matter. Rejecting a 500-row CSV because one amount has a stray dollar sign is hostile; showing which rows failed and letting the user edit or skip each one keeps support tickets out of the queue.

```html
<!-- features/transactions/import/transaction-import.component.html -->
<fin-drop-zone accept=".csv,text/csv" (filesAccepted)="onFile($event[0])" />

@if (result(); as r) {
  <p>{{ r.valid.length }} valid rows, {{ r.invalid.length }} errors.</p>

  @if (r.invalid.length > 0) {
    <section aria-labelledby="errors-heading">
      <h3 id="errors-heading">Rows that need attention</h3>
      <ul>
        @for (err of r.invalid; track err.row) {
          <li>Row {{ err.row }}: {{ err.errors.join('; ') }}</li>
        }
      </ul>
    </section>
  }

  <button type="button" [disabled]="r.valid.length === 0" (click)="commit(r.valid)">
    Import {{ r.valid.length }} transactions
  </button>
}
```

Because the Zod schema mirrors the transaction model from [Chapter 6](ch06-signal-forms.md), valid rows flow into the normal submission pipeline and reuse every downstream validator, duplicate-detector, and category mapper.

---

## CSV Export

Export is the symmetric operation. `papaparse`'s `unparse` serializes, and the browser handles the rest via an object URL bound to a synthesized anchor.

```typescript
// libs/shared/file-utils/src/lib/csv/csv-export.service.ts
@Injectable({ providedIn: 'root' })
export class CsvExportService {
  exportTransactions(transactions: Transaction[], filename: string): void {
    const rows = transactions.map(t => ({
      date: t.date, description: t.description,
      amount: t.amount.toFixed(2), type: t.type, category: t.category,
    }));
    const csv = '\uFEFF' + Papa.unparse(rows, { header: true });
    this.triggerDownload(new Blob([csv], { type: 'text/csv;charset=utf-8' }), filename);
  }

  private triggerDownload(blob: Blob, filename: string): void {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = filename; a.style.display = 'none';
    document.body.appendChild(a); a.click(); a.remove();
    URL.revokeObjectURL(url);
  }
}
```

`URL.createObjectURL` allocates an opaque URL pointing at the in-memory blob. Assigning it to an anchor's `href` and programmatically clicking triggers the native download flow -- default download directory, filename sanitization, and virus scanner hooks included. Two details are mandatory: `URL.revokeObjectURL` releases the blob (the browser holds it in memory until the URL is revoked or the document unloads, and a long-lived SPA leaks fast), and the UTF-8 BOM (`\uFEFF`) is Excel-on-Windows insurance -- without it, non-ASCII merchant names display as `Ã¤` where `ä` should be.

---

## PDF Generation with pdf-lib

Account statements are the classic "generate a PDF" use case. A server round-trip works, but it adds latency and a whole category of templating decisions. For FinancialApp's monthly statement, `pdf-lib` produces a complete document client-side in under a hundred lines.

```typescript
// libs/shared/file-utils/src/lib/pdf/pdf-statement.service.ts
import { PDFDocument, StandardFonts, rgb } from 'pdf-lib';

export interface StatementInput {
  account: Account;
  periodStart: string;
  periodEnd: string;
  transactions: Transaction[];
}

@Injectable({ providedIn: 'root' })
export class PdfStatementService {
  async generate(input: StatementInput): Promise<Blob> {
    const doc = await PDFDocument.create();
    const font = await doc.embedFont(StandardFonts.Helvetica);
    const bold = await doc.embedFont(StandardFonts.HelveticaBold);
    let page = doc.addPage([595, 842]);
    let y = 800;

    page.drawText('Account Statement', { x: 50, y, size: 20, font: bold });
    y -= 30;
    page.drawText(`${input.account.name} (${input.account.type})`, { x: 50, y, size: 12, font });
    y -= 18;
    page.drawText(`Period: ${input.periodStart} to ${input.periodEnd}`,
      { x: 50, y, size: 10, font, color: rgb(0.4, 0.4, 0.4) });
    y -= 30;

    for (const [label, x] of [['Date', 50], ['Description', 130], ['Amount', 450]] as const) {
      page.drawText(label, { x, y, size: 10, font: bold });
    }
    y -= 16;

    let total = 0;
    for (const t of input.transactions) {
      if (y < 80) { page = doc.addPage([595, 842]); y = 800; }
      const signed = t.type === 'credit' ? t.amount : -t.amount;
      total += signed;
      page.drawText(t.date, { x: 50, y, size: 9, font });
      page.drawText(t.description.slice(0, 50), { x: 130, y, size: 9, font });
      page.drawText(signed.toFixed(2), {
        x: 450, y, size: 9, font,
        color: signed >= 0 ? rgb(0, 0.4, 0) : rgb(0.6, 0, 0),
      });
      y -= 14;
    }

    y -= 20;
    page.drawText(`Net change: ${total.toFixed(2)} ${input.account.currency}`,
      { x: 50, y, size: 11, font: bold });

    return new Blob([await doc.save()], { type: 'application/pdf' });
  }
}
```

Call sites feed the blob into the same `triggerDownload` helper used for CSV exports. `pdf-lib` has no dependencies on Node, Canvas, or a headless browser -- pure TypeScript on the main thread. For statements with thousands of transactions, push generation into a Web Worker to keep the UI thread responsive.

---

## Authenticated File Downloads

Public downloads work with a plain `<a href>`. Authenticated downloads do not: an anchor click bypasses `HttpClient` and its interceptors, so no bearer token is attached. The pattern is to fetch the file as a blob through `HttpClient` -- letting interceptors apply auth and XSRF headers -- then trigger the download from the resulting blob.

```typescript
// libs/shared/file-utils/src/lib/download/authenticated-download.service.ts
@Injectable({ providedIn: 'root' })
export class AuthenticatedDownloadService {
  private http = inject(HttpClient);

  async download(url: string, filename: string): Promise<void> {
    const blob = await firstValueFrom(this.http.get(url, { responseType: 'blob' }));
    const objectUrl = URL.createObjectURL(blob);
    try {
      const a = document.createElement('a');
      a.href = objectUrl; a.download = filename; a.click();
    } finally {
      URL.revokeObjectURL(objectUrl);
    }
  }
}
```

`responseType: 'blob'` tells `HttpClient` to keep the response as binary rather than JSON-parsing it. For files that fit in memory -- PDF statements, CSV exports, receipt thumbnails -- this is the pattern. For a 400 MB archival export, buffering the entire file defeats the point; prefer a short-lived signed URL obtained via an authenticated call, then redirect the browser to that URL so the download streams from the CDN without touching JavaScript memory.

---

## Client-Side Image Processing

Receipts arrive as 8-megapixel phone photos. Uploading 4 MB of JPEG when a 600-pixel-wide preview would suffice wastes bandwidth and storage. Canvas-based downscaling happens entirely on the client.

```typescript
// libs/shared/file-utils/src/lib/image/image-resize.service.ts
@Injectable({ providedIn: 'root' })
export class ImageResizeService {
  async resize(file: File, maxWidth: number, maxHeight: number, quality = 0.85): Promise<Blob> {
    const bitmap = await createImageBitmap(file);
    try {
      const scale = Math.min(1, maxWidth / bitmap.width, maxHeight / bitmap.height);
      const width = Math.round(bitmap.width * scale);
      const height = Math.round(bitmap.height * scale);

      const canvas = document.createElement('canvas');
      canvas.width = width; canvas.height = height;
      canvas.getContext('2d')!.drawImage(bitmap, 0, 0, width, height);

      return await new Promise<Blob>((resolve, reject) => {
        canvas.toBlob(b => b ? resolve(b) : reject(new Error('toBlob failed')), 'image/jpeg', quality);
      });
    } finally {
      bitmap.close();
    }
  }
}
```

`createImageBitmap` decodes off the main thread and returns a handle you can draw to a canvas. Call `bitmap.close()` when you are done -- bitmaps hold real GPU/decoder resources, and a component that processes fifty receipts without closing leaks aggressively. `canvas.toBlob` re-encodes with the requested MIME type and quality. For previews, wrap the blob in an object URL, assign to an `<img>` src, and revoke the URL when the preview unmounts.

---

## Testing File Handling

Unit tests create `File` instances directly with the constructor -- no fixtures, no filesystem:

```typescript
// libs/shared/file-utils/src/lib/csv/csv-import.service.spec.ts
it('parses valid transactions and collects row errors', async () => {
  const csv = [
    'date,description,amount,type',
    '2026-01-04,Coffee,4.50,debit',
    'not-a-date,Lunch,12.00,debit',
    '2026-01-05,Refund,not-a-number,credit',
  ].join('\n');
  const file = new File([csv], 'transactions.csv', { type: 'text/csv' });

  const result = await new CsvImportService().parse(file);

  expect(result.valid).toHaveLength(1);
  expect(result.invalid).toHaveLength(2);
  expect(result.invalid[0].errors[0]).toMatch(/date/i);
});
```

Drag-and-drop events require a synthesized `DataTransfer`. jsdom does not implement the constructor, so tests build a minimal stand-in:

```typescript
function makeDropEvent(files: File[]): DragEvent {
  const dataTransfer = { files, types: ['Files'] } as unknown as DataTransfer;
  return new DragEvent('drop', { bubbles: true, cancelable: true, dataTransfer });
}

it('rejects files larger than maxSize', () => {
  const fixture = TestBed.createComponent(FinDropZoneComponent);
  fixture.componentRef.setInput('maxSize', 100);
  fixture.detectChanges();

  const rejected = vi.fn();
  fixture.componentInstance.filesRejected.subscribe(rejected);

  const big = new File([new Uint8Array(200)], 'big.bin');
  fixture.nativeElement.dispatchEvent(makeDropEvent([big]));

  expect(rejected).toHaveBeenCalledWith([
    expect.objectContaining({ file: big, reason: expect.stringContaining('limit') }),
  ]);
});
```

For upload progress, `HttpTestingController` inspects outgoing requests and emits synthetic `HttpProgressEvent`s. A service test can assert that `reportProgress: true` was passed, that the request body is the expected `FormData`, and that the exposed signal reaches 100% after a scripted event sequence -- all without a real network.

---

## Summary

Files are where the Angular app meets the rest of the operating system. Every pattern in this chapter exists to keep that boundary thin, fast, and recoverable.

- **`<input type="file">`** is the universal primitive: reset `value` after every change for re-selection, convert the `FileList` to an array, and treat `accept` as UX polish rather than real validation.
- **Drag and drop** requires `preventDefault()` on `dragover` and a depth counter to tame flickering `dragleave` events. Validate MIME types, extensions, and size in the handler.
- **Accessibility** uses a visually hidden input inside a styled `<label>`, paired with `LiveAnnouncer` announcements of selections and rejections, following [Chapter 22](ch22-accessibility-aria.md).
- **Chunked uploads** use `Blob.slice()` for zero-copy views and upload sequentially with retry. A server-side manifest makes uploads resumable across dropped connections.
- **`HttpClient` progress** requires `reportProgress: true` on an `HttpRequest`, mapped through `HttpEventType.UploadProgress` and exposed as a signal.
- **CSV import** combines `papaparse` with Zod schemas from [Chapter 31](ch31-advanced-typescript-openapi.md) for row-level validation, surfacing errors per row so users fix individual problems. The flow composes with the transaction form from [Chapter 6](ch06-signal-forms.md).
- **CSV export** serializes via `papaparse`'s `unparse`, wraps it in a blob, and triggers a download through a synthesized anchor. Remember the UTF-8 BOM for Excel and `URL.revokeObjectURL` to prevent leaks.
- **PDF generation** with `pdf-lib` runs entirely client-side, composing account statements with headers, tables, and totals.
- **Authenticated downloads** route through `HttpClient` with `responseType: 'blob'` so interceptors apply auth; short-lived signed URLs are the escape hatch for files too large to buffer.
- **Canvas resizing** compresses camera-quality images before upload via `createImageBitmap`, a scratch canvas, and `toBlob`. Close bitmaps and revoke object URLs.
- **Tests** use the `File` constructor for fixtures and a synthetic `DataTransfer` for drag-and-drop; `HttpTestingController` covers progress without a real network.

A user who cannot import their year of transactions, export their statement for their accountant, or attach a receipt quickly stops being a user. The patterns here turn those flows into reusable building blocks across every domain the application grows into.
