---
title: "K17 CTF non-AI division"
date: 2026-09-24 10:30:00 +0700
categories: [CTF, Cryptography]
tags: [ctf, cryptography, blowfish, cbc, xor, gf2, gaussian-elimination]
description: "A CTF writeup on abusing per-block signatures and CBC encryption/decryption oracles to forge an admin JSON message."
math: true
mermaid: true
toc: true
---

## Intro

I'm currently polishing my blog, and what can be better than a writeup post? :DD

It has been long since I last attended a CTF without any AI to slop and I had fun trying to solve this problem. I did encounter a rabbit hole as im attempting to solve it with bit-flipping lmao.

---

## Reading the Challenge

The challenge gives us a very large "fish", together with a signature:

```python
print(f"Fish: {iv_fish.hex()}")
print(f"Signature: {sign(iv_fish)}")
```

Then we get two operations:

1. submit a **signed plaintext** and ask the server to encrypt it;
2. submit a **signed ciphertext** and ask the server to decrypt it.

The flag is returned if our submitted plaintext passes the signature check and parses as JSON with

```json
{"admin": true}
```

after the first 8 bytes:

```python
def is_admin(plaintext_hex):
    try:
        return json.loads(bytes.fromhex(plaintext_hex)[8:]).get("admin") is True
    except:
        return False
```

So the goal is pretty clear:

> construct a validly signed message whose bytes after the first 8 bytes are `{"admin":true}`.

The first 8 bytes are not parsed as JSON because the challenge treats them as the IV.

At first this looks annoying: we do not know either the Blowfish key or the signing key. Fortunately, we do not need to recover any of them.

---

## The First Bug: The Signature Is Per Block

The signing function is:

```python
def sign(raw_fish):
    return "".join(
        sha256(SECRET_SIGNING_KEY + raw_fish[i:i+8]).hexdigest()
        for i in range(0, len(raw_fish), 8)
    )
```

This is the first important observation.

If a message is split into 8-byte blocks

$$
M = B_0 \Vert B_1 \Vert \cdots \Vert B_n,
$$

then the challenge does **not** compute one MAC over the whole message. It computes

$$
T_i = H(K_s \Vert B_i)
$$

for every block independently, and the final "signature" is simply

$$
T_0 \Vert T_1 \Vert \cdots \Vert T_n.
$$

That means a signature belongs to a **block**, not really to a message.

Once I know a pair

```text
(block, signature_of_that_block)
```

I can reuse that pair somewhere else. There is no block index, no chaining, and no message-level authentication tying all blocks together.

So I started treating everything the server gave me as a collection of signed chunks:

```python
def split_items(raw, sighex):
    n = (len(raw) + 7) // 8
    return [
        (raw[i*8:(i+1)*8], sighex[i*64:(i+1)*64])
        for i in range(n)
    ]
```

Blowfish has an 8-byte block size, which is also exactly the chunk size used by the challenge.[^blowfish]

The problem is that the initial fish obviously does not contain the exact blocks I want:

```python
target1 = b'{"admin"'       # 8 bytes
target2 = b':true}\x00\x00' # 8 bytes for now
```

So I still need a way to manufacture **new signed blocks**.

This is where CBC becomes useful.

---

## CBC Refresher

I had to write the CBC equations down before continuing. NIST SP 800-38A Section 6.2 is the clean reference if you also forget the indexing every single time.[^nist-cbc]

For CBC encryption,

$$
C_1 = E_K(P_1 \oplus IV),
$$

and then

$$
C_i = E_K(P_i \oplus C_{i-1}).
$$

For decryption,

$$
P_1 = D_K(C_1) \oplus IV,
$$

and

$$
P_i = D_K(C_i) \oplus C_{i-1}.
$$

The challenge uses the first 8 bytes of our input directly as the IV:

```python
def encrypt(plaintext):
    iv = plaintext[:8]
    return iv + Blowfish.new(
        SECRET_KEY,
        Blowfish.MODE_CBC,
        iv=iv
    ).encrypt(pad(plaintext[8:]))
```

and similarly for decryption:

```python
def decrypt(ciphertext):
    iv = ciphertext[:8]
    return iv + unpad(
        Blowfish.new(
            SECRET_KEY,
            Blowfish.MODE_CBC,
            iv=iv
        ).decrypt(ciphertext[8:])
    )
```

The IV being attacker-controlled is already suspicious, but the much more useful fact here is that the server gives us **both** an encryption oracle and a decryption oracle, as long as every input block has a valid signature.

And every output is signed again.

That lets us turn CBC into a small algebra machine.

---

## Building an XOR-3 Oracle

Assume I already own three valid signed 8-byte blocks

$$
A,\quad B,\quad C,
$$

and one signed filler block

$$
F.
$$

First, I ask the server to encrypt

```text
A || B || F
```

where `A` becomes the IV.

The returned blocks are

$$
A,\quad D,\quad E,
$$

with

$$
D = E_K(B \oplus A)
$$

and

$$
E = E_K(F \oplus D).
$$

The server also gives me valid signatures for `D` and `E`.

Now I submit

```text
C || D || E
```

to the decryption oracle, using `C` as the new IV.

The first decrypted plaintext block becomes

$$
\begin{aligned}
P_1
&= D_K(D) \oplus C\\
&= (B \oplus A) \oplus C\\
&= A \oplus B \oplus C.
\end{aligned}
$$

The next block is

$$
\begin{aligned}
P_2
&= D_K(E) \oplus D\\
&= (F \oplus D) \oplus D\\
&= F.
\end{aligned}
$$

So the whole encrypt-then-decrypt dance gives me a **new signed block** equal to

$$
\boxed{A \oplus B \oplus C}.
$$

This is the core exploit.

The corresponding function in my solve script is:

```python
def xor3(a, b, c, strip=False):
    enc = encrypt([a, b, filler])
    d, e = enc[1], enc[2]
    items.extend([d, e])

    dec = decrypt([c, d]) if strip else decrypt([c, d, e])
    got = dec[1]

    target = bytes(x ^ y ^ z for x, y, z in zip(a[0], b[0], c[0]))
    expected = target.rstrip(b"\x00") if strip else target
    assert got[0] == expected

    items.append(got)
    return got
```

I choose `filler` to be a full signed block whose last byte is non-zero:

```python
filler = next(it for it in items if it[0][-1] != 0)
```

because the challenge's "unpadding" is literally

```python
def unpad(data):
    return data.rstrip(b'\x00')
```

and I do not want it randomly eating bytes from the last filler block during normal `xor3()` calls.

At this point I effectively have the following primitive:

```text
signed A + signed B + signed C
              |
              v
       signed (A XOR B XOR C)
```

No Blowfish key recovery required.

---

## What Exactly Do We Need to Forge?

The admin check parses everything after the first 8 bytes:

```python
json.loads(bytes.fromhex(plaintext_hex)[8:])
```

So I want the message to be

```text
[8-byte IV] || {"admin":true}
```

Blowfish works with 8-byte blocks, therefore I split the JSON as

```python
target1 = b'{"admin"'       # exactly 8 bytes
target2 = b':true}\x00\x00' # padded to 8 bytes
```

The first target is easy from a formatting perspective.

The second one has a problem: the real JSON ends at

```python
b':true}'
```

which is only 6 bytes. If I submit the two zero bytes too, `json.loads()` sees garbage after the JSON document and fails.

I will deal with those two bytes later. First I only need to solve the harder problem:

> given many signed 8-byte blocks and an XOR-3 oracle, how can I build a specific 8-byte target?

---

## XOR Is Linear Algebra over $GF(2)$

An 8-byte block is a 64-bit value. A 64-bit value can be viewed as a vector in

$$
GF(2)^{64}.
$$

Over $GF(2)$, vector addition is XOR.

So if I have blocks

$$
B_1,B_2,\ldots,B_m,
$$

finding a subset whose XOR equals a target $T$ is just solving

$$
x_1B_1 \oplus x_2B_2 \oplus \cdots \oplus x_mB_m = T,
$$

where

$$
x_i \in \{0,1\}.
$$

This is a Gaussian-elimination / linear-basis problem over $GF(2)$.

I was a bit rusty with XOR-basis stuff since AI makes it way too easy to skip reimplementing these things, so to write it again without AI I went back to this old Stack Overflow question: **"How to find which subset of bitfields xor to another bitfield?"**[^stackoverflow-xor]

The nice thing about integer bit-vectors is that Gaussian elimination becomes very small: pick a pivot bit, XOR basis vectors to remove that pivot, repeat.

However, there is one extra constraint in this challenge.

---

## Why the Subset Must Have Odd Size

My primitive combines exactly three blocks:

$$
A,B,C
\quad\longrightarrow\quad
A\oplus B\oplus C.
$$

Every call replaces 3 current blocks by 1 block.

Therefore the number of current blocks changes by

$$
3\rightarrow1,
$$

i.e.

$$
n\rightarrow n-2.
$$

Parity never changes.

So if I start with an even number of blocks, repeatedly applying `xor3()` can never leave exactly one block.

To reduce everything to one final signed target, the subset must contain an **odd** number of source blocks.

This is where the slightly weird 65th bit in the solver comes from.

For every 64-bit block $B_i$, instead of inserting only

$$
B_i,
$$

I insert the augmented vector

$$
(B_i,1) \in GF(2)^{65}.
$$

The target is also augmented as

$$
(T,1).
$$

Now, if Gaussian elimination finds

$$
\bigoplus_{i\in S}(B_i,1)=(T,1),
$$

then the lower 64 bits give

$$
\bigoplus_{i\in S}B_i=T,
$$

while the final bit gives

$$
\bigoplus_{i\in S}1=1.
$$

But XORing one bit `1` once per chosen vector is just the parity of the number of selected vectors. Therefore

$$
|S|\equiv1\pmod2.
$$

Exactly what I need.

In code:

```python
x = block_to_int(it[0]) | (1 << 64)
```

and for the target:

```python
x = block_to_int(target8) | (1 << 64)
```

That one extra bit turns "find a subset XORing to the target" into

> find an **odd-sized** subset XORing to the target.

I like this trick a lot more than trying to patch parity after solving.

---

## Recovering the Actual Subset

A normal XOR basis only needs to remember the basis vector itself. Here I also need to know **which original blocks produced it**, because those are the signed chunks I must feed into `xor3()` later.

So each basis entry stores

```python
(vector, mask)
```

where `mask` is a bitmask over the original item indices.

When two vectors are XORed during elimination, their masks are XORed too:

```python
x ^= self.basis[p][0]
mask ^= self.basis[p][1]
```

At the end, if the target reduces to zero, the mask directly tells me which signed blocks form the target:

```python
idxs = [
    i for i in range(len(items))
    if (mask >> i) & 1
]
```

The solver also verifies both conditions:

```python
if acc != block_to_int(target8) or len(idxs) % 2 != 1:
    raise AssertionError("linear algebra bug")
```

So now I can take an odd-sized list

```text
B1, B2, B3, ..., B_(2k+1)
```

and repeatedly reduce triples:

```python
while len(cur) > 1:
    a = cur.pop()
    b = cur.pop()
    c = cur.pop()
    cur.append(xor3(a, b, c))
```

until only one signed block remains.

That block is the target.

---

## What If the Current Blocks Do Not Span the Target?

The challenge initially gives us a huge signed fish, so we already have a lot of valid blocks.

Still, there is no reason the current set must span every target I want.

Luckily the encryption oracle can also generate new signed ciphertext blocks.

If I encrypt known signed blocks, the server returns ciphertext blocks and signs them for me. Those ciphertext values give the basis more independent-looking 64-bit vectors.

My final solver just bootstraps more of them whenever the target is not in the current span:

```python
def bootstrap(rounds=40):
    full_items = [it for it in items if len(it[0]) == block_size]

    for _ in range(rounds):
        a, b, _ = random.sample(full_items, 3)
        enc = encrypt([a, b, filler])

        items.extend([enc[1], enc[2]])
        full_items.extend([enc[1], enc[2]])
```

Then I retry the basis solve.

Again, there is no cryptanalysis of Blowfish here. I am abusing the fact that the challenge willingly gives me signatures on oracle outputs.

---

## The Last Annoyance: Signing a 6-Byte Block

After the linear solve I can obtain a valid signature for

```python
b':true}\x00\x00'
```

but the actual JSON must end at

```python
b':true}'
```

The challenge's broken zero-padding comes back to help.

Suppose `full` is the signed block

```python
b':true}\x00\x00'
```

Then I call

```python
xor3(full, filler, filler, strip=True)
```

Algebraically,

$$
full \oplus filler \oplus filler = full.
$$

So the XOR result does not change.

But with `strip=True`, my helper only sends two ciphertext blocks into the decryption oracle:

```python
dec = decrypt([c, d])
```

Therefore the XOR result is now the **last plaintext block** returned by `decrypt()`.

And the server executes

```python
unpad(data) = data.rstrip(b'\x00')
```

before signing that plaintext.

So

```text
:true}\x00\x00
```

becomes

```text
:true}
```

and the server gives me a valid signature for that final **6-byte chunk**.

This is why `sign()` accepting a partial final block also matters: it signs `raw_fish[i:i+8]` even when the last slice is shorter than 8 bytes.

Now I have everything.

---

## Final Forgery

Let

```python
admin_key   = sign_target(b'{"admin"')
admin_value = sign_target(b':true}\x00\x00', strip=True)
```

and keep the original first signed block as the IV:

```python
iv = items[0]
```

Then construct

```python
raw = iv[0] + admin_key[0] + admin_value[0]
sig = iv[1] + admin_key[1] + admin_value[1]
```

The bytes after the IV are now exactly

```json
{"admin":true}
```

and every chunk has a valid signature.

When I choose option 1, the server first checks

```python
verify_signature(plaintext_hex, signature_hex)
```

which succeeds, and then checks

```python
is_admin(plaintext_hex)
```

**before** doing any encryption.

So we never even need the final forged message to be meaningful CBC plaintext. It only needs to be correctly signed and parse as admin JSON.

The whole exploit path is:

```mermaid
flowchart TD
    A[Initial fish + per-block signatures]
    B[Collect signed 8-byte chunks]
    C[Encryption oracle]
    D[Get new signed ciphertext chunks]
    E[GF(2) odd-parity XOR basis]
    F[Find odd subset XORing to target]
    G[xor3 CBC gadget]
    H[Signed target block]
    I[Strip trailing zeros for :true}]
    J[Forge IV || {"admin":true}]
    K[Signature passes]
    L[is_admin == True]
    M[Flag]

    A --> B
    B --> E
    B --> C --> D --> E
    E --> F --> G --> H
    H --> I
    H --> J
    I --> J
    J --> K --> L --> M
```

---

## The Core Solver

Below is the important part of my final solve. I removed some challenge boilerplate and kept the pieces responsible for the exploit.

```python
import random

block_size = 8
items = []
filler = None


def block_to_int(b):
    return int.from_bytes(b, "big")


def xor3(a, b, c, strip=False):
    enc = encrypt([a, b, filler])
    d, e = enc[1], enc[2]
    items.extend([d, e])

    dec = decrypt([c, d]) if strip else decrypt([c, d, e])
    got = dec[1]

    target = bytes(x ^ y ^ z for x, y, z in zip(a[0], b[0], c[0]))
    expected = target.rstrip(b"\x00") if strip else target
    assert got[0] == expected

    items.append(got)
    return got


class OddXorBasis:
    def __init__(self):
        self.basis = {}
        self.used_upto = 0
        self.rank = 0

    def _insert(self, idx, it):
        if len(it[0]) != block_size:
            return False

        # 64 data bits + one parity bit.
        x = block_to_int(it[0]) | (1 << 64)
        mask = 1 << idx

        while x:
            p = x.bit_length() - 1

            if p not in self.basis:
                self.basis[p] = (x, mask)
                self.rank += 1
                return True

            x ^= self.basis[p][0]
            mask ^= self.basis[p][1]

        return False

    def extend(self, items):
        for idx in range(self.used_upto, len(items)):
            self._insert(idx, items[idx])
        self.used_upto = len(items)

    def solve(self, items, target8):
        self.extend(items)

        x = block_to_int(target8) | (1 << 64)
        mask = 0

        while x:
            p = x.bit_length() - 1

            if p not in self.basis:
                return None

            x ^= self.basis[p][0]
            mask ^= self.basis[p][1]

        idxs = [
            i for i in range(len(items))
            if (mask >> i) & 1
        ]

        assert len(idxs) % 2 == 1
        return idxs


linear_basis = OddXorBasis()


def bootstrap(rounds=40):
    full_items = [it for it in items if len(it[0]) == block_size]

    for _ in range(rounds):
        a, b, _ = random.sample(full_items, 3)
        enc = encrypt([a, b, filler])

        items.extend([enc[1], enc[2]])
        full_items.extend([enc[1], enc[2]])


def sign_target(target8, strip=False):
    for _ in range(6):
        idxs = linear_basis.solve(items, target8)
        if idxs is not None:
            break
        bootstrap(40)
    else:
        raise RuntimeError("target is still outside the span")

    cur = [items[i] for i in idxs]

    while len(cur) > 1:
        a = cur.pop()
        b = cur.pop()
        c = cur.pop()
        cur.append(xor3(a, b, c))

    full = cur[0]

    if strip:
        return xor3(full, filler, filler, strip=True)

    return full
```

The network interaction is just Pwntools around this logic. I used `sendlineafter()`, `recvuntil()`, and friends from the `pwnlib.tubes` API.[^pwntools]

---

## Why the Challenge Breaks

There are several bugs, but none of them alone describes the whole exploit.

### 1. Message authentication is block-local

The signature authenticates blocks independently instead of authenticating the complete message structure.

This lets us collect and splice valid `(block, signature)` pairs.

### 2. Encryption and decryption both re-sign their outputs

That turns the server into a machine for generating more authenticated intermediate values.

### 3. CBC is algebraically malleable

CBC decryption contains an XOR with the previous ciphertext block or IV. By carefully crossing the encryption and decryption oracles, we make the block cipher cancel out and expose

$$
A\oplus B\oplus C.
$$

### 4. The padding is just `rstrip(b'\x00')`

This lets us deliberately convert an 8-byte signed target ending in zeros into a shorter signed final chunk.

### 5. The final admin check only cares about parsing

Once a valid signature exists for

```text
IV || {"admin":true}
```

the game is over.

---


## References

[^blowfish]: PyCryptodome documentation, **Blowfish**. Blowfish has a fixed 8-byte block size and supports CBC mode. <https://pycryptodome.readthedocs.io/en/latest/src/cipher/blowfish.html>

[^nist-cbc]: NIST SP 800-38A, **Recommendation for Block Cipher Modes of Operation: Methods and Techniques**, Section 6.2, Cipher Block Chaining mode. <https://csrc.nist.gov/pubs/sp/800/38/a/final>

[^stackoverflow-xor]: Stack Overflow, **“How to find which subset of bitfields xor to another bitfield?”** The problem is solved as a linear system over $GF(2)$ using mod-2 Gaussian elimination. <https://stackoverflow.com/questions/3855479/how-to-find-which-subset-of-bitfields-xor-to-another-bitfield>

[^pwntools]: Pwntools documentation, **pwnlib.tubes — Talking to the World!** <https://docs.pwntools.com/en/stable/tubes.html>
