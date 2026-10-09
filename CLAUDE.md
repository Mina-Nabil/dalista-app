# Dalista App (Flutter) — Agent Instructions

Flutter mobile app for Dalista, a shopping-list / shop-catalog sharing app with item comparison and an AI agent. This file is the standing context for any Claude Code session working in this repo — read it before starting any ticket.

## Architecture — fixed convention, no exceptions

**Clean Architecture with Feature-First organization**, **flutter_bloc** for state management. Every feature follows the same layered structure; don't improvise a different pattern for a feature that feels simple.

### Layering (per feature)

- **Presentation** — pages, feature-specific widgets, Bloc/Cubit.
- **Domain** — entities, repository interfaces, use cases. Zero framework/IO dependencies.
- **Data** — repository implementations, remote/local data sources, DTOs/models.
- Dependency rule: Presentation → Domain ← Data. Domain never imports Data or Presentation.

### Directory structure

```
lib/
├── core/
│   ├── error/              # Failure classes
│   ├── network/            # network utilities, interceptors
│   ├── utils/               # utility functions and extensions
│   └── widgets/             # reusable widgets
├── features/
│   ├── auth/
│   ├── lists/               # Lists, Grouped Lists, Categories, Items
│   ├── sharing/              # Sharing & Invites
│   ├── public_lists/         # Public Lists & Discovery, stars
│   ├── comparison/           # Item Comparison
│   ├── periodic_lists/       # Periodic Lists & "Today's List"
│   ├── activity/             # Activity, Comments & Timeline
│   ├── ai_agent/             # AI Agent chat/interaction
│   ├── billing/              # native app store billing only — see Billing section below
│   └── notifications/        # push handling (client side)
│       └── (each with data/domain/presentation as above)
└── main.dart
```

This maps directly onto the product modules — each module becomes at minimum one `features/` folder.

### State management (flutter_bloc)

- Bloc for event-driven/complex flows (real-time list sync, AI agent conversation); Cubit for simpler state.
- States and Events are immutable, built with **Freezed**, union types (initial/loading/success/error).
- `BlocProvider` for DI; `BlocObserver` for logging/debugging.
- `BlocBuilder` with `buildWhen` for optimized rebuilds; side effects via `BlocListener`.
- No business logic in widgets — Blocs/Cubits own it, UI only renders state.

### Dependency injection

**GetIt** as the service locator, registered per feature in dedicated init files (`injection_container.dart` per feature, aggregated at app startup). Lazy singletons for services/repositories, factories (transient) for Blocs.

### Error handling

**Dartz `Either<Failure, Success>`** throughout — no exceptions crossing layer boundaries. Base `Failure` class (Equatable) with subclasses: `ServerFailure`, `CacheFailure`, `NetworkFailure`, `ValidationFailure` — extend per feature as needed (e.g. `PermissionFailure` for editor/viewer role checks). Repositories map data-layer exceptions to domain `Failure`s. UI pattern-matches on `Either` (`fold`/`when`) to render loading/error/success.

### Repository pattern

Repository = single source of truth, abstracting over remote (Golang API) and local (cache) data sources. Sharing & Invites and Activity/Timeline are live/collaborative — repositories should serve cached list state offline and reconcile on reconnect. Repositories map DTOs to Domain entities; Domain never sees a raw API model.

### Testing

Unit tests for domain use cases, repositories, Blocs (Given-When-Then). Widget tests for UI components, integration tests per feature. Mocking via `mocktail` or `mockito`.

### Code quality & performance

`flutter_lints`, functions under ~30 lines, SOLID principles, `const` constructors, `ListView.builder` for lists, `compute()` for expensive work, pagination for large lists (public-list discovery feeds, activity timelines).

## Design source

Visual style (colors, spacing, typography, components) comes from the existing Adobe XD file (`src/design/Shopanizer App UI.xd` in the Dalista project folder), already extracted to pixel-exact specs. Screen flows and widget composition are rebuilt from scratch against the confirmed data model (Grouped List → List → Item, direct sharing, public lists, timeline, comparison, AI agent) — do not implement the XD's old Group/participants flows as-is; only the visual language carries over.

## Auth model

Four sign-in methods: Google OAuth, Facebook OAuth, email/password, and **phone number + OTP as its own standalone method** (no email required on that path). Every account needs a verified phone number eventually — for social/email signups this is a post-signup verification screen, not a blocker to initial signup.

## Billing — platform-conditional purchase flow

v1 uses native app store billing exclusively on both platforms — no Stripe:

- **iOS**: `in_app_purchase` plugin (StoreKit) — mandatory per Apple guideline 3.1.1.
- **Android**: Google Play Billing via the same `in_app_purchase` plugin (Play billing add-on).
- Single `PurchaseDataSource` interface in the Domain layer, two Data-layer implementations (`ApplePurchaseDataSource`, `GooglePlayPurchaseDataSource`) selected at DI-registration time by platform. Domain/Presentation stay provider-agnostic — this is what makes adding Stripe later (e.g. for web) a new Data-layer implementation rather than a rework.
- The client never trusts its own purchase confirmation. Send the raw receipt/purchase token to the backend for server-side validation before treating the tier as active — entitlement state lives on the backend, not on-device.

## Known open items (don't block on these, but flag rather than silently guessing)

1. Real-time sync mechanism for collaborative lists — not yet decided on the backend, so the `sharing`/`activity` Data layers may need to accommodate whichever lands (websockets/SSE/polling).
2. AI agent LLM provider + invocation model — affects whether the `ai_agent` feature's chat screen needs to handle a background-job-style async response vs. a synchronous one.
3. Exact push notification event list — not yet finalized.

## Where the full spec lives

The complete requirements digest, module breakdown, technical architecture doc, and phased build plan live in the "Claude outputs" folder of the Dalista project, and the full ticket backlog is in Linear (team: Dalista, projects: API / Mobile App / Dashboard, labeled Phase 0 through Phase 12). When a ticket's one-line description isn't enough context, check those docs before guessing.
