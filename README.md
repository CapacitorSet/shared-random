# shared-random

`shared-random` is a library for generating verifiably-random numbers between multiple parties.

A real-world example is an online casino, where the user and the server don't trust each other. `shared-random` enables them to generate a number which is guaranteed to be random.

## Usage

Suppose the library is used in a web app for playing roulette. The server and the client must agree on a random number between 0 and 36; however, neither trust each other.

 1. The server generates a random passphrase, for instance `ICKOKJuL8hxjp4EQ`. It sends its hash to the client (the result of `choose_passphrase("ICKOKJuL8hxjp4EQ")`, which is `fd29c8e8...`).
>This is a guarantee that the server chose a passphrase, and will not change it.

 2. The client also generates a random passphrase, for instance `sfc9tT0wO9jhDNii`, and sends its hash to the server (`choose_passphrase("sfc9tT0wO9jhDNii")`, which is `60dabc0f...`).

 3. Each party discloses his passphrase to the other: the client to the server, and the server to the client.

 4. Each party verifies that the passphrase they received is correct (matches the hash). For instance, the client calls `verify_passphrase("ICKOKJuL8hxjp4EQ", "fd29c8e8...")`, which must return `true`.
>This is a guarantee that the server did not change his passphrase after reading the client's (doing so would have enabled the server to skew the random number in its favour). At this time, there are no two strings that produce the same hash.

 5. Each party generates the random number from the two passphrases, by calling `generate_random(["ICKOKJuL8hxjp4EQ", "sfc9tT0wO9jhDNii"], 37)`. They get the same result, `11`.
>The range parameter is `37` because we want a number between 0 and 36.

## Algorithm

In the first stage, each party generates a seed: `S1`, `S2`, `S3`...

To ensure that no party manipulates their seed in order to skew the random generator, it is required that everyone share the SHA1 hash of their seed, producing `H1`, `H2`, `H3`... Any observer can thus verify that the seed disclosed in the next stage was not manipulated; however, it doesn't reveal anything about the seeds themselves, ensuring that every party chooses their seed fairly.

Then, each party discloses their seed publicly, and verifies that every seed matches with its hash (`hash(H1) == S1`, `hash(H2) == S2`, ...).

Once every seed is known, each party can calculate the random number, by evaluating `SHA1(sort(S1, S2, S3...).concatenate()) modulo n` (where `+` represents concatenation, and `n` is the range for the random number `[0, n)`). Because the list of seeds is sorted, the calculation is completely deterministic (no two observers can calculate different numbers).

>The second step is fundamental to the fairness of the algorithm. If parties weren't required to disclose their hashes, an attacker could manipulate his seed to get the desired result.

>For instance, suppose that a number between 0 and 100 is generated, that the attacker wants the result to be 30, and that parties aren't required to disclose their hashes. The attacker waits for the other party to disclose their seed, for instance `foobar`. Then, the attacker calculates `generate_random(["foobar", my_seed], 101)` for many values of `my_seed`, until he finds a value (eg. `abc14`) that generates `30`, and discloses this seed to the other party.

## Known weaknesses

 * The library relies on the impossibility to find a collision for SHA1 in a sufficiently short time. At this time, no collisions are known, and no algorithms are known that can find a collision in a short time (eg. less than 10 minutes).
 * Although the library forces every party to be honest, it can't require everyone to collaborate (for instance, one could refuse to disclose his hash). This is a problem that must be handled externally.
 * The random generator relies on calculating a SHA1 hash modulo n. The result is perfectly unbiased if n is a power of two; else, it introduces an *extremely* tiny bias.
>Let `k = (2 ** 160) % n`. The first `k` outputs (`0, 1, 2, ... k-1`) have a higher probability to be selected, in particular, `b = 2 ** (-160)` more.

>For instance, if `n = 100`, the numbers `0, 1, 2, ... 75` have a probability `P' = 0,10000000000000000000000000000000000000000000000164214664...`; the numbers `76, 77, ... 99` have a probability `P = 0,00999999999999999999999999999999999999999999999947998689...`, for a difference of one part in 14615016373309029182036848327162830196559325429.

>If `n = 2 ** 100`, the difference is of one part in 1152921504606846976.

Essentially, the random generator introduces an *extremely* tiny bias, which is only relevant for *extremely* large values of `n` (several billion billion billion billions).