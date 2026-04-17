# Chapter 50: Command Palette & Keyboard Shortcuts

A senior analyst opens FinancialApp on Monday morning with forty-three browser tabs already competing for attention. She needs to pull up the Hargrove portfolio, check yesterday's flagged transactions, run a risk report, and jot down a note about a client call -- all before the 9:15 standup. Clicking her way through four nested menus is not going to happen. She taps `Cmd+K`. A dialog opens in the center of the screen. She types "harg," hits Enter, and she is on the portfolio before her coffee has cooled.

For analysts and advisors who spend six or more hours a day inside a single application, keyboard-first navigation is the difference between getting through the work and drowning in it. A command palette collapses dozens of menu drill-downs into a single muscle memory: summon, type a few letters, confirm. In a financial app where the same handful of destinations are visited thousands of times per day, that multiplier compounds into real hours saved. Command palettes are the difference between power users and everyone else.

This chapter builds a complete command palette for FinancialApp: the dialog, a contributable command registry, fuzzy search, a vim-style hotkey service, and accessibility and discovery treated as first-class concerns rather than afterthoughts.

> **Companion code:** `financial-app/libs/shared/ui/command-palette/` with `FinCommandPalette`, `CommandRegistryService`, `HotkeysService`; wired into the app shell with FinancialApp commands.

---

## Why Now

A decade ago, command palettes lived mostly in code editors. Today they are table stakes. Linear, GitHub, Notion, Figma, and Raycast all respond to `Cmd+K` with a searchable action surface, and users have been trained to try it everywhere. When a user hits `Cmd+K` on their first day and nothing happens, they register that as a missing feature -- often before they have clicked a single menu item. The technology arrived with the expectation: signals make reactive result lists trivial, `@angular/cdk/a11y` provides focus trapping and live announcements, and fuzzy search libraries are a few kilobytes. There is no excuse left to ship an enterprise app without one.

FinancialApp's palette needs to do four things well: **navigate** (jump to any account, portfolio, client, or page), **transact** (start a wire transfer, categorize a transaction), **search** (find a specific transaction or holding), and **configure** (toggle theme, change locale, sign out). Those four verbs shape every design choice that follows.

---

## Designing `FinCommandPalette`

The palette is a dialog-style overlay rendered through `@angular/cdk/overlay`. An input at the top drives a result list grouped by category. Up/Down moves the highlight, Enter executes, Escape closes, Tab is trapped inside the dialog. The component uses the [Chapter 27](ch27-material-design-system.md) design system for visuals -- `FinIcon` for glyphs and token-backed SCSS for colors and spacing.

```typescript
// libs/shared/ui/command-palette/src/lib/fin-command-palette.component.ts
import { Component, computed, effect, inject, signal } from '@angular/core';
import { A11yModule, LiveAnnouncer } from '@angular/cdk/a11y';
import { FinIconComponent } from '@financial-app/shared/design-system';
import { CommandRegistryService, FinCommand } from './command-registry.service';
import { scoreCommands, ScoredCommand } from './fuzzy';

@Component({
  selector: 'fin-command-palette',
  imports: [A11yModule, FinIconComponent],
  host: {
    'role': 'dialog', 'aria-modal': 'true',
    'aria-labelledby': 'fin-cmdk-title', '(keydown)': 'onKeydown($event)',
  },
  template: `
    <div class="palette" cdkTrapFocus cdkTrapFocusAutoCapture>
      <h2 id="fin-cmdk-title" class="sr-only">Command palette</h2>
      <div class="palette__input">
        <fin-icon name="search" [size]="18" />
        <input type="text" role="combobox" aria-expanded="true"
               aria-controls="fin-cmdk-results"
               [attr.aria-activedescendant]="activeId()"
               placeholder="Type a command or search..."
               [value]="query()" (input)="onInput($event)" />
        <kbd>esc</kbd>
      </div>
      @if (grouped().length === 0) {
        <p class="palette__empty">No matches for "{{ query() }}"</p>
      } @else {
        <ul id="fin-cmdk-results" role="listbox">
          @for (group of grouped(); track group.category) {
            <li role="presentation">
              <span class="palette__group">{{ group.category }}</span>
              <ul role="group" [attr.aria-label]="group.category">
                @for (cmd of group.commands; track cmd.id) {
                  <li [id]="'cmd-' + cmd.id" role="option"
                      [class.is-active]="cmd.id === activeCommand()?.id"
                      [attr.aria-selected]="cmd.id === activeCommand()?.id"
                      (mousedown)="execute(cmd)">
                    @if (cmd.icon) { <fin-icon [name]="cmd.icon" [size]="18" /> }
                    <span [innerHTML]="cmd.highlighted"></span>
                    @if (cmd.shortcut) { <kbd>{{ cmd.shortcut }}</kbd> }
                  </li>
                }
              </ul>
            </li>
          }
        </ul>
      }
    </div>`,
  styleUrl: './fin-command-palette.component.scss',
})
export class FinCommandPaletteComponent {
  private registry = inject(CommandRegistryService);
  private announcer = inject(LiveAnnouncer);

  readonly query = signal('');
  private readonly index = signal(0);

  private readonly scored = computed<ScoredCommand[]>(() =>
    scoreCommands(this.registry.commands(), this.query()));

  protected readonly grouped = computed(() => {
    const byCategory = new Map<string, ScoredCommand[]>();
    for (const cmd of this.scored()) {
      (byCategory.get(cmd.category) ?? byCategory.set(cmd.category, []).get(cmd.category)!).push(cmd);
    }
    return [...byCategory].map(([category, commands]) => ({ category, commands }));
  });

  private readonly flat = computed(() => this.grouped().flatMap(g => g.commands));
  protected readonly activeCommand = computed(() => this.flat()[this.index()]);
  protected readonly activeId = computed(() =>
    this.activeCommand() ? `cmd-${this.activeCommand()!.id}` : null);

  constructor() {
    effect(() => {
      const n = this.flat().length;
      this.announcer.announce(`${n} command${n === 1 ? '' : 's'}`, 'polite');
      if (this.index() >= n) this.index.set(0);
    });
  }

  onInput(e: Event): void { this.query.set((e.target as HTMLInputElement).value); this.index.set(0); }

  protected onKeydown(e: KeyboardEvent): void {
    const max = this.flat().length - 1;
    switch (e.key) {
      case 'ArrowDown': this.index.update(i => Math.min(i + 1, max)); break;
      case 'ArrowUp':   this.index.update(i => Math.max(i - 1, 0));   break;
      case 'Enter':     if (this.activeCommand()) this.execute(this.activeCommand()!); break;
      case 'Escape':    this.registry.close(); break;
      default: return;
    }
    e.preventDefault();
  }

  protected execute(cmd: FinCommand): void { cmd.handler(); this.registry.close(); }
}
```

Three details matter. First, the `host` object declares `role="dialog"` and `aria-modal="true"` directly -- the component *is* the dialog. Second, the input uses `role="combobox"` with `aria-activedescendant` so highlighted options announce without DOM focus leaving the text field. Third, `(mousedown)` is preferred over `(click)` on list items so selection commits before the input's blur tears down the palette.

---

## The Command Registry

A hardcoded command list would force every new feature to edit a shared file -- the kind of centralized coupling that [Chapter 14](ch14-monorepos-libraries.md)'s boundary rules exist to prevent. Instead, features contribute commands through a multi-provider `COMMAND_CONTRIBUTIONS` token at bootstrap, and the palette reads from the registry:

```typescript
// libs/shared/ui/command-palette/src/lib/command-registry.service.ts
import { computed, inject, Injectable, InjectionToken, signal } from '@angular/core';

export interface FinCommand {
  id: string; title: string;
  category: 'Navigate' | 'Transact' | 'Search' | 'Settings';
  keywords?: readonly string[]; description?: string;
  icon?: string; shortcut?: string;
  handler: () => void | Promise<void>;
}

export const COMMAND_CONTRIBUTIONS = new InjectionToken<ReadonlyArray<() => FinCommand[]>>(
  'FIN_COMMAND_CONTRIBUTIONS', { providedIn: 'root', factory: () => [] },
);

@Injectable({ providedIn: 'root' })
export class CommandRegistryService {
  private readonly contributions = inject(COMMAND_CONTRIBUTIONS);
  private readonly dynamic = signal<Map<string, FinCommand>>(new Map());
  readonly isOpen = signal(false);

  readonly commands = computed<FinCommand[]>(() => {
    const all = new Map<string, FinCommand>();
    for (const f of this.contributions) for (const c of f()) all.set(c.id, c);
    for (const [id, c] of this.dynamic()) all.set(id, c);
    return [...all.values()];
  });

  registerCommand(cmd: FinCommand): () => void {
    this.dynamic.update(m => new Map(m).set(cmd.id, cmd));
    return () => this.dynamic.update(m => { const n = new Map(m); n.delete(cmd.id); return n; });
  }

  open(): void { this.isOpen.set(true); }
  close(): void { this.isOpen.set(false); }
  toggle(): void { this.isOpen.update(v => !v); }
}
```

Feature libraries contribute by providing factory functions alongside their routes:

```typescript
// libs/features/transactions/src/lib/transactions.commands.ts
export const transactionCommands: Provider = {
  provide: COMMAND_CONTRIBUTIONS, multi: true,
  useFactory: () => {
    const router = inject(Router);
    const dialog = inject(TransactionDialogService);
    return (): FinCommand[] => [
      { id: 'tx.new', title: 'New transaction', category: 'Transact',
        keywords: ['add', 'create'], icon: 'add_circle', shortcut: 'N',
        handler: () => dialog.openCreate() },
      { id: 'tx.import', title: 'Import transactions from CSV', category: 'Transact',
        keywords: ['upload', 'csv'], icon: 'upload_file',
        handler: () => router.navigate(['/transactions/import']) },
    ];
  },
};
```

The multi-provider pattern keeps features decoupled: `app.config.ts` lists each feature's provider, the registry observes contributions, and no feature knows about the others. Adding the portfolio feature tomorrow means adding one provider line, not modifying a central registry. `registerCommand()` complements the static token for truly dynamic commands -- "Open account: Hargrove Investments" entries generated from the user's account list -- with the returned disposer handling unregistration on destroy.

---

## Fuzzy Search

Exact substring matching ("Navig" matches "Navigate") breaks down fast. A user who types "portacc" should still find "Portfolio: Accounts overview" -- those are their wrists firing the right letters in roughly the right order, and the palette should reward that.

Two libraries dominate this space. **`fuse.js`** (~10 KB gzipped) offers configurable weight per field and includes match-index output for highlighting. **`fzf.js`** (~4 KB) is a faithful port of the `fzf` algorithm that commits to typed scoring and is markedly faster on long lists. FinancialApp ships a few hundred commands at most and needs highlighting, so `fuse.js` is the right default. Teams with thousands of candidates -- file pickers, symbol lookup -- often pick `fzf.js` for the speed.

Scoring follows the literature: **title matches outrank keyword matches outrank description matches**. The wrapper returns commands with highlight markup:

```typescript
// libs/shared/ui/command-palette/src/lib/fuzzy.ts
import Fuse from 'fuse.js';
import { FinCommand } from './command-registry.service';

export interface ScoredCommand extends FinCommand { highlighted: string; }

const options: Fuse.IFuseOptions<FinCommand> = {
  keys: [{ name: 'title', weight: 0.6 }, { name: 'keywords', weight: 0.3 }, { name: 'description', weight: 0.1 }],
  includeMatches: true, threshold: 0.4, ignoreLocation: true,
};

export function scoreCommands(commands: readonly FinCommand[], query: string): ScoredCommand[] {
  if (!query.trim()) return commands.map(c => ({ ...c, highlighted: escape(c.title) }));
  return new Fuse(commands, options).search(query).map(({ item, matches }) => ({
    ...item,
    highlighted: highlight(item.title, matches?.find(m => m.key === 'title')?.indices ?? []),
  }));
}

function highlight(text: string, indices: readonly [number, number][]): string {
  if (indices.length === 0) return escape(text);
  let html = '', cursor = 0;
  for (const [s, e] of indices) {
    html += escape(text.slice(cursor, s)) + `<mark>${escape(text.slice(s, e + 1))}</mark>`;
    cursor = e + 1;
  }
  return html + escape(text.slice(cursor));
}

function escape(s: string): string {
  return s.replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[c]!);
}
```

Characters the user typed appear wrapped in `<mark>` tags, which the design system styles with a token-colored background -- visual feedback that reinforces the mental model: the palette noticed what you typed, and here is why each result rose to the top.

---

## Global Hotkeys Beyond `Cmd+K`

Power users want more than a palette. They want `G P` to jump to Portfolio the way Gmail lets `G I` jump to Inbox, and `?` to summon a cheat sheet. The `HotkeysService` captures those chords and coordinates with the palette so global shortcuts do not fire while a modal has focus:

```typescript
// libs/shared/ui/command-palette/src/lib/hotkeys.service.ts
import { DOCUMENT } from '@angular/common';
import { Injectable, inject } from '@angular/core';
import { CommandRegistryService } from './command-registry.service';

interface Hotkey { description: string; handler: () => void; }

@Injectable({ providedIn: 'root' })
export class HotkeysService {
  private readonly doc = inject(DOCUMENT);
  private readonly registry = inject(CommandRegistryService);
  private readonly hotkeys = new Map<string, Hotkey>();
  private buffer: string[] = [];
  private timeout: ReturnType<typeof setTimeout> | null = null;

  initialize(): void { this.doc.addEventListener('keydown', this.onKeydown); }

  register(sequence: string, description: string, handler: () => void): () => void {
    const key = sequence.toLowerCase();
    this.hotkeys.set(key, { description, handler });
    return () => this.hotkeys.delete(key);
  }

  all(): ReadonlyArray<{ sequence: string; description: string }> {
    return [...this.hotkeys].map(([sequence, { description }]) => ({ sequence, description }));
  }

  private readonly onKeydown = (e: KeyboardEvent) => {
    if (this.registry.isOpen() || this.isTyping(e.target)) return;
    if (e.metaKey || e.ctrlKey || e.altKey) return;
    this.buffer.push(e.key.toLowerCase());
    if (this.timeout) clearTimeout(this.timeout);
    this.timeout = setTimeout(() => (this.buffer = []), 1000);
    const combo = this.buffer.join(' ');
    const hit = this.hotkeys.get(combo);
    if (hit) { e.preventDefault(); this.buffer = []; hit.handler(); return; }
    if (![...this.hotkeys.keys()].some(k => k.startsWith(combo + ' '))) this.buffer = [];
  };

  private isTyping(t: EventTarget | null): boolean {
    return t instanceof HTMLElement && (t.isContentEditable
      || t.matches('input, textarea, select, [role="textbox"], [role="combobox"]'));
  }
}
```

The service is deliberately conservative. It ignores sequences while an `<input>` is focused (typing "g" in a search box must not navigate), while a modifier key is held (reserved for palette-style single chords), and while the palette is open. Sequences time out after one second, matching Gmail's behavior. Feature code registers shortcuts at bootstrap:

```typescript
hotkeys.register('g p', 'Go to Portfolio', () => router.navigate(['/portfolio']));
hotkeys.register('g t', 'Go to Transactions', () => router.navigate(['/transactions']));
hotkeys.register('?',   'Show keyboard shortcuts', () => shortcutsDialog.open());
```

For single-component shortcuts -- "Save this form with `Cmd+S`" -- a full hotkey registration is overkill. Angular's host binding does the job in one line:

```typescript
@Component({
  selector: 'fin-transaction-form',
  host: {
    '(document:keydown.meta.s)': 'onSave($event)',
    '(document:keydown.control.s)': 'onSave($event)',
  },
})
export class TransactionFormComponent {
  onSave(e: KeyboardEvent): void { e.preventDefault(); this.save(); }
}
```

Angular's style guide prefers the `host` object over `@HostListener` because it keeps the host contract visible on the component metadata rather than scattered across decorators, and because it plays better with AOT type-checking.

---

## The Help Overlay

Power users learn shortcuts by discovery -- they try something, see it work, and remember. But no one reads release notes. The `?` key opens a help overlay enumerating every registered shortcut, grouped by category, and the palette exposes the same list as a "Show keyboard shortcuts" command for users who do not yet know the `?` trick:

```typescript
// libs/shared/ui/command-palette/src/lib/fin-shortcuts-overlay.component.ts
@Component({
  selector: 'fin-shortcuts-overlay',
  imports: [A11yModule],
  host: { 'role': 'dialog', 'aria-modal': 'true', 'aria-labelledby': 'shortcuts-title' },
  template: `
    <div class="overlay" cdkTrapFocus cdkTrapFocusAutoCapture>
      <h2 id="shortcuts-title">Keyboard shortcuts</h2>
      @for (group of grouped(); track group.category) {
        <section>
          <h3>{{ group.category }}</h3>
          <dl>
            @for (item of group.items; track item.sequence) {
              <dt><kbd>{{ item.sequence }}</kbd></dt>
              <dd>{{ item.description }}</dd>
            }
          </dl>
        </section>
      }
    </div>`,
})
export class FinShortcutsOverlayComponent {
  protected grouped = computed(() => groupByCategory(inject(HotkeysService).all()));
}
```

The list is authoritative: whatever is registered in `HotkeysService` is what appears. Documentation and reality cannot drift, because they are the same source.

---

## Accessibility

A command palette that only works with a mouse defeats its own purpose. [Chapter 22](ch22-accessibility-aria.md) established the building blocks; the palette composes them into the most ARIA-dense widget in the application.

- **`role="dialog"` with `aria-modal="true"`** marks the overlay as modal, pausing announcement of elements outside the dialog until it closes.
- **`cdkTrapFocus` and `cdkTrapFocusAutoCapture`** handle Tab/Shift+Tab cycling and restore focus to the originating element on close.
- **`role="combobox"` with `aria-activedescendant`** lets the highlighted result change without moving DOM focus away from the input -- users keep typing while the selected option updates around them.
- **`role="listbox"` with `role="option"` children** maps the palette onto the WAI-ARIA combobox-with-listbox-popup pattern, the same contract `@angular/aria` Combobox from [Chapter 22](ch22-accessibility-aria.md) implements for `FinAccountSelector`.
- **`LiveAnnouncer`** politely announces the result count after each keystroke ("12 commands," "0 commands") -- the audible equivalent of watching the list shrink as you type.
- **`role="presentation"`** on the category group wrapper hides the grouping `<li>` from the accessibility tree; the inner `<ul role="group">` provides the category semantics.
- **`<kbd>` elements** for shortcut hints are the semantic choice over styled `<span>`s; screen readers read "`Cmd K`" verbatim.

A complete walkthrough with VoiceOver -- opening, searching, executing, typing-and-hearing -- belongs in [Chapter 49](ch49-screen-reader-testing.md). Running that script against the palette uncovers any gaps before shipping.

---

## Testing

Following the pyramid from [Chapter 25](ch25-e2e-playwright.md), **unit tests** cover the registry and fuzzy search as plain TypeScript -- assertions on pure functions and signal state:

```typescript
it('ranks title matches above keyword matches', () => {
  const ranked = scoreCommands([
    { id: 'a', title: 'Import transactions', category: 'Transact', handler: () => {} },
    { id: 'b', title: 'Export report', category: 'Settings',
      keywords: ['import'], handler: () => {} },
  ], 'import');
  expect(ranked.map(c => c.id)).toEqual(['a', 'b']);
});

it('favors dynamic commands over static contributions on id collision', () => {
  const registry = TestBed.inject(CommandRegistryService);
  registry.registerCommand({ id: 'tx.new', title: 'Override',
    category: 'Transact', handler: () => {} });
  expect(registry.commands().find(c => c.id === 'tx.new')?.title).toBe('Override');
});
```

**E2E tests** exercise the keyboard contract end-to-end with Playwright:

```typescript
test('Cmd+K opens palette, typing filters, Enter executes', async ({ page }) => {
  await page.goto('/');
  await page.keyboard.press('Meta+K');
  const palette = page.getByRole('dialog', { name: 'Command palette' });
  await expect(palette).toBeVisible();
  await page.keyboard.type('portfolio');
  await expect(palette.getByRole('option').first()).toHaveAccessibleName(/portfolio/i);
  await page.keyboard.press('Enter');
  await expect(page).toHaveURL(/\/portfolio/);
});
```

Two tests cover the 80% case. Exhaustive keyboard tests -- every arrow key, every escape path, every modifier combination -- live at the unit level where they run in milliseconds.

---

## Performance

At a few hundred commands the palette is fast by construction: signal-driven change detection recomputes only the result list, Fuse.js scans a list this size in well under a millisecond, and the view updates a handful of DOM nodes per keystroke. Two optimizations matter anyway.

**Debounce the search input** when the registry grows large or when descriptions are expensive to compute. A 50 ms debounce inside `onInput` keeps every-keystroke work tiny; the fuzzy scan only runs after the user pauses.

**Virtualize long result lists** when the registry exceeds a couple hundred items -- typically because it includes per-entity commands like "Open transaction T-12345." The CDK `ScrollingModule` from [Chapter 24](ch24-performance.md) is the drop-in answer: replace the inner `<ul>` with `<cdk-virtual-scroll-viewport itemSize="40">` plus `*cdkVirtualFor`. Keyboard navigation still works because the viewport scrolls the active row into view through `scrollToIndex`, called from the same `effect()` that resets the index.

---

## Discovery

The best command palette in the world is worthless to users who never learn it exists. FinancialApp treats discovery as a feature, not a footnote.

- **First-visit toast.** On the user's first session after the palette ships, a non-intrusive toast reads "Press `Cmd+K` to jump anywhere. Press `?` for all shortcuts." It dismisses on click, auto-dismisses after eight seconds, and is keyed per-user.
- **Persistent hint in the top nav.** The shell includes a `FinIcon` button whose tooltip reads "Command palette (`Cmd+K`)." Users who never try the shortcut still have a visible affordance.
- **Help menu entry.** The Help menu lists "Keyboard shortcuts" as an item, opening the same overlay that `?` produces. Help-menu-first users and keyboard-first users converge on the same surface.
- **Empty-state hints on slow pages.** When an analyst loads a large portfolio, a line below the spinner reads "Tip: press `Cmd+K` to search accounts while this loads." Context-aware teaching beats a generic onboarding tour.

These are cheap to build and compound in impact. The goal is not to push the feature on the user but to make it impossible not to notice over the course of a typical week.

---

## Summary

- **Command palettes are table stakes for power-user applications.** Linear, GitHub, Notion, and Raycast have trained users to expect `Cmd+K` everywhere; missing it reads as a missing feature.
- **`FinCommandPalette`** is a dialog-role overlay with a fuzzy-searching combobox input and grouped listbox, built on `@angular/cdk/overlay` and `@angular/cdk/a11y` and styled with the [Chapter 27](ch27-material-design-system.md) design system.
- **`CommandRegistryService`** is the single source of truth. Features contribute through the `COMMAND_CONTRIBUTIONS` multi-provider token; dynamic commands use `registerCommand()` with a disposer for lifecycle.
- **`fuse.js`** covers most apps; `fzf.js` is preferable for multi-thousand-entry pickers. Weight titles above keywords above descriptions; render matches wrapped in `<mark>` tags.
- **`HotkeysService`** dispatches vim-style sequences (`g p`, `?`) with conflict-safe defaults: ignore typing targets, modifier chords, and anything while the palette is open. Single-component shortcuts go through host bindings like `(document:keydown.meta.s)` rather than `@HostListener`.
- **Accessibility** composes `role="dialog"`, `cdkTrapFocus`, `role="combobox"` with `aria-activedescendant`, and `LiveAnnouncer` announcements -- cross-referenced against [Chapter 22](ch22-accessibility-aria.md) and exercised with screen readers per [Chapter 49](ch49-screen-reader-testing.md).
- **Testing** layers unit tests for the registry and fuzzy scoring plus E2E tests for the keyboard contract (Playwright, [Chapter 25](ch25-e2e-playwright.md)). Debounce the input and reach for CDK virtual scroll from [Chapter 24](ch24-performance.md) only when the registry exceeds a few hundred entries.
- **Discovery is a feature.** A first-visit toast, a persistent nav hint, a Help menu entry, and context-aware inline tips ensure the palette is not a secret.

An analyst who learns `Cmd+K`, `g p`, and `n` finishes her morning checklist in a fraction of the time it took with menus. Shipping that keyboard-first experience is not about showing off the latest pattern -- it is about respecting the users who spend their day inside the application you built.

> **Companion code:** `financial-app/libs/shared/ui/command-palette/` with `FinCommandPalette`, `CommandRegistryService`, `HotkeysService`; wired into the app shell with FinancialApp commands.
