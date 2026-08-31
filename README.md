# Samy's Number Names Generator :3

A single-page site that names extremely large numbers, in both directions.
Type a number in and get its name, type a name or its short abbreviation
in and get the number back.

Live at: `https://samyfiorini.github.io`

## The naming system

This site uses a hybrid system: the base range (thousand through the 
hundreds tier, up to 1e303) follows the additive construction Sbiis
Saibian laid out in his Extensible Illion System, and everything past that
uses a dedicated recursive connector so it can keep going indefinitely.
  
### Why additive, not prefix-mashed

The traditional approach (Conway-Wechsler) builds names by welding three
Latin prefixes together and appending "-illion" once at the end: "un" +
"viginti" + "illion" gives "unvigintillion". That works, but it has a real
flaw. Latin's own word for 300 happens to already start with the same
letters as "tre" (3) glued to "centi" (hundred), so both 103 and 300
collide on the exact same word: "trecentillion". That's not a typo or an
edge case I introduced, it's baked into the Latin roots themselves.

Saibian's fix is to build names the way English actually reads compound
numbers: additively, one recognizable word at a time. Twenty-one isn't
"twenty-and-one" mashed into a single prefix, it's "twenty" then "one" as
separate words. So in this system:

- `n = 21` is **vigintimillion** (viginti, twenty, plus the full word
  "million"), not a mashed prefix.
- `n = 103` is **centitrillion** (centi, hundred, plus the full word for
  3, "trillion").
- `n = 300` is **trecentillion** (the standalone root for "three
  hundred").

Those last two no longer collide, because they're built through
completely different paths (one glues "centi" onto a whole existing
word, the other uses a dedicated three-hundred root). I checked this
isn't a coincidence: none of the 1,000 names from n = 0 to 999 collide
with each other. The base range (n = 0-20: thousand through vigintillion)
is the same canonical dictionary words either system would use.

### The construction rules, briefly

- **n = 0-20:** irregular, canonical dictionary words (thousand, million,
  ..., vigintillion). No formula, just memorized.
- **n = 21-99, not a round ten:** tens-root (viginti, triginti, ...,
  nonaginti) plus the *whole word* for the ones digit, concatenated
  directly with no letters dropped. Example: vigintitrillion (23).
- **n = 30, 40, ..., 90:** a dedicated tens-only word, built the usual way
  (root plus "illion", with the one vowel-drop needed to avoid a double
  vowel). Example: trigintillion (30).
- **n = 100-999:** a hundreds-prefix (centi for 100, or an ones-root
  fused directly onto centi for 200-900, like "trecenti" for 300) glued
  onto the whole word for whatever's left over. Example: centidecillion
  (110), duocentitrigintiquadrillion (234).

## Past centillion: the recursive extension

The base system above only covers up to n = 999 (1e3000), since that's as
far as Saibian's own examples go. Beyond that, this site does **not** use
his original extension method. His approach reuses "million", "billion",
etc. as tier-multiplier words on top of their normal meaning as numbers,
and that double duty reintroduces exactly the kind of ambiguity the
additive system was built to avoid. His own patch for it (inserting "de",
then "dode", then "triade" for each deeper level) is a set of one-off
fixes rather than a real formula, so it can't extend to arbitrarily large
numbers.

Instead, this site chains the additive base names using two words that
are never used to mean an actual number: "milli" and "nilli". Neither one
ever collides with anything, because neither one is a real value in the
base vocabulary to begin with. A number is broken into groups of 1000
(same idea as commas in "1,000,000"), and each group beyond the last is
named using the additive system above, then chained together with
"milli"/"nilli". For example:

- **1e3003** (n = 1000) is millinillion
- **1e6003** (n = 2000) is billinillion
- **1e3,000,003** (n = 1,000,000) is millinillinillion

This was tested by round-tripping every name from n = 0 to 999, plus a
wide sample of much larger numbers (into the billions and beyond), both
generating names and parsing them back into numbers, with zero
collisions and zero failures.

One honest note: exact powers of 1000 (a million illions, a billion
illions, and so on) do degenerate into a repetitive "milli...nilli...
nillion" chain. Most numbers don't have this problem; a "messy" number
like 1,234,567 still produces a genuinely varied name using real roots at
every level, not a repeated word.

### Abbreviations

Each root also has a short abbreviation (Dc for decillion, Vg for
vigintillion, Qi for quintillion, Qig for quinquagintillion, and so on).
This is a separate, self-consistent shorthand of my own, unit letters
plus tens letters plus hundreds letters, and isn't tied to either naming
convention above. The recursive tier uses "Mi" for a group of 1000 and
"Ni" as the zero marker.

## The two lookup tools

### Number to name

Type a number in three ways:

- **A plain integer**, however long. The code counts its digits to work
  out the exponent.
- **Scientific notation**, like `4.2e45`.
- **Power of ten shorthand**, like `10^63` or `2.5*10^63`.

Whatever you type gets normalized into a "mantissa times 10 to an
exponent" form. The exponent is rounded down to the nearest multiple of 3,
the leftover remainder is folded back into the mantissa, and the
resulting illion index runs through the naming logic above.

### Name to number

Type a full name (like "vigintitrillion") or its abbreviation (like
"TVg") and get the number back. Since the additive system never drops or
elides letters at a chain boundary, every word's own letters stay fully
intact inside a bigger name, so the parser works by repeatedly finding the
longest recognized ending and stripping it off, working right to left
until nothing's left.

## File structure

Everything lives in one file, `index.html`:

- **`<style>`**: the visual design (the dark "specimen cabinet" theme,
  fonts, card layout).
- **HTML body**: the two lookup rows (number-to-name and name-to-number),
  the filter bar, and four empty `<div>` containers (`grid1` through
  `grid4`) that JavaScript fills in.
- **`<script>`**: the naming engine, both parsers, and the code that
  builds each drawer's cards when the page loads.

Nothing is hardcoded as a giant list of names. The drawers are generated
on the fly by looping over index values and calling the same naming
functions the lookup tools use, and the reverse-lookup dictionaries
(`CHAINFORM_MAP`, `DICTFORM_MAP`, `CHAINABBR_MAP`, `DICTABBR_MAP`) are
built once at page load by running those same forward functions across
the base 0-999 range.
