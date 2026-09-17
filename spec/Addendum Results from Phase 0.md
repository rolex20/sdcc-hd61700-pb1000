# Title: Addendum - Results from Phase 0

## Results from Phase 0, Prompt 5

- No undocumented surprises; all truth-tables from Prompt 1 remain correct.
- Initially the results showed no flag effects on Bits 0, 1, 2 and 3.  These prompted the following updates to the assembly guide:
- Those apparent unnafected flag bits were included in the documentation as not used for 2 hardware bits (bit 0 and 1), `APO` (bit 2) and `SW` (bit 3)

## Results from Phase 0, Prompt 6

The following section captures the results from Shift/Rotate Prompt 6 with an extension that mirrors what Prompt 5 did, but for ADW, SBW, ADCW, SBCW

Prompt 6 captures in a single CALL raw Flags and result bytes for: 

- 8-bit shift/rotate: ROU A, ROD A, BIU A, BID A

- 16-bit shift/rotate: ROUW A, RODW A, BIUW A, BIDW A

- 16-bit arithmetic: ADW A,B, SBW A,B, ADCW A,B, SBCW A,B

### Verified Flag-Side-Effects Not Covered Elsewhere

This section covers only those details that were **absent** from the provided documentation.


#### Nibble-Zero Flags with 16-bit ALU Instructions

| Instruction | What the manual said | Missing detail (now confirmed) |
|-------------|---------------------|--------------------------------|
| `ADW dst,src` `SBW dst,src` | “Sets Z, C, LZ, UZ.” | **LZ and UZ look only at the low byte** of the 16-bit result.  They ignore the high byte entirely. |

Practical impact: to test whether a 16-bit result is zero you must examine **Z**, not LZ/UZ.



#### `ADCW` / `SBCW` (Add-Check-Word / Sub-Check-Word)

* Documented purpose: “Performs arithmetic only to update the flag register; destination is unchanged.”

As previously documented in the section "5.3 Implementing Comparisons on the HD61700 using `SBC`/`SBCW`" of the Assembly Guide, the ADCW and ADC act as 'Add-Check' similar to `SBC`/`SBCW`.

These instructions, only set the flags and `SBC`/`SBCW` in particular are the recommended operations for Compariso

* These commands **do not read the incoming Carry flag**.  
* It sets **Z, C, LZ, UZ** exactly as `ADW`/`SBW` would have, based solely on `dst −/+ src`.  
* The destination register pair remains untouched, as documented.

Compiler note: multi-word addition/subtraction cannot rely on an “add-with-carry” chain; propagate carry/borrow manually.


#### Shift / Rotate Instructions – Flag Matrix

| Class | Z flag | C flag | LZ / UZ source | Notes |
|-------|--------|--------|----------------|-------|
| 8-bit `ROU A`, `ROD A`, `BIU A`, `BID A` | Full 8-bit result | Bit that falls off (MSB for left, LSB for right) | Same 8-bit result | – |
| 16-bit `ROUW`, `RODW`, `BIUW`, `BIDW` | Full 16-bit result | Lost bit (bit 15 or bit 0) | **Low byte only** | High byte never influences LZ/UZ |

(The manuals stated only that “flags are affected,” without the byte-selection rule.)


#### Carry Flag after 16-bit Arithmetic

Empirical confirmation:

* `ADW` sets **C = 1** when there is a carry out of bit 15.  
* `SBW` sets **C = 1** when a borrow occurs from bit 15.  

This matches the 8-bit rules and can be used directly for unsigned compares (`JR C` / `JR NC`).


### Final remarks

These clarifications close the final undocumented gaps revealed by Prompt 6 ant it's extension and are sufficient for correct code-generation at 16-bit and multi-word precision.

## Results from Phase 0, Prompt 7
- Pending

## Results from Phase 0, Prompt 8
- Pending

## Results from Phase 0, Prompt 9
- Pending