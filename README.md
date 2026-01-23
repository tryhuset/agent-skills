# TRY Agent Skills

Practical skills for AI coding agents, built from real-world workflows.

```bash
npx skills add tryhuset/agent-skills
```

## Skills

### commit-organizer

Analyzes uncommitted changes and organizes them into logical, well-structured commits.

**Use when:**
- You've made multiple unrelated changes in a session
- Your working directory has accumulated changes that need sorting
- You want clean, meaningful commit history before pushing

**What it does:**
- Groups changes by type (features, fixes, refactors, docs)
- Orders commits logically (infrastructure first, then features, then tests)
- Writes clear commit messages in imperative mood
- Excludes sensitive files automatically

[View skill →](./skills/commit-organizer/)

```bash
npx skills add tryhuset/agent-skills --skill commit-organizer
```

---

### swift-composable-architecture

Expert guidance for building iOS/macOS apps with The Composable Architecture (TCA) from Point-Free.

**Use when:**
- Building new features with TCA
- Refactoring existing code to TCA patterns
- Debugging state or effect issues
- Writing tests with TestStore

**What it covers:**
- Feature anatomy (State, Action, Reducer, Store)
- Effects and cancellation patterns
- Dependency injection with DependencyKey
- Navigation (tree-based and stack-based)
- Testing with TestStore and TestClock
- Modern TCA macros (@Reducer, @ObservableState, @CasePathable)

[View skill →](./skills/swift-composable-architecture/)

```bash
npx skills add tryhuset/agent-skills --skill swift-composable-architecture
```

---

### swiftui-animations

Expert guidance for building smooth, performant, and accessible animations in SwiftUI.

**Use when:**
- Building animations for iOS/macOS apps
- Choosing between animation APIs (withAnimation, PhaseAnimator, KeyframeAnimator)
- Implementing gesture-driven interactions
- Debugging animation issues

**What it covers:**
- Core principles (state-driven animation, transactions, Animatable protocol)
- Implicit animations (withAnimation, .animation modifier, timing curves)
- Explicit animations (AnimatableModifier, custom shapes)
- Transitions (built-in, combined, asymmetric, custom)
- Gesture-driven animations with spring physics
- Modern iOS 17+ APIs (PhaseAnimator, KeyframeAnimator)
- Accessibility (reduced motion support)

[View skill →](./skills/swiftui-animations/)

```bash
npx skills add tryhuset/agent-skills --skill swiftui-animations
```

## License

MIT

---

Built by **[TRY](https://try.no)** — Creators, designers, advisors, and technologists. Norway's top-ranked agency for 22 consecutive years. Anthropic partner for Claude AI implementation.
