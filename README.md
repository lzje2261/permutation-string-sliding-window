# permutation in string: sliding window explained with working code, the bugs that fail submissions, and the follow-ups interviewers ask

The problem, as LeetCode #567 states it: given two strings `s1` and `s2`, return `true` if `s2` contains a permutation of `s1`. In other words, one of `s1`'s permutations has to appear as a contiguous substring of `s2`.

text
Input:  s1 = "ab", s2 = "eidbaooo"
Output: true        // "ba" is a permutation of "ab" and sits at index 3

Input:  s1 = "ab", s2 = "eidboaoo"
Output: false


Constraints: both strings are between 1 and 10,000 characters, lowercase English letters only.

That's the whole setup. Everything hard about this problem lives in the gap between "sounds easy" and "should run in one pass."

## Why the two obvious approaches both fail

**Generate every permutation of `s1` and search for each one in `s2`.** A string of length 10 has 3.6 million permutations. Length 20 has more permutations than you can write down before the interview ends. This is a non-starter, and knowing *why* it's a non-starter is the first step toward the real solution.

**Check every substring of `s2` with the right length and compare characters.** Better, but still slow if you rebuild the counts from scratch each time: you'd touch `len(s2)` windows and `len(s1)` characters per window, giving O(n·m).

Both approaches share the same blind spot. They treat each candidate as unrelated to the previous one, when in fact consecutive windows of the same length overlap almost entirely.

## The insight that collapses the problem

Two strings are permutations of each other exactly when they contain the same characters with the same frequencies. "ba" and "ab" are permutations because both are one `a` and one `b`. Nothing about order matters, only counts.

Combine that with the contiguity requirement and the problem becomes concrete:

> Scan `s2` with a window of exactly `len(s1)` characters. If any window's letter counts equal `s1`'s letter counts, return `true`.

Since `s1` is length 2 in the example, every candidate window is length 2. Slide across `eidbaooo`:

| Window | Counts | Match? |
| --- | --- | --- |
| `ei` | e:1, i:1 | no |
| `id` | i:1, d:1 | no |
| `db` | d:1, b:1 | no |
| `ba` | b:1, a:1 | yes → return true |

The window length is fixed by the input rather than by a running constraint. That distinction matters, and it's the thing that separates this problem from the other string problems that also use a window.

## Working code, simple version first

Resist the urge to write the cleverest version immediately. Get a correct O(n) solution on the board, then tighten it.

python
def checkInclusion(s1: str, s2: str) -> bool:
    if len(s1) > len(s2):
        return False

    need = [0] * 26
    window = [0] * 26

    for ch in s1:
        need[ord(ch) - ord('a')] += 1

    # Build the first window by hand
    for i in range(len(s1)):
        window[ord(s2[i]) - ord('a')] += 1

    if need == window:
        return True

    # Slide: add the entering character, remove the leaving one
    for right in range(len(s1), len(s2)):
        window[ord(s2[right]) - ord('a')] += 1
        window[ord(s2[right - len(s1)]) - ord('a')] -= 1
        if need == window:
            return True

    return False


Two arrays of 26 integers, updated with two operations per slide. No window is ever rebuilt, which is exactly the waste the naive version created. The constant 26 comes from the constraint that inputs are lowercase English letters; if you're handed arbitrary Unicode, you'd swap the arrays for hash maps and the space bound changes from O(1) to O(k) where k is the number of distinct characters.

### The optimized match-check

Comparing two 26-element arrays on every slide means up to 26 integer comparisons per position. That's O(26n), which is genuinely O(n) because 26 is a constant, so it's not a correctness problem. Interviewers still like the version that does O(1) work per slide, which tracks how many of the 26 letters currently have equal counts.

python
def checkInclusion(s1: str, s2: str) -> bool:
    if len(s1) > len(s2):
        return False

    need = [0] * 26
    have = [0] * 26

    for i in range(len(s1)):
        need[ord(s1[i]) - ord('a')] += 1
        have[ord(s2[i]) - ord('a')] += 1

    matches = sum(1 for i in range(26) if need[i] == have[i])
    if matches == 26:
        return True

    for right in range(len(s1), len(s2)):
        idx_in = ord(s2[right]) - ord('a')
        have[idx_in] += 1
        if need[idx_in] == have[idx_in]:
            matches += 1
        elif need[idx_in] + 1 == have[idx_in]:
            matches -= 1

        idx_out = ord(s2[right - len(s1)]) - ord('a')
        have[idx_out] -= 1
        if need[idx_out] == have[idx_out]:
            matches += 1
        elif need[idx_out] - 1 == have[idx_out]:
            matches -= 1

        if matches == 26:
            return True

    return False


The bookkeeping is easy to get subtly wrong: you increment or decrement `matches` only when a count crosses the equality boundary in the relevant direction. Adding a character that makes two counts equal is a match gained; adding one that pushes a previously equal count out of alignment is a match lost. Same logic in reverse for the character leaving the window.

**Complexity for both versions:** O(n) time where n = `len(s2)`, O(1) space (two fixed arrays of 26, or O(k) with hash maps over an unbounded alphabet).

## The bugs that actually cost people the interview

Most failed submissions on this problem are not conceptual failures. They're one of these five.

**Skipping the length check.** If `s1` is longer than `s2`, no window can ever fit. Without the guard you either return a wrong answer or index out of bounds while building the first window.

**Off-by-one on the window.** The window ending at index `right` has length `len(s1)` and its leftmost character is at `right - len(s1)`. Getting this wrong by one produces windows of the wrong size, and the failure is intermittent enough to look like a logic bug in the comparison rather than an indexing slip.

**Zero-valued keys in a dictionary.** `{'a': 0, 'b': 1}` and `{'b': 1}` are not equal in Python, so a dictionary-based solution that decrements a count to zero and leaves the key in place will miss valid windows. Either delete the key when its count hits zero, or use fixed-size arrays and sidestep the issue entirely.

**Rebuilding counts inside the loop.** Correct, but O(n·m), and on n = 10,000 it may still pass while signaling that you ddin't notice the overlap between consecutive windows. Interviewers read that as a missed optimization, not a stylistic preference.

**Sorting each candidate window.** Sorting is O(k log k) per window where k = `len(s1)`. With a 26-letter alphabet, counting is strictly cheaper and simpler.

## What interviewers ask after your code runs

The follow-up conversation is usually more revealing than the solution itself.

**"What's the space complexity, and does it change if the input is Unicode?"** The 26-array answer only holds because the alphabet is bounded. With arbitrary characters you need a hash map sized by the number of distinct characters, and the bound becomes O(k) instead of O(1).

**"What if I want every starting index, not just a boolean?"** That's LeetCode #438, *Find All Anagrams in a String*. Identical mechanics, except you collect `right - len(s1) + 1` instead of returning early. If you can solve 567 cleanly, 438 is a two-line change.

**"What if the pattern doesn't have to match exactly?"** Now the window stops being fixed-length and becomes LeetCode #76, *Minimum Window Substring*. You grow the window until it satisfies the requirement and shrink it while it still does, which is the variable-length version of the same idea. Same skeleton, different loop structure.

**"Why is a fixed window correct here?"** Because a substring of `s2` must be contiguous and its length must equal `len(s1)` for it to be a permutation of `s1`. There's no shrinking or growing decision to make.

### Related problems worth solving in the same sitting

| Problem | Number | What changes |
| --- | --- | --- |
| Valid Anagram | 242 | Same counts idea, no window at all |
| Find All Anagrams in a String | 438 | Return all indices instead of `true`/`false` |
| Minimum Window Substring | 76 | Variable-length window |
| Longest Substring Without Repeating Characters | 3 | Variable-length window, no target string |
| Longest Repeating Character Replacement | 424 | Variable window plus a frequency condition |

Doing 242 and 438 back to back after 567 takes maybe 20 extra minutes and makes the pattern feel like one idea instead of three separate memorized solutions.

## Recognizing this pattern before you start coding

The practical question in an interview isn't "how do I implement sliding window," it's "how do I know this is a sliding window problem." A decision rule that holds up: if the substring you're looking for has a fixed length determined by the input, use a fixed-size window. If the valid substring's length depends on a condition you're trying to satisfy or optimize, use a variable-size window. Add the word "anagram," "permutation," or "same characters" to the problem statement and you're almost certainly looking at frequency counting inside a window.

One caveat about how often this specific problem comes up: at least one community-maintained pattern repository tags #567 as *occasional* rather than high-frequency in interview loops. Treat it as a pattern-teaching problem more than a must-memorize one. The mechanics, not the exact question, are what recur.

## Where structured practice helps, and where it doesn't

If this problem felt obvious to you from the first paragraph, you don't need a course. Solve 438 and 76 to confirm the transfer, then move on.

If it didn't, the useful thing isn't another explanation of sliding window. It's a way to practice the recognition step repeatedly, because the failure mode in real interviews isn't writing the loop, it's burning the first ten minutes deciding which technique applies.

AlgoMonster is built around that specific gap. It's a pattern-based course created by ex-Google engineers, and the official subscribe page lists access to 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus an AI assistant and a "Speedrun" mode. There's a public editorial walkthrough for problem 567 with solutions across multiple languages and complexity notes alongside it: 👉 [read the permutation in string walkthrough](https://algo.monster/liteproblems/567?fpr=li69).

A few things about the platform worth knowing before you pay:

- **The format is text and diagrams, with no video.** Every review of it says the same thing. If you learn by watching someone code, this is the wrong tool and NeetCode or AlgoExpert is the right one.
- **The curated problem set is small.** Review sites consistently put it around 230 problems. That's a fraction of LeetCode's 3,000+, and the sensible workflow most people describe is AlgoMonster for the frameworks, LeetCode for volume.
- **System design coverage is introductory.** For L5+ loops, plan on a separate resource.
- **There's a free tier.** It's limited, but the Monster 50 and Blind 75 practice lists can be worked through on a free account, which is a reasonable way to test whether the teaching style fits before spending anything.
- **Speedrun is drilling, not solving.** You answer multiple-choice questions about which technique and template apply to a problem, which targets recognition speed directly. It's a different activity from writing the code, and it's the part that maps most closely to the problem above.

The honest framing: AlgoMonster is a good fit if your bottleneck is *pattern selection* and you have a 4–8 week window to prepare. It's a poor fit if you can already name the right technique on sight and just need reps, or if you need live mock interviews and pressure simulation, neither of which it provides.

## Current plans and pricing

AlgoMonster doesn't sell a monthly subscription. The subscribe page lists a free option and two Pro plans:

| Plan | What you get | Price | Billing | Get started |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; free practice lists | $0 | — | [Create a free account](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; company question sets updated monthly; 500+ lessons across 6 courses; 12 months of access; member discount on live bootcamps | $99/year (~$8.25/month) | Billed annually, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in Pro Yearly, paid once, plus lifetime updates and future bonus courses | $189 | One-time payment | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two notes on those numbers. AlgoMonster runs promotions often, and third-party review sites quote figures ranging from roughly $89 to $299 a year and $99 to $459 for lifetime depending on when they checked, so the checkout page is the only source that counts. And because there's no monthly option, the annual plan is really a one-year commitment; if you're prepping for interviews two months from now and expect to interview again later, the lifetime plan is the version that doesn't expire while you're at a job you like.

Bonus courses are included with a paid plan at no extra cost, which currently means the Foundation Course (data structures and algorithms fundamentals) and a "Build an LLM" course covering tokenization, attention, and training.

## The short version

`s2` contains a permutation of `s1` when some window of length `len(s1)` in `s2` has identical letter frequencies to `s1`. Maintain those frequencies with two 26-element arrays, slide one character at a time, and return `true` the moment the counts align. Check the length first, watch the window boundaries, and don't rebuild counts you can update.

Then prove you understood it by doing 438 next, because the only real test of pattern recognition is whether it transfers to a problem you haven't memorized.
