# BBS+ Signature Scheme

BBS+ signatures enable selective disclosure of the signed messages, meaning that we can show only part of the signed
payload to the verifier and that the verifier does not get to see the signature.

This document is meant as a somewhat high level overview of how the scheme works (going to the lowest level for the
brave souls), as there is no explanation of it in plain English. I assume that the reader has 0 knowledge of
cryptography and how it works, however, if any part bores you, feel free to skip it and return back to it if you need
it.

## Boring math basics

### Groups

Groups allow us to do one way operations, so that if I give you some number, you don't know which number I used to get
to the number I have given you. For example:
$$2^x \bmod 11 = 10$$
10 is the number you are shown, you know that I used 2 as the base, you know that I used 11 to do the modulus, but there
exists no formula which would tell you what the $x$ was. This is called **discrete logarithm problem**. This $x$ is
known as my private key, 10 is my public key. Yes, that wall of text that you see as private key is actually a number. A
**HUGE** one.

<details>
    <summary>More boring details</summary>

You can think of groups as a set of numbers to which we can do some operation and the product of the operation is still
part of this group. Groups that interest cryptographers are usually natural numbers modulus some big number.
Let's take a group of 5 numbers as an example: 0, 1, 2, 3, 4 - we will throw away 0, because no one likes 0. If we use
an operation such as exponentiation over 2, we will observe that we map each number to a new number, but we never
repeat.

1. $2^1 \pmod 5 = 2$
2. $2^2 \pmod 5 = 4$
3. $2^3 \pmod 5 = 3$
4. $2^4 \pmod 5 = 1$
5. $2^5 \pmod 5 = 2$
6. $2^6 \pmod 5 = 4$
7. $2^7 \pmod 5 = 3$
8. ...

So from above we can observe that we repeat after four elements and that we map 1 => 2, 2 => 4, 3 => 3, 4 => 1. The base
2 is called generator, because by using it, we can get every other element of the group (1, 2, 3, 4). The cool thing
about this is that there is no logarithm to "reverse engineer" the exponent (without generating the whole group). The
group above is called a *cyclic group*, because we can generate the same number of elements as we are moding by
-> 5 (minus 1).

Mathematically we would write this as $\mathbb{Z}_{5}^*$. The * is there because $\mathbb{Z}_5$ also includes 0.

</details>

### Elliptic curves

Elliptic curve cryptography is the same shit, but different. Instead of exponentiation, we use point multiplication,
which we can compute efficiently. What do I mean by point multiplication? We have a *generator* $G$ (a point from which
we can get all the other points on the curve), which we "add to itself" => $G + G + G ... + G \equiv x \times G$.
The $x$ is a private key... And it is HUGE! (but not nearly as huge as the RSA keys are.) Why is this important?
Additions are fast-ish, but not so fast, that we would do addition $x$ times. That is why we multiply $x \times G$ which
is way faster.

How exactly does this work? There are plenty of sources that explain this, for the purposes of this doc, just know that
we add and multiply points instead of multiplying and exponentiating, but they have the same role respectively.

Formulas for the signature magic actually use multiplication and exponentiation. I don't know why, but I don't make the
rules around here, just follow them.

### Pairing friendly curves

Pairing friendly curves are curves that have some additional properties. Mainly, they allow us to "pair" two groups
together using some operation $e$. What does "pair" mean? It means that when we do $e(G_{1}^x, G_{2}^y)$ that is the
same as $e(G_{1}, G_{2})^{xy}$ -> we can lift the "exponents" away from the points, meaning that
$e(G_{1}, G_{2}^{xy}) \equiv e(G_{1}^{xy}, G_{2})$, which will enable us to do some cool shit later on.

## The actual algorithm

### The curve

We use BLS12-381, but we could use any other pairing friendly curve for this.

### Signing

#### BLS

I'll first introduce the simpler BLS signatures, just for you to see how signing works. Signing is essentially just
multiplication of a point by some number. So a basic BLS signature looks something like this:

- we have two groups $G_{1}$ and $G_{2}$ with generators $g_{1}$ and $g_{2}$
- $x$ is a private key
- $g_{2}^x \equiv w$ is a public key in $G_{2}$
- $m$ is a message we want to sign represented as a point in $G_{1}$

To sign we just do $m^x$, which I'll refer to as $S$. To verify the signature, we compare:
$$e(S, g_{2}) \equiv e(m, w)$$

<details>
    <summary>Why does this work???</summary>

At first this might look like random shit thrown together... And it kind of is, at least from the eyes of whoever is
verifying the signature. We can however expand the random letters and get the following:

- $e(S, g_{2}) \equiv e(m, w)$
- $e(m^x, g_{2}) \equiv e(m, g_{2}^x)$ - substitute the letters with how they were constructed
- $e(m, g_{2})^x \equiv e(m, g_{2})^x$ - this is the cool part of pairing friendly curves: we can extract the exponent
  out
  of parenthasis

But remember, as a verifier we are only shown $S$ and $w$, we cannot calculate the $x$ out of them.

</details>

#### BBS--

Now let's try to do this for multiple messages. I'll start with the basic version and add some details later:

- $x$ is a private key
- $g_{2}^x \equiv w$ is a public key
- $M$ is a set of messages represented as numbers
- $H$ is a set of random points in $G_{1}$ that we will multiply by messages
- $|H| = |M|$ -> We generate the same number of random points as we have messages

First we will multiply the random points by messages and sum them together
$$h_{1}^{m_{1}} \times ... \times h_{i}^{m_{i}} = \prod_{i=1}^{|M|} h_i^{m_i}$$

and then also multiply them by generator $g_{1}$. The result is some point in $G_{1}$. If we now exponentiate this point
by private key $x$, we will get our basic signature $A$. So
$$A = (g_{1} \times h_{1}^{m_{1}} \times ... \times h_{i}^{m_{i}})^{\frac{1}{x}}$$
$$A = (g_{1} \times \prod_{i=1}^{|M|} h_i^{m_i})^{\frac{1}{x}}$$

So how would we verify this? Well... First, we have to send a bit more information to the verifier, namely:

- $w$ - BLS public key
- $H$ - Random bases
- $A$ - Signature value
- $M$ - Messages

With this info, the verifier can compute the value $b$:
$$b = g_{1} \times \prod_{i=1}^{|M|} h_i^{m_i}$$
and verify:
$$e(A, w) \equiv e(b, g_{2})$$

<details>
    <summary>How does this work?</summary>

If we write everything out, we will get the following

- $e(A, w) \equiv e(b, g_{2})$
- $e(b^{\frac{1}{x}}, g_{2}^x) \equiv e(b, g_{2})$
- $e(b, g_2)^{\frac{1}{x} \times x} \equiv e(b, g_{2})$
- $e(b, g_2) \equiv e(b, g_2)$

</details>

#### BBS+

Now to the actual implementation. Most of the stuff above will remain the same, we will just add some additional
parameters more or less.

So, the signer starts with:

- $x$ is a private key
- $g_{2}^x \equiv w$ is a public key
- $M$ is a set of messages represented as numbers
- $H$ is a set of semi random points in $G_{1}$ which we will multiply by messages
- $h_{0}$, $e$ and $s$ are some random parameters
- $|M| + 1 = |H|$ - We added one more random point to $H$

Notice $h_{0}$, $e$ and $s$ are new random values. Also, $H$ points are now "semi random", because we will use some
publicly known function $f$ and pass public key $w$ to it and generate $H$:
$$f(w) = H$$
This way we will not need to send the $H$ parameters to the verifier, saving us some space, but losing us some time for
computation of $H$.

The $b$ is now computed as:
$$b = g_{1} \times h_{0}^{s} \times \prod_{i=1}^{|M|} h_i^{m_i}$$
We added $h_{0}^{s}$. The exponent of the signature is a bit modified so that it also uses $e$:
$$A = b^{\frac{1}{e + x}}$$

The verifier will now get:

- $w$ - BLS public key
- $A$ - Signature value
- $e$ and $s$ - Random parameters
- $M$ - Messages

He will then calculate $H$ by using $f(w)$ and $b$ and do:
$$e(A, w \times g_{2}^e) \equiv e(b, g_{2})$$

<details>
    <summary>How does this work?</summary>

Mostly the same as above, the only difference is that now we have to add $g_{2}^e$ on the left side, because we signed
the messages with $\frac{1}{x+e}$.

- $e(A, w \times g_{2}^e) \equiv e(b, g_{2})$
- $e(b^{\frac{1}{x+e}}, g_{2}^x \times g_{2}^e) \equiv e(b, g_{2})$
- $e(b^{\frac{1}{x+e}}, g_{2}^{x+e}) \equiv e(b, g_{2})$
- $e(b, g_{2})^{\frac{1}{x+e} \times (x+e)} \equiv e(b, g_{2})$
- $e(b, g_{2}) \equiv e(b, g2)$

</details>

### Proofs

We have now arrived at the hardest part of the scheme to understand, which is how do we prove the signature without
showing it to the verifier and also how do we do this while only showing some of the messages the signer signed. I'll
again start by a simplified version, which will not use zero knowledge proofs.

#### BBS--

So now we have the signature, which is composed of:

- $A = b^{\frac{1}{x + e}}$ - Signature value
- $s$ and $e$ - random values

To make the proof, we will first randomize $A$ so that we get $A' = r_1 \times A$ ($r_1$ is a random value), then we
will calculate $\bar{A} = A'^{-e} \times b^{r_1}$. Next, we will calculate $d = b^{r_1} \times h_{0}^{-r_2}$ ($r_2$ is a
random value) and $s' = s - r_2 \times \frac{1}{r_{1}}$. That's it! We now have all the pieces of the proof.

What exactly are we proving? Three things:

1. $e(A', w) \equiv e(\bar{A}, g_2)$
2. $\frac{\bar{A}}{d} \equiv A'^{-e} \times h_{0}^{r_{2}}$
3. $g_1 \times \prod_{i \in D} h_i^{m_i} \equiv d^{\frac{1}{r_1}} \times h_0^{-s'} \times \prod_{j \notin D} h_j^{-m_j}$

Where $D$ is a set of disclosed messages. The equations were given by god himself and are thus true. If you don't trust
that, you can "prove" them yourself, as we have done up until now.

So what does verifier need? Well, "too much", but we will give it to him anyway, for now. Our proof thus consists of:

- $A'$ - ok
- $\bar{A}$ - ok
- $d$ - ok
- $M_{hidden}$ - nok (these are hidden messages after all)
- $-e$ - nok
- $r_2$ - nok
- $r_1$ - nok
- $s'$ - nok

Important thing to note here is that left side of all equations contains the data, which should be visible to the
verifier.

#### BBS+

Now for the brain fuck that are ZKP schemes. We want to hide all the "not ok" parts of the proof from the verifier.
Well, all of them are exponents and there exists a non-interactive ZKP scheme - Fiat-Shamir.

<details>
   <summary>What is Fiat-Shamir?</summary>

Alice wants to prove to Bob that she knows secret $s$ satisfying $y \equiv g^s$, without telling Bob what $s$ is.

1. She picks random value $r \in \mathbb{Z}_q^*$ and computes commitment $C = g^r$
2. She computes challenge $X = H(g, y, t)$ where $H$ is a hash function
3. She computes response $v = r - s \times X$. The result is proof (C, v)
4. Bob now checks if $C \equiv g^v \times y^X$.

This works, because:
$$g^v \times y^X \equiv g^{r-sX} \times y^X \equiv g^{r-sX} \times g^{sX} \equiv g^{r-sX+sX} \equiv g^r \equiv C$$

... Yes, magic.
</details>

I will show how this is done for proving the 2. point, as it is shorter, but the same stuff applies to point 3.

Looking at the right side of this equation, everything is public except $-e$ and $r_2$
$$\frac{\bar{A}}{d} \equiv A'^{-e} \times h_0^{r_2}$$

First, prover will create a commitment by randomizing the bases: $C = A'^{r_x} \times h_0^{r_y}$ - $r_x$ and $r_y$ are
some random numbers known as **blinding factors** - they hide $A'$ and $h_0$ from us, get it? And more importantly,
these will also help in hiding the values of $-e$ and $r_2$.

For Fiat-Shamir we also need a challenge $X$, which both prover and verifier can compute (similar to computing the $H$,
but instead of using public key, they use the proof itself). We want to hide $-e$ and $r_2$ exponents from the verifier,
so we will make the following responses $r_x - (-e) \times X$ and $r_y - r_2 \times X$. We put both commitment and
responses into proof.

Verifier now gets back the following proof (without point 3 for the sake of simplicity):

- $A'$
- $\bar{A}$
- $d$
- $C$
- $responses = [r_x - (-e) \times X, r_y - r_2 \times X]$

To validate point 2, we have the following formula:

$$C \equiv A'^{responses[0]} \times h_0^{responses[1]} \times (\frac{\bar{A}}{d})^X$$

Basically commitment (sum of randomized bases) is equal to sum of bases multiplied by responses plus left side of the
equation multiplied by the challenge.

<details>
    <summary>WTF? Why?</summary>

Let's write everything out slowly:

- $C = A'^{responses[0]} \times h_0^{responses[1]} \times (\frac{\bar{A}}{d})^X$
- $C = A'^{r_x - (-eX)} \times h_0^{r_y - r_2X} \times (\frac{\bar{A}}{d})^X$ - substitute responses
- $C = A'^{r_x + eX} \times h_0^{r_y - r_2X} \times (\frac{A'^{-e} \times b^{r_1}}{b^{r_1} \times h_{0}^{-r_2}})^X$ -
  substitute $\bar{A}$ and $d$
- $C = A'^{r_x + eX} \times h_0^{r_y - r_2X} \times (\frac{A'^{-e}}{h_{0}^{-r_2}})^X$ - $b^{r_1}$ cancels itself out
- $A'^{r_x} \times h_0^{r_y} = A'^{r_x + eX} \times h_0^{r_y - r_2X} \times (\frac{A'^{-e}}{h_{0}^{-r_2}})^X$ -
  substitute commitment $C$
- $A'^{r_x} \times h_0^{r_y} = A'^{r_x + eX} \times h_0^{r_y - r_2X} \times (A'^{-e} \times h_{0}^{r_2})^X$ - resolve
  fraction
- $A'^{r_x} \times h_0^{r_y} = A'^{r_x + eX} \times h_0^{r_y - r_2X} \times A'^{-eX} \times h_{0}^{r_2X}$ - resolve
  exponentiation
- $A'^{r_x} \times h_0^{r_y} = A'^{r_x + eX - eX} \times h_0^{r_y - r_2X + r_2X}$ - collapse same bases
- $A'^{r_x} \times h_0^{r_y} = A'^{r_x} \times h_0^{r_y}$

Magic!
</details>

More generally, we could write:

- $r^*$ - blinding factors
- $Z \equiv Y$ - $Y$ is of shape $\prod a_i^{b_i}$
- $C = Y_{bases}^{r^*}$
- $responses[i] = r_i^* - Y_{exponents[i]} \times challenge$
  $$C = (\prod_{i=1}^{|responses|} Y_{bases[i]}^{responses[i]}) \times Z^{challenge}$$
