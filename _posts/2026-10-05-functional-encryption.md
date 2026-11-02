---
title: "Functional Encryption"
date: 2026-10-05 00:00:00 +0700
categories: [Cryptography, Encryption]
tags: [functional-encryption, registered-fe, bounded-collusion, security-models]
description: "An introduction to functional encryption, with a focus on registered FE for circuits and the security model behind bounded collusion."
math: true
toc: true
---

Ordinary encryption gives a key holder access to the whole plaintext. **Functional encryption (FE)** changes the decryption key: each key is associated with a function, and decrypting reveals that function's output on the encrypted data.

The key idea is simple:

$$
\text{ciphertext} + \text{key for } f
\quad\longrightarrow\quad
f(\text{plaintext}).
$$

The security goal is to control what a key holder can learn from the ciphertext. To make that goal precise, we also have to say which kind of FE we mean and what powers the attacker has.

## The basic FE interface

A simplified FE system has four algorithms:

1. **Setup** creates public parameters and a master secret key.
2. **Key generation** uses the master secret key to create a functional key for a chosen function $f$.
3. **Encryption** encrypts an input $x$.
4. **Decryption** evaluates $f$ on that input.

Writing the public key as $mpk$ and the functional key as $sk_f$, correctness means

$$
\operatorname{Dec}(sk_f,\operatorname{Enc}(mpk,x))=f(x).
$$

For example, suppose the encrypted input is a vector $x=(x_1,x_2,x_3)$ and a user receives a key for

$$
f_y(x)=\langle x,y\rangle.
$$

That user learns the inner product $\langle x,y\rangle$. The user does not receive a general-purpose key that decrypts the entire vector.

The output is intentionally revealed. If a function returns the whole plaintext, then its key holder is allowed to learn the whole plaintext. FE protects information beyond the authorized function outputs; it cannot make those outputs reveal nothing.

## Which type of FE?

Functional encryption is an umbrella term. Schemes can support different function families, such as predicates, inner products, or Boolean circuits. They can also differ in how keys are issued and how many keys may collude.

This post focuses on **registered functional encryption for circuits with bounded collusion**. These phrases describe different parts of the design:

- **For circuits** describes the functions the keys can represent. A function is supplied as a Boolean circuit.
- **Registered FE** describes how keys enter the system. Each user creates its own key pair and registers its public key and function; a curator or aggregation procedure maintains the public information needed for encryption and decryption.
- **Bounded collusion** describes the attacker. Security is proved only while at most $Q$ users' functional keys are compromised.

One concrete example of this flavor is a registered FE scheme for polynomial-size circuits with a bound $Q$ on corrupted users.[^rfe]

In a central-authority FE scheme, one authority creates every functional key from a master secret. Registered FE shifts key creation to the users: they generate their own keys and register them. The curator processes registrations but does not need to hold a master secret that can derive every user's decryption key.[^rfe]

The collusion bound is a real condition on the theorem, not a footnote. A $Q$-bounded scheme promises security against coalitions of up to $Q$ key holders. It makes no promise for a larger coalition unless the scheme has a separate unbounded-collusion guarantee.

## What is a security model?

A security model is a formal test for a security claim. It specifies what the adversary sees, what queries or keys it can obtain, and what counts as breaking the system.

For FE, a useful indistinguishability game works as follows:

1. The challenger sets up the scheme and gives the public information to the adversary.
2. The adversary selects two candidate inputs, $x_0$ and $x_1$, and obtains permitted functional keys.
3. For every function $f$ attached to a key held by the adversary, the challenge inputs must have the same permitted output:
   
   $$
   f(x_0)=f(x_1).
   $$

4. The challenger chooses a hidden bit $b$ and returns an encryption of $x_b$.
5. The adversary tries to guess $b$.

Security means that every efficient adversary's advantage over a random guess is negligible as the security parameter $\lambda$ grows:

$$
\left|\Pr[b'=b]-\frac{1}{2}\right|\leq \operatorname{negl}(\lambda).
$$

The equal-output condition is essential. If the adversary holds a key for a function that gives different answers on $x_0$ and $x_1$, it can evaluate that function and distinguish the challenge. That is authorized leakage, not a failure of confidentiality.

In a bounded-collusion model, the experiment also limits how many function keys or users the adversary can control. In a registered scheme, the experiment must say how registration works and whether corrupted users may submit adversarial public keys. The timing matters too: a **static** model fixes the corrupted users before the challenge, while an **adaptive** model lets the adversary choose them during the experiment.

This game is a useful way to understand FE privacy, but it is not the only possible definition. Boneh, Sahai, and Waters showed that a natural indistinguishability game can be too weak for some function families. A simulation-based definition instead asks whether an attacker’s view could be generated using only the outputs its authorized functions reveal.[^bsw] A security theorem should name its exact definition; the words “FE-secure” alone leave out important details.

For comparison, the [Shamir post's security section]({% post_url 2026-09-03-secret-sharing-and-threshold-cryptography %}#why-t-1-shares-reveal-nothing) defines privacy by asking whether fewer than $T$ shares reveal the secret; Shamir provides perfect privacy. FE asks whether function keys reveal anything beyond their permitted outputs, usually under a computational assumption.

## What makes an FE scheme secure?

A functional-encryption scheme is not secure merely because it decrypts to $f(x)$. Its security claim depends on the definition, its assumptions, and the parameter choices.

A clear claim should specify at least:

- **Allowed leakage:** Which functions and outputs may each key reveal?
- **Collusion bound:** How many users or functional keys may the attacker combine?
- **Registration rules:** Can the attacker register its own keys or choose malicious public keys?
- **Challenge timing:** Must the attacker choose its challenge before setup, or can it do so adaptively?
- **Proof model:** Does the proof use explicit algorithms in the standard model, or idealize something such as a hash function as a random oracle?
- **Hardness assumption:** Which computational problem must remain difficult for efficient attackers?

The proof then connects the game to the assumption. Informally, it shows that an attacker who learns more than the allowed function outputs could be used to solve the assumed-hard problem. The strength of the conclusion is limited by every condition in that reduction: the function family, the collusion bound, the registration rules, and the proof model.

For example, one registered FE construction for circuits proves adaptive simulation security with a collusion bound $Q$ fixed at setup. Its paper gives an instantiation under the $k$-Lin assumption in the standard model and another under LWE and evasive LWE in the random oracle model.[^rfe] This is the useful shape of a security statement: it names the privacy definition, attacker power, assumption, and proof setting.

## What FE security does not promise

FE does not automatically hide the function associated with a key. Hiding the function itself is a separate property called function privacy.

FE also does not erase leakage from authorized outputs. Several individually harmless-looking function values may together reveal sensitive facts about the input. The security definition must allow the outputs needed by the application while ruling out additional leakage.

Finally, a bounded-collusion theorem does not protect against arbitrarily many colluding key holders. A static theorem does not automatically cover adaptive corruption. A proof in the random oracle model is a theorem about an idealized hash function, not an automatic guarantee for every concrete hash implementation.

## Takeaway

Functional encryption lets a key holder compute a specified function on encrypted data without giving that holder an unrestricted decryption key. **Registered FE for circuits with bounded collusion** is one particular flavor: users register keys and functions, circuits describe the permitted computations, and the proof limits how many users may collude.

What makes the scheme secure is the complete statement around it: the permitted outputs, the adversary's keys and queries, the collusion bound, the registration model, the hardness assumption, and the proof setting. Those details say exactly what the theorem protects.

---

# References

[^bsw]: Dan Boneh, Amit Sahai, and Brent Waters, “Functional Encryption: Definitions and Challenges,” TCC 2011. [IACR ePrint 2010/543](https://eprint.iacr.org/2010/543).

[^rfe]: Yijian Zhang et al., “Bounded Collusion-Resistant Registered Functional Encryption for Circuits,” [IACR ePrint 2024/1572](https://eprint.iacr.org/2024/1572).
