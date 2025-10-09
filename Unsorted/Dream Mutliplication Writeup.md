# “idog.py” CTF Write-up (Dream Multiplication & AES)

## Challenge Overview

This challenge provided a short Python script `idog.py` along with a meme image. At first glance the image suggested a joke multiplication problem, but the code revealed a deeper puzzle involving a custom multiplication function and AES decryption.

When executed, the script:

* Prompts for an **8-digit number** `x` and a **7-digit number** `y`.
* Enforces two constraints:

  1. `x * y` must not equal `381404224402842` (the “meme” product shown in the image).
  2. A custom check: `dream_multiply(x, y) == x * y`.
* Uses `(x, y)` to derive an AES key via
  `sha256(str((x, y)).encode()).digest()`.
* Decrypts a fixed ciphertext under AES-ECB and prints the flag.

---

## Key Observations (Understanding the Mechanism)

1. **`dream_multiply` logic**
   The function concatenates digit-wise products instead of doing true multiplication:

   ```
   digits = x[0] | (x[1]*y[0]) | (x[2]*y[1]) | ... | (x[7]*y[6])
   ```

   This string, when interpreted as an integer, must equal the real product `x * y`.

2. **Digit constraints from product length**
   An 8-digit number multiplied by a 7-digit number yields a 14–15 digit product.
   For the concatenated string to match, nearly all digit-pair products must be **two digits** (≥10).
   This prunes the search space significantly:

   * If `x[i] = 2`, then `y[i-1] ≥ 5`.
   * If `x[i] = 3`, then `y[i-1] ≥ 4`.
   * For `x[i] ≥ 5`, even `y[i-1] = 2` is enough.

3. **Exclusion of the obvious pair**
   The assertion banning `381404224402842` ensured we couldn’t just plug in the meme numbers.

4. **AES-ECB with SHA-256 key**
   Once a valid `(x, y)` is found, the AES key is deterministic:

   ```
   key = sha256(b"(x, y)").digest()
   ```

   Only the correct `(x, y)` yields a meaningful plaintext flag.

---

## Exploitation (Finding `(x, y)`)

### Step 1 — Constraints

We required:

* `x` has 8 digits, `y` has 7 digits.
* `dream_multiply(x, y) == x * y`.
* Each digit pair `x[i] * y[i-1]` must yield two digits to keep the concatenation length consistent.

### Step 2 — Reasoning and Brute Force

While it was possible to reason about digit-by-digit possibilities, the more reliable path was to **brute force with pruning**:

* For each possible digit in `x[1..7]`, restrict `y`’s digits according to the ≥10 rule above.
* Construct candidate `x` and `y` pairs.
* Verify the equality `dream_multiply(x,y) == x*y`.

> Community hints (such as Reddit threads) also pointed toward brute force being a viable and intended approach.

### Step 3 — Solution Pair

The correct pair was discovered:

```
x = 49228443
y = 9773647
```

Verification:

* `dream_multiply(49228443, 9773647) = 481141424241621`
* `49228443 * 9773647 = 481141424241621`

They match exactly, satisfying the condition.

### Step 4 — Decryption

With `(x, y)` fixed, compute:

```
key = sha256(b"(49228443, 9773647)").digest()
```

Decrypting the provided ciphertext under AES-ECB with this key yields:

```
Here is your flag: SEKAI{iSOgenni_in_mY_D1234M5;_iS_it_T00_s00n}
```

---

## Running It End-to-End

```bash
$ python3 -O idog.py
Enter an 8-digit multiplicand: 49228443
Enter a 7-digit multiplier: 9773647
Here is your flag: SEKAI{iSOgenni_in_mY_D1234M5;_iS_it_T00_s00n}
```

---

## Final Flag

```
sekai{iSOgenni_in_mY_D1234M5;_iS_it_T00_s00n}
```

---

## Why This Challenge Is Neat

* It disguised a real math + crypto puzzle behind a meme.
* It forced you to either reason digit-by-digit or brute force cleverly, both valid approaches.
* It tied the math to a cryptographic primitive (AES-ECB with SHA-256 keying), providing a clean final check.
* Online hints nudged solvers toward brute force, but understanding the digit constraints provided insight into why the brute force works.

