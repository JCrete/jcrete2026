# Do you need ID in the era of AI Agents?

What do we use the IDE for?
* Running the code
* Refactoring deterministically without spending tokens
* Navigate through code
* Refactoring
* Debugging
* Reviewing the code in the IDE (incl navigating)
* Renaming
* Git tooling
* Navigate & understanding

```
If noone looks at the code, we don't need an IDE. For code we do look at, we will still need an IDE.
```

## Workflow
- ChatGPT for ideas
- Claude for generating code
  - Generate Specs
  - Read specs
  - Generate code
  - Read that code (the closer you get to production)
- Still writing 5% of code, more than previously.

Peter: "Generate code with AI, but then carefully rewrite it manually"

Cliff: "Generate code with AI, read and edit in Emacs, debug with IntelliJ IDEA"


### Notes: 

If the code from the AI would be perfect we wouldnt need an IDE, but we know its not.

Everything but writing code? Some people still (re)write code. 

Someone uses AI more for debugging and profiling. But the IDE for reviewing and (re)writing.

You need IDE to navigate the code (for languages that have good IDE support)

Mary: "This is a room full of people already used to using one. Its part of our process. What about the next generation of developers?"

Integrating ArchUnit to visualise PlantUML diagrams

Arno Would expect the IDE to help with understanding what happened (in a large PR)

Alex would use whatever the best harnass is for a specific type of project, e,g Java, Maven, Kotlin, 

Evgeny - cannot follow the rationale behind the implemenentation done by AI

We need IDEs for coding agents:
- Should expose the functionality of the IDE to agents.
- Understanding the impact of the changes; reviewing the PR is the hardest task for a developer.
- IDE hasn't changed (enough) to support this workflow.
- Understanding the code.
- Writing the code was never the bottleneck, it was always the understanding.
- Code was never the job. 

Maurice: "need to see the code to understand it"

Yorgos - wants something to tailor to his preferences (theme, colors etc)

Working on things in parallel -> git worktrees

Renaming - the IDE does it deterministically, instead of grep.

More efficient with some models, but not others. 

Question from Alex: what is JB doing?

Question about agents and tools exposed

Pasha would like to have a skill to discover the tools from MCP (?)