# Code Reading & JFlattener

**Day 4, Session 12 - [Eleftherios Chrysochoidis](https://www.linkedin.com/in/lefterisxris/) (Main Room)**

*Why code reading is harder than code writing, and what we can do about it (Thu, 30 Jul 26)*

## About the Speaker

Eleftherios (Lefteris) Chrysochoidis is a Lead API Software Engineer at Chubb, co-organizer of the [SKG Java Meetup](https://www.meetup.com/thessaloniki-not-only-java/) (1,600+ members), and creator of the [CodeTour IntelliJ plugin](https://plugins.jetbrains.com/plugin/19227-codetour).

## Background: The Onboarding Problem

- As a member of a startup that scaled up, onboarding new joiners became a significant bottleneck
- Too much effort was required to bring people up to speed on the codebase
- This led to the **CodeTour** idea: guided walkthroughs of source code

### CodeTour Plugin

- Found an existing VS Code tool, then created a more enhanced version for IntelliJ: [CodeTour on JetBrains Marketplace](https://plugins.jetbrains.com/plugin/19227-codetour)
- Allows you to **record and play back guided tours** in your source code
- Worked great for onboarding, but **hard to maintain** as tours are attached to specific lines (community feedback confirmed this pain point)

## The Code Reading Problem

Today, AI writes ~90% of the code. Developers must **read, comprehend, and evaluate** - which is more challenging than writing.

Common anti-patterns:
- Developers tend to blame code they don't own, without knowing the context or tradeoffs
- The classic reaction: *"Let's rewrite it"* - when the real issue is lack of understanding

### Why Is Code Reading Challenging?

- Good software design including **OOP**, **Design Patterns**, and **Dependency Injection**, makes code more modular, reusable, and maintainable. These are essential practices.
- However, they also mean that behavior is distributed across many files, interfaces, and abstraction layers.
- For developers who are **new to a project** (not necessarily junior - even senior engineers joining a new codebase), this creates a real challenge: the logic is split into many puzzle pieces, and you have to put them all together in your head to understand a single flow.

**Example - a simple order placement:**

```
OrderService.placeOrder()
├── inventoryService.reserve()                → open file 2
├── pricingService.calculateTotal()           → open file 3
│   └── discountService.findDiscount()        → open file 4
│       └── ? CouponDiscount | LoyaltyDiscount | SeasonalDiscount   ← which impl?
├── paymentService.charge()                   → open file 5
│   └── ? StripeGateway | PayPalGateway | BraintreeGateway          ← which impl?
└── notificationService.send()                → open file 7
    └── templateEngine.render()               → open file 8
```

**8 files. 4 interfaces. 3 layers of abstraction. Multiple possible implementations behind each interface. One order.**

To understand what happens, you navigate through method after method, switch between interfaces and implementations, jump into injected services, inspect variables, come back, remember everything, and reconstruct the execution flow in your head.

This requires **cognitive and spatial skills** - not just technical knowledge.

Speed matters, but **quality of comprehension** matters more.

## Idea: JFlattener

A **read-only** tool concept (does not exist yet) that could help developers understand code faster. JFlattener does **not** rewrite, refactor, or modify the code in any way - it is purely a **reading aid** that presents existing code in a flattened view.

- **Flatten the code on demand** - inline method calls into a single readable flow, for reading purposes only
- **IDE plugin** (IntelliJ) - integrated into the developer workflow as a read-only view
- **AI skill** - cheaper and faster for AI to read a single big flow than to navigate across files
- **Stacktrace to flat** - perfect for debugging and finding issues in production logs
- **Key benefit**: help developers keep the execution flow organized and visible, not only in their heads. Faster comprehension without changing a single line of code.

## Open Discussion & Feedback

The session opened into a rich discussion with the group:

| Question | Key Points |
|----------|------------|
| **How real is the "code reading issue"?** | Many people struggle indeed - widely acknowledged in the room |
| **How do you read code? Selectively or in-depth?** | Depends on the situation. Running tests helps. Debugging helps. Reading generic info first rather than rushing into the specific problem |
| **Do you use bookmarks, breakpoints, etc.?** | Yes, commonly used navigation aids |
| **What tools do you use?** | [Marit van Dijk](https://www.youtube.com/@maritvandijk) has many excellent talks about code reading tools and techniques |
| **How do you train code reading?** | Marit's talks are great. Some books exist. The [Code Reading Club](https://codereading.club/) is an excellent resource - experience how others read and understand the same code, gaining different perspectives |
| **Should IDEs optimize for navigation or understanding?** | Both - and it's already in progress |
| **Is code reading now a bigger bottleneck than code writing?** | Yes, absolutely |
| **How much flattening is useful before it becomes overwhelming?** | Depends on context and depth |
| **Could this improve onboarding for large enterprise codebases?** | Yes. Also suggested: use Claude (or other AI) to generate tours for the CodeTour plugin to onboard you in a project. It works well |
| **Would you use flattening during code reviews?** | For large features - yes. Otherwise, probably not |
| **Is flattening language-specific?** | The concept could be language-agnostic |

## Resources

- [CodeTour IntelliJ Plugin](https://plugins.jetbrains.com/plugin/19227-codetour) - Record and play guided code walkthroughs
- [Code Reading Club](https://codereading.club/) - Community for practicing code reading together
- [Marit van Dijk's YouTube](https://www.youtube.com/@maritvandijk) - Talks on code reading and developer tools
