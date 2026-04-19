# Chapter 49: AI in Angular -- Runtime & Engineering

AI shows up in Angular projects at two distinct layers, and this chapter covers both. The first is the **runtime** layer -- AI features your users interact with, like chat assistants that call tools, render live Angular components, and produce charts from natural-language queries. The second is the **engineering** layer -- AI that helps you *build* the Angular application itself, through editor rules, MCP servers that expose your workspace to a coding assistant, and Agent Skills that encode procedural knowledge about the framework. The two layers use the same underlying techniques -- structured prompting, tool calling, context injection -- but the audience and the failure modes differ. Part 1 covers runtime AI with Hashbrown; Part 2 covers the engineering side with Cursor rules, the Angular CLI MCP server, and Agent Skills.

> **Stability warning -- read before adopting.**
> This chapter covers a fast-moving corner of the Angular ecosystem, and every piece of it is
> evolving rapidly. Hashbrown is a niche, community-driven library maintained by LiveLoveApp;
> its `chatResource`, `uiChatResource`, and `structuredCompletionResource` APIs reflect the
> library at the time of writing (spring 2026) and may change in future releases. The Angular
> CLI MCP server and the broader Model Context Protocol ecosystem are likewise in active
> development -- tool names, flags, and server configurations shift between releases, and what
> starts as "experimental" this quarter may become stable (or disappear) the next. Pin your
> dependency versions, monitor the relevant changelogs
> ([Hashbrown](https://hashbrown.dev), [angular.dev/ai](https://angular.dev/ai), and the MCP
> specifications), and write integration tests around the specific APIs you depend on. The
> patterns in this chapter -- tool-calling boundaries, structured output, runtime sandboxing,
> and context injection via rules and MCP servers -- are durable even if the specific libraries
> implementing them are not.

---

## Part 1 -- Runtime AI

A financial dashboard that answers "show my spending by category this month" by rendering a real chart component -- not a blob of markdown -- is qualitatively different from one that opens a separate chat tab. The pattern is called *generative UI*: the language model selects and populates Angular components at runtime.

This part explores three levels of integration using Hashbrown: text chat with tool calling, generative UI that streams real components into the conversation, and on-the-fly code generation for ad-hoc analytical queries. All three build on the signal-based component model from [Chapter 2](ch02-signal-components.md) and the reactive patterns from [Chapter 3](ch03-reactive-signals.md).

> **Companion code:** `financial-app/apps/financial-app/src/app/features/assistant/chat.component.ts`

---

### Implementing an Assistant with Tool Calling

#### Setting up Hashbrown

Hashbrown ships as scoped packages. Install the core library, the Angular binding, and a provider for your LLM of choice:

```bash
npm install @hashbrownai/{core,angular,openai}
```

Swap `@hashbrownai/openai` for `@hashbrownai/google`, `@hashbrownai/azure`, or another provider -- every provider implements the same streaming interface, so Angular code stays the same. API keys must never reach the browser; Hashbrown expects a thin backend proxy. Register the proxy URL at bootstrap:

```typescript
// main.ts
import { provideHashbrown } from '@hashbrownai/angular';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(),
    provideHashbrown({
      baseUrl: '/api/chat',
    }),
  ],
});
```

The proxy receives `Chat.Api.CompletionCreateParams`, injects the API key, optionally overrides the model or system prompt (a cost-control lever), and streams the response back.

#### Using the chatResource

`chatResource` is an Angular resource that manages conversation lifecycle -- message history, streaming, and tool-call orchestration:

```typescript
// chat.component.ts
import { chatResource } from '@hashbrownai/angular';

@Component({
  selector: 'fin-assistant-chat',
  templateUrl: './chat.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class AssistantChatComponent {
  message = signal('');

  chat = chatResource({
    model: 'gpt-4.1-mini',
    system: `
      You are FinBot, a helpful assistant for a personal finance app.
      - Voice: clear, concise, respectful.
      - Only use the provided tools to look up data.
      - Never fabricate account numbers or balances.
    `,
    tools: [searchTransactionsTool, getAccountBalanceTool],
  });

  submit(): void {
    const text = this.message();
    this.message.set('');
    this.chat.sendMessage({ role: 'user', content: text });
  }
}
```

The `value` signal contains the message array. Each message carries a `role` and `content`; assistant messages may include a `toolCalls` array:

```html
<!-- chat.component.html -->
@for (msg of chat.value(); track $index) {
  <article [class]="'msg ' + msg.role">
    <div class="bubble">
      {{ msg.content }}
      @if (msg.role === 'assistant') {
        @for (tc of msg.toolCalls; track tc.toolCallId) {
          <span class="tool-badge">{{ tc.name }}</span>
        }
      }
    </div>
  </article>
}
```

`sendMessage()` appends the user message and fires a request. Because `chatResource` is stateful, every request carries the full conversation, so the model can resolve references like "that account" or "the same date range."

#### Providing Tools

Define tools with `createTool`, using Hashbrown's Skillet schema language to describe arguments:

```typescript
// tools/search-transactions.tool.ts
import { createTool } from '@hashbrownai/angular';
import { s } from '@hashbrownai/core';

export const searchTransactionsTool = createTool({
  name: 'searchTransactions',
  description: 'Searches for transactions matching the given filters.',
  schema: s.object('filters for the transaction search', {
    accountId: s.number('the account to search'),
    category: s.string('spending category, e.g. "groceries"').optional(),
    startDate: s.string('ISO date for range start').optional(),
    endDate: s.string('ISO date for range end').optional(),
  }),
  handler: async (input) => {
    const http = inject(HttpClient);
    const params: Record<string, string> = { accountId: String(input.accountId) };
    if (input.category) params['category'] = input.category;
    if (input.startDate) params['date_gte'] = input.startDate;
    if (input.endDate) params['date_lte'] = input.endDate;
    return firstValueFrom(http.get<Transaction[]>('/api/transactions', { params }));
  },
});

export const getAccountBalanceTool = createTool({
  name: 'getAccountBalance',
  description: 'Returns the current balance for a given account.',
  schema: s.object('account lookup', {
    accountId: s.number('ID of the account'),
  }),
  handler: async (input) => {
    const http = inject(HttpClient);
    const acct = await firstValueFrom(http.get<Account>(`/api/accounts/${input.accountId}`));
    return { balance: acct.balance, currency: acct.currency };
  },
});
```

The `description` on both the tool and each schema field is critical. The model reads these descriptions at inference time to decide which tool to invoke and how to populate its arguments. Vague descriptions lead to unreliable tool selection; specific ones produce consistent behavior.

Handlers run in the Angular injection context, so `inject()` works for services, stores, and the router. When a handler returns a value, Hashbrown includes it in a `tool` role message sent back to the model.

#### Under the Hood

Each request includes the full message history plus JSON Schema definitions derived from Skillet. The exchange follows a standard loop:

1. The frontend sends the user message.
2. The model responds with one or more `toolCalls`.
3. Hashbrown executes each handler and appends a `tool` role message with the result.
4. Hashbrown sends the updated history back to the model.
5. The model responds with a final `assistant` message incorporating the tool results.

Steps 2--4 may repeat if the model chains multiple tool calls. Hashbrown handles this loop internally -- the component only observes the `value` signal updating as messages arrive.

---

### Generative UI and Component Control

#### UI chat with uiChatResource

`uiChatResource` replaces `chatResource` when you want the model to respond with Angular components. The API is identical except for a required `components` array:

```typescript
chat = uiChatResource({
  model: 'gpt-4.1-mini',
  system: `
    You are FinBot. Answer every question using the provided components.
    Always include a messageWidget for textual context. When displaying
    transactions, use the transactionCard component instead of describing
    them in prose.
  `,
  tools: [searchTransactionsTool, getAccountBalanceTool],
  components: [transactionCardWidget, balanceSummaryWidget, messageWidget],
});
```

Render assistant messages with `hb-render-message`, which resolves structured output into live Angular components:

```html
@for (msg of chat.value(); track $index) {
  <article [class]="'msg ' + msg.role">
    @if (msg.role === 'assistant') {
      <hb-render-message [message]="msg" />
    } @else {
      <div class="bubble">{{ msg.content }}</div>
    }
  </article>
}
```

Because `uiChatResource` forces component-only responses, you need a catch-all `messageWidget` for plain text. Without it, the model cannot say "I didn't find any matching transactions."

#### Components for chat: Dumb Components with Smart Wrappers

The components registered with `uiChatResource` follow a clear separation. A *dumb* component handles pure display. A *smart wrapper* injects services and handles interactions:

```typescript
// transaction-card-widget.component.ts
@Component({
  selector: 'fin-transaction-card-widget',
  imports: [TransactionCardComponent, CurrencyPipe, DatePipe],
  template: `
    <fin-transaction-card [transaction]="transaction()">
      <button (click)="viewDetails()">View Details</button>
    </fin-transaction-card>
  `,
})
export class TransactionCardWidgetComponent {
  private router = inject(Router);

  transaction = input.required<Transaction>();

  viewDetails(): void {
    this.router.navigate(['/transactions', this.transaction().id]);
  }
}
```

The inner `TransactionCardComponent` is the same dumb component used elsewhere in the app. The widget wrapper bridges the gap between the model's structured output and the application's navigation layer.

#### Describing Components

Hashbrown's `exposeComponent` function pairs a component with a Skillet schema so the model knows what it can render and what inputs to provide:

```typescript
import { exposeComponent } from '@hashbrownai/angular';
import { s } from '@hashbrownai/core';

export const transactionCardWidget = exposeComponent(TransactionCardWidgetComponent, {
  name: 'transactionCard',
  description: 'Displays a single transaction. Use instead of describing transactions in text.',
  input: {
    transaction: s.object('the transaction to display', {
      id: s.number('transaction ID'),
      amount: s.number('transaction amount'),
      type: s.enumeration('credit or debit', ['credit', 'debit']),
      category: s.string('spending category'),
      date: s.string('ISO date string'),
      description: s.string('transaction description'),
    }),
  },
});

export const messageWidget = exposeComponent(MessageWidgetComponent, {
  name: 'messageWidget',
  description: 'Displays a text message to the user.',
  input: { data: s.string('the message text') },
});
```

#### Under the Hood: Structured Output

When components are registered, Hashbrown instructs the model to respond with JSON instead of plain text. The `content` of each assistant message becomes a JSON string containing a `ui` array -- each entry names a component and supplies its `$props`. `hb-render-message` deserializes this array, resolves each component by name, and renders it with the specified inputs. The developer never handles this parsing directly.

#### Supporting Different Models

Some providers (notably Google Gemini) do not support structured output combined with tool calling. Hashbrown provides `emulateStructuredOutput`, which rewrites component selection as a pseudo-tool call:

```typescript
provideHashbrown({
  baseUrl: '/api/chat',
  emulateStructuredOutput: true,
});
```

This makes `uiChatResource` compatible with any model that supports tool calling, even without native structured output.

#### Applying Few-Shot Prompting

Smaller models (Gemini Flash, GPT-4.1-mini) benefit from explicit examples in the system prompt. Without them, they may omit the `messageWidget`, malform inputs, or call the same tool repeatedly:

```typescript
chat = uiChatResource({
  model: 'gpt-4.1-mini',
  system: `
    You are FinBot, a financial assistant.

    ## Rules
    - Always respond with a messageWidget for textual context.
    - When transactions are available, also include transactionCard components.
    - Never call the same tool twice with identical parameters.

    ## Example
    - User: What did I spend on groceries last week?
    - Assistant:
      - Tool: searchTransactions({accountId: 1, category: "groceries",
              startDate: "2026-04-07", endDate: "2026-04-13"})
      - UI: messageWidget("Here are your grocery transactions.")
      - UI: transactionCard({transaction: {id: 10, amount: -45.20, ...}})

    ## Negative Example (do not repeat identical tool calls)
    - Tool: searchTransactions({accountId: 1, category: "groceries"})
    - Tool: searchTransactions({accountId: 1, category: "groceries"})
  `,
  tools: [searchTransactionsTool, getAccountBalanceTool],
  components: [transactionCardWidget, balanceSummaryWidget, messageWidget],
});
```

Providing one positive and one negative example -- *one-shot prompting* at minimum, *few-shot* when you add more -- significantly improves consistency. Embed examples in component descriptions too, especially for inputs the model must infer from context.

---

### Natural Language Queries with Code Generation

#### Approach

Tool calling and generative UI handle pre-defined operations well, but some requests are inherently open-ended: "show my spending by category this month as a pie chart" requires aggregation logic no single tool anticipates. The model is well-suited to *describe* these steps as code but poorly suited to *perform* the arithmetic itself.

Hashbrown separates these concerns. The model generates JavaScript that calls application-provided functions (`loadTransactions`, `generateChart`). A WebAssembly-based runtime executes the code in a sandbox with no direct DOM or service access -- the runtime functions are the only bridge.

#### Implementation with Hashbrown

`structuredCompletionResource` handles single-shot requests -- the user provides a query, the model responds with a structured object containing generated code, and a tool executes it in the runtime:

```typescript
import { createRuntime, createRuntimeFunction, createToolJavaScript,
  structuredCompletionResource } from '@hashbrownai/angular';

@Component({ selector: 'fin-report-builder', templateUrl: './report-builder.component.html' })
export class ReportBuilderComponent {
  query = signal('');
  input = signal<string | undefined>(undefined);
  chartData = signal<{ name: string; value: number }[]>([]);

  runtime = createRuntime({
    functions: [this.loadTransactionsFn(), this.generateChartFn()],
  });

  generator = structuredCompletionResource({
    model: 'gpt-4.1-mini',
    input: this.input,
    system: REPORT_SYSTEM_PROMPT,
    schema: s.object('code generation result', {
      type: s.enumeration('outcome', ['success', 'error']),
      message: s.string('user-facing feedback'),
      code: s.string('the generated JavaScript code'),
    }),
    tools: [createToolJavaScript({ runtime: this.runtime })],
  });

  submit(): void { this.input.set(this.query()); }
}
```

Unlike `chatResource`, this resource is stateless -- each submission produces a single structured response.

#### Runtime Functions

Runtime functions are the API surface visible to generated code. `loadTransactions` retrieves data; `generateChart` pushes results into a signal:

```typescript
private loadTransactionsFn() {
  return createRuntimeFunction({
    name: 'loadTransactions',
    description: 'Loads transactions for a given account and date range.',
    args: s.object('query parameters', {
      accountId: s.number('account ID'),
      startDate: s.string('ISO start date').optional(),
      endDate: s.string('ISO end date').optional(),
    }),
    result: s.array('matching transactions', TransactionSchema),
    handler: async (input) => {
      const http = inject(HttpClient);
      const params: Record<string, string> = { accountId: String(input.accountId) };
      if (input.startDate) params['date_gte'] = input.startDate;
      if (input.endDate) params['date_lte'] = input.endDate;
      return firstValueFrom(http.get<Transaction[]>('/api/transactions', { params }));
    },
  });
}

private generateChartFn() {
  return createRuntimeFunction({
    name: 'generateChart',
    description: 'Renders a chart from name/value pairs.',
    args: s.object('chart data', {
      data: s.array('data points', s.object('a single data point', {
        name: s.string('label for this slice or bar'),
        value: s.number('numeric value'),
      })),
    }),
    handler: async (input) => { this.chartData.set(input.data); },
  });
}
```

The `result` schema on `loadTransactions` tells the model what fields the returned data contains, so generated code can reference `t.category` and `t.amount` reliably. `generateChart` is a *sink* -- it writes processed data into the component's signal state.

#### System Prompt with One-Shot Prompting

The system prompt defines the code-generation contract with a concrete example:

```typescript
const REPORT_SYSTEM_PROMPT = `
You are FinBot-Reports, an assistant that generates charts from
financial data.

## Your Task
1. Read the user's request and generate JavaScript code that:
   a) Calls loadTransactions as many times as needed to retrieve data.
   b) Aggregates the data according to the request.
   c) Passes the result to generateChart as an array of {name, value} objects.
2. Submit the code to the JavaScript runtime.

## Example
- User: Show my spending by category this month.
- Code:
  const txns = loadTransactions({
    accountId: 1,
    startDate: '2026-04-01',
    endDate: '2026-04-30'
  });
  const totals = {};
  for (const t of txns) {
    if (t.type === 'debit') {
      totals[t.category] = (totals[t.category] || 0) + Math.abs(t.amount);
    }
  }
  const data = Object.entries(totals).map(([name, value]) => ({ name, value }));
  generateChart({ data });
- Message: Here is your spending breakdown by category.

## Rules
- Never access external resources. Only use the provided functions.
- Always pass generated code to the JavaScript runtime.
- Replace zero values with 0.1 so they remain visible in charts.
`;
```

This one-shot example anchors the model's output format. Without it, models often produce valid code that forgets to call `generateChart`. Add more examples (few-shot) if the model struggles with specific query patterns.

---

### Summary

Hashbrown provides three progressive levels of AI integration for Angular applications. `chatResource` delivers reactive chat with tool calling -- the model invokes application functions like `searchTransactions` and incorporates live results into its responses. `uiChatResource` extends this with generative UI, letting the model select and populate Angular components at runtime through structured output. `structuredCompletionResource` paired with `createRuntime` handles open-ended analytical queries by generating JavaScript code that executes in a sandboxed WebAssembly environment.

Across all three levels, the model never has direct access to application internals. It operates through described interfaces -- tool schemas, component schemas, and runtime function signatures -- that the developer controls. This boundary is what makes agentic UI practical rather than chaotic.

Where Hashbrown embeds intelligence into the product your users interact with, the next part of this chapter looks at the other side of the coin: embedding intelligence into the workflow you use to build that product.

---

## Part 2 -- Engineering AI

Part 1 embedded intelligence inside the FinancialApp -- an assistant that streams Angular components, calls tools to query transaction data, and generates charts from natural language. That is one layer of AI in Angular: runtime features your users interact with. There is a second layer that never ships to production but shapes every file that does: developer-time AI tooling that helps you build the application itself.

This part covers that second layer. We will configure AI coding assistants to generate accurate, idiomatic Angular v21 code. We will wire the Angular CLI's MCP server into Cursor so the agent can query documentation, list workspace projects, and retrieve best practices without hallucinating outdated patterns. We will install Angular Agent Skills that give the assistant deep domain knowledge about component architecture and project scaffolding. And we will explore signal-based design patterns -- `resource`, `linkedSignal`, streaming -- that make AI-driven features reactive and composable within Angular's change detection model.

> **Companion code:** `.cursor/rules/angular.mdc` and MCP server configurations live in `financial-app/`. Angular Agent Skills are installed at the workspace level.

---

### Angular's "Build with AI" Ecosystem

The Angular team maintains a dedicated section at [angular.dev/ai](https://angular.dev/ai) with resources for integrating AI assistants into your development workflow. The centerpiece is a best-practices prompt -- a condensed set of rules that steer language models toward correct Angular v21 code generation.

#### The Best Practices Prompt

Without explicit guidance, language models generate Angular code trained on years of Stack Overflow answers, blog posts, and tutorials spanning every Angular version since 2016. The result is a grab bag of patterns: `NgModule` declarations, `@Input()` decorators, Zone.js assumptions, `*ngIf` structural directives. The best-practices prompt eliminates this drift by giving the model a concise, authoritative ruleset.

The key rules include:

- **Standalone by default.** All components, directives, and pipes are standalone. Do not add `standalone: true` to the decorator -- it is the default in Angular v21 and including it is noise.
- **Signal-first reactivity.** Use `signal()`, `computed()`, `linkedSignal()`, and `resource()` for state management. Use `input()`, `output()`, and `model()` instead of `@Input()`, `@Output()`, and two-way binding decorators.
- **Dependency injection with `inject()`.** Prefer `inject()` over constructor injection. It works in any injection context and composes better with functional patterns.
- **OnPush change detection.** Set `changeDetection: ChangeDetectionStrategy.OnPush` on every component. Signal reads automatically trigger change detection in the correct scope.
- **Built-in control flow.** Use `@if`, `@else`, `@for`, `@switch` instead of `*ngIf`, `*ngFor`, and `ngSwitch` directives.
- **`NgOptimizedImage` for images.** Use the `NgOptimizedImage` directive with `width` and `height` attributes for all `<img>` elements.
- **No `ngOnInit` for data fetching.** Use `resource()` or `rxResource()` to tie data loading to signal-derived request parameters.

#### Where to Put the Prompt

The prompt is only useful if your AI assistant sees it on every interaction. Where you place it depends on your editor:

**Cursor** stores instructions in `.cursor/rules/` as `.mdc` files. Rules can be scoped to glob patterns, so an Angular rule applies only to `.ts`, `.html`, and `.scss` files.

**VS Code with Copilot** uses `.github/copilot-instructions.md` at the repository root. The file supports markdown with code blocks, and Copilot includes it as system context for every chat and inline completion.

**JetBrains IDEs** accept custom AI instructions in the Settings > AI Assistant > Custom Instructions panel, or through a `.junie/guidelines.md` file in the project root.

The specific file format varies, but the content is the same: the Angular best-practices prompt, plus any project-specific conventions you want enforced.

---

### Cursor Rules for Angular

Cursor's rule system goes beyond a single prompt file. The `.cursor/rules/` directory supports multiple `.mdc` files, each with frontmatter that controls when the rule activates. For the FinancialApp, we create a rule that fires whenever the agent works on Angular source files.

#### The Rule File

```markdown
---
description: Angular v21 conventions for FinancialApp
globs: ["**/*.ts", "**/*.html", "**/*.scss"]
alwaysApply: false
---

# Angular v21 — FinancialApp Conventions

## Component Patterns
- All components are standalone (do not add `standalone: true` — it is the default).
- Use `changeDetection: ChangeDetectionStrategy.OnPush` on every component.
- Use signal inputs: `input()`, `input.required()`. Never use `@Input()`.
- Use signal outputs: `output()`. Never use `@Output()` or `EventEmitter`.
- Use `model()` for two-way binding instead of the input/output pair pattern.
- Use `inject()` for dependency injection. Do not use constructor parameters.
- Use `viewChild()`, `viewChildren()`, `contentChild()`, `contentChildren()` instead of decorator-based queries.

## State & Reactivity
- Prefer `signal()` and `computed()` over BehaviorSubject.
- Use `linkedSignal()` when a signal needs to reset based on another signal.
- Use `resource()` for async data fetching tied to signal parameters.
- Use `rxResource()` only when interoperating with existing RxJS streams.

## Templates
- Use `@if` / `@else if` / `@else`, `@for` (with `track`), `@switch` / `@case`.
- Never use `*ngIf`, `*ngFor`, `ngSwitch`, or `<ng-template [ngIf]>`.
- Use `NgOptimizedImage` for all `<img>` elements with explicit width/height.

## File Naming
- Components: `feature-name.component.ts`
- Services: `feature-name.service.ts`
- Models/interfaces: `feature-name.model.ts`
- Guards/interceptors: `feature-name.guard.ts`, `feature-name.interceptor.ts`
- Place feature components under `features/<feature>/` directories.

## Design System
- Wrap PrimeNG components in `fin-*` prefixed wrapper components.
- Wrapper components live in `libs/shared/ui/`.
- Never import PrimeNG directly in feature components.

## Testing
- Use Vitest with `@analogjs/vitest-angular` for unit tests.
- Test files are co-located: `feature-name.component.spec.ts`.
- Use `render()` from `@testing-library/angular` for component tests.
- Prefer testing behavior over implementation details.
```

Save this as `financial-app/.cursor/rules/angular.mdc`. The `globs` frontmatter ensures the rule activates when the agent touches TypeScript, HTML, or SCSS files -- but not when it is editing `package.json`, CI configuration, or markdown documentation. The `alwaysApply: false` setting means the rule is included only when glob patterns match, keeping the context window clean for unrelated tasks.

Project-specific rules like the design system wrapper convention and the PrimeNG constraint are what distinguish a useful rule from a generic prompt. The Angular best-practices prompt tells the model *how to write Angular*. The Cursor rule tells it *how to write Angular for this project*.

---

### Angular CLI MCP Server

#### What MCP Is

The Model Context Protocol (MCP) is a standardized interface for AI agents to discover and invoke tools exposed by external servers. Instead of baking knowledge into prompts, MCP lets an agent call a tool at runtime -- querying live documentation, listing projects in a workspace, or retrieving version-specific best practices. The agent receives structured data it can reason about, rather than relying on memorized (and potentially outdated) training data.

#### Setting Up the Angular CLI MCP Server

Angular v21 ships an MCP server as part of the CLI. Enable it with a single command:

```bash
ng mcp
```

This prints the configuration block you need. For Cursor, add it to `.cursor/mcp.json` at the workspace root:

```json
{
  "mcpServers": {
    "angular-cli": {
      "command": "npx",
      "args": ["-y", "@angular/mcp@latest", "--read-only"]
    }
  }
}
```

The `--read-only` flag restricts the server to tools that do not modify your project -- documentation queries, project listing, best practices retrieval. This is the recommended starting point. For VS Code, the equivalent goes in `.vscode/mcp.json`. JetBrains IDEs, Firebase Studio, and Gemini CLI each have their own configuration surface, but the server command is identical.

#### Available Tools

The MCP server exposes six stable tools:

**`get_best_practices`** retrieves the Angular best-practices guide, the same content discussed earlier in this chapter but delivered dynamically. The agent calls this tool when it needs to verify a pattern rather than guessing from training data.

**`search_documentation`** searches angular.dev and returns relevant documentation sections. When the agent needs to understand `resource()` options or `provideRouter` configuration, it queries the live docs instead of hallucinating an API.

**`find_examples`** searches a curated database of code examples. This is particularly useful for patterns that are hard to describe in prose -- the agent retrieves a working example and adapts it to your codebase.

**`list_projects`** lists all Angular projects in the current workspace, including their types (application, library), build targets, and source roots. In the FinancialApp's Nx workspace, this returns the main application and every shared library.

**`onpush_zoneless_migration`** analyzes your codebase and produces a migration plan for adopting `OnPush` change detection and removing Zone.js. It identifies components still using the default change detection strategy and flags Zone.js-dependent patterns.

**`ai_tutor`** provides an interactive learning interface. It is less relevant for experienced developers but valuable for teams onboarding junior engineers who interact with AI assistants frequently.

#### Experimental Tools

Passing `--experimental-tool` unlocks additional capabilities that modify the project:

```json
{
  "mcpServers": {
    "angular-cli": {
      "command": "npx",
      "args": [
        "-y", "@angular/mcp@latest",
        "--experimental-tool", "build",
        "--experimental-tool", "devserver.start",
        "--experimental-tool", "devserver.stop",
        "--experimental-tool", "devserver.wait_for_build",
        "--experimental-tool", "test",
        "--experimental-tool", "e2e",
        "--experimental-tool", "modernize"
      ]
    }
  }
}
```

With these enabled, the agent can start and stop the dev server, trigger builds, run test suites, and apply automated modernization codemods -- all within the chat conversation. The `modernize` tool is especially powerful: it applies the same migrations that `ng update` performs, but scoped to specific patterns the agent identifies during a refactoring session.

Use `--local-only` if you want to prevent the server from making any network requests (no documentation search, no example fetching). This is useful in air-gapped environments or when you want deterministic, offline-only behavior.

You can combine flags: `--read-only --experimental-tool test` gives the agent permission to run tests but not to start dev servers or modify project files. Start conservative and expand access as you build trust in the workflow.

---

### Angular Agent Skills

#### What Skills Are

Agent Skills are a layer above prompts and MCP tools. A skill is a structured set of domain-specific instructions -- a `SKILL.md` file -- that an AI agent reads and follows when performing a particular type of task. Where a Cursor rule says "use `OnPush` everywhere," a skill says "here is how to architect a new feature module in Angular, step by step, including the file structure, the route configuration, the service layer, and the test setup."

#### Official Angular Skills

The Angular team publishes two official skills:

**`angular-developer`** covers code generation and architecture guidance. When the agent is asked to create a component, add a service, or refactor a feature module, this skill provides the step-by-step procedure -- ensuring the result follows Angular v21 conventions, uses the correct file structure, and wires up routes and providers correctly.

**`angular-new-app`** handles project creation from scratch. It guides the agent through `ng new` configuration, initial dependency installation, environment setup, and baseline architecture scaffolding.

#### Installing Skills

Skills are installed with the `skills` CLI:

```bash
npx skills add https://github.com/angular/skills
```

This clones the skill definitions into your local skills directory (typically `~/.cursor/skills/` or a workspace-level `.skills/` directory, depending on your configuration). Once installed, the agent discovers and reads the skill file automatically when it encounters a matching task.

#### Community Skills and Custom Authoring

The skills ecosystem is open. Community-maintained skills cover frameworks, libraries, and workflows beyond Angular's official set. You can also author custom skills for your organization using the `skills.sh` platform or by creating `SKILL.md` files directly.

A custom skill for the FinancialApp might encode the wrapper-component pattern for PrimeNG integration, the Vitest testing conventions, or the specific state management approach using signal stores. The skill file is prose, not code -- it describes the *procedure* the agent should follow, referencing project conventions and file paths.

The distinction between a Cursor rule and a skill is scope. A rule is a set of constraints the agent applies passively -- "always use `OnPush`," "never import PrimeNG directly." A skill is an active procedure the agent follows when it recognizes a matching task -- "when asked to create a new feature, follow these seven steps." Rules shape every interaction; skills activate on demand.

---

### AI Design Patterns in Angular

The patterns in this section apply whether you are building a chat interface, a search assistant, or any feature that calls an AI backend. They use Angular's signal-based primitives to manage the lifecycle of AI requests: triggering, accumulating, streaming, and rendering.

#### Triggering Requests with Signals

A common mistake is triggering an AI request on every keystroke. The `resource` API solves this with a two-signal pattern: one signal holds the current input, a second holds the *submitted* input. The resource reads only from the second signal.

```typescript
// features/assistant/prompt.component.ts
@Component({
  selector: 'fin-prompt',
  template: `
    <textarea [ngModel]="draft()" (ngModelChange)="draft.set($event)"></textarea>
    <button (click)="submit()" [disabled]="pending()">Send</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class PromptComponent {
  private readonly http = inject(HttpClient);

  readonly draft = signal('');
  readonly submitted = signal<string | undefined>(undefined);
  readonly pending = signal(false);

  readonly response = resource({
    params: () => {
      const prompt = this.submitted();
      return prompt ? { prompt } : undefined;
    },
    loader: async ({ params }) => {
      this.pending.set(true);
      const result = await firstValueFrom(
        this.http.post<{ answer: string }>('/api/assistant', params)
      );
      this.pending.set(false);
      return result.answer;
    },
  });

  submit(): void {
    this.submitted.set(this.draft());
    this.draft.set('');
  }
}
```

Typing in the textarea updates `draft`, but the resource does not fire. Only when the user clicks Send does `submit()` copy the value into `submitted`, which `resource`'s `params` function reads. This prevents wasteful API calls and gives the user explicit control over when a request is sent.

#### Preparing LLM Data for Templates

Chat interfaces accumulate messages over time. Each response from the model should append to the conversation history, not replace it. `linkedSignal` bridges this by letting you derive a signal's value from another signal while preserving the ability to imperatively update it:

```typescript
// features/assistant/chat.store.ts
readonly latestResponse = signal<AssistantMessage | null>(null);

readonly messages = linkedSignal<AssistantMessage | null, AssistantMessage[]>({
  source: this.latestResponse,
  computation: (newMsg, previous) => {
    const history = previous?.value ?? [];
    return newMsg ? [...history, newMsg] : history;
  },
});
```

When `latestResponse` updates with a new message from the model, `linkedSignal` appends it to the existing array. The template reads `messages()` and always sees the full conversation. If you need to clear the history -- starting a new conversation, for example -- call `messages.set([])` directly.

#### Streaming Responses

For long-running model responses, streaming chunks to the UI as they arrive is a better experience than waiting for the complete answer. Angular's `resource` API supports a stream-based loader with `ResourceStreamItem<T>`:

```typescript
// features/assistant/streaming-prompt.component.ts
readonly streamedAnswer = resource({
  params: () => {
    const prompt = this.submitted();
    return prompt ? { prompt } : undefined;
  },
  stream: true,
  loader: async function* ({ params }) {
    const response = await fetch('/api/assistant/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(params),
    });
    const reader = response.body!.getReader();
    const decoder = new TextDecoder();
    let accumulated = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      accumulated += decoder.decode(value, { stream: true });
      yield { value: accumulated };
    }
  },
});
```

The `loader` is an async generator. Each `yield` pushes a `ResourceStreamItem` that updates the resource's `value()` signal. The template re-renders progressively as chunks arrive, giving the user immediate feedback. The `accumulated` variable concatenates chunks so the template always displays the full response so far, not just the latest fragment.

#### Template Patterns for AI Content

AI responses have three states -- loading, success, and error -- and the template should handle all three explicitly:

```html
<!-- features/assistant/response.component.html -->
@if (response.isLoading()) {
  <div class="skeleton-block" aria-busy="true">Thinking...</div>
} @else if (response.error()) {
  <div class="error-banner" role="alert">
    <p>Something went wrong. Please try again.</p>
    <button (click)="response.reload()">Retry</button>
  </div>
} @else if (response.hasValue()) {
  <div class="response-body">
    {{ response.value() }}
  </div>
}
```

The `reload()` method on `resource` re-executes the loader with the same parameters -- a natural fit for retry-after-error. For scoped loading indicators, place the `resource` in the component that renders the response rather than in a global service. This keeps the loading state local, so other parts of the UI remain interactive while the AI request is in flight. For SSR considerations, AI-generated content should be wrapped in a `@defer` block or rendered only on the client, since model responses are non-deterministic and should not be server-rendered into the initial HTML payload. See [Chapter 22](ch31-defer-ssr-hydration.md) for hydration strategies and [Chapter 40](ch19-error-handling.md) for broader error and retry patterns.

---

### Nx MCP Server

The Angular CLI MCP server focuses on Angular-specific concerns: documentation, best practices, project structure. In an Nx monorepo like the FinancialApp, the **Nx MCP server** complements it with workspace-level tooling.

The Nx MCP server provides tools organized around several concerns:

**Workspace exploration.** The agent can query the project graph to understand which libraries exist, how they depend on each other, and what targets each project supports. In the FinancialApp, this means the agent knows that `shared/ui` depends on `shared/models` before it tries to add an import in the wrong direction.

**Documentation retrieval.** Like the Angular CLI MCP's `search_documentation`, the Nx MCP provides access to Nx-specific docs -- generator options, executor configuration, `nx.json` schema, and caching behavior.

**Task monitoring.** The agent can inspect running tasks, view their output, and check build status. During a refactoring session, it can kick off `nx affected -t test` and monitor which tests pass or fail without leaving the chat.

**CI analytics.** For workspaces connected to Nx Cloud, the MCP exposes pipeline performance data -- task durations, cache hit rates, and historical trends. The agent can identify which projects are slowing down CI and suggest targeted optimizations.

**Generator discovery.** The agent can list available generators and their options, then invoke them to scaffold new libraries, components, or configuration files following the workspace's conventions.

Setup is automatic if you use the Nx Console extension for Cursor or VS Code -- the extension registers the MCP server on your behalf. For manual configuration:

```json
{
  "mcpServers": {
    "nx": {
      "command": "npx",
      "args": ["nx-mcp@latest"]
    }
  }
}
```

The two servers -- Angular CLI and Nx -- coexist without conflict. The agent queries whichever server is appropriate for the task at hand. When generating a new feature component, it pulls Angular best practices from the CLI MCP and workspace structure from the Nx MCP. When analyzing CI performance, only the Nx MCP is involved.

For a deeper treatment of Nx workspace management, task orchestration, and CI optimization, see [Chapter 37](ch15-advanced-nx.md).

---

### Storybook MCP

If the FinancialApp uses Storybook for its component library -- as discussed in [Chapter 28](ch26-storybook.md) -- the **Storybook MCP server** exposes that library's context to AI agents. The agent can query which components exist, what their inputs and variants are, and how they are documented in stories.

The practical benefit is component reuse. Without Storybook MCP, an agent asked to "add a transaction summary card" might generate a new component from scratch. With it, the agent discovers that `fin-transaction-card` already exists in the shared UI library, reads its story to understand its API, and reuses it -- or extends it with a new variant rather than duplicating it.

This matters more than it sounds. Duplication is the most common failure mode in AI-assisted codebases. The agent does not inherently know what your project already contains -- it only knows what it has been told or can query. MCP servers are the query mechanism. Without them, the agent relies on file search and pattern matching, which is slow and incomplete. With Storybook MCP, the agent has a structured, searchable catalog of every component in the design system, complete with documentation and usage examples.

This closes a loop in the AI-assisted workflow: Cursor rules define the conventions, the Angular CLI MCP provides framework knowledge, the Nx MCP provides workspace structure, and the Storybook MCP provides the component catalog. Each server contributes a different facet of project context, and the agent synthesizes them into coherent, project-aware code generation.

---

### Summary

AI tooling for Angular operates at two distinct layers. The runtime layer -- Part 1's generative UI, tool calling, and code generation -- embeds intelligence into the product your users interact with. The developer-time layer covered in this part embeds intelligence into the workflow you use to build that product.

**Cursor rules** encode project-specific conventions in `.cursor/rules/angular.mdc`, ensuring the agent generates code that matches your architecture, file naming, testing patterns, and design system constraints.

**The Angular CLI MCP server** gives agents structured access to live documentation, best practices, code examples, and workspace metadata through the Model Context Protocol. Its stable tools (`get_best_practices`, `search_documentation`, `find_examples`, `list_projects`, `onpush_zoneless_migration`, `ai_tutor`) keep the agent grounded in current Angular patterns rather than outdated training data. Experimental tools extend this to builds, dev server management, and automated modernization.

**Angular Agent Skills** provide procedural domain knowledge -- step-by-step instructions for scaffolding applications, creating feature modules, and following architectural patterns. The official `angular-developer` and `angular-new-app` skills cover the most common workflows; custom skills encode your organization's specific practices.

**Signal-based AI patterns** -- the two-signal trigger for `resource`, `linkedSignal` for accumulating chat history, async generator streaming, and explicit loading/error/success templates -- give AI features the same reactive, composable behavior as the rest of an Angular application.

**Nx MCP** and **Storybook MCP** round out the tooling layer with workspace-level intelligence and component catalog awareness, ensuring the agent understands not just Angular conventions but the specific structure and assets of your project.

The through-line across all of these tools is *context*. A language model without project context generates plausible but generic code. A model with Cursor rules, MCP servers, and agent skills generates code that fits your architecture, follows your conventions, reuses your components, and respects your dependency boundaries. The investment in configuring this context pays for itself on the first refactoring session.

---

## Bridging the two

The runtime and engineering halves of this chapter are converging on the same substrate. The Hashbrown tools in Part 1 -- `searchTransactions`, `getAccountBalance`, `loadTransactions` -- expose application capabilities to a language model through typed, schema-described interfaces. The Angular CLI MCP server in Part 2 does the exact same thing for the development workspace: typed tools, schema-described, invoked by an LLM over a standard protocol. As MCP matures, the boundary between "tools my coding agent calls at build time" and "tools my user-facing assistant calls at runtime" is dissolving. Teams increasingly ship a single catalogue of tools, served over MCP and consumed by whichever agent needs them. Investing in clear schemas, precise descriptions, and well-scoped handlers today pays off on both sides of that line.
