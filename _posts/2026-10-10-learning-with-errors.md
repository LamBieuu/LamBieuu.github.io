---
title: "Learning With Errors: Why the Noise Matters"
date: 2026-10-10 00:00:00 +0700
categories: [Cryptography, Lattices]
tags: [lwe, lattices, noise, public-key-encryption, security-models]
description: "A worked introduction to LWE, from noisy linear equations to encryption, rounding, and the problem of keeping errors under control."
math: true
toc: true
---

Suppose I give you a few linear equations and ask you to find the unknowns. You would probably reach for Gaussian elimination. Fair enough; that is what it is for.

Now suppose every answer is slightly wrong.

The unknowns have not changed, and neither have the coefficients. But before handing you each equation, I add a small random error to its right-hand side. Can you still recover the unknowns?

This is the starting point of **Learning With Errors**, usually shortened to **LWE**. The arithmetic happens modulo an integer, and the interesting instances have many more dimensions than the examples below. Still, a small example is enough to see what the error is doing.

I want to understand two things here: why noisy equations are useful for encryption, and why the same noise becomes a problem when we compute with ciphertexts.

## Start with equations that are exact

We will work modulo $q=97$. For this example, the hidden vector is

$$
s=\begin{bmatrix}3\\5\end{bmatrix},
\qquad
A=\begin{bmatrix}
2&7\\
6&1\\
4&3
\end{bmatrix}.
$$

Without any error, the public answers would be

$$
As=
\begin{bmatrix}
2\cdot3+7\cdot5\\
6\cdot3+1\cdot5\\
4\cdot3+3\cdot5
\end{bmatrix}
=\begin{bmatrix}41\\23\\27\end{bmatrix}
\pmod{97}.
$$

Someone who knows $A$ and $As$ can solve for $s$: the first two rows already form an invertible matrix over $\mathbb F_{97}$. Keeping $s$ out of the public-key file would not help much.

Now add the error vector

$$
e=\begin{bmatrix}1\\-1\\0\end{bmatrix}.
$$

The public answers become

$$
b=As+e
=\begin{bmatrix}42\\22\\27\end{bmatrix}
\pmod{97}.
$$

The first equation says $2s_1+7s_2=42$, although the true inner product is $41$. The second has been nudged in the other direction. An attacker receives $A$ and $b$, but does not receive the errors that would let it undo those nudges.

Elimination still combines the equations. It also combines their unknown errors, so its output is no longer an exact solution for the original secret.

> These numbers are deliberately tiny. We could recover this particular secret by brute force. The example shows the arithmetic, not a secure parameter choice.

## What exactly is the LWE problem?

In the matrix convention used here, we sample

$$
A\leftarrow\mathbb Z_q^{m\times n},
\qquad s\leftarrow\mathbb Z_q^n,
\qquad e\leftarrow\chi^m,
$$

and publish

$$
(A,b),\qquad b=As+e\pmod q.
$$

Here $n$ is the secret dimension, $m$ is the number of samples, and $\chi$ is a specified distribution of small integer errors. Each row gives one noisy inner product with the same secret.

**Search LWE** asks an attacker to recover $s$. **Decision LWE** asks it to distinguish $(A,As+e)$ from $(A,u)$, where $u$ is an independent uniform vector. Regev introduced LWE and established hardness connections to worst-case lattice problems for particular parameter regimes.[^regev]

The dimensions, modulus, error distribution, and number of samples belong to the assumption. Saying "there is some noise" is not enough to claim hardness. With zero error, we are back to linear algebra. With badly chosen parameters, adding nonzero error may still leave an easy problem.

There is also a useful distinction from secret sharing. LWE is a **computational** assumption: it concerns what efficient attackers can do. It does not promise the information-theoretic privacy of Shamir sharing with truly random coefficients.

## Small modulo what?

An error of $-1$ is small. But its usual residue modulo $97$ is $96$.

If we measure the size of that residue as an ordinary nonnegative integer, it looks enormous. This is why we use **centered representatives** when discussing noise:

$$
96\equiv-1\pmod{97},\qquad
95\equiv-2\pmod{97}.
$$

For odd $q$, the centered representatives lie between $-(q-1)/2$ and $(q-1)/2$.

It helps to imagine the residues arranged around a circle. Zero and $96$ are neighbours; the printed numbers hide that fact. A decoding rule must respect this wraparound too.

The distribution $\chi$ often comes from a discrete Gaussian. One common parameterization assigns an integer $z$ probability proportional to

$$
\exp(-\pi z^2/\sigma^2).
$$

Most mass sits near zero, with increasingly unlikely larger values. The width parameter $\sigma$ and the tail bounds must be interpreted using the chosen convention. In particular, "usually small" is not the same as "always bounded by one." Peikert's survey develops the Gaussian and lattice background in more detail.[^peikert]

## How can the owner decrypt noisy data?

At first this sounds awkward. If the error hides the secret, why does it not also ruin decryption?

The trick is to encode the message with enough space around it. For a bit $\mu\in\{0,1\}$, let

$$
\Delta=\lfloor q/2\rfloor.
$$

We encode zero near $0$ and one near $\Delta$. With $q=97$, these centres are $0$ and $48$.

The following is the basic Regev-style public-key encryption pattern.[^regev] Let $(A,b=As+e)$ be the public key and $s$ the secret key. To encrypt a bit, choose fresh binary randomness

$$
r\leftarrow\{0,1\}^m
$$

and form

$$
u=A^Tr\pmod q,
\qquad
v=b^Tr+\Delta\mu\pmod q.
$$

The ciphertext is $(u,v)$. The owner computes

$$
\begin{aligned}
v-s^Tu
&=(As+e)^Tr+\Delta\mu-s^TA^Tr\\
&=\Delta\mu+e^Tr
\pmod q.
\end{aligned}
$$

The secret-dependent part cancels. The message encoding remains, plus a small accumulated error. Decryption chooses whichever encoding centre is closer around the modulus circle.

This cancellation explains correctness. Establishing security also requires the appropriate LWE parameters and enough randomness in $r$; it does not follow from the cancellation alone.

### Try it with our numbers

Take the same $A,b,s,e$ as above, and choose

$$
r=\begin{bmatrix}1\\0\\1\end{bmatrix},
\qquad \mu=1.
$$

Then

$$
u=\begin{bmatrix}6\\10\end{bmatrix},
\qquad
v=42+27+48=117\equiv20\pmod{97}.
$$

The owner obtains

$$
v-s^Tu
=20-(3\cdot6+5\cdot10)
=-48\equiv49\pmod{97}.
$$

The result $49$ is one step away from the centre $48$, so the owner recovers bit $1$. Indeed,

$$
e^Tr=1,
\qquad 49=48+1.
$$

For bit $0$, the same illustrative randomness would instead give a residual of $1$, which decodes to zero. Actual encryptions sample fresh randomness each time.

## The noise has a budget

Suppose decryption leaves

$$
d=\Delta\mu+E\pmod q.
$$

As long as $E$ is small enough, rounding still finds the right centre. A simple sufficient condition for nearest-centre decoding is

$$
|E|<\frac{\Delta}{2}.
$$

For $q=97$, this is $|E|<24$. You will often see the approximate condition $|E|<q/4$. Using $\Delta/2$ here keeps the odd-modulus boundary and possible ties out of our example.

The phrase **noise budget** refers to this room for error. Computations can consume it.

For example, if we combine noisy values with integer weights $w_i$, their errors combine too:

$$
E_{\mathrm{combined}}=\sum_i w_iE_i.
$$

If every $|E_i|\leq B$, the triangle inequality gives

$$
\left|E_{\mathrm{combined}}\right|
\leq B\sum_i|w_i|.
$$

This bound is conservative: some errors may cancel. But it tells us what we can guarantee without relying on a lucky cancellation. A coefficient of $100$ can turn an error of $1$ into an error of $100$.

For vectors, we will use

$$
\|e\|_\infty=\max_i|e_i|.
$$

Thus $|r^Te|\leq\|r\|_1\|e\|_\infty$. Since $r$ is binary, $\|r\|_1$ is just the number of selected samples. These simple inequalities will be useful when we reach threshold reconstruction.

### Can we just increase the modulus?

Increasing $q$ can give decoding more room if the other quantities stay fixed. It also increases the number of bits needed to store residues, and changes the modulus-to-noise ratio in the hardness assumption. Other dimensions or noise widths may need to change with it.

So parameter selection has two jobs: make decryption work with overwhelming probability, and meet the security requirements of the construction. A successful toy decryption only checks the first job for that example.

## Where this leads

The part I find useful to keep in mind is the cancellation:

$$
\text{secret-dependent terms cancel}
\quad\Longrightarrow\quad
\text{message encoding} + \text{remaining error}.
$$

It explains both why LWE encryption can work and why further computation needs care. We are allowed some error, but the decoder has a limited margin.

In the [Shamir post]({% post_url 2026-09-03-secret-sharing-and-threshold-cryptography %}), reconstruction multiplies shares by Lagrange coefficients. If the values being reconstructed still contain LWE noise, those coefficients multiply the noise as well. The next post works through [what happens when Shamir meets LWE]({% post_url 2026-10-12-when-shamir-meets-lwe %}).

The [Decomposed LWE post]({% post_url 2026-09-08-decomposed-lwe %}) takes a different next step: it introduces structured public matrices needed by more advanced constructions. Ordinary LWE and its decoding arithmetic are the starting point for that discussion.

## References

[^regev]: Oded Regev, *On Lattices, Learning with Errors, Random Linear Codes, and Cryptography*, STOC 2005; expanded journal version, 2009. [Author's paper](https://cims.nyu.edu/~regev/papers/qcrypto.pdf), especially the LWE definition and public-key cryptosystem.

[^peikert]: Chris Peikert, *A Decade of Lattice Cryptography*, 2016. [IACR ePrint 2015/939](https://eprint.iacr.org/2015/939). Background on lattice distributions, LWE, and cryptographic constructions.
