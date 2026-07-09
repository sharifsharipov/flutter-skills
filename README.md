# Flutter Master

> An enterprise-grade Flutter/Dart engineering standard, packaged as a Claude skill.

**Flutter Master** makes an AI assistant behave like a Staff/Principal Flutter Engineer performing a code review *before* writing any code. It is not a feature factory — it turns code generation into **code engineering**: decide the right shape first, then produce production-grade code suitable for large-scale enterprise apps.

Whenever you write, refactor, review, architect, test, secure, or optimize Flutter/Dart code, this standard is applied automatically.

## The Golden Rule

> **Never write code just because it works.**
> Always write production-grade code. Prioritize readability over cleverness. Every architectural decision must be justified by scalability, maintainability, and testability. Minimize coupling, maximize cohesion, follow SOLID, and align with official Flutter and Dart guidelines.

If a request violates this rule, the assistant says so plainly and proposes the correct shape. Being a good engineer sometimes means pushing back.

## Default Tech Stack

Unless the project dictates otherwise:

- **Architecture** — Clean Architecture, feature-first. `presentation → domain ← data`. The domain layer depends on nothing.
- **State** — BLoC/Cubit with immutable state. `BlocSelector` / `context.select` for granular rebuilds.
- **Models** — Freezed for data classes, unions, and `copyWith`. Value objects for domain invariants.
- **Errors** — Return `Either<Failure, T>` (or a sealed `Result`) from the domain boundary. No exceptions crossing layers.
- **DI** — `get_it` + `injectable`. Constructor injection everywhere.
- **Networking** — Dio with interceptors; a Mapper between DTOs and domain entities. DTOs never leak into the UI.
- **Immutability** — Prefer `StatelessWidget`, `const`, `final`, and immutable state.

## What's Inside

The skill uses **progressive disclosure**: `SKILL.md` is an orchestrator that loads only the reference files relevant to the task, instead of loading everything at once.

| File | Covers |
| --- | --- |
| [`SKILL.md`](SKILL.md) | Orchestrator — Golden Rule, pre-code checklist, default stack, routing table, self-review protocol |
| [`architect.md`](architect.md) | Clean Architecture, the dependency rule, feature-first folders, SOLID, DDD, use cases |
| [`patterns.md`](patterns.md) | `Either`/Result monad, Repository, Mapper (DTO⇄Entity), UseCase, Freezed unions, DI, Strategy/Factory/Adapter |
| [`performance.md`](performance.md) | Rebuild scoping (`const`, `BlocSelector`, `RepaintBoundary`), list/sliver builders, memory leaks & disposal, isolates, image cache |
| [`security.md`](security.md) | Secrets handling, secure token storage, Dio auth + 401 refresh, HTTPS + certificate pinning, authn/authz, log hygiene |
| [`testing.md`](testing.md) | Test pyramid, mocktail/`bloc_test`, widget/golden/integration tests, >90% business-logic coverage, TDD |
| [`refactor.md`](refactor.md) | KISS/DRY/YAGNI, widget extraction, god-class fixes, magic-value constants, smell→fix table |
| [`review.md`](review.md) | PR review order, full checklist, smell detectors, AI self-review protocol |
| [`enterprise.md`](enterprise.md) | CI/CD gates, flavors, melos monorepo, ADRs, semver + CHANGELOG, observability, dependency hygiene |

## How It Works

1. A task touches a `.dart` file or a Flutter concept → the standard triggers (even if you never say "architecture" or "review").
2. The orchestrator picks the 2–3 relevant reference files by intent. A new feature is usually `architect` + `patterns` + `testing`; a review is `review` plus the domain the code lives in.
3. The design is decided first, then production-grade code is written.
4. A self-review checklist runs at the end of every code response. Rule violations are flagged, never applied silently.

## Prefer / Avoid at a Glance

**Prefer:** `StatelessWidget` · `const` constructors · composition over inheritance · extensions · value objects · Freezed · immutable state · `BlocSelector` · Repository pattern · UseCase pattern · feature-first structure.

**Avoid:** huge widgets · god classes · business logic in the UI · magic numbers/strings · deep widget trees · nested `FutureBuilder`/`BlocBuilder` · unnecessary `Container` · global mutable state.

## Usage

This is a [Claude](https://claude.com/claude-code) skill. Place the folder where your assistant discovers skills, and it activates automatically for Flutter/Dart work. The reference files are also useful as a standalone Flutter engineering handbook.

## License

MIT
