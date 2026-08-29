# Samy's Number Names Generator :3

A single-page site that names extremely large numbers. Type one in and get its name,
or browse the reference drawers to see how the naming system climbs from a thousand
all the way past centillion.

Live at: `https://SamyFiorini.github.io`

## How the naming system works

Every "-illion" name beyond million corresponds to a position `n` in a numbered
sequence, related to its exponent by:

```
exponent = 3n + 3
```

So million is `n = 1` (1e6), billion is `n = 2` (1e9), decillion is `n = 10`
(1e33), and so on. The names fall into two groups:

- **n = 0–9 (thousand through nonillion):** irregular. Each is its own
  historical word, there's no formula, they're just memorized, like
  irregular verbs.
- **n = 10 and up (decillion onward):** compositional. The name is built by
  putting together some Latin root pieces plus the suffix `"illion"`:
  - a **units** root (un, duo, tre, quattuor, quin, sex, septen, octo, novem)
  - a **tens** root (deci, viginti, triginta, quadraginta, quinquaginta,
    sexaginta, septuaginta, octoginta, nonaginta)
  - a **hundreds** root (centi, ducenti, trecenti, quadringenti, quingenti,
    sescenti, septingenti, octingenti, nongenti)
    
  and so on.

  For example,
  `n = 17` → units "septen" + tens "deci" → **septendecillion** (1e54)
  `n = 101` → units "un" + hundreds "centi" → **uncentillion** (1e306)

This is the standard "Conway–Wechsler style" system used by dictionaries and
mathematicians for naming large numbers, not something invented for this site.

### The vowel-elision rule

When you put together two of those pieces together and the join would put two vowels
back to back, the first piece drops its trailing vowel. That's why
"deci" + "illion" becomes **decillion** (not "deciillion"), and
"viginti" + "illion" becomes **vigintillion**. The code does this with a
small `joinParts()` function that checks the last letter of one piece and the
first letter of the next.

### Abbreviations

Each root also has a short abbreviation (Dc for decillion, Vg for
vigintillion, etc.), combined the same way as the full names. This
abbreviation scheme is a common convention, the same one used in a lot of
big-number calculators and games.

## How the lookup tool works

You can type a number in three ways:

- **A plain integer**, however long — the code counts its digits to figure
  out the exponent.
- **Scientific notation**, like `4.2e45`.
- **Power of ten**, like `10^33` or `2.5*10^43`.

Whatever you type gets normalized into a "mantissa × 10^exponent" form (a
number between 1 and 10, times a power of ten). The exponent is then rounded
down to the nearest multiple of 3 (since illion names only exist at multiples
of a thousand), the leftover remainder gets folded back into the mantissa,
and the resulting `n` gets run through the same naming logic as the drawers.

## File structure

Everything lives in one file, `index.html`:

- **`<style>`** all the visual design (the dark "specimen cabinet" theme,
  fonts, card layout).
- **HTML body** the lookup box, the filter bar, and three empty `<div>`
  containers (`grid1`, `grid2`, `grid3`) that JavaScript fills in.
- **`<script>`** the naming logic, the input parser, and the code that
  builds each drawer's cards when the page loads.

Nothing is hardcoded as a giant list of names, the drawers are generated on
the fly by looping over `n` values and calling the same `nameForN()` and
`abbrForN()` functions the lookup tool uses.
