# How Samy's Number Names Generator Works

A complete technical reference: the math, the naming rules, the parsing
algorithms, and the code, section by section. This applies to version 0.4.0.

---

## 1. What the site does

The site does three things:

1. **Number → name.** Type a number (in almost any format) and get back its
   name, type "1e87" and get "1 vigintioctillion".
2. **Name → number.** Type a name or its abbreviation and get back its
   number, type "vigintioctillion" and get "1e87"
3. **Browse.** Five drawers show representative names across the whole
   range the site supports.

Everything runs client-side in a single `index.html` file with no server, no
build step and no dependencies beyond three Google Fonts.
All the logic (the naming rules, both parsers, and the code that
builds the drawers) lives in one `<script>` block.

---

## 2. The mathematical foundation

Every name beyond "thousand" corresponds to a position in a numbered
sequence. If `n` is that position (called the **illion index** throughout
the code and this document), the number it names is:

```
value = 10^(3n + 3)
```

So:

| n | value | name |
|---|---|---|
| 0 | 10^3 | thousand |
| 1 | 10^6 | million |
| 2 | 10^9 | billion |
| 10 | 10^33 | decillion |
| 999 | 10^3000 | novemcentinonagintinonillion |
| 1000 | 10^3003 | myriillion |

This relationship is fixed and never changes, no matter which "system" is
generating the *word* for a given `n`. In the code it's two one-line
functions:

```js
function exponentForN(n){ return 3*n + 3; }
function exponentForNBig(n){ return 3n*n + 3n; }   // BigInt version, used for huge n
function nForExponent(e){ return Math.round((e-3)/3); }
```

The entire rest of this document is about one question: **given `n`, what
string of letters do we produce?** The site answers that differently
depending on how big `n` is, in three layers, described in Sections 4, 6,
and 7.

---

## 3. Where the naming system came from

The base of this system started as an implementation of **Sbiis Saibian's
Extensible Illion System**, from his web book *One to Infinity: A Guide to
the Finite* (the specific page is his "Sbiis Saibian's -illions", part of
a series on large-number naming conventions). His core idea, and the whole
reason this site doesn't use the more common Conway-Wechsler naming
convention, is this:

**Conway-Wechsler** builds names by welding three Latin prefixes together
and appending "-illion" once, at the very end: `un` + `viginti` + `illion`
→ *unvigintillion*. This has a real flaw. Latin's own word for "three
hundred" happens to already start with the same letters as "three" (tre)
glued to "hundred" (centi), so **both 103 and 300 collide on the exact
same word, "trecentillion."** That's not an edge case introduced by this
project; it's baked into the Latin roots themselves.

**Saibian's fix** is to build names the way English actually reads
compound numbers: additively, one whole recognizable word at a time.
Twenty-one isn't "twenty-and-one" mashed into a single prefix, it's
"twenty" then "one," as two separate words. Applied to illion names, `n =
21` becomes *vigintimillion* (viginti + the complete word "million"), not
a mashed prefix.

This site's naming system has since been **substantially extended past
what Saibian's own (unfinished) notes cover** - his page is explicitly
marked "under construction". Sections 6 and 7 describe the extension built for this
site, including two real bugs found and fixed during development that
aren't in Saibian's original notes at all.

---

## 4. Layer 1 - the base system (n = 0 to 999)

This covers everything from *thousand* up to *novemcentinonagintinonillion*
(10^3000). It has three sub-rules depending on `n`.

### 4.1 The irregular base (n = 0-20)

These are canonical dictionary words, memorized rather than constructed:
thousand, million, billion, trillion, quadrillion, quintillion, sextillion,
septillion, octillion, nonillion, decillion, undecillion, duodecillion,
tredecillion, quattuordecillion, quindecillion, sexdecillion,
septendecillion, octodecillion, novemdecillion, vigintillion.

In code, `n = 1` through `9` come from a lookup table:

```js
const LOW_NAMES = {0:"thousand",1:"million",2:"billion",3:"trillion",
  4:"quadrillion",5:"quintillion",6:"sextillion",7:"septillion",
  8:"octillion",9:"nonillion"};
```

`n = 10` through `20` are technically *compositional* (built from Latin
roots, see 4.4) but happen to be the same words a Conway-Wechsler system
would produce, so no special casing is needed for them beyond the general
composition rule.

### 4.2 The tens only words (n = 30, 40, ..., 90)

Each multiple of ten from 30 to 90 gets its own dedicated word, built from
a tens root plus "-illion":

```js
const SAIBIAN_TENS = ["", "deci","viginti","triginti","quadraginti",
  "quinquaginti","sexaginti","septaginti","octoginti","nonaginti"];
```

Notice this differs from Conway-Wechsler's own tens roots in two ways:
the endings are `-i` instead of `-a` (Saibian's own choice, he explicitly
rejects the `-a` spelling), and `septaginti` is missing the "u" that
Conway-Wechsler's `septuaginta` has. Both are deliberate, verified against
his stated text.

### 4.3 The composite tens+ones words (n = 21-29, 31-39, ..., 91-99)

This is where the additive philosophy shows up directly. Rather than
mashing a units prefix onto the tens root, the *whole word* for the ones
digit is attached:

```js
function saibianWord0to99(v){
  if(v <= 20) return nameForN(BigInt(v));
  const t = Math.floor(v/10), u = v % 10;
  if(u === 0) return joinParts([SAIBIAN_TENS[t], "illion"]);
  return SAIBIAN_TENS[t] + nameForN(BigInt(u));
}
```

So `n = 23` is `viginti` (twenty) + `trillion` (the complete word for 3) =
**vigintitrillion**. Not "trevigintillion." No letters are dropped or
elided at this join, the two words are concatenated exactly as they are.

### 4.4 The hundreds tier (n = 100-999)

This is the layer that actually resolves the 103 vs 300 problem. There are
two cases depending on the hundreds digit `h`:

**If h = 1** (n = 100-199): the prefix is simply `"centi"`, and it's glued
onto the *whole word* for whatever's left over:

```js
const hundredPrefix = h === 1 ? "centi" : joinParts([ONES[h], "centi"]);
```

So `n = 103` is `centi` + `trillion` = **centitrillion**.

**If h = 2-9** (n = 200-999): an ones-prefix (`ONES[h]`) is fused directly
onto `centi` to make a compound hundred word, and *that* is either used
standalone (if nothing's left over) or glued onto the remaining word:

So `n = 300` (h=3, nothing left over) uses `joinParts(["tre","centi"])` =
"trecenti", then eliding into "-illion" gives **trecentillion**, see
Section 5 for what "eliding" means here.

The full function:

```js
function saibianWord0to999(n){
  if(n <= 20) return nameForN(BigInt(n));
  const h = Math.floor(n/100), rem = n % 100;
  if(h === 0) return saibianWord0to99(n);
  const hundredPrefix = h === 1 ? "centi" : joinParts([ONES[h], "centi"]);
  if(rem === 0) return joinParts([hundredPrefix, "illion"]);
  return hundredPrefix + saibianWord0to99(rem);
}
```

**Why this avoids the collision:** 103 and 300 are built by *completely
different code paths*. 103 goes through the `h === 1` branch and attaches
the whole word "trillion." 300 goes through the `h >= 2` branch and
attaches bare "centi" directly to a fused ones prefix. There's no shared
step where they could accidentally produce the same string. This was
verified computationally by generating and comparing all 1,000 possible
names for n = 0 to 999: there are **zero** collisions.

### 4.5 The "combines as" root, shown in Drawer I

Drawer I shows a small subtitle on each card (million through nonillion)
reading "combines as: [root]". This is the units prefix: `un, duo, tre,
quattuor, quin, sex, septen, octo, novem`. That gets used when that digit
is folded into a *bigger* compound name, which is different from the
word's own standalone form. Septillion (7) stands alone under that name,
but when 7 is a units digit inside something bigger, the fragment used is
`septen` (as in septendecillion). This is surfaced because it directly
explains vocabulary that shows up elsewhere on the page.

---

## 5. Vowel elision

When two pieces are joined *directly into the bare suffix "-illion"* (not
into a whole other word), and the join would put two vowels back to back,
the first piece drops its trailing vowel. This is why "deci" + "illion"
becomes **decillion**, not "deciillion."

```js
function isVowel(ch){ return "aeiou".includes(ch.toLowerCase()); }
function joinParts(parts){
  const nonEmpty = parts.filter(p => p.length > 0);
  if(nonEmpty.length === 0) return "";
  let result = nonEmpty[0];
  for(let i=1;i<nonEmpty.length;i++){
    const next = nonEmpty[i];
    if(isVowel(result[result.length-1]) && isVowel(next[0])) result = result.slice(0,-1) + next;
    else result = result + next;
  }
  return result;
}
```

This elision rule is used in exactly two places: building the tens only
words (Section 4.2) and building the hundred only words (Section 4.4).
Everywhere else (attaching a tensroot to a whole ones word, attaching a
hundred prefix to a whole remainder word, and everything in the Phase 1
and outer tier extensions below) uses **plain concatenation with no
elision at all**. This was a deliberate simplification made partway
through development: since elision only ever changes cosmetic spelling
(it doesn't change *which* number a name refers to, because the reverse
parser strips spaces and dashes before matching anyway), skipping it
everywhere except the two places above kept the code simpler with zero
cost to correctness.

---

## 6. Layer 2 - Phase 1 (n = 1,000 to 10^33 − 1)

The base system runs out of dedicated vocabulary at n = 999. Phase 1
extends it using **ten dedicated "block" words**, each one a thousand
times bigger than the last:

```js
const P = ["", "myri","million","billion","trillion","quadrillion",
  "quintillion","sextillion","septillion","octillion","nonillion"];
```

`P[1]` through `P[10]` correspond to place values 1000^1 through 1000^10.
Since 1000^10 = 10^30, and each place can hold a coefficient from 0 to 999,
this scheme covers illion indices up to just under 1000^11 = **10^33**.

### 6.1 Decomposing n into base 1000 "digits"

Any `n` in this range is broken into eleven digits `c₀, c₁, ..., c₁₀`,
each 0-999, the same way you'd break a number into ones/thousands/millions
by grouping digits in threes:

```js
function phase1Name(N){ // N: BigInt, 0 <= N < 10^33
  const digits = [];
  let rem = N;
  for(let i=0; i<11; i++){ digits.push(Number(rem % 1000n)); rem = rem / 1000n; }
  const c0 = digits[0];
  ...
}
```

### 6.2 Rule 1 - zero digits are simply skipped

If `cᵢ = 0` for some `i ≥ 1`, nothing is emitted for that position at all,
not even a placeholder. This is safe *specifically* because the ten
words in the `P` array are all different from each other, so there's never
a gap that could be misread. (Compare this to the outer tier in Section 7,
which *does* need an explicit zero-marker, because it reuses the same word
at every level.)

### 6.3 Rule 2 - the coefficient attached to each block-word

For `i ≥ 1` with `cᵢ > 0`, a coefficient is built and glued directly in
front of `P[i]`. This coefficient uses a **compact, bare-root form**,
notably *different* from the readable, whole word style used for the
final group (Section 6.4). It's built from up to three bare Latin
fragments with no embedded "-illion" anywhere:

```js
function compactCoefficient(v){ // v: 1..999
  if(v === 1) return "";
  const h = Math.floor(v/100), t = Math.floor(v/10)%10, u = v%10;
  const hundredPart = h === 0 ? "" : (h === 1 ? "centi" : ONES[h] + "centi");
  const tensPart = t === 0 ? "" : SAIBIAN_TENS[t];
  const onesPart = u === 0 ? "" : ONES[u];
  return hundredPart + tensPart + onesPart;
}
```

The special case at the top, `v === 1` returns an empty string, means a
coefficient of exactly 1 is *implicit*: you don't say "one-million-illion,"
you just say "millionillion."

This compact style was discovered by checking Saibian's own worked
example for n = 1,234,567 letter by letter. He writes it as
"...duocentitrigintiquattuormilli...", using bare "quattuor," not the
whole word "quadrillion", which revealed that coefficients on a block
word use a totally different construction than the readable style used
everywhere else.

### 6.4 Rule 3 - the final group (c₀)

The lowest digit, `c₀`, always uses the **full additive style** from
Section 4, a complete, readable word ending in "-illion":

```js
if(c0 === 0) return prefix === "" ? null : prefix + "illion";
return prefix + saibianWord0to999(c0);
```

If `c0` is zero, a bare "-illion" is appended directly to whatever
block word prefix came before it.

### 6.5 Full worked example: n = 1,234,567

```
1,234,567 in base 1000: c₂=1, c₁=234, c₀=567

i=2 (c₂=1, attached to P[2]="million"):
  compactCoefficient(1) = "" (implicit)
  chunk: "million"

i=1 (c₁=234, attached to P[1]="myri"):
  h=2, t=3, u=4
  hundredPart = "duo"+"centi" = "duocenti"
  tensPart    = "triginti"
  onesPart    = "quattuor"
  compactCoefficient(234) = "duocentitrigintiquattuor"
  chunk: "duocentitrigintiquattuormyri"

c0 = 567 (full additive style):
  saibianWord0to999(567) = "quincenti" + "sexagintiseptillion"
                          = "quincentisexagintiseptillion"

Final: million + duocentitrigintiquattuormyri + quincentisexagintiseptillion
     = millionduocentitrigintiquattuormyriquincentisexagintiseptillion
```

(This matches Saibian's own worked example exactly, letter for letter,
except his version uses "milli" where this site uses "myri", see Section
8.1 for why that word was changed.)

---

## 7. Layer 3 - the outer tier (n ≥ 10^33)

Phase 1's ten block words are exhausted once `n` reaches 10^33. Past that
point, the site switches to a **dedicated recursive connector pair**:
`vasti` (for a nonzero group) and `nulla` (for a zero group), plus a fixed
terminal word `nullaillion` for when the very last group is zero.

### 7.1 The recursive structure

`n` is decomposed in base `PHASE1_LIMIT` (= 1000^11 = 10^33) exactly the way
Phase 1 decomposes numbers in base 1000:

```js
const PHASE1_LIMIT = 1000n ** 11n;

function outerChainForm(v){
  if(v === 1n) return VASTI;
  if(v < PHASE1_LIMIT) return phase1ChainForm(v) + VASTI;
  const Q = v / PHASE1_LIMIT, R = v % PHASE1_LIMIT;
  const pq = outerChainForm(Q);
  return R === 0n ? pq + NULLA : pq + outerChainForm(R);
}

function bigName(n){
  if(n < PHASE1_LIMIT) return phase1NameOrThousand(n);
  const q = n / PHASE1_LIMIT, r = n % PHASE1_LIMIT;
  const p = outerChainForm(q);
  return r === 0n ? p + NULLA_TERMINAL : p + phase1Name(r);
}
```

Here, `phase1ChainForm(v)` takes a full Phase 1 name and truncates its
"-illion" ending down to "-illi" (dropping the final "on"), so it can be
embedded as a prefix before the next connector, the same truncation
trick used throughout the naming system.

Because this uses `BigInt` throughout (JavaScript's arbitrary precision
integer type) rather than ordinary numbers, there's no meaningful ceiling
on how large a number this can name, ordinary numbers lose precision
past about 9 quadrillion, but `BigInt` has no such limit, so the only
real constraint is a deliberate safety cap (Section 11) to keep the
*output text* from becoming unreasonably long to render.

### 7.2 Why "vasti" and "nulla" specifically

Any two words could theoretically serve this role, but they can't be
picked casually, see Section 8.1 for a real bug that came from an
insufficiently careful choice. Before committing to "vasti" and "nulla,"
every candidate word was checked by computer against the entire Phase 1
vocabulary (all ten block words, every one of the 999 possible compact
coefficients, and every one of the 999 possible full base words) to
confirm it never appears as a literal substring anywhere in that
vocabulary. Only after that check passed clean were they used.

---

## 8. Two real bugs, found by testing, not by inspection

### 8.1 "milli" is a prefix of "million"

Both words come from the same Latin root ("mille," thousand), Saibian's
own extension used "milli" as the very first block-word, but "milli" is
literally the first five letters of "million." A left to right parser
would sometimes misread "millionmillion" (which should mean n = 1,000,001)
by matching the *first* five letters as if they were the connector
"milli," corrupting the rest of the parse. This was caught by running
22,367 generated names back through the parser and finding ~1,000
failures, all sharing this one root cause.

**The fix**: rather than patch around it, the word itself was swapped,
`P[1]` is `"myri"` instead of `"milli"`, after verifying computationally
that "myri" doesn't collide with *anything* else in the vocabulary. This
is the one deliberate departure from Saibian's actual word choice anywhere
in the base 1000 block system.

### 8.2 A trailing "million" is genuinely ambiguous

Even after the fix above, a *second* class of failures turned up: a
number like 103 × 10^30 + 1 generates a name ending in "...nonillionmillion",
and a naive left to right separator scan would misread that trailing
"million" as *another* block-word (implying c₀ = 0), rather than
recognizing it as the complete word for the final group (c₀ = 1, since
"million" is also simply the standalone word for the number 1). Both
readings are valid-looking parses of the same letters, but they mean
different numbers.

**The fix**: the parser was restructured into two stages. It first
identifies the final group (`c₀`) using the same *longest suffix match*
technique already used successfully elsewhere on the site (checking the
input against a table of all 999 possible base words and picking the
longest one that matches as a literal suffix), and only *then* parses
whatever text is left over as the block word chain. Extracting the known,
enumerable piece first removes the ambiguity entirely.

---

## 9. Reverse parsing: name (or abbreviation) → number

This is the more intricate direction, since it has to work backward
through everything above. The entry point is `parseNumberName(raw)`.

### 9.1 Deciding which layer a name belongs to

```js
function parseNumberName(raw){
  const input = raw.trim().toLowerCase().replace(/[\s-]/g, "");
  ...
  let n = null;
  if(!input.includes(VASTI) && !input.includes(NULLA)){
    n = parsePhase1(input);
  } else {
    // outer-tier parsing (Section 9.3)
  }
  if(n !== null) return {n};
  return parseAbbr(raw);   // fall back to trying it as an abbreviation
}
```

If the input doesn't contain "vasti" or "nulla" anywhere, it's assumed to
be a pure Phase 1 name (or the special case "thousand") and handled by
`parsePhase1`. If it does, it's routed to the outer tier parser. If
neither succeeds, the input is tried as an *abbreviation* instead (Section
10), so the same box accepts "vigintitrillion" or "TVg" interchangeably.

### 9.2 Parsing a Phase 1 name - `parsePhase1`

This directly mirrors the two stage fix from Section 8.2:

**Stage 1 - find c₀ first.** A precomputed map of all 999 possible base
words (plus the literal string "illion" standing in for c₀ = 0) is
checked against the *end* of the input, taking whichever entry is the
longest match:

```js
function findLongestSuffix(str, sortedKeys){
  for(const k of sortedKeys){ if(str.length >= k.length && str.endsWith(k)) return k; }
  return null;
}
```

(`sortedKeys` is sorted longest first, so the first match found is
guaranteed to be the longest one, this matters because, for example,
"nonillion" is also a valid ending of many longer words, so checking
short candidates first could grab a false partial match.)

**Stage 2 - parse whatever's left as the block-word chain.** Since
`myri`/`million`/`billion`/etc. no longer collide with each other or with
c₀'s vocabulary (Section 8.1 fixed that), a straightforward left to right
scan for each of the ten block-words, from `nonillion` down to `myri`, is
now safe:

```js
for(let i = 10; i >= 1; i--){
  const sep = P[i];
  const idx = remaining.indexOf(sep);
  if(idx === -1) continue;
  const prefix = remaining.slice(0, idx);
  const c = parseCompactCoefficient(prefix);
  ...
}
```

`parseCompactCoefficient` looks up the bare root prefix text in another
precomputed table (built by generating `compactCoefficient(v)` for every
v from 1 to 999) to recover the numeric coefficient.

### 9.3 Parsing an outer-tier name

The overall structure is: `[outer chain prefix] + [final group text]`,
where the final group is either the literal word `nullaillion` (meaning
the outermost remainder is zero) or a complete Phase 1 name.

```js
if(input.endsWith(NULLA_TERMINAL)){
  r = 0n;
  prefix = input.slice(0, input.length - NULLA_TERMINAL.length);
} else {
  const lastVasti = input.lastIndexOf(VASTI);
  const lastNulla = input.lastIndexOf(NULLA);
  const boundary = Math.max(/* whichever ends later */);
  prefix = input.slice(0, boundary);
  const rText = input.slice(boundary);
  r = parsePhase1(rText);
}
```

Since neither "vasti" nor "nulla" can ever appear *inside* a genuine
Phase 1 name (that was verified when the words were chosen), the last
occurrence of either one always marks the true boundary between the outer
chain and the final Phase 1 range group.

The remaining `prefix`, the outer chain itself, is parsed by
`parseOuterChain`, which scans left to right for every occurrence of
`vasti` or `nulla`, treating the text before each one as either an
implicit 1 (empty), a zero (for `nulla`), or a recursively parsed Phase 1
value (for `vasti`, after appending "on" back to reconstruct the
truncated "-illi" ending into "-illion"):

```js
value = value * PHASE1_LIMIT + digit;
```

...accumulating the result the same way you'd read a multi digit number
left to right.

---

## 10. Reverse parsing: abbreviations

Abbreviations are a **completely separate system**, independent of which
naming convention (Phase 1, outer tier) produced the full word. They're
built from three short letter tables:

```js
const ONES_ABBR = ["", "U","D","T","Qa","Qi","Sx","Sp","Oc","N"];
const TENS_ABBR = ["", "Dc","Vg","Tg","Qag","Qig","Sxg","Spg","Ocg","Ng"];
const HUNDREDS_ABBR = ["", "Ce","Dce","Tce","Qace","Qice","Sxce","Spce","Once","Nce"];
const LOW_ABBR = {0:"K",1:"M",2:"B",3:"T",4:"Qa",5:"Qi",6:"Sx",7:"Sp",8:"Oc",9:"No"};
```

These aren't tied to Saibian's system or any official standard, they're
a consistent shorthand invented for this site.

The recursive extension beyond n = 999 uses `Mi` for a group of 1000 and
`Ni` as the zero marker, the abbreviation-system's own version of the
old milli/nilli connectors, kept unchanged even after the full word naming
system moved on to Phase 1 and the outer tier, since abbreviations were
deliberately never tied to either naming convention.

### 10.1 A genuine, unavoidable collision

Two of these abbreviation tables happen to overlap: "Dce" is both the
abbreviation for 200 (`HUNDREDS_ABBR[2]`) *and* what you get by writing
"D" (=2) followed by "Ce" (=100), i.e., 102. This mirrors the 103/300
naming collision from Section 4.4, but here it genuinely can't be
avoided, the abbreviation scheme's structure just runs into it. It's
resolved by generic policy rather than special casing: whenever two
different values would produce the same abbreviation, the one with fewer
non-zero digit groups (i.e., the "rounder" number) wins:

```js
function nonzeroDigitCount(n){
  return (n%10n!==0n?1:0) + ((n/10n)%10n!==0n?1:0) + ((n/100n)%10n!==0n?1:0);
}
```

So "Dce" resolves to 200, not 102 - matching the same round-number
preference used for the "trecentillion" naming collision.
(which will probably be changed on a future version)

---

## 11. Number parsing: number → name direction

Typing a number accepts three formats, all funneled through `parseInput`:

- **A plain integer**, however long (e.g. `100000000000000000000000`) - the
  exponent is simply the digit count minus one.
- **Scientific notation** (`4.2e45`).
- **Power of ten shorthand** (`10^63`, or `2.5*10^63`).

All three are normalized into a "mantissa × 10^exponent" form (mantissa
between 1 and 10), using `BigInt` for the exponent so there's no precision
ceiling even for absurdly large inputs:

```js
function normalize(mantissa, exponent){
  if(!isFinite(mantissa) || mantissa === 0) return {mantissa: 0, exponent: 0n};
  while(mantissa >= 10){ mantissa /= 10; exponent += 1n; }
  while(mantissa < 1){ mantissa *= 10; exponent -= 1n; }
  return {mantissa, exponent};
}
```

From there, `describe()` rounds the exponent down to the nearest multiple
of 3 (since illion names only exist at multiples of a thousand), folds
the leftover remainder back into the mantissa, computes the illion index
`n = (exponent − 3) / 3`, and hands `n` to `bigName()`/`bigAbbr()`, the
exact same functions used everywhere else on the page.

A safety cap, `MAX_N_DIGITS = 1000`, rejects lookups where the resulting
illion index itself would need more than 1,000 digits. This isn't a
mathematical limitation (the naming system has no real ceiling), it's
there because a name for such an index could run to tens of thousands of
characters, which would be unpleasant to render. A 1,000-digit index still
produces a name of roughly 10,000 characters, which is already far beyond
anything meaningful to read, so the cap is generous by any practical measure.
(this cap will probably change on a future update)

---

## 12. Testing methodology

Every layer of this naming system was verified the same way, computationally
rather than by inspection, because inspection alone missed real bugs (see
Section 8):

- **Collision checks**: generate names for a large, structured sample of
  index values (every single coefficient placement, every pairing of two
  nonzero digits at every pair of positions, the specific 103/300 pattern
  embedded at every possible position and every recursion depth, plus
  thousands of random samples), and confirm no two different indices ever
  produce the same string.
- **Round-trip checks**: for every generated name, run it back through the
  reverse parser and confirm it recovers the exact original number.

The base 0-999 range was checked exhaustively (all 1,000 values). Phase 1
and the outer tier, which can't be checked exhaustively (there are 10^33
possible Phase 1 values alone), were checked with tens of thousands of
structured and random samples, including deliberately adversarial cases
built from the known ambiguity patterns.

---

## 13. The lookup tools (UI layer)

### 13.1 Number → name

`runLookup()` reads the input box, calls `describe()`, and populates the
result "plaque": the catalog number (`1eN`), the name, the abbreviation
badge, and a short note about which tier the number falls into (irregular,
additive, or the recursive extension).

### 13.2 Name → number

`runReverseLookup()` calls `parseNumberName()` and displays the reverse
information: the illion index (`n = ...`), the resulting `1eN` value, and
the abbreviation.

Both share the same result display (`#plaque`), and both apply
`overflow-wrap`/`word-break` CSS so that extremely long generated names,
which can run to thousands of characters at extreme scale, wrap onto
multiple lines instead of overflowing the page.

### 13.3 The filter bar

A single text input filters every visible card across all five drawers at
once, matching against each card's precomputed `dataset.search` string
(which bundles the name, abbreviation, and exponent together in
lowercase).

---

## 14. The five drawers

Each drawer is built by calling `makeCard(n, featured, showRoot)` for a
curated list of index values, and appending the resulting card to a grid
`<div>`.

| Drawer | Range | What it shows |
|---|---|---|
| I | n = 0-9 | The irregular base words, each with its "combines as" compositional root |
| II | n = 10, 20, ..., 90 | The main tens roots |
| III | n = 100, 200, ..., 900 | The hundreds tier, including the 300 vs. 103 disambiguation |
| IV | n = 1,000 up to 10^30 | Phase 1's ten block words (myri through nonillion) |
| V | n ≥ 10^33 | The outer tier (vasti/nulla), including deliberately re testing the 103/300 pattern one level up |

`makeCard` builds each card's HTML directly:

```js
function makeCard(n, featured, showRoot){
  const nBig = BigInt(n);
  ...
  const exp = exponentForNBig(nBig);
  const name = bigName(nBig);
  const abbr = bigAbbr(nBig);
  ...
}
```

Every card's exponent, name, and abbreviation are computed live by calling
the exact same functions the lookup tools use, nothing on the page is a
hardcoded string of pre written names.

---

## 15. Visual design

The page uses a dark "specimen cabinet" theme: deep ink-green background
(`--ink: #0e1512`), brass accents (`--brass: #c9a24b`) for interactive and
highlighted elements, and a muted teal (`--verdigris: #6fae9b`) for
secondary text like exponents. Three typefaces divide labor:

- **Fraunces** (serif), the big display name in the result plaque and
  each card's word, giving names a bit of typographic weight.
- **Inter** (sans-serif), body text, labels, buttons.
- **IBM Plex Mono** (monospace), anything numeric or code like: the
  exponent readouts, abbreviation badges, input fields.

The whole layout is a single `<main>` column capped at 900px wide, with
each drawer as its own `<section>` containing a responsive CSS grid
(`repeat(auto-fill, minmax(150px, 1fr))`) that reflows card count based on
viewport width automatically.

---

## 16. File structure

Everything lives in one file, `index.html`:

- **`<style>`** (head), the entire visual design described in Section 15.
- **HTML body**, the header, the two lookup input rows, the filter bar,
  and five empty grid containers waiting to be filled by JavaScript.
- **`<script>`** (end of body), every function described in this
  document, in roughly this order: letter tables and constants, the base
  0-999 naming system, abbreviations, Phase 1, the outer tier, both
  reverse parsers, number-input parsing, and finally the UI wiring
  (`runLookup`, `runReverseLookup`, `makeCard`, and the code that
  populates all five drawers on page load).

There's no build step, no bundler, no external JavaScript files, opening
`index.html` in any modern browser runs the whole thing as is.
---

## 17. Known limitations

- **Two unavoidable naming collisions exist**: 103/300 at the
  full name level (Section 4.4's fix makes these two *specific* numbers
  distinguishable, but the same underlying Latin coincidence could in
  principle recur elsewhere in the 900-999 hundreds range if a similar
  spelling accident existed, none was found in testing) and the
  Dce abbreviation collision (Section 10.1). Both are resolved by a
  documented, generic tie-breaking rule rather than papering over them.
- **The safety cap** (Section 11) is a practical rendering limit, not a
  mathematical one; the underlying system has no real ceiling given
  `BigInt` arithmetic.

  This is being worked on.
