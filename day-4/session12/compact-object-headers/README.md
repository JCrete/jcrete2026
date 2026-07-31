# From Compressed Oops to Compact Headers: Inside the JVM

## Slides

[Download the slides (PDF)](./Compact_Object_Headers_Slides_v1.1_EN_2026-07-30.pdf)

## Recording

I recorded this talk and made it available on YouTube: [Talk on YouTube](https://www.youtube.com/watch?v=8mjttkyf4Fk)

This video is a bit older:

- [JEP 534 (Compact Object Headers by Default)](https://openjdk.org/jeps/534) had not yet been targeted to a Java version.
- At around minute 32, I gave a wrong reason for the GC forwarding pointer. 
**Correction**: The forwarding pointer is used only during a stop-the-world phase.
Serial, Parallel, and G1 have to stop the world to move objects, because they don’t use a read barrier.
ZGC, on the other hand, can move objects concurrently with the running application, because it can follow moved objects via a read barrier. ZGC doesn’t use a forwarding pointer for this, but instead stores the forwarding information in a side table.
