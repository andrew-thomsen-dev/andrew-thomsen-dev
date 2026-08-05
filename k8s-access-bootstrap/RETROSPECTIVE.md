# Lessons Learned: Building an Access Setup Tool for a Non-Technical Team

This is a companion piece to the [case study](./case-study-k8s-onboarding.md) and [architecture concepts](./architecture-concepts.md) documents. Where those focus on the problem and the design, this one is about what the process of building and iterating on the tool actually taught — including the parts that didn't work the first time.

## 1. Rewrite the process before you automate it

The instinct when a manual process is painful is to jump straight to scripting it. The more valuable step happened before that: rewriting the underlying runbook itself, for its actual audience, before writing a single line of automation. Doing that first surfaced exactly which steps were genuinely necessary decisions a human had to make, versus steps that only felt necessary because the original documentation buried the actual logic in unrelated detail. Automating a bad process just makes the bad process faster. Clarifying it first is what made automating it straightforward.

## 2. Don't trust a shared document as if it were an API contract

The tool depends on a wiki page maintained by hand, by multiple people, over time. Early on, it would have been easy to treat that page's attachments as clean, structured data. They weren't. Real revisions included an accidentally duplicated configuration section that broke a downstream tool's parser with an unhelpful generic error, and a byte-order-mark character — invisible in every normal way of inspecting the file — that broke parsing in a way that looked, to a human, like nothing was wrong at all.

The lesson wasn't "fix those two specific things." It was: **any boundary where a human-maintained document becomes an input to automated tooling needs its own validation layer**, because the failure modes at that boundary are different from normal software bugs — they're data-quality problems that only show up when a person made an editing mistake, and they need to be caught and explained at the point of ingestion, not several steps later when a completely unrelated tool chokes on the result.

## 3. Platform-specific failures need platform-specific understanding, not pattern matching

A cryptic low-level error (a binary refusing to execute at all) turned out to have nothing to do with the tool being installed incorrectly. It was caused by the terminal session itself running under an emulation layer for a different chip architecture than the machine actually had — a distinction that isn't visible from the error message at all, and that a first, reasonable-looking theory (guessing it was a leftover incompatible binary) didn't fully explain.

Getting this right required actually understanding the mechanism — the difference between a machine's real hardware and the architecture a *specific running process* is currently emulating — rather than pattern-matching the error text to a plausible-sounding fix. The generalizable lesson: when a failure looks unrelated to the change that supposedly caused it, that's a signal to go find the actual mechanism, not to apply the most common fix for similar-looking symptoms.

## 4. The most important debugging skill is knowing what you haven't ruled out yet

The hardest bug in this project was a final verification step intermittently reporting "not found" for something confirmed to be running perfectly well, seconds later, by hand. Over several rounds, multiple plausible explanations were tested and each one legitimately made the check *more correct* in some way — and none of them changed the actual outcome:

- **First theory:** the check couldn't tell a real connection failure from an empty result. True, and worth fixing on its own merits, but not the cause here — there was no connection failure.
- **Second theory:** the check was only looking in the wrong location (a namespace-scoping gap). Also a real, worth-fixing gap in the code — but a direct side-by-side comparison showed the *exact same command*, run immediately after, succeeded without it.
- **Third theory:** it was a timing issue immediately after a fresh authentication, so a short automatic retry was added. Better-targeted, but still didn't resolve it in practice.

At that point, the right move wasn't a fourth increasingly-elaborate theory. It was stepping back, collecting direct evidence (the literal command the person could run themselves, the exact context configuration, the exact tool resolution) rather than more inference, and being willing to simplify the implementation back down to the exact, known-to-work manual command rather than keep layering cleverness on top of a mechanism that wasn't actually understood yet.

**The lesson:** each fix attempt should be treated as a hypothesis with a falsifiable prediction, not a patch. If reality doesn't match the prediction, that's more valuable information than the fix itself — and it's a much stronger signal to slow down and gather direct evidence than to try a fifth variation on the same idea.

## 5. Sometimes the correct fix is to remove sophistication, not add it

Related to the above: the final version of the verification step is deliberately *less* defensive than an earlier version — it dropped a retry loop and a broader search scope that had both been added in good faith to fix the exact symptom being reported. Neither one turned out to be addressing the real mechanism, and both added complexity and runtime cost without a corresponding benefit. Reverting to the simplest possible version, matching exactly what was already confirmed to work by hand, was the right call — not because simple is always better, but because in this case the added logic had been shown, empirically, not to matter.

It's worth being comfortable walking back a "fix" once the evidence says it isn't one, rather than defending it because it represents completed work.

## 6. Small friction compounds heavily for a non-expert audience

None of these individually sound like they'd matter much, but each one came directly from a real person's confusion or a real missed step:

- A masked password-style input with *zero* visual feedback is indistinguishable, to someone unfamiliar with terminals, from a broken input field. Showing a character of feedback per keystroke (without revealing the actual value) removed that ambiguity.
- A URL mentioned once in a paragraph, several screens of output earlier, is easy to miss entirely. Repeating it directly in the prompt where it's needed — and offering to just open it in a browser — removed the need to find it at all.
- Unlabeled, continuous terminal output reads as one undifferentiated, possibly-broken process to someone who doesn't know what "normal" output is supposed to look like. Numbering the steps turned the same output into something with a visible beginning, middle, and end.

None of these are hard engineering problems. They're the kind of thing that's easy to skip because they don't affect whether the tool *works* — but they heavily affect whether the intended audience is willing and able to use it at all, which was the entire point.

## 7. Design explicitly for the repeat case, not just the first run

The underlying authentication system this tool wraps requires re-authenticating roughly once a day — a permanent, ongoing cost, not a one-time setup step. Treating "I already did this once" as a first-class case (checking whether the expensive parts are still fresh before repeating them) turned a recurring daily cost into something that takes seconds. It's easy to design a tool around "the happy path of running it for the first time" and let every subsequent run pay the same cost as the first — worth explicitly asking, for anything meant to be run repeatedly, what the *second* run should cost, not just the first.

## 8. A changelog that explains *why*, not just *what*, is worth the overhead

Because this tool evolved through real, sometimes contradictory, user reports, keeping a running record of each change *and the reasoning behind it* — including the two changes that were later reverted — made it possible to avoid re-litigating settled questions, and made it much easier to reconstruct, later, why the current version looks the way it does. A changelog entry that just says "fixed pod check" is nearly worthless six iterations later; one that says what was tried, what evidence contradicted it, and what replaced it is what actually let the next round of debugging start from where the last one left off instead of from scratch.

## Closing thought

Nothing here was a single clever fix. It was a lot of ordinary decisions — validate untrusted input, understand mechanisms instead of pattern-matching symptoms, treat each fix as a testable claim, and take real user friction as seriously as real bugs — applied consistently, to a tool whose actual success metric was never "does the code run" but "will the person it's for actually use it."
