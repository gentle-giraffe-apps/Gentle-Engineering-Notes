
# Mini‑TCA “State Machine MVVM”
## A Pragmatic, Team‑Friendly Alternative to Full TCA Adoption (TCA‑Inspired, Observation‑Friendly)

## 1. Introduction

Modern SwiftUI development often gravitates toward MVVM because it is approachable and fits neatly with Apple’s new `Observation` framework (`@Observable`, `@Bindable`, etc.). View models become simple observable types, and SwiftUI reacts automatically to state changes.

On the other end of the spectrum lies **The Composable Architecture (TCA)**—a powerful, opinionated system that introduces explicit state machines, reducers, effect management, dependency control, and feature composition.

TCA is excellent… but **overwhelming**. It requires:

- new vocabulary  
- new abstractions  
- a mental model shift  
- significant onboarding time  

And, crucially: **TCA is not taught by Apple**, nor is it widely covered by mainstream Swift education (WWDC talks, common YouTube playlists, etc.).

For many engineering teams—especially those with mixed experience levels—introducing full TCA can *hurt* maintainability, adoption, and team cohesion.

This article proposes a middle path:

> **Mini‑TCA “State Machine MVVM”: the TCA *paradigm* without the TCA *package*, implemented using Apple’s modern Observation system.**  
>  
> You gain the structure, clarity, and testability of reducer‑based logic while keeping:
> - MVVM familiarity  
> - zero external dependencies  
> - `@Observable` instead of `@Published`/Combine  
> - intuitive naming  
> - low onboarding cost  

This model is intentionally **team‑friendly** and appropriate for organizations where unilateral architecture changes can create friction or anxiety.

---

## 2. Why Full TCA Is Overwhelming for Many Teams

TCA doesn't just introduce a new architecture — it introduces an entire mental framework:

- `State`  
- `Action`  
- `Reducer`  
- `Store`  
- `Effect`  
- dependency injection system  
- scoping and composition  
- navigation as state  

For an engineer who knows MVVM from tutorials and Apple’s Observation docs, this is a *lot*.

### Common adoption risks

- **Engineers feel confused or insecure**, fearing they must watch hours of videos to contribute.  
- **Hiring becomes harder** because TCA knowledge is niche.  
- **Mixed architectural styles** lead to fragmentation (MVVM here, TCA there, some homegrown pattern elsewhere).  
- **One engineer becomes the “gatekeeper”**, creating team imbalance and making others afraid to touch “the TCA parts.”  

A heavy‑handed TCA rollout can unintentionally **alienate** teammates—something many devs learn the hard way.

---

## 3. The Middle Path: Mini‑TCA “State Machine MVVM”

Instead of importing TCA, we borrow just the *core ideas* and implement them with Apple's Observation (`@Observable`) and plain Swift.

### ✔️ Use the *paradigm*:

- A `ViewState` struct describing all the state for the screen  
- An `Event` (Action) enum listing all the events that can occur  
- A `reduce` function that updates state for each event in a single, central place  

### ✔️ Retain MVVM shape:

- A view‑model type still exists (now using `@Observable`)  
- SwiftUI binds to `@Bindable` properties or the `viewState`  
- Async tasks still live in the view‑model  
- No global `Store` type or scoping machinery required  

### ✔️ Keep everything readable to non‑TCA engineers

Use friendly naming:

- `CheckoutViewState` (instead of just `State`)  
- `CheckoutEvent` (instead of `Action`)  
- `send(_:)` or `handle(_:)` (instead of “dispatch” or a named `Reducer` type)  

To someone who has never heard of TCA, this looks like:

> “Oh, this view‑model uses a small state machine so all mutations are in one place. That makes it easier to test and reason about.”

No new framework, no magic.

---

## 4. Baseline: A Non‑TCA MVVM View‑Model (Observation)

Here is a relatively typical modern SwiftUI view‑model using `@Observable` and async/await:

```swift
import Observation

@Observable
final class CheckoutViewModel {
    enum Step {
        case enterDetails
        case completed
    }

    var isLoading = false
    var errorMessage: String?
    var step: Step = .enterDetails

    private let service: CheckoutService

    init(service: CheckoutService) {
        self.service = service
    }

    func submit() {
        isLoading = true
        errorMessage = nil

        Task {
            do {
                try await service.performPayment()
                await MainActor.run {
                    self.step = .completed
                    self.isLoading = false
                }
            } catch {
                await MainActor.run {
                    self.errorMessage = "Payment failed."
                    self.isLoading = false
                }
            }
        }
    }
}
```

This is fine and idiomatic—but it has some drawbacks when systems get more complex:

- Logic is **spread out** across methods.  
- There's no explicit model of “all events that can happen.”  
- You can test it, but often via **integration‑style tests** that exercise both async work and state changes together.  
- As more states and branches accumulate, it becomes harder to reason about all possible transitions.

---

## 5. Mini‑TCA “State Machine MVVM” Version (Observation‑Friendly)

Let’s rework the same feature using the Mini‑TCA pattern:

### 5.1 View State + Event

We first define the state for this screen and the events that can occur.

```swift
struct CheckoutViewState {
    enum Step {
        case enterDetails
        case completed
    }

    var isLoading = false
    var errorMessage: String?
    var step: Step = .enterDetails
}

enum CheckoutEvent {
    case onAppear
    case submitTapped
    case paymentSucceeded
    case paymentFailed(String)
}
```

### 5.2 Reducer

Next, we centralize **all state mutations** in a single `reduce` function:

```swift
func reduce(state: inout CheckoutViewState, event: CheckoutEvent) {
    switch event {
    case .onAppear:
        state.errorMessage = nil

    case .submitTapped:
        state.isLoading = true
        state.errorMessage = nil

    case .paymentSucceeded:
        state.isLoading = false
        state.step = .completed

    case let .paymentFailed(message):
        state.isLoading = false
        state.errorMessage = message
    }
}
```

This is the “mini‑reducer.” It’s just a function:

> `(inout State, Event) -> Void`

No framework required.

### 5.3 Observation‑Based View‑Model

Finally, we wrap this in an `@Observable` view‑model that SwiftUI can bind to:

```swift
import Observation

@Observable
final class CheckoutViewModel {
    var viewState = CheckoutViewState()

    private let service: CheckoutService

    init(service: CheckoutService) {
        self.service = service
    }

    func send(_ event: CheckoutEvent) {
        reduce(state: &viewState, event: event)
    }

    func onAppear() {
        send(.onAppear)
    }

    func submit() {
        send(.submitTapped)

        Task {
            do {
                try await service.performPayment()
                await MainActor.run {
                    self.send(.paymentSucceeded)
                }
            } catch {
                await MainActor.run {
                    self.send(.paymentFailed("Payment failed. Try again."))
                }
            }
        }
    }
}
```

SwiftUI can now watch `viewState` or its fields via `@Bindable`:

```swift
struct CheckoutView: View {
    @Bindable var viewModel: CheckoutViewModel

    var body: some View {
        VStack {
            if viewModel.viewState.isLoading {
                ProgressView()
            }

            if let message = viewModel.viewState.errorMessage {
                Text(message).foregroundStyle(.red)
            }

            // ...
            Button("Submit") {
                viewModel.submit()
            }
        }
        .task { viewModel.onAppear() }
    }
}
```

From SwiftUI’s perspective, this is **just an `@Observable` view‑model**.  
From an architectural perspective, you now have an explicit **state machine**.

---

## 6. Why This Is So Much Easier to Unit Test

The big win: you can test the reducer **independently** of async work and Observation.

### 6.1 Pure Reducer Tests

```swift
import XCTest

final class CheckoutReducerTests: XCTestCase {
    func testPaymentFailureUpdatesErrorAndStopsLoading() {
        var state = CheckoutViewState(isLoading: true,
                                      errorMessage: nil,
                                      step: .enterDetails)

        reduce(state: &state, event: .paymentFailed("Oops"))

        XCTAssertFalse(state.isLoading)
        XCTAssertEqual(state.errorMessage, "Oops")
        XCTAssertEqual(state.step, .enterDetails)
    }

    func testSubmitTappedStartsLoadingAndClearsError() {
        var state = CheckoutViewState(isLoading: false,
                                      errorMessage: "Old error",
                                      step: .enterDetails)

        reduce(state: &state, event: .submitTapped)

        XCTAssertTrue(state.isLoading)
        XCTAssertNil(state.errorMessage)
    }
}
```

These tests:

- Don’t touch `CheckoutViewModel`  
- Don’t depend on Observation  
- Don’t run async work  
- Don’t need test doubles for `CheckoutService`  

You’re simply testing: **Given this state and this event, what should the new state be?**

This is *exactly* the core TCA benefit, preserved in a tiny, approachable form.

---

## 7. MVVM vs Mini‑TCA vs Full TCA

### 7.1 When MVVM (with `@Observable`) is enough

Use plain MVVM for:

- Simple CRUD screens  
- Lists and detail screens  
- Simple forms with light validation  
- Screens that mostly read from a repository/service and display results  

If your `@Observable` view‑model:

- has only a few properties  
- has straightforward logic  
- doesn’t model a complex flow  

…then plain MVVM is fine.

---

### 7.2 When Mini‑TCA “State Machine MVVM” shines

Reach for this pattern when:

- The **flow itself** is complex (not just the data)  
- You have multi‑step user journeys (onboarding, checkout, multi‑phase forms)  
- There are many possible states and transitions (draft/saving/synced/error)  
- There are tricky combinations of user events and async results  
- You want rock‑solid unit tests for business logic  

It’s especially compelling for:

- Payment flows  
- Identity/KYC flows  
- Complex editors (text, layout, drawing)  
- Retry/heavy error handling scenarios  

You keep:

- MVVM shape  
- `@Observable` view‑models  
- No external dependency  

But your logic is now a **small, explicit state machine**.

---

### 7.3 When (and if) to consider full TCA

Full TCA makes sense only when:

- You have many features using this pattern  
- You want to compose reducers across modules  
- You need strict, testable dependency injection across the app  
- Navigation is fully state‑driven and non‑trivial  
- You want advanced effect management and cancellation patterns  

At that point:

- You’re no longer just solving one complex screen  
- You’re building a whole app around reducer‑based architecture  

Only then is the TCA package’s extra complexity likely to pay off.

Until then, the **TCA‑inspired Mini‑TCA pattern** is more than enough.

---

## 8. Why “Paradigm Over Package” Is Safer for Teams

The TCA *paradigm* is timeless:

- explicit events  
- explicit state transitions  
- value semantics  
- deterministic logic  

The TCA *package* is optional.

By teaching the paradigm using familiar tools (`@Observable`, enums, and functions):

- Team members stay calm  
- No one feels forced to learn a big external framework just to edit a file  
- Your architecture stays **approachable** and **onboard‑able**  
- You avoid creating an accidental “TCA priesthood” inside the team  

You get most of the benefit without the social and educational cost.

---

## 9. Adoption Strategy: A Safe, Political‑Aware Path

Here’s a concrete way to adopt this pattern without scaring anyone:

1. **Start with a single, complex flow**  
   Choose a place where bugs are painful and state is already messy.

2. **Introduce `ViewState` + `Event` + `reduce` gently**  
   Keep the view‑model’s public interface the same wherever possible.

3. **Use friendly naming and comments**  
   ```swift
   /// This view model uses a small reducer-style state machine:
   /// - `CheckoutViewState` holds all UI state
   /// - `CheckoutEvent` lists everything that can happen
   /// - `reduce(state:event:)` is the only place that mutates state
   ///
   /// This makes the flow easier to reason about and unit test.
   @Observable
   final class CheckoutViewModel { ... }
   ```

4. **Share a couple of tiny reducer tests in your PR**  
   Let reviewers see how straightforward it is.

5. **Avoid mentioning TCA unless asked**  
   If someone is curious, you can say:
   > “This is loosely inspired by The Composable Architecture,  
   > but implemented with no external dependency and minimal ceremony.”

6. **Only consider the TCA package if the pattern spreads naturally**  
   If several features benefit and you start re‑implementing infrastructure, *then* you can have a thoughtful TCA‑adoption conversation.

---

## 10. Summary

Mini‑TCA “State Machine MVVM” (TCA‑inspired, Observation‑friendly) is a **practical, low‑risk architectural upgrade** for SwiftUI teams:

- It keeps MVVM and `@Observable` as the baseline.  
- It introduces **explicit state + events + reducer** where flows are complex.  
- It gives you **TCA‑style testability** and **predictability** without the cost of full TCA adoption.  
- It respects team psychology, onboarding, and hiring constraints.

Use it:

- where flows are gnarly,  
- where correctness matters,  
- and where you want your future self (or your teammates) to be able to reason about the code in five minutes instead of fifty.

It’s not about being clever or elite.  
It’s about making **complex behavior feel simple, testable, and safe** for everyone who works in your codebase—today and two years from now.

---

## 🤖 Tooling Note

Portions of drafting and editorial refinement in this repository were accelerated using large language models (including ChatGPT, Claude, and Gemini) under direct human design, validation, and final approval. All technical decisions, code, and architectural conclusions are authored and verified by the repository maintainer.
