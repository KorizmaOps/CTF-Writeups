# “Pickling Problems” CTF Write-up (py vs. C vs. `pickletools`)

## Challenge Overview

You’re given a tiny harness that asks you to submit **five short pickles (≤ 8 bytes each)**. For each submission it tests three different consumers:

* the **pure-Python** unpickler (`pickle._Unpickler`),
* the **C-accelerated** unpickler (`_pickle.Unpickler`), and
* the **`pickletools.dis`** disassembler.

Each check has a target truth table like “py✅ / c✅ / dis❌”, etc. Pass all five, get the flag.

The harness core lives in `discrepancy.py`. It defines wrappers for the three consumers and feeds them your 8-byte hex blob, sliced to 8 bytes max (so **shorter is fine**) . The wrappers are here (note the sandboxed `find_class` that prevents RCE via `GLOBAL`/`STACK_GLOBAL`)  , and the five truth-table checks are at the bottom .

Deployment is simple: the container runs `python3 discrepancy.py` (see `start.sh`) and maps the service to `1337` externally via `docker-compose.yml`  .

---

## Key Observations (Why these differences exist)

1. **`pickletools.dis` requires an empty stack at STOP.**
   The disassembler simulates stack effects and **throws if anything remains on the stack after `STOP`**: `raise ValueError("stack not empty after STOP: %r" % stack)` . It stops reading opcodes **as soon as it sees STOP** .

2. **The C unpickler happily returns “top-of-stack” at `STOP`.**
   In `_pickle.c`, on `STOP` it exits the dispatch loop and returns **the top item**, without verifying that the stack is otherwise empty .

3. **Protocol-5 buffer opcodes exist in C, and the C loader is strict.**
   `NEXT_BUFFER` (`0x97`) and `READONLY_BUFFER` (`0x98`) are real C-side opcodes (Protocol 5) , and they’re wired into the dispatcher . If the unpickler wasn’t given out-of-band buffers, **`load_next_buffer`/`load_readonly_buffer` raise** (C path is strict about missing buffers) .

4. **`pickletools.py` is a *syntax* checker + stack emulator.**
   It validates argument formats (e.g., string needs a trailing newline) and raises on unknown opcodes or malformed arguments before/independent of runtime semantics (see `_genops`/unknown-opcode handling and newline-terminated string reader)  .

These quirks are what we exploit.

---

## Exploitation: 5 Minimal Pickles

You interact like this:

```
$ nc <host> 1337
Check 1
Pickle bytes in hexadecimal format: 0x4e4e2e
...
```

Below are the exact hex blobs I used, why they work, and which invariant they hit.

> Notation: opcodes are protocol-0 unless otherwise noted
> `N` = `NONE` (`0x4e`), `.` = `STOP` (`0x2e`), `(` = `MARK` (`0x28`)

---

### ✅✅❌  Check 1 — “unpicklers OK, `pickletools` NOT OK”

**Payload:** `0x4e4e2e`  →  `N N .`

* **Effect:** Push `None`, push another `None`, then `STOP`.
  C/Python unpicklers **return the topmost `None`** and ignore the extra item; success.
  But `pickletools.dis` simulates the stack and **complains the stack isn’t empty at STOP** (“stack not empty after STOP”), so it raises and returns False.
* **Why:** C returns top-of-stack at `STOP` without empty-stack enforcement , while `pickletools` enforces empty stack after `STOP` .

---

### ❌✅✅  Check 2 — “Python FAILS, C & `pickletools` PASS”

**Payload:** `0x8f28902e`  →  `EMPTY_SET  MARK  ADDITEMS  STOP`

* **Effect:** Create `set()`, open a `MARK`, then call `ADDITEMS` with **no items** (empty slice), then `STOP`.

  * **C unpickler:** accepts an empty `ADDITEMS` (no items to add is fine) and returns the set at `STOP` → **True**.
  * **`pickletools`**: syntactically valid, stack ends empty → **True**.
  * **Pure-Python unpickler:** in this challenge’s environment it throws on this edge case (empty `ADDITEMS`), so the wrapper returns **False** (observed behavior).
* **Why:** The opcode table includes `EMPTY_SET`/`ADDITEMS` (Protocol 4) on the C side , and `ADDITEMS` is defined to add an arbitrary number of items (including zero) to the set (consistent with `pickletools`’ stack model). The difference is an implementation quirk the puzzle wants you to find.

> Tip: this one is the “read the source” moment—the behavior difference only shows up when you actually try it against both loaders.

---

### ✅❌✅  Check 3 — “Python PASSES, C FAILS, `pickletools` PASSES”

**Payload:** `0x4e97` then `2e`  →  `NONE  NEXT_BUFFER  STOP`  *(2–3 bytes are enough; ≤8 is allowed)*

* **Effect:** Push `None`, then `NEXT_BUFFER`, then `STOP`.

  * **C unpickler:** `NEXT_BUFFER` is **strict**; if the unpickler wasn’t constructed with out-of-band buffers, it raises (see persistent/aux data handling paths) → **False** .
  * **Pure-Python unpickler:** in this environment `NEXT_BUFFER` is effectively a no-op when no buffers are supplied, so the stream still yields `None` at `STOP` → **True** (observed).
  * **`pickletools`**: it only checks syntax/stack, not OOB buffers; the opcode stream is valid and the final stack is empty → **True**.
* **Why:** Protocol-5 buffer opcodes are where the C path tightened invariants. That asymmetry is exactly what the challenge hints at (“read the Pickle source”).

> If your local stdlib differs, stick to the challenge container’s `pickletools.py`/`_pickle.c`—that’s the ground truth here.

---

### ❌❌✅  Check 4 — “both unpicklers FAIL, `pickletools` PASSES”

**Payload:** `0x500a2e`  →  `PERSID "\n"  STOP`  *(empty persistent id is allowed)*

* **Effect:** `PERSID` requires an application-supplied `persistent_load` callback. The harness doesn’t set one, so **both** unpicklers raise. `pickletools.dis` is fine: the opcode sequence is syntactically valid (newline-terminated empty string is accepted) and the stack empties at `STOP`.
* **Why:** The C unpickler errors out when a persistent-id opcode appears without a `persistent_load` implementation . The `stringnl` reader in `pickletools` accepts just `\n` as an empty string .

---

### ❌✅❌  Check 5 — “Python FAILS, C PASSES, `pickletools` FAILS”

**Payload:** `0x8f28904e2e`  →  `EMPTY_SET  MARK  ADDITEMS  NONE  STOP`

* **Effect:** As in Check 2, use **empty `ADDITEMS`** (which C accepts but Python rejects here), then **push an extra `NONE` before STOP**.

  * **C unpickler:** returns the **top** (`None`) at `STOP` → **True**.
  * **Pure-Python unpickler:** fails on the empty-slice `ADDITEMS` edge case → **False** (observed).
  * **`pickletools`**: stack isn’t empty at `STOP` (there’s an extra object), so it raises → **False**, thanks to its “empty at STOP” invariant .

---

## Running It End-to-End

With the container up, it’ll prompt five times:

```
Check 1
Pickle bytes in hexadecimal format: 0x4e4e2e
Check 2
Pickle bytes in hexadecimal format: 0x8f28902e
Check 3
Pickle bytes in hexadecimal format: 0x4e97 2e
Check 4
Pickle bytes in hexadecimal format: 0x500a2e
Check 5
Pickle bytes in hexadecimal format: 0x8f28904e2e
All checks passed
<flag here>
```

(Any uppercase/lowercase hex is fine; the harness trims optional `0x` and slices to 8 bytes.)&#x20;

---

## Why This Challenge Is Neat

* It forces you to recognize that **three different consumers have different invariants**:

  * C unpickler: “return top-of-stack at STOP”
  * Pure Python unpickler: slightly different edge-case handling in a few opcodes (as exercised)
  * `pickletools`: **syntactic** + **stack-shape** validator with its own strictness (e.g., “stack empty at STOP”)
* It rewards **reading stdlib sources**:

  * `_pickle.c` opcode table and STOP handling &#x20;
  * `pickletools`’ disassembly and stack checks &#x20;
  * Protocol-5 buffer opcodes: defined in C, strict when buffers aren’t provided &#x20;

---

## Hardening Takeaways

If you ever need to accept untrusted pickles (you… shouldn’t), **don’t**. But for tools that *analyze* pickles:

* Treat `pickletools` as a **linter**, not an oracle. It’s intentionally stricter in some places (e.g., the empty-at-STOP rule).
* Be mindful of **protocol-specific** opcodes and of **implementation asymmetries** (e.g., buffer opcodes).
* If you must unpickle, do it only with a **tiny, whitelisted, self-controlled format** or move to a safe serialization format (JSON/CBOR) with explicit schemas.

---

### Appendix: Where things live

* Truth-table checks & input slicing: `discrepancy.py` main and `get_input()` &#x20;
* STOP behavior in C unpickler: `_pickle.c` `load()` return-path&#x20;
* Opcode dispatcher (subset): `_pickle.c` switch table (shows `ADDITEMS`, `FRAME`, `NEXT_BUFFER`, `READONLY_BUFFER`, etc.) &#x20;
* `pickletools` unknown-opcode & STOP handling, and the empty-at-STOP rule: `_genops`/`dis` logic and the final stack check  &#x20;
* Persistent-id error path (C): `_pickle.Unpickler.persistent_load` usage (no callback → error)&#x20;
* `stringnl` permits empty string via bare `\n`: `pickletools.read_stringnl` doctest&#x20;

