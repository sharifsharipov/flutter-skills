# Refactoring — KISS, DRY, YAGNI, Extraction

Read this when cleaning up existing code, breaking down a large widget/class, or
removing duplication. Refactoring changes structure, **not behavior** — so it is
only safe with tests (see `testing.md`). Refactor in small, verifiable steps.

## 1. The three principles that drive most refactors

- **KISS — Keep It Simple.** The simplest solution that fully solves the problem
  wins. Clever one-liners that need a comment to explain are usually a net loss.
  Readability over cleverness (the Golden Rule).
- **DRY — Don't Repeat Yourself.** Duplicated *knowledge* is the enemy, not
  duplicated *lines*. Extract when the same rule appears in multiple places, so a
  change happens once. But don't over-abstract two things that merely look
  similar today (see AHA: "Avoid Hasty Abstractions").
- **YAGNI — You Aren't Gonna Need It.** Don't build for imagined future
  requirements. Delete speculative generality, unused params, and "just in case"
  hooks. The best code is the code you didn't write.

## 2. Widget extraction — break down huge `build` methods

A `build` method longer than a screen, or nested more than ~3–4 levels, is a
refactor target. Extract cohesive subtrees into their own **widget classes**
(not helper methods returning `Widget`).

```dart
// BEFORE — one 200-line build with a deep tree
Widget build(BuildContext context) {
  return Scaffold(
    body: Column(children: [ /* header 40 lines */, /* list 80 lines */, /* footer 60 lines */ ]),
  );
}

// AFTER — cohesive, const-able, independently rebuildable pieces
Widget build(BuildContext context) {
  return const Scaffold(
    body: Column(children: [_TripsHeader(), Expanded(child: _TripsList()), _TripsFooter()]),
  );
}
```

**Prefer extracting to a widget class over a `_buildX()` method.** A widget class
can be `const`, gets its own build boundary (so it rebuilds independently), and
is testable in isolation. A helper method rebuilds with the parent every time and
can't be `const`.

## 3. God classes & long methods

Symptoms: a class with 15+ methods and mixed concerns, a method over ~30–40
lines, or a name containing "Manager"/"Helper"/"Util" doing five unrelated
things.

Refactor moves:
- **Extract Class** — pull a cohesive cluster of fields+methods into its own
  class (Single Responsibility).
- **Extract Method** — name a block of logic; the name documents intent.
- **Move business logic out of the UI** — a bloc/cubit or use case owns it, the
  widget just renders state and dispatches events.
- **Replace conditional with polymorphism** — a growing `switch` on a `type`
  becomes subclasses / a Strategy (Open/Closed).

## 4. Magic numbers & strings

Named constants make intent explicit and changes safe. Route strings, keys,
durations, and sizes should never be scattered literals.

```dart
// BEFORE
if (status == 3) { ... }
Future.delayed(const Duration(milliseconds: 300));
Navigator.pushNamed(context, '/trip-details');

// AFTER
if (status == TripStatus.delivered) { ... }
Future.delayed(AppDurations.debounce);
Navigator.pushNamed(context, Routes.tripDetails);
```

Group constants meaningfully (`AppSpacing`, `AppDurations`, `Routes`,
`AppColors`) rather than one giant `Constants` bucket.

## 5. Common Flutter smells & their fixes

| Smell | Fix |
| --- | --- |
| Unnecessary `Container` | Use `Padding`/`SizedBox`/`DecoratedBox`/`Align` directly |
| Deeply nested widget tree | Extract widget classes; use `Spacer`/`Gap` |
| Nested `FutureBuilder`/`BlocBuilder` | Move async into a bloc; emit discrete states |
| `setState` in a big `StatefulWidget` | Localize state to the smallest widget, or lift to a bloc |
| Business logic in `onPressed` | Dispatch an event/call a use case |
| `if (x != null) x!.foo()` chains | Null-aware ops `?.`, `??`, pattern matching |
| Repeated padding/margins | Design tokens (`AppSpacing`) + shared widgets |
| `print()` for logging | A logger gated by `kDebugMode` |
| Passing data via global singletons | Constructor injection / bloc / route args |

## 6. Refactoring safely — the loop

1. Ensure there's a test covering the current behavior. If not, write a
   **characterization test** first (assert what it does now, even if ugly).
2. Make **one** small structural change.
3. Run analyzer + tests. Green? Commit. Red? Revert or fix immediately.
4. Repeat. Never mix a refactor and a behavior change in the same commit — it
   makes review and bisecting impossible.

## 7. When NOT to refactor

- No tests and no time to add them for a risky area → add tests first, or leave
  it.
- "It's ugly but isolated and never changes" → low value; spend effort where
  churn is high.
- Refactoring to introduce an abstraction you *might* need → YAGNI; wait for the
  second real use case (Rule of Three) before abstracting.
