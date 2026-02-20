# Senior iOS Job Search — Process Map (Anonymized & Approximate)

The job seeking process consists of a series of sequential phases. Below is a structural breakdown of the modern senior-level iOS hiring funnel as observed across multiple processes.

## A. Discovery & Application Phase

### 1. Finding Open Positions

- **Tools used:**
  - **LinkedIn** — using the "Jobs where you're more likely to hear back" section.
    - Always make sure that I'm applying to a real company and not a staffing agency or consultancy.
    - I seldom use the "Easy Apply" button, as that's a black hole, unless it's a highly desirable position and then I'll mark it as a top choice.
    - I make sure I match 95% of the required items. If not, I fear the ATS system will filter me out.
    - I scan for no-gos, i.e. "React Native, Kotlin Multiplatform, or Flutter/Dart, Core Data, or Swift Data".
      - Depending on the context, I may skip applying.
    - I target "Senior" and below. "Staff" positions in general weren't working for me. There seems to be role inflation happening across the board now.
    - Local hybrid or full time roles have higher response rates than remote, but I have gotten responses from remote positions, especially lower paid ones, despite the 200+ candidates application counts, with 40 per day.
  - **Jobright.ai** — posts new openings that are usually buried or harder to find in LinkedIn. I cross-reference with LinkedIn and can see the number of applicants.
  - **Indeed** — I used this a couple months ago, but seems like it's dried up.

### 2. Applying for Those Positions

- I fill out the app.
- For the website field, I put my GitHub repo link.
- I don't customize my resume per position since they would detect differences by looking at LinkedIn.
- I haven't been sending cover letters, but it might be something I consider.
- It feels like they are either interested or not.

## B. Initial Response & Early Filtering

### 3. Engaging After a Response

- My response rate for applications is around:
  - 1 in 10 gets a recruiter or automated advancement response.
  - 3 in 10 gets an eventual rejection.
  - 6 in 10 are silent, and I never hear anything.

#### 3a. With a recruiter in a brief 30-minute meeting

- I try to respond with times same or next day. About half of recruiters have a convenient calendar where I can pick a half hour slot. If not I have to propose a series of times and dates.
- I get through about 7 out of 10 of these.
  - Reasons for non-advancement can be the recruiter is overwhelmed and doesn't respond, the position closes, etc. Or there's a role clarification, or higher expectations than conveyed on the job description.

#### 3b. With an automated testing service (1–2 hour exam)

- Automated testing services include: CodeSignal and HackerRank.
- I've been through two CodeSignal rounds in Swift. They take about 2 to 3 hours and can be challenging the first time.
- I turned down a HackerRank one due to exhaustion over the weekend.

#### 3c. With an automated GitHub project evaluation

- A GitHub repo is sent; you are asked to submit a PR against the repo after 3 hours. You are asked to agree not to use outside help or AI.
  - Projects can be ambitious for someone rusty in SwiftUI. Including SwiftUI gotchas and the ability to unit test Combine code with promises as just some of the aspects needed to complete.
  - Without the ability to use outside help or AI, it's a matter of whether one knows the material or not, rather than ability to research and complete a task like a normal dev.

#### 3d. With a multiple choice skills assessment from a standards organization

- Tests Swift and legacy UIKit and AVFoundation knowledge.
- Can be outdated and test edge cases like variables being captured and modified within a closure.
- Can be very heavy on practical Apple knowledge and light on architecture, reasoning, etc.

## C. Early Evaluation Rounds (Human & Automated)

### 4. Engaging in the Next Step

- **A technical interview with an internal developer.**
  - About 7 in 10 times.

- **An engineering manager or future manager interview.**
  - About 2 in 10 times.
  - This usually delves into whether your previous experience matches their expectations for the current role, and whether there's alignment / chemistry in best practices, conflict resolution, mentoring, etc.
  - Sometimes there's a good fit, the engineering manager is a true engineer, with real scars, and alignment on unit testing coverage, having someone who isn't biased testing releases, and various other important metrics.
  - Other times, the engineering manager comes from a product background and there isn't alignment, or they are looking someone leaning towards staff or principal level responsibility in a senior role.
  - It's often a good time to see how teams function, whether there are other experts available, or you're going to be the only expert, how failures are handled, how well represented the engineering team is within management, whether the culture and company is healthy or not, whether people are encouraged to be honest and surface issues early or there are optics issues. What's the technical debt load looking like. Is the product complicated with a "magical sync engine". How are difficult architectural issues resolved, etc.

- **An automated technical assessment with an external vendor with a real human interview via Karat.**
  - About 1 in 20 times.
  - This can be an hour and half including technical discussions around iOS and Swift topics, a PR review, and coding challenge. It's fairly comprehensive and very technical.

- **An automated technical assessment with CodeSignal.**
  - About 1 in 30 times.
  - This is a web-portal with Swift compiler (usually with autocomplete). The web-IDE can take some getting used to. There is usually a project with various objectives.
  - Problems tend to be practical but involve some thought and involve basic data structures.

- **An automated technical assessment with Byteboard.**
  - About 1 in 30 times.
  - Very practical. Had a positive experience with this.

- **A semi-technical prototyping round with a product manager.**
  - About 1 in 50 times.
  - Lots of fun, but depending on position, it could be for something very competitive, and most of the candidates eliminated.
  - In my case, I checked the existing people, and competition for the role and it was off the charts. Current iOS employees were 1 in a 500, i.e. conference speakers and book authors.

## D. Multi-Round Technical & Behavioral Assessment

### 5. Engaging in a Multi-Round Technical and Behavioral Assessment

The canonical multi-round interview is an "onsite virtual zoom". This is really just a series of back-to-back zoom sessions with different interviewers or panels. Usually they are organized as such:

**1. Interview with a Product Manager (30 minutes)**

- Friendly.
- How do you work with people in your team?
- How do you resolve conflicts?
- How do you escalate issues when something goes wrong?
- What's an example where something didn't work out, and how did you address it within your pod's structure and across teams?
- etc. etc.

**2. Interview with a Senior or Staff iOS Engineer — someone above you technically (1 hour)**

- **Live coding with Xcode.** Types:
  - Variation on an Item list in SwiftUI. *5 out of 10.*
    - Fetched from the Network or a given service. *8 out of 10.*
    - Fetched from the file system. *2 out of 10.*
  - Model and Page decoding from a service. *5 out of 10.*
  - A given project of medium complexity with unit tests with various issues. *2 out of 10.*
  - Login and details screen with username and password rules. *1 out of 10.*
  - List of models from canned JSON accessed from the project as a text file. *1 out of 10.*
  - A given project of large complexity based on TCA (The Composable Architecture) using SwiftData. *1 out of 10.*
  - A given project of medium complexity with SwiftUI gotchas and Combine unit tests. *1 out of 10.*
- **An interactive discussion without any live coding.** *2 out of 10.*
- **A playground where an objective is given**, i.e. design an API manager, which is then gradually made more complex escalating to discuss concurrency, in-flight cancellation, etc.

**3. Secondary interview with another Senior or Staff iOS Engineer**

- A different aspect not covered by the first interview.

**4. A final round with an engineering manager or hiring manager**

### Other Observed Interview Formats

- A team and collaborative interview with Product Manager.
- A one-on-one live coding interview with an iOS expert (UIKit, SwiftUI).
- A one-on-one technical conceptual interview covering iOS, Swift, SwiftUI, concurrency (new and legacy), architecture, and best practices.
- A paired coding exercise fixing issues on a complex codebase.
- An interview with the engineering manager.
- An interview around networking and model decoding.
- A system design interview centered on client concerns.
- A multi-panel interview with 5 staff-level engineers and an architect, along with a LeetCode-style question at the end.
- A one-on-one live coding interview with an expert staff-level platform member (staff-shaped coding exercise, e.g., task coalescing, caching, concurrency edge cases).
- A one-on-one live coding interview fixing advanced Combine unit tests on a challenging codebase.

## E. Alignment & Executive Review

### 6. Engaging in Alignment Interviews Following Multi-Round Interviews

- An interview with the Chief Financial Officer.
- An interview with VP of Product.
- A behavioral and process interview with future boss and another engineering manager.

## F. Offer & Decision Phase

### 7. Offer Arrival

- 24-hour next business day offer arriving by email at 1pm with significant due diligence required.
