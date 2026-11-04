# Documentation agent: token cost of one run and of a 10-iteration error loop

## Measurement

The system prompt was measured with the tokenization tool for **GPT-5.x and o1/o3** (the `o200k` family used by those models).

| Piece | Tokens |
| --- | ---: |
| System prompt | 58 |
| Code file injected after the prompt | 15,000 |
| Assistant response, each iteration | 500 |

Rates used for every figure below:

- Input: **$5 / 1,000,000 tokens**
- Output: **$15 / 1,000,000 tokens**

No other messages are added between iterations. The only growth is the previous 500-token response, which is kept in the history and sent again on the next call.

## Single execution

Input is the prompt plus the code file. Output is the one response.

```
input  = 58 + 15,000 = 15,058 tokens
output = 500 tokens

input cost  = 15,058 × 5  / 1,000,000 = 0.07529
output cost =    500 × 15 / 1,000,000 = 0.00750
total       = 0.08279
```

**One execution costs $0.08279.**

The same total as a single fraction:

```
(15,058 × 5 + 500 × 15) / 1,000,000 = 82,790 / 1,000,000 = 0.08279
```

## Error loop: context grows by the last response

Iteration `n` (from 1 to 10) re-sends everything already sent, plus every earlier response:

```
input_n  = 15,058 + (n − 1) × 500
output_n = 500

cost_n = (5 × input_n + 15 × 500) / 1,000,000
       = (82,790 + 2,500 × (n − 1)) / 1,000,000
```

Each new iteration therefore costs **$0.0025** more than the one before it: 500 extra input tokens × $5 / 1,000,000. Output cost stays at $0.00750 because every response is 500 tokens.

| Iteration | Input tokens | Output tokens | Input cost | Output cost | Iteration cost | Cumulative |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 15,058 | 500 | 0.07529 | 0.00750 | 0.08279 | 0.08279 |
| 2 | 15,558 | 500 | 0.07779 | 0.00750 | 0.08529 | 0.16808 |
| 3 | 16,058 | 500 | 0.08029 | 0.00750 | 0.08779 | 0.25587 |
| 4 | 16,558 | 500 | 0.08279 | 0.00750 | 0.09029 | 0.34616 |
| 5 | 17,058 | 500 | 0.08529 | 0.00750 | 0.09279 | 0.43895 |
| 6 | 17,558 | 500 | 0.08779 | 0.00750 | 0.09529 | 0.53424 |
| 7 | 18,058 | 500 | 0.09029 | 0.00750 | 0.09779 | 0.63203 |
| 8 | 18,558 | 500 | 0.09279 | 0.00750 | 0.10029 | 0.73232 |
| 9 | 19,058 | 500 | 0.09529 | 0.00750 | 0.10279 | 0.83511 |
| 10 | 19,558 | 500 | 0.09779 | 0.00750 | 0.10529 | 0.94040 |

**Cumulative cost over 10 iterations: $0.94040.**

## Closed form (why the cumulative figure is exact)

Input tokens form an arithmetic series. With `k = n − 1`:

```
total input  = Σ (15,058 + 500k)  for k = 0..9
             = 10 × 15,058 + 500 × (0 + 1 + … + 9)
             = 150,580 + 500 × 45
             = 173,080

total output = 10 × 500 = 5,000
```

```
input cost  = 173,080 × 5  / 1,000,000 = 0.86540
output cost =   5,000 × 15 / 1,000,000 = 0.07500
cumulative  = 0.94040
```

The same result from the per-iteration numerators, which avoids rounding:

```
Σ (82,790 + 2,500k)  for k = 0..9
  = 10 × 82,790 + 2,500 × 45
  = 827,900 + 112,500
  = 940,400

940,400 / 1,000,000 = 0.94040
```

Cross-check with the first and last terms of the cost series (common difference 0.0025):

```
10 / 2 × (0.08279 + 0.10529) = 5 × 0.18808 = 0.94040
```

All three paths agree, so the cumulative cost accounts for the growing history and is exact at five decimal places. Every cent is an integer number of hundred-thousandths of a dollar; nothing was rounded along the way.

Ten independent calls, each forgetting the previous response, would cost `10 × 0.08279 = 0.82790`. Keeping the history adds `22,500` extra input tokens (`500 × 45`), which is `22,500 × 5 / 1,000,000 = 0.11250`. That is the whole gap:

```
0.82790 + 0.11250 = 0.94040
```

## Self-assessment and how to stop the growth

The cumulative total is the sum of an arithmetic series, not `10 ×` the first call. Iteration 10 already costs $0.10529, about 27% more than iteration 1, and the extra is only repeated input. Output spend is flat at $0.07500 for the whole loop. The growth term `500 × n(n − 1) / 2` is quadratic in the number of iterations, so the same loop at 100 iterations would resend `500 × 4,950 = 2,475,000` historical tokens and the premium alone would be $12.375.

Context truncation removes that quadratic term. Keep the stable prefix (the 58-token prompt and the 15,000-token file) and only the latest 500-token response. Iteration 1 is unchanged. Iterations 2 through 10 each send 15,558 input tokens instead of climbing to 19,558.

```
input tokens = 15,058 + 9 × 15,558 = 155,080
cost         = (155,080 × 5 + 5,000 × 15) / 1,000,000 = 0.85040
```

That is $0.09000 less than the full-history loop ($0.94040 − $0.85040), and only $0.02250 above ten stateless calls. Dropping every previous response, when the next user message already contains the error, returns the loop to `10 × 0.08279 = 0.82790`.

Truncation bounds the history. It does not remove the 15,058-token prefix, which is identical on every call and is still the bulk of the bill (about $0.75 of the $0.94). A prompt cache on that prefix would cut the repeated input rate without changing the token counts above. The two fixes apply to different parts of the bill: truncation stops the history from growing, and caching discounts the prefix that has to be sent anyway.
