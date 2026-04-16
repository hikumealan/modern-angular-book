# Chapter 13: Agentic UI & AI Assistants with Hashbrown

> **Stability warning -- read before adopting.**
> Hashbrown is a niche, community-driven library maintained by LiveLoveApp. Its API surface
> is still evolving rapidly. The `chatResource`, `uiChatResource`, and `structuredCompletionResource`
> functions documented here reflect the library at the time of writing (spring 2026) and may
> change in future releases. Pin your dependency versions, monitor the
> [Hashbrown changelog](https://hashbrown.dev), and write integration tests around the resource
> APIs you depend on. For the framework-native AI integration point, see
> [Angular's CLI MCP Server](https://angular.dev/ai/mcp), which ships six stable tools and
> several experimental ones in Angular v21. The MCP Server gives AI coding assistants direct
> access to your workspace structure, documentation, and best-practice guides -- a complementary
> concern to the runtime UI integration that Hashbrown provides.

A financial dashboard that answers "show my spending by category this month" by rendering a real chart component -- not a blob of markdown -- is qualitatively different from one that opens a separate chat tab. The pattern is called *generative UI*: the language model selects and populates Angular components at runtime.

This chapter explores three levels of integration using Hashbrown: text chat with tool calling, generative UI that streams real components into the conversation, and on-the-fly code generation for ad-hoc analytical queries. All three build on the signal-based component model from [Chapter 2](ch02-signal-components.md) and the reactive patterns from [Chapter 3](ch03-reactive-signals.md).

> **Companion code:** `financial-app/apps/financial-app/src/app/features/assistant/chat.component.ts`

---

## Implementing an Assistant with Tool Calling

### Setting up Hashbrown

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

### Using the chatResource

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

### Providing Tools

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

### Under the Hood

Each request includes the full message history plus JSON Schema definitions derived from Skillet. The exchange follows a standard loop:

1. The frontend sends the user message.
2. The model responds with one or more `toolCalls`.
3. Hashbrown executes each handler and appends a `tool` role message with the result.
4. Hashbrown sends the updated history back to the model.
5. The model responds with a final `assistant` message incorporating the tool results.

Steps 2--4 may repeat if the model chains multiple tool calls. Hashbrown handles this loop internally -- the component only observes the `value` signal updating as messages arrive.

---

## Generative UI and Component Control

### UI chat with uiChatResource

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

### Components for chat: Dumb Components with Smart Wrappers

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

### Describing Components

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

### Under the Hood: Structured Output

When components are registered, Hashbrown instructs the model to respond with JSON instead of plain text. The `content` of each assistant message becomes a JSON string containing a `ui` array -- each entry names a component and supplies its `$props`. `hb-render-message` deserializes this array, resolves each component by name, and renders it with the specified inputs. The developer never handles this parsing directly.

### Supporting Different Models

Some providers (notably Google Gemini) do not support structured output combined with tool calling. Hashbrown provides `emulateStructuredOutput`, which rewrites component selection as a pseudo-tool call:

```typescript
provideHashbrown({
  baseUrl: '/api/chat',
  emulateStructuredOutput: true,
});
```

This makes `uiChatResource` compatible with any model that supports tool calling, even without native structured output.

### Applying Few-Shot Prompting

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

## Natural Language Queries with Code Generation

### Approach

Tool calling and generative UI handle pre-defined operations well, but some requests are inherently open-ended: "show my spending by category this month as a pie chart" requires aggregation logic no single tool anticipates. The model is well-suited to *describe* these steps as code but poorly suited to *perform* the arithmetic itself.

Hashbrown separates these concerns. The model generates JavaScript that calls application-provided functions (`loadTransactions`, `generateChart`). A WebAssembly-based runtime executes the code in a sandbox with no direct DOM or service access -- the runtime functions are the only bridge.

### Implementation with Hashbrown

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

### Runtime Functions

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

### System Prompt with One-Shot Prompting

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

## Summary

Hashbrown provides three progressive levels of AI integration for Angular applications. `chatResource` delivers reactive chat with tool calling -- the model invokes application functions like `searchTransactions` and incorporates live results into its responses. `uiChatResource` extends this with generative UI, letting the model select and populate Angular components at runtime through structured output. `structuredCompletionResource` paired with `createRuntime` handles open-ended analytical queries by generating JavaScript code that executes in a sandboxed WebAssembly environment.

Across all three levels, the model never has direct access to application internals. It operates through described interfaces -- tool schemas, component schemas, and runtime function signatures -- that the developer controls. This boundary is what makes agentic UI practical rather than chaotic.

For AI integration at the *development tooling* layer rather than the runtime UI layer, Angular v21's CLI MCP Server provides a complementary story. Its stable tools (`list_projects`, `get_best_practices`, `search_documentation`, `find_examples`, `ai_tutor`, `onpush_zoneless_migration`) give AI coding assistants structured access to your workspace and documentation. Where Hashbrown embeds intelligence into the product your users interact with, the MCP Server embeds intelligence into the workflow you use to build it.
