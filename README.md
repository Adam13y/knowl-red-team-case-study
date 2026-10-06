# I tried to poison an AI agent's memory. Here is what broke and what got fixed.

The project is [Knowl](https://github.com/dat999zx/knowl), an open source memory server for coding agents. I tested it in August and September 2026, and I am naming it here with the maintainer's permission.

## The problem

A lot of AI memory systems replace an old fact when a newer note about the same thing comes in. That is what you want for real updates. It also means one wrong note can quietly replace a true fact, and after that the agent gives the wrong answer with full confidence.

The maintainer, dat999zx, put the sharper question to me in [issue #165](https://github.com/dat999zx/knowl/issues/165): can you tell a malicious overwrite apart from a real correction at the moment it is written? I set out to answer that.

## What I tested

I wrote 12 facts that a coding agent really stores, things like token lifetime, the deploy approval rule, backup retention and the data region. I saved them as verified, so I knew the right answer to every one.

Then I sent realistic wrong writes at them. Some were plain contradictions. Some carried a fake "observed" label. Some were built exactly like a real correction. And some were loophole writes that keep the true rule but add an exception, like "two reviewer approvals or an admin override".

I sent them through the normal store path. After the fixes I also tested two channels separately: direct, where the agent calls the store tool itself, and automatic, which covers capture and ingest.

Every test ran 3 times on a fresh store. With 12 facts and 3 runs that is 36 writes per test, and every run gave the same result. For each write I checked three things: was the true fact removed, did anything flag it, and what answer does the agent actually get back when it asks.

## What I found on version 5.16.0

All of this is on commit 2f97ae5.

1. 216 of 216 attack writes removed the verified fact. After every one of them, the true fact was no longer in the top 3 answers, and in 207 of the 216 the false value was the top answer.
2. Nothing flagged it. The conflict check only compared facts that were still active, and the true fact was not active anymore.
3. You could not catch it by looking at the write. A fake write and a real correction for the same fact were identical on every field the system checks.
4. A write that says nothing false could still erase a fact. Putting "see the ops runbook" in place of the number removed the real value 36 of 36 times.
5. The "exclusive" protection could be skipped by leaving one field out. That worked 36 of 36 times.

The full report and the test harness are in [issue #165](https://github.com/dat999zx/knowl/issues/165).

## What got fixed

I made five recommendations and the maintainer shipped four of them. I posted my report on 24 September, and the fixes were merged on 28 September, four days later. They went out in [v5.24.0](https://github.com/dat999zx/knowl/releases/tag/v5.24.0), and the release notes open with a credit to this test.

Here is what changed:

1. Automatic writes can no longer remove a verified fact ([#319](https://github.com/dat999zx/knowl/pull/319)).
2. "Exclusive" facts stay protected even when a write leaves the key out ([#320](https://github.com/dat999zx/knowl/pull/320)).
3. `knowl conflicts` now lists verified facts that were removed, and what replaced them ([#321](https://github.com/dat999zx/knowl/pull/321)).
4. A vague restatement no longer removes a fact that has a real value ([#322](https://github.com/dat999zx/knowl/pull/322)).

The one that did not ship is R4, recording which channel a write came in through. The maintainer notes that it would remove the ranking trade off I describe further down, for real corrections on the automatic channel.

When I reran my tests on the fixed version, I found one more gap. On the automatic channel the true fact was now kept, but a planted note could still rank above it ([#323](https://github.com/dat999zx/knowl/issues/323)). I wrote a patch for that. The maintainer reviewed it, added a guard, and merged it with me as co-author in [PR #325](https://github.com/dat999zx/knowl/pull/325), which shipped in [v5.24.1](https://github.com/dat999zx/knowl/releases/tag/v5.24.1). In the PR he wrote:

> "He found the hole and wrote the fix; I reviewed it, added one guard and ran the checks he could not."

## Before and after

Every count here is out of 36 writes.

1. An automatic write removing a verified fact went from 36 in every attack type to 0 in every attack type.
2. An exclusive fact being removed when the key is left out went from 36 to 0.
3. Removed facts that anyone other than the writer could see afterwards went from none to all of them. In my rerun, all 216 removals on the direct channel showed up in `knowl conflicts`, which lists removals from the last 14 days.
4. The "see the runbook" write went from 36 removals to 9. The other 27 are now kept side by side with the real fact. The 9 that still remove it are the cases where the real value is written in plain words instead of numbers, so that one is still an open gap.
5. On the automatic channel, the false value coming out as the top answer went from 21 to 0 for the forged label attack, from 12 to 0 for the superset title, from 9 to 0 for the plain contradiction and from 3 to 0 for the terse write. The "before" numbers here were measured after the first fixes and before the ranking fix in v5.24.1.

## What is still open, on purpose

If an agent is persuaded to call the store tool directly with a lie shaped like a correction, the lie still replaces the fact. Nothing can tell that apart from a real fix at the moment it is written. What changed is that it is now visible to someone other than the writer.

In my rerun on v5.24.1, the control case of "X" against "X no longer" still put the wrong one first in 6 of 36 runs. All 36 were marked contested, so the agent is told there are two answers.

A real correction that comes in through the automatic channel now ranks below the old fact until someone resolves the pair. The maintainer documented this trade off himself.

## What I took away from it

"Newest wins" is also an attack path. Anything your agent reads, whether a README, an email or a web page, can end up as a "newer" fact.

Checking the write is not enough. A good lie looks exactly like a good correction. What helps is knowing where the write came from, and making sure a human can see when a trusted fact changes.

Test with known answers. Save a few facts you know are true, send a believable wrong version through the same path your agent uses, then ask again. If you get the wrong answer back, you have this problem too.

## Links

1. The report, the harness and the whole conversation: [issue #165](https://github.com/dat999zx/knowl/issues/165)
2. The ranking fix I wrote, merged with me as co-author: [PR #325](https://github.com/dat999zx/knowl/pull/325)
3. The release with the four fixes: [v5.24.0](https://github.com/dat999zx/knowl/releases/tag/v5.24.0)

I do this kind of testing for AI agents, chatbots and automations, so if you want yours checked the same way, you can find me on GitHub as [@Adam13y](https://github.com/Adam13y).
