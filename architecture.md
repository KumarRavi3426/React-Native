# Welth — Full Architecture & Interview Prep Guide

> **App name:** Welth | **Stack:** React Native + Expo 58, Supabase, Clerk, Gemini AI
> **Platform:** Android & iOS (New Architecture enabled)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack & Why Each Was Chosen](#2-tech-stack--why-each-was-chosen)
3. [Folder Structure](#3-folder-structure)
4. [Navigation Architecture (Expo Router)](#4-navigation-architecture-expo-router)
5. [Authentication Flow (Clerk)](#5-authentication-flow-clerk)
6. [Database & Backend (Supabase)](#6-database--backend-supabase)
7. [State Management Architecture](#7-state-management-architecture)
8. [Data Fetching Layer (TanStack Query)](#8-data-fetching-layer-tanstack-query)
9. [AI Features (Gemini API)](#9-ai-features-gemini-api)
10. [Supabase Edge Functions (Backend Jobs)](#10-supabase-edge-functions-backend-jobs)
11. [Form Handling & Validation](#11-form-handling--validation)
12. [Core Domain Types](#12-core-domain-types)
13. [Screen-by-Screen Breakdown](#13-screen-by-screen-breakdown)
14. [Key Design Decisions & Trade-offs](#14-key-design-decisions--trade-offs)
15. [Interview Q&A — Ready Answers](#15-interview-qa--ready-answers)

---

## 1. Project Overview

**Welth** is a cross-platform personal finance tracker with AI-powered transaction entry. Users can:

- Track income & expenses across multiple accounts (Cash, Bank, Credit Card, Savings)
- Log transactions **three ways**: manually, by scanning a receipt photo (AI), or by voice note (AI)
- Set a monthly budget and receive email alerts at 80% and 100% usage
- Ask a conversational AI assistant questions about their finances
- Export the last 30 days of transactions as a CSV
- Receive weekly AI-generated money tips by email

---

## 2. Tech Stack & Why Each Was Chosen

| Technology | Role | Why |
|---|---|---|
| **React Native 0.88** | UI framework | Cross-platform iOS + Android from one codebase |
| **Expo 58 (New Arch)** | Build tooling + native modules | Managed workflow, EAS builds, native module access |
| **Expo Router** | File-based navigation | Same mental model as Next.js; typed routes; layout nesting |
| **Clerk** | Authentication | Passwordless, OAuth, JWT issuing; integrates with Supabase RLS |
| **Supabase** | Postgres DB + Edge Functions | Row-Level Security by user; real-time capable; hosted Postgres |
| **TanStack Query v5** | Server state / data fetching | Caching, background refetch, mutation + cache invalidation |
| **Zustand v5** | Client state | Minimal, no boilerplate; used for session-scoped UI state |
| **Gemini API (Flash Lite)** | AI extraction & chat | Fast, cheap model; structured JSON output via `responseSchema` |
| **NativeWind v4** | Styling | Tailwind CSS utility classes in React Native |
| **React Hook Form + Zod** | Form validation | Schema-first validation, type-safe form values |
| **Resend** | Transactional email | Used inside Supabase Edge Functions |
| **react-native-gifted-charts** | Data visualization | Pie chart for category breakdown |
| **date-fns v4** | Date utilities | Lightweight, tree-shakeable |
| **expo-file-system + expo-sharing** | CSV export | Write to cache dir; native share sheet |

---

## 3. Folder Structure

```
React-Native/
├── app/                        # Expo Router screens (file-based routes)
│   ├── _layout.tsx             # Root layout — providers
│   ├── index.tsx               # Entry redirect
│   ├── (auth)/                 # Public auth screens
│   │   ├── _layout.tsx
│   │   ├── sign-in.tsx
│   │   └── sign-up.tsx
│   └── (root)/                 # Authenticated screens
│       ├── _layout.tsx         # Auth guard + onboarding redirect
│       ├── onboarding.tsx      # First-launch setup
│       └── (tabs)/             # Bottom tab navigator
│           ├── _layout.tsx
│           ├── index.tsx       # Home dashboard
│           ├── transactions.tsx
│           ├── add-transaction.tsx
│           ├── assistant.tsx
│           └── profile.tsx
│
├── components/                 # Shared UI components
├── constants/
│   ├── categories.ts           # All 15 expense + 6 income categories
│   └── theme.ts
├── hooks/
│   ├── useSupabase.ts          # Creates Clerk-authed Supabase client
│   ├── useUserSync.ts          # Syncs Clerk user → Supabase DB on mount
│   ├── queries/                # TanStack Query read hooks
│   │   ├── useAccountsQuery.ts
│   │   ├── useTransactionsQuery.ts
│   │   └── useBudgetQuery.ts
│   └── mutations/              # TanStack Query write hooks
│       ├── useTransactionMutations.ts
│       ├── useAccountMutations.ts
│       └── useBudgetMutations.ts
│
├── lib/
│   ├── supabase.ts             # Supabase client factory
│   ├── query/
│   │   ├── client.ts           # QueryClient singleton
│   │   └── keys.ts             # Typed query key factory
│   ├── schemas/                # Zod validation schemas
│   │   ├── transaction.ts
│   │   └── onboarding.ts
│   ├── services/               # Pure async service functions
│   │   ├── accounts.ts
│   │   ├── transactions.ts
│   │   ├── budgets.ts
│   │   ├── assistant.ts        # Gemini AI chat
│   │   └── extractTransaction.ts  # Gemini receipt + voice extraction
│   └── utils/
│       ├── index.ts            # formatPrice etc.
│       └── exportTransactions.ts
│
├── store/
│   └── userStore.ts            # Zustand: currency + onboarding flag
│
├── supabase/
│   └── functions/
│       ├── check-budget-alerts/index.ts   # Deno edge fn — daily budget emails
│       ├── weekly-tips/index.ts           # Deno edge fn — weekly Gemini tips
│       └── _shared/
│           ├── supabaseAdmin.ts
│           ├── resend.ts
│           └── emailLayout.ts
│
├── types/                      # Shared TypeScript types
│   ├── account.ts
│   ├── transaction.ts
│   └── budget.ts
│
└── notes/                      # Developer notes (you are here)
```

---

## 4. Navigation Architecture (Expo Router)

Expo Router turns the `app/` folder into a file-system router (like Next.js for mobile).

### Route Tree

```
/                       → app/index.tsx (redirects based on auth)
  (auth)/               → Public, unauthenticated
    sign-in
    sign-up
  (root)/               → Protected, requires Clerk sign-in
    onboarding          → Shown if user hasn't set currency yet
    (tabs)/             → Bottom tab bar
      index             → Home dashboard
      transactions      → Full transaction list
      add-transaction   → Add new transaction (3 input modes)
      assistant         → AI chat
      profile           → User settings & accounts
```

### Group folders `(auth)` and `(root)`

Parentheses `()` make a folder a **route group** — the folder name does not appear in the URL. Each group has its own `_layout.tsx` that wraps child screens.

### Auth Guard — `app/(root)/_layout.tsx`

This is the most important layout. It enforces three rules **in order**:

```
1. Not loaded?         → show nothing (Clerk still initializing)
2. Not signed in?      → redirect to /sign-in
3. Needs onboarding?   → redirect to /onboarding
4. Already onboarded + on /onboarding? → redirect to tabs
5. Otherwise           → render <Slot /> (the matched child screen)
```

The `needsOnboarding` flag is a tri-state: `null` (unknown), `true`, `false`. It starts as `null` so we don't redirect prematurely before the user record is checked.

### Tab Bar — Platform-Adaptive

```tsx
// iOS: NativeTabs (expo-router/unstable-native-tabs) — SF Symbols
// Android: Tabs (standard JS tab bar) — Feather icons
const useNativeTabs = Platform.OS === "ios";
```

5 tabs: Home, Transactions, Add, Assistant, Profile.

---

## 5. Authentication Flow (Clerk)

### Setup (`app/_layout.tsx` — Root Provider)

```tsx
<GestureHandlerRootView>
  <QueryClientProvider client={queryClient}>
    <ClerkProvider publishableKey={...} tokenCache={tokenCache}>
      <Slot />
    </ClerkProvider>
  </QueryClientProvider>
</GestureHandlerRootView>
```

`tokenCache` uses `expo-secure-store` to persist the Clerk session token securely on-device (Keychain on iOS, EncryptedSharedPreferences on Android).

### Clerk ↔ Supabase JWT Bridge

The key integration: Clerk issues a JWT, Supabase uses it to enforce Row-Level Security.

```ts
// lib/supabase.ts
export function createClerkSupabaseClient(getToken: () => Promise<string | null>) {
  return createClient(supabaseUrl, supabaseAnonKey, {
    async accessToken() {
      return getToken(); // Clerk JWT is passed as the Supabase auth token
    },
  });
}
```

```ts
// hooks/useSupabase.ts
export function useSupabase() {
  const { getToken } = useAuth(); // Clerk hook
  const client = useMemo(
    () => createClerkSupabaseClient(() => getToken()),
    []
  );
  return client;
}
```

**How RLS works:** Supabase's RLS policies check `auth.jwt()->>'sub'` (the Clerk user ID) against each row's `user_id`. Every query is automatically scoped to the signed-in user — no server-side filtering needed in application code.

### User Sync — `hooks/useUserSync.ts`

Called on every mount of the `(root)` layout. It:

1. Checks if a `users` row exists for the Clerk user ID
2. If **yes** → reads currency → sets `needsOnboarding` based on whether currency is set
3. If **no** → upserts the user record (name, email, avatar) → creates a default "Cash" account → sets `needsOnboarding = true`

The `upsert` with `onConflict: "clerk_id"` makes this idempotent — safe to call multiple times.

---

## 6. Database & Backend (Supabase)

### Tables

| Table | Key Columns | Notes |
|---|---|---|
| `users` | `clerk_id` (PK-equivalent), `email`, `name`, `image_url`, `currency` | Synced from Clerk on first sign-in |
| `accounts` | `id`, `user_id`, `name`, `type`, `balance`, `is_default` | Types: CASH, BANK, CREDIT_CARD, SAVINGS |
| `transactions` | `id`, `user_id`, `account_id`, `type`, `amount`, `category`, `date`, `input_method`, `voice_transcript`, `is_flagged` | Types: INCOME, EXPENSE. Input: MANUAL, RECEIPT_SCAN, VOICE |
| `budgets` | `id`, `user_id`, `amount`, `last_alert_sent`, `last_alert_threshold` | One row per user (upserted) |

### Row-Level Security

All tables have RLS enabled. Policies use:
```sql
auth.jwt()->>'sub' = user_id  -- matches Clerk user ID
```

### No ORM — Direct Supabase Client Calls

Service functions in `lib/services/` receive a `SupabaseClient` instance as a parameter (dependency injection pattern). This makes them testable and keeps business logic separate from React hooks.

```ts
export async function getTransactions(
  supabase: SupabaseClient,
  userId: string,
  filters: TransactionFilters = {}
) { ... }
```

### Balance Update Strategy

There is **no Postgres RPC/transaction** wrapping the transaction insert + balance update. It's two sequential awaited calls:

```
1. INSERT into transactions
2. SELECT current balance from accounts
3. UPDATE accounts SET balance = balance + delta
```

If step 3 fails after step 1 succeeds, the error is **surfaced to the UI** rather than swallowed. This is an acknowledged trade-off — noted in the source code comments.

**Why not use a Postgres function?** Speed of development. The simplest fix for production would be a `BEGIN/COMMIT` Postgres RPC called via `supabase.rpc()`.

### Account Deletion — Safe Two-Step Default Management

```ts
// First unset all other defaults, THEN set the target
// A failure between the two never leaves two accounts marked default
await supabase.from("accounts").update({ is_default: false }).neq("id", accountId)
await supabase.from("accounts").update({ is_default: true }).eq("id", accountId)
```

For account deletion, a `force: false` dry-run first counts linked transactions and returns that count so the UI can show a confirmation dialog before actually deleting.

---

## 7. State Management Architecture

Welth uses **two separate state mechanisms** for different concerns:

### Zustand — Client/Session State (`store/userStore.ts`)

```ts
interface UserStore {
  currency: string;         // e.g. "INR", "USD"
  setCurrency: (v) => void;
  needsOnboarding: boolean | null;  // null = not yet determined
  setNeedsOnboarding: (v) => void;
}
```

Used for lightweight UI state that doesn't come from the server: the user's preferred currency and the onboarding gate. It's initialized at app start and updated by `useUserSync`.

### TanStack Query — Server State

All data from Supabase (accounts, transactions, budget) lives in TanStack Query's cache. Never stored in Zustand or React state.

**Why separate?**  
Server state has completely different concerns: loading, error, staleness, background refetch, cache invalidation. TanStack Query handles all of these automatically.

---

## 8. Data Fetching Layer (TanStack Query)

### Query Key Factory (`lib/query/keys.ts`)

```ts
export const queryKeys = {
  accounts:     (userId?) => ["accounts", userId],
  transactions: (userId?, filters?) => ["transactions", userId, filters],
  budget:       (userId?) => ["budget", userId],
};
```

Typed, hierarchical keys make cache invalidation precise and safe.

### Query Hooks Pattern

```ts
// hooks/queries/useTransactionsQuery.ts
export function useTransactionsQuery(filters = {}) {
  const { user } = useUser();
  const supabase = useSupabase();
  return useQuery({
    queryKey: queryKeys.transactions(user?.id, filters),
    queryFn: () => getTransactions(supabase, user!.id, filters),
    enabled: !!user,  // don't fetch until authenticated
  });
}
```

The `enabled: !!user` guard prevents fetching before Clerk loads.

### Mutation + Cache Invalidation Pattern

```ts
// hooks/mutations/useTransactionMutations.ts
return useMutation({
  mutationFn: (payload) => createTransaction(supabase, payload),
  onSuccess: (result) => {
    if (result.error) return;
    queryClient.invalidateQueries({ queryKey: ["transactions"] }); // refetch all transactions
    queryClient.invalidateQueries({ queryKey: ["accounts"] });     // refetch accounts (balance changed)
  },
});
```

After creating or deleting a transaction, **both** `transactions` and `accounts` queries are invalidated because the account balance has changed.

### Pull-to-Refresh

The Home screen exposes `refetch` from each query and combines them into a single `onRefresh` handler passed to `<RefreshControl>`.

---

## 9. AI Features (Gemini API)

All AI calls use **Gemini 3.1 Flash Lite** (`gemini-3.1-flash-lite`) — fast and cost-efficient.

### Feature 1: Receipt Scanner (`lib/services/extractTransaction.ts`)

**Flow:**
1. User taps "Scan Receipt" → `ReceiptScannerModal` opens (uses `expo-camera`)
2. User captures photo → returned as base64 + mimeType
3. `extractTransactionFromReceipt(base64, mimeType)` calls Gemini with:
   - The base64 image as `inlineData`
   - A structured prompt telling Gemini it's reading a receipt
   - **`responseSchema`** (JSON Schema) so Gemini returns structured JSON guaranteed to match `ExtractedTransaction`
4. The returned fields are applied to the form with `setValue()` (React Hook Form)
5. Missing fields (amount, category) trigger an `Alert` asking the user to fill them in

**Key — Structured Output:**
```ts
generationConfig: {
  responseMimeType: "application/json",
  responseSchema: RESPONSE_SCHEMA,  // { type, amount, category, description, date, transcript }
}
```
This ensures Gemini always returns valid JSON in the right shape, not free-form text.

### Feature 2: Voice Entry (`lib/services/extractTransaction.ts`)

**Flow:**
1. User taps "Voice Log" → `VoiceRecorderModal` opens (uses `expo-audio`)
2. User speaks → audio recorded as base64
3. `extractTransactionFromVoice(base64Audio, mimeType)` sends audio + prompt to Gemini
4. Prompt includes today's date so Gemini can resolve relative dates ("yesterday" → actual ISO date)
5. Same structured output + `applyExtraction()` → pre-fills the form

**Input method tracking:** The `input_method` field on each transaction is set to `"RECEIPT_SCAN"` or `"VOICE"` based on which AI was used. For voice, `voice_transcript` is also saved.

### Feature 3: AI Financial Assistant (`lib/services/assistant.ts`)

**Flow:**
1. User types a question on the Assistant screen
2. `askAssistant(question, transactions, budget, currency)` is called
3. `buildContext()` aggregates the last 30 days of data:
   - Total income / expense
   - Spending by category (sorted by amount)
   - Budget status
   - Last 40 transactions as human-readable lines
4. Full context + user question is sent to Gemini as a single prompt
5. Gemini's text response is displayed as a chat bubble

**Stateless design:** No conversation history is passed to Gemini — each question is independent. The "chat" UI is local React state only.

**Why not stream?** Simplicity. `fetch` returns the full response; no streaming plumbing needed for MVP.

---

## 10. Supabase Edge Functions (Backend Jobs)

Edge Functions run on Deno runtime, deployed to Supabase's global edge network.

### `check-budget-alerts` — Daily Budget Email

**Trigger:** Scheduled daily via a cron job (pg_cron in Supabase).

**Logic:**
```
For each budget row:
  1. Calculate this month's expenses
  2. Compute percentage: spent / budget.amount * 100
  3. Check thresholds [100, 80] (highest first — prevents double-firing)
  4. Skip if this threshold was already alerted THIS MONTH
     (tracked via last_alert_sent + last_alert_threshold on the budget row)
  5. Send email via Resend
  6. Update last_alert_sent + last_alert_threshold
```

**Idempotency:** The `last_alert_threshold` field prevents the 80% email from firing again if the user crosses 100% later in the same month. The month boundary resets the guard.

### `weekly-tips` — Weekly AI Finance Tips Email

**Trigger:** Scheduled weekly via cron.

**Logic:**
```
For each user:
  1. Fetch last 7 days of transactions
  2. Skip if no recent activity
  3. Build spending breakdown by category
  4. Call Gemini with structured output (2-4 tips as a JSON array)
  5. Send email with weekly summary + AI tips via Resend
```

Gemini's `responseSchema` forces tips to be a `string[]` with 2-4 items — no parsing needed.

### Shared Utilities (`_shared/`)

- **`supabaseAdmin.ts`** — creates a Supabase client with the `service_role` key (bypasses RLS — safe only on the server)
- **`resend.ts`** — wraps the Resend API for sending emails
- **`emailLayout.ts`** — shared HTML email wrapper/template

---

## 11. Form Handling & Validation

### Stack: React Hook Form + Zod + `@hookform/resolvers`

```ts
// lib/schemas/transaction.ts
export const transactionSchema = z.object({
  type:        z.enum(["EXPENSE", "INCOME"]),
  amount:      z.string().min(1).refine(v => parseFloat(v) > 0, "Enter a valid amount."),
  category:    z.custom<CategoryKey>(v => typeof v === "string"),
  accountId:   z.string().min(1, "Select an account."),
  description: z.string().optional(),
  date:        z.date(),
});

export type TransactionFormValues = z.infer<typeof transactionSchema>;
```

**Key pattern — `amount` as string:** The form stores amount as a string (not number) to support comma-separated input like "1,000". It's parsed to a float only at submit time.

**AI pre-filling:** When Gemini extracts transaction data from a receipt or voice, it calls `setValue()` for each extracted field — React Hook Form tracks these changes reactively.

---

## 12. Core Domain Types

### Transaction

```ts
type TransactionType = "INCOME" | "EXPENSE";
type InputMethod = "MANUAL" | "RECEIPT_SCAN" | "VOICE";

type Transaction = {
  id: string;
  user_id: string;
  account_id: string;
  type: TransactionType;
  amount: number;
  category: CategoryKey;       // e.g. "food", "salary"
  description: string | null;
  date: string;                // ISO 8601
  status: string;
  input_method: InputMethod;
  voice_transcript: string | null;
  is_flagged: boolean;
  flag_reason: string | null;
  created_at: string;
  updated_at: string;
};
```

### Account

```ts
type AccountType = "CASH" | "BANK" | "CREDIT_CARD" | "SAVINGS";

type Account = {
  id: string;
  user_id: string;
  name: string;
  type: AccountType;
  balance: number;
  is_default: boolean;
  created_at: string;
};
```

### Budget

```ts
type Budget = {
  id: string;
  user_id: string;
  amount: number;
  last_alert_sent: string | null;
  created_at: string;
  updated_at: string;
};
```

### Categories (`constants/categories.ts`)

21 categories total: 15 expense + 6 income. Each has:
- `label` (human-readable)
- `icon` (emoji)
- `color` (hex, used in charts)
- `type` ("EXPENSE" | "INCOME")

`CategoryKey` is a TypeScript union of all 21 keys — fully type-safe.

---

## 13. Screen-by-Screen Breakdown

### Home (`app/(root)/(tabs)/index.tsx`)

**Data:** `useAccountsQuery`, `useTransactionsQuery`, `useBudgetQuery`

**Computed (all via `useMemo`):**
- `totalBalance` — sum of all account balances
- `monthTransactions` — filtered by current month (`date-fns isSameMonth`)
- `monthIncome` / `monthExpense` — sums of month transactions by type
- `recentTransactions` — first 5 items (pre-sorted by date desc from API)
- `expenseBreakdown` — grouped by category, sorted by amount → passed to `PieChart`

**Budget progress bar:** Color-coded:
- Green if < 80% of budget used
- Yellow if 80–99%
- Red if 100%+

**Quick actions:** 3 buttons (AI Receipt Scan, Voice Entry, Add Manually) each navigate to `add-transaction` with an `action` URL param.

### Add Transaction (`app/(root)/(tabs)/add-transaction.tsx`)

The most complex screen. Manages:
- `useForm` with Zod schema
- `inputMethod` state: `"MANUAL" | "RECEIPT_SCAN" | "VOICE"`
- `voiceTranscript` state: saved with the transaction
- Modal open/close state for `ReceiptScannerModal` and `VoiceRecorderModal`
- `scanning` overlay state while Gemini processes a receipt

**URL param handling:** When navigated from Quick Actions with `?action=scan` or `?action=voice`, the screen automatically opens the appropriate modal via `useEffect`.

### Transactions (`app/(root)/(tabs)/transactions.tsx`)

Lists all transactions with filter support (by type, by account). Uses `useTransactionsQuery(filters)` — filters are part of the query key so changing them triggers a new fetch.

### Assistant (`app/(root)/(tabs)/assistant.tsx`)

Chat UI with `FlatList`. Messages are local React state (`useState<ChatMessage[]>`). On each send:
1. Adds user message to state
2. `await Promise.all([refetchTransactions(), refetchBudget()])` — always uses fresh data
3. Calls `askAssistant()` with the refreshed data
4. Adds assistant reply to state

**Suggested prompts** shown when conversation has ≤ 1 message.

### Profile (`app/(root)/(tabs)/profile.tsx`)

- Shows Clerk user info (name, email, avatar)
- Avatar upload: `expo-image-picker` → base64 → `user.setProfileImage()` (Clerk API)
- Lists all accounts via `useAccountsQuery`
- Currency picker → updates `users` table in Supabase + Zustand `currency`
- Account management via `AccountModal` (create, edit, set default, delete)
- Sign out via `useAuth().signOut()`

### Onboarding (`app/(root)/onboarding.tsx`)

One-time setup after first sign-in. Collects:
- Starting balance (required)
- Preferred currency (picker)

On submit:
1. Updates `users.currency` in Supabase
2. Fetches the auto-created "Cash" account
3. Inserts a "Starting balance" INCOME transaction
4. Updates the Cash account balance
5. Sets `needsOnboarding = false` in Zustand → triggers redirect to tabs

---

## 14. Key Design Decisions & Trade-offs

### Decision 1: Clerk for Auth (not Supabase Auth)

**Why:** Clerk provides a better out-of-box UI/UX for sign-in flows in React Native (OAuth, passkeys, etc.). The JWT bridge to Supabase lets us keep RLS while using Clerk.

**Trade-off:** Two auth systems to understand. The Clerk JWT must be valid when Supabase requests are made — handled by the `accessToken()` callback in the client factory.

### Decision 2: TanStack Query + Zustand (not a single store)

**Why:** Server state (async, cached, stale) has fundamentally different semantics from client state (synchronous, persistent). Using one store for both leads to complex manual cache management.

**Trade-off:** Two mental models. Developers need to know when to use `useQuery` vs Zustand.

### Decision 3: Gemini API called client-side

**Why:** Faster to build; no need for a separate API layer. The API key is prefixed `EXPO_PUBLIC_` so it's bundled in the app.

**Trade-off:** API key is exposed to the app bundle. For production, Gemini calls should move behind an Edge Function or API route. This is an MVP simplification.

### Decision 4: No Postgres transactions for balance updates

**Why:** Supabase's JavaScript client supports `supabase.rpc()` for Postgres functions, but setting them up takes more time. Sequential calls are simpler and cover 99.9% of cases.

**Trade-off:** A network failure between insert and balance update can cause inconsistency. The error is surfaced (not swallowed), so the user is informed.

### Decision 5: Stateless AI Assistant

**Why:** No conversation history keeps the implementation simple and cost-free (no token accumulation). Each question is answered independently against fresh financial data.

**Trade-off:** Assistant can't refer to previous messages in the conversation (e.g., "What about last month?" after "How much did I spend on food?").

### Decision 6: `useMemo` for all dashboard computations

The Home screen does all aggregation client-side from the cached transaction list. `useMemo` ensures these only recompute when the `transactions` or `accounts` arrays change, not on every render.

---

## 15. Interview Q&A — Ready Answers

### Q: How does authentication work in this app?

**A:** We use Clerk for authentication (sign-in, sign-up, session management). Clerk issues a JWT for each signed-in user. When making Supabase database calls, we pass that JWT as the `accessToken` to the Supabase client. Supabase's RLS (Row-Level Security) policies check `auth.jwt()->>'sub'` — the Clerk user ID — against each row's `user_id`. This means every query is automatically scoped to the current user without us needing to filter manually in code.

---

### Q: Why did you use TanStack Query instead of just `useEffect` + `useState`?

**A:** `useEffect` + `useState` for server data means you have to manually handle loading, error, caching, deduplication, refetching on focus, and cache invalidation across components. TanStack Query handles all of that. For example, when you add a transaction, calling `invalidateQueries(["transactions"])` automatically refetches every component using `useTransactionsQuery` — zero prop drilling or global event needed.

---

### Q: How do the AI features work?

**A:** There are three AI features, all using Google Gemini Flash Lite:
1. **Receipt scanner** — takes a base64 image, sends it to Gemini with a structured JSON schema, Gemini returns `{ type, amount, category, description, date }`, and those values pre-fill the form.
2. **Voice entry** — sends base64 audio to Gemini, same structured output, the transcript is also saved with the transaction.
3. **AI assistant** — builds a text summary of the last 30 days of transactions and budget, appends the user's question, sends to Gemini, displays the text reply in a chat UI.

The `responseSchema` parameter forces Gemini to return structured JSON, so we never need to parse free-form text.

---

### Q: How do you prevent the budget alert from firing multiple times a month?

**A:** The `budgets` table has two columns: `last_alert_sent` (timestamp) and `last_alert_threshold` (number — 80 or 100). When an alert fires, both are updated. On the next daily run, we check: was an alert already sent *this month* for a threshold *at or above* the current one? If yes, skip. This means: if you get an 80% alert on the 15th, you won't get another 80% alert, but you will still get a 100% alert later. At the start of a new month, `last_alert_sent` will be a different month, so both thresholds reset.

---

### Q: What is the onboarding flow and why does `needsOnboarding` have three states?

**A:** When a new user signs in, `useUserSync` runs and checks if the user has a row in Supabase with a `currency` set. If not, `needsOnboarding = true`. If yes, `needsOnboarding = false`. We use `null` as the initial state to mean "we haven't checked yet." The auth guard in `(root)/_layout.tsx` renders a loading spinner while `needsOnboarding === null`, preventing a flash of either the main app or the onboarding screen before the check completes.

---

### Q: How does the transaction export work?

**A:** `exportTransactionsToCsv` filters the last 30 days of transactions, builds a CSV string with `Date, Type, Category, Description, Amount, Input Method` columns, writes it to the device's cache directory using `expo-file-system`, then opens the native share sheet with `expo-sharing`. The file is overwritten if it already exists.

---

### Q: What's the platform difference in the tab bar?

**A:** On iOS we use `NativeTabs` from `expo-router/unstable-native-tabs`, which renders a true native UITabBar with SF Symbols (e.g., `house.fill`, `brain.head.profile`). On Android, we fall back to Expo Router's standard JS `Tabs` with Feather vector icons. This is detected with `Platform.OS === "ios"`.

---

### Q: How is form validation handled?

**A:** React Hook Form manages form state and touch tracking. Zod schemas define the validation rules. The `zodResolver` from `@hookform/resolvers` bridges them. The `transactionSchema` stores `amount` as a string (not number) to handle comma-formatted input like `"1,000"` — it's only parsed to `parseFloat` at submit time and in the Zod `.refine()` validator.

---

### Q: If you were to improve this app, what would you change?

**A:** Key improvements for production:
1. **Move Gemini API calls to Edge Functions** — the API key is currently bundled in the app.
2. **Use a Postgres function (`supabase.rpc()`) for transaction + balance updates** — to make them atomic.
3. **Add real conversation history to the assistant** — maintain context across messages for follow-up questions.
4. **Add optimistic updates** — currently mutations wait for the server round-trip before updating the UI.
5. **Add Supabase Realtime** — so the balance updates live across tabs/devices without manual refresh.
6. **Replace sequential default-account logic** with a Postgres trigger or single atomic update.

---

*Last updated: November 2026*

