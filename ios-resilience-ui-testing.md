# UI Torture Testing with XCUITest: A Minimal, Practical Approach (V1)

*Jonathan Ritchey --- Gentle Giraffe Apps*

------------------------------------------------------------------------

## Why Torture Testing Still Matters in the Age of AI

Modern AI tools can now generate impressive unit tests in seconds. Yet
despite this, many of the most expensive and embarrassing bugs in
production apps still come from:

-   Asynchronous race conditions\
-   View lifecycle timing issues\
-   Cancellation bugs\
-   Background / foreground transitions\
-   Partial state restoration after suspension

These are *not* happy‑path bugs. They emerge under **chaotic,
long‑running, real‑world interaction**.

This is where **UI torture testing** still shines.

Rather than checking correctness under perfect conditions, torture
testing deliberately stresses the app under unpredictable usage
patterns: random taps, navigation, backgrounding, and relaunching---over
and over.

This article presents a **minimal, production‑practical V1 torture
harness** that:

-   Requires **no app code changes**
-   Runs entirely in **XCUITest**
-   Works with any existing UI‑testable iOS app
-   Can be adopted **today**, not next quarter

------------------------------------------------------------------------

## The Core Idea

At its heart, a UI torture test does just a few things:

1.  Launch the app
2.  Randomly perform user‑like actions
3.  Periodically suspend and resume the app
4.  Log everything with timestamps
5.  Let the test run long enough for rare failures to surface

Even this simple model is enough to uncover: - Navigation crashes -
Zombie views and leaked tasks - Half‑completed async pipelines -
Corrupted local state during lifecycle transitions

You do *not* need a perfect mocking setup to begin.

------------------------------------------------------------------------

## A Minimal UI Torture Harness (Runs Today)

This is the smallest useful torture test I've found in real projects. It
intentionally avoids any app‑specific assumptions.

``` swift
final class UITortureTests: XCTestCase {

    func testUITorture() {
        let app = XCUIApplication()
        app.launch()

        let start = Date()

        func log(_ message: String) {
            let delta = Date().timeIntervalSince(start)
            print(String(format: "[%.3fs] %@", delta, message))
        }

        for i in 0..<300 {

            // Periodically background and resume
            if i % 40 == 0 {
                XCUIDevice.shared.press(.home)
                log("suspend")
                sleep(1)
                app.activate()
                log("resume")
                continue
            }

            let roll = Int.random(in: 0..<100)

            if roll < 70 {
                let elements = app.descendants(matching: .any)
                    .allElementsBoundByIndex
                    .filter { $0.isHittable }

                if let element = elements.randomElement() {
                    element.tap()
                    log("tap: \(element.label)")
                }
            } else {
                app.swipeUp()
                log("swipe up")
            }

            RunLoop.current.run(until: Date().addingTimeInterval(0.2))
        }
    }
}
```

This test:

-   Randomly taps hittable UI elements
-   Randomly swipes
-   Regularly suspends and resumes the app
-   Emits a timestamped execution trace

Despite its simplicity, this pattern alone has uncovered real production
bugs that dozens of deterministic UI tests never touched.

------------------------------------------------------------------------

## What This Catches Exceptionally Well

Torture tests are especially effective at revealing:

-   Swift concurrency cancellation bugs\
-   Navigation controllers or SwiftUI stacks in invalid states\
-   Race conditions during token refresh\
-   Memory pressure issues caused by backgrounding\
-   Incomplete teardown of async tasks on suspension

These failures rarely appear in short, scripted tests.

They emerge when: - The app is interrupted at just the wrong moment -
Multiple async systems are active simultaneously - The lifecycle does
something "impolite"

Which is exactly how real users behave.

------------------------------------------------------------------------

## About Authentication and Setup

One of the hardest parts of large‑scale UI testing is reliably getting
the app into a **logged‑in, realistic state**.

In practice, teams use a mix of:

-   Launch arguments to bypass login in test builds
-   Injected tokens stored directly in the Keychain
-   Test‑only backend endpoints
-   Pre‑seeded local databases

Every production app differs in: - Auth provider - Token storage -
Backend topology - CI security constraints

For that reason, this article intentionally **does not prescribe a
universal login strategy**. Instead, it focuses on the torture harness
itself---the chaos generator.

Once a harness exists, higher‑fidelity setup (mocked vs pre‑prod
environments) becomes an **incremental upgrade**, not a prerequisite.

------------------------------------------------------------------------

## Mocked vs Integrated Torture Testing

In real workflows, torture testing often splits into two useful modes:

### Offline / Mocked Torture

-   No real networking
-   Deterministic data
-   Finds intrinsic UI and state‑machine failures
-   Fast and stable

### Pre‑Production / Integrated Torture

-   Real networking and tokens
-   Finds serialization, latency, auth‑refresh issues
-   Higher signal, higher flakiness

Both are valuable. Most teams eventually use **both**.

------------------------------------------------------------------------

## Logging and Video Capture

Xcode already records full **MP4 video of UI tests**.

A powerful next step is to: - Log each action with timestamps - Convert
logs into `.srt` subtitle files - Overlay subtitles on recorded video

This produces a complete, searchable execution record: what happened,
when it happened, and what the UI looked like at the time.

This V1 article intentionally leaves that as an extension point rather
than core complexity.

------------------------------------------------------------------------

## What This Approach Is *Not*

This minimal harness is not:

-   A replacement for deterministic UI tests
-   A CI pipeline solution by itself
-   A general mocking framework

It is a **chaos amplifier**.

It finds the bugs you didn't think to write a test for.

------------------------------------------------------------------------

## Why This Still Matters in 2025

AI can generate unit tests. AI can scaffold UI flows. AI can mock entire
networking stacks.

What it cannot yet reliably do is: - Reason about long‑running async
systems under lifecycle chaos - Predict emergent state corruption -
Simulate weeks of real user entropy

Torture testing remains one of the most effective ways to **manufacture
rare failures on demand**.

------------------------------------------------------------------------

## Final Thoughts

You do not need a perfect infrastructure to begin. You only need:

-   A runnable harness
-   A long time horizon
-   A willingness to let chaos do its work

Start simple. Let bugs surface. Then iterate.

That is how durable test systems are actually built in practice.

------------------------------------------------------------------------

*Jonathan Ritchey --- Gentle Giraffe Apps* Senior iOS Engineer --- Swift
• SwiftUI • Concurrency
