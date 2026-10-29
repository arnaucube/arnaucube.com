# How Aerie aggregates Falcon signatures efficiently
*2026-10-29*

Almost two years ago (late 2024, pre-AI coding era), with [Ahmad Afuni](https://ahmadafuni.com) we implemented the (post-quantum) [Falcon signature](https://falcon-sign.info) verification in a Plonky2 circuit. We used a variety of tricks to make it efficient and reduce the amount of needed circuit constraints, but one of the pain points was that in Plonky2 we work over the Goldilocks field $\mathbb{F}_p$, with $p=2^{64} - 2^{32} + 1 = 18446744069414584321$,
whereas Falcon signatures work over the much smaller finite field $\mathbb{F}_q$, with $q=12289$, which means that inside our Plonky2 circuits we had to compute the modulo reduction check every few operations, to ensure that despite working over $p$, the values were actually smaller than $q$.

A couple of weeks ago ([announcement](https://x.com/SuccinctJT/status/2111136119593082925)) Alberto Centelles, Quang Dao, Mohammadtaghi Badakhshan and Justin Thaler published [Aerie: Efficient Aggregation of Falcon Signatures](https://github.com/a16z/aerie/blob/main/paper/aerie.pdf). Among many cool techniques, Aerie uses an interesting approach to ensure that the aggregated Falcon signatures satisfy the Falcon signature verification, including the modulo $q$ check over Aerie's field ($p= 2^{64} - 59$). The full scheme is quite complex, and would not fit in a blog post, but in this post I'll focus on the technique used for the modulo $q$ checks, which I found really nice.

<blockquote style="font-size:105%;color:black;">

It is the kind of cases where you can optimize the zk circuits a lot, but there is an alternative path relying on algebra which drastically improves performance, so it's always interesting to learn!

</blockquote>



### Falcon signature verification

Falcon signatures are post-quantum signatures based on lattice cryptography, currently being standarized by NIST (FN-DSA). For each Falcon signature, we need to check

$$i)~ s_1 + h \cdot s_2 \equiv c \pmod {q,~ X^n +1}$$
$$ii)~ || s_1 ||_2^2 + ||s_2||_2^2 + ||e||_2^2 = \lfloor \beta^2 \rfloor$$
$$iii)~ | s_{2,j} | \leq 2047,~~ \sum_j \lfloor \frac{ |s_{2,j}| }{ 128 } \rfloor \leq L$$

> <div style="font-size:80%;">
> Here $e \in \mathbb Z^4$ is an auxiliary slack vector introduced by Aerie (not at the original Falcon signatures) to transform original Falcon's inequality into the equality we see at $(ii.)$ (works thanks to the <a href="https://en.wikipedia.org/wiki/Lagrange%27s_four-square_theorem" target="_blank">Lagrange's four-square theorem</a>).
> </div>

For the norm bounds check $(ii.)$, LaBRADOR (lattice-based SNARK scheme) has a similar norm check and already uses a projection technique, Johnson-Lindenstrauss projection, which I covered at the [LaBRADOR notes post](/blog/labrador.html#2.-johnson-lindenstrauss projection). LaBRADOR uses it to (probabilistically) bound the Euclidean norm of a committed witness without checking its coefficients individually, and Aerie uses a variant of it.

Condition $(iii.)$ enforces the signature encoding requirements from Falcon.

For the check $(i.)~ s_1 + h \cdot s_2 \equiv c \pmod{q, X^n +1}$, we're interested in proving that the coefficients of $s_1 + h \cdot s_2$ and $c$ are congruent modulo $q$ after reduction modulo $X^n + 1$.


In this post we will focus on the check $(i)$.

### One MLE to rule them all. ie. batching many verifications


<blockquote style="font-size:85%;">

For an introduction to MLEs (multilinear extensions), highly recommended <a href="https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.html" target="_blank">Justin Thaler's book</a> (Chapter 3.5). Here's a quick recap:

<br>
A multivariate polynomial $g$ is multilinear if the degree of the polynomial in each variable is at most one.

<br>
Any function $f: \{0,1\}^n \longrightarrow \mathbb{F}$ has a unique multilinear extension (MLE) over $\mathbb{F}$, denoted $F$. That is, $F$ is the unique multilinear polynomial over $\mathbb F$ satisfying

$$F(x_1, \ldots, x_n) = f(x_1, \ldots, x_n) ~~\forall~ (x_1, \ldots, x_n) \in \{0,1\}^n$$

<br>
For each Boolean vector $b=(b_1, \ldots, b_n) \in \{0,1\}^n$, we define the basis polynomial

$$
\chi_b(X_1,\ldots,X_n)
= \prod_{i=1}^{n}
\begin{cases}
X_i, & b_i = 1,\\
1-X_i, & b_i = 0.
\end{cases}
$$

This polynomial is interesting because $\chi_b(x) =
\begin{cases}
1 & x=b\\
0 & x \neq b
\end{cases}
~~\forall~ x \in \{0,1\}^n
$

<br>
Then, the MLE can be obtained as

$$F(X) = \sum_{b \in \{0,1\}^n} f(b) \cdot \chi_b(X)$$

The key idea here, is that the MLE and the original function agree at the same values when evaluated at any value of the <i>Boolean hypercube</i> (fancy way to say all possible values of $\{0,1\}^n$), and outside the Boolean hypercube the original function is undefined but it's MLE has a well-defined value.

<br>
Notation: usually the MLE is denoted as $\tilde{f}(X)$, but here I've followed Aerie's notation $F(X)$.

</blockquote>


In Aerie, since we're batching many Falcon signatures, we're not working on the original equation

$$s_1 + h \cdot s_2 \equiv c \pmod {q,~ X^n +1}$$

(one for each signature), but encoding all the signatures values into batched multilinear extensions (MLEs). This is obtained by computing the MLEs of the Falcon polynomials we're batching:

$$S_1(j, i) = s_{1,i,j} \pmod p;
~~~~~~~~~~~~H(j, i) = ctr_q (h_i)_j \pmod p$$
$$S_2(j, i) = s_{2,i,j} \pmod p;
~~~~~~~~~~~~C(j, i) = ctr_q (c_i)_j \pmod p$$

> <span style="font-size:85%">
> Here $p= 2^{64} - 59$, which is Aerie's field modulus, and $ctr_q(\cdot)$ returns the centered integer representative modulo $q$, defined as
> 
> $$ctr_q: \mathbb F_q \longrightarrow \mathbb Z, ~ \text{such that}$$
> 
> $$ctr_q(a) \equiv a \pmod q, ~~ -\frac{q}{2} < ctr_q(a) < \frac{q}{2}$$
> </span>


Each MLE ($S_1, S_2, H, C$) is a batched MLE which encodes the values of the various original Falcon values. You can imagine each $S_1$ encoding a table where each column has a signature associated and encodes the coefficients of the signature of that column, ie.

|          | $i=0$        | $1$          | $\ldots$ | $N-1$          |
| ------   | ------------ | ---          | -----    | -----          |
| $j=0$    | $S_1(0,0)$   | $S_1(0,1)$   | $\ldots$ | $S_1(0,N-1)$   |
| $1$      | $S_1(1,0)$   | $S_1(1,1)$   | $\ldots$ | $S_1(1,N-1)$   |
| $\vdots$ | $\vdots$     | $\vdots$     | $\ddots$ | $\vdots$       |
| $n-1$    | $S_1(n-1,0)$ | $S_1(n-1,1)$ | $\ldots$ | $S_1(n-1,N-1)$ |

For $N$ batched signatures, each original polynomial with $n$ coefficients. And similarly for $S_2(j,i), H(j,i), C(j,i)$.



> <div style="font-size:85%;">
> Also, recall that the MLEs, while they preserve the original function's outputs when evaluated at the boolean hypercube, they do not preserve the polynomial multiplication; that is, $MLE(h \cdot s_2) \neq MLE(h) \cdot MLE(s_2)$.
> <br><br>
> In Falcon, we're operating inside the polynomial ring $\mathbb{F}_q[X]/(X^n +1)$, where multiplication means computing the negacyclic convolution, which is equivalent to doing ordinary polynomial multiplication and then reducing modulo $X^n +1$. In Aerie's paper, they use the operator symbol $\circledast$ for when performing negacyclic convolution on their Boolean-domain coefficent vectors, to denote clearly the negacyclic convolution of the coefficient vectors rather than ordinary multiplication of the MLEs.
> </div>






Then, from the batch equation of the Falcon signatures verification,

<!-- $$S_1(j,i) + H(j,i) \circledast S_2(j,i) \equiv C(j,i) \pmod{q, X^n+1}$$ -->

Aerie defines the *virtual residual* $\Delta$ as the batched MLE which when evaluated at the boolean hypercube matches the original Falcon residual coefficients modulo $p$,

$$\Delta(\cdot, i) = S_1(\cdot, i) + H(\cdot, i) \circledast S_2(\cdot, i) - C(\cdot, i), ~~~~\text{over}~ \mathbb{F}_p$$

> <div style="font-size:85%">
> Notation: here the paper uses the $(\cdot, i)$ notation, which indicates that the Falcon signature index $i$ is fixed, while the coefficient $j$ ranges over $\{ 0, \ldots, n-1 \}$, so that $S_1(\cdot, i)$ denotes the vector of coefficients belonging to signature $i$:
>
> $$S_1(\cdot, i) = ( S_1(0,i),~ S_1(1,i),~ \ldots, S_1(n-1, i) )$$
> So, while $S_1(j, i)$ refers to one coefficient, $S_1(\cdot, i)$ refers to all coefficients for the signature $i$. therefore $S_1(\cdot, i)$ is a vector of length $n$.
>
> </div>


For the Falcon check, we need to check divisibility by $q$ over the integers, but in Aerie we're operating $\Delta$ over $\mathbb{F}_p$. Since $q \nmid p$, reducing an integer modulo $p$ could change its divisibility by $q$, so what Aerie does is to introduce the *integer interpretation of the residual*, $\Delta^{(\mathbb Z)}$, and we want to establish $ctr_p(\Delta)=\Delta^{(\mathbb Z)}$.

The paper defines $\Delta^{(\mathbb Z)}$ as

$$\Delta^{(\mathbb{Z})} = ctr_p(S_1) + H \circledast S_2 - C$$

where the operations are over $\mathbb Z$. Here we use $ctr_p(S_1)$ to interpret the committed field elements of $S_1$ as signed integers, while $H, S_2, C$ are already in their bounded integer representatives (bounded in $(-p/2, p/2)$).

With this, we have that $\Delta^{(\mathbb{Z})}\pmod p = \Delta$ coefficientwise on the Boolean domain, but this does not imply
$\operatorname{ctr}_p\Delta=\Delta^{(\mathbb Z)}$.


To ensure the equality, we need to ensure the integer residual ($\Delta^{(\mathbb Z)}$) does not wrap around modulo $p$, which is guaranteed by the bound $| \Delta^{(\mathbb{Z})} (j, i) | < p/2$, so that we can recover the original integer value by the centered reduction, $ctr_p(\cdot)$, implying $ctr_p(\Delta)=\Delta^{(\mathbb Z)}$. This bound is not checked coefficientwise but instead is derived indirectly through bounding $S_1, S_2, e$ (while $H, C$ are publicly bounded), using the **triangle inequality** and **Cauchy-Schwarz inequality**.

> <div style="font-size:85%">
> <i>I learned the following Proposition and Corollary from <a href="https://sites.google.com/view/matanprasmashomepage/notes" target="_blank">Matan Prasma</a>'s notes on Lattices (<a href="https://drive.google.com/file/d/1Z0Nh6u1hxdpdddgixJgEWjvZZ7OPHIFf/view" target="_blank">link</a>).</i>
> <br><br>
> <b>Proposition (Cauchy-Schwarz inequality):</b><br>
>
> $$\text{for}~~ v, w \in \mathbb R^n,~ |\langle v, w \rangle| \leq ||v|| \cdot ||w||$$
>
> <br>
> <i>Proof:</i><br>
> If either $v$ or $w$ is $0$, then it is trivial.
> <br>
> Set $a=- \langle v,w \rangle,~ b=\langle v,v \rangle =||v||^2$.
> <br>
> Then,
>
> $$0 \leq \langle av + bw, av+bw \rangle$$
> $$=\langle av, av \rangle + \langle av, bw \rangle + \langle bw, av \rangle + \langle bw, bw \rangle$$
> $$=a^2 \underbrace{\langle v, v \rangle}_{||v||^2} + ab \underbrace{\langle v, w \rangle}_{-a} + ab \underbrace{\langle w, v \rangle}_{-a} + b^2 \underbrace{\langle w, w \rangle}_{||w||^2}$$
> $$= a^2 \underbrace{||v||^2}_{b} - aba -aba + b^2 ||w||^2$$
> $$=\underbrace{a^2b - a^2b}_{0} - a^2 b + b^2 ||w||^2$$
> $$=-a^2b + b^2 ||w||^2$$
> $$=b (-a^2 + b ||w||^2)$$
>
> Since $v \neq 0$, we know that $b > 0$, so that
>
> $$0 \leq -a^2 + b ||w||^2$$
>
> which means that $~~| \langle v, w \rangle | \leq ||v|| \cdot ||w||~~$
> as desired. $\quad\quad\quad\square$
>
> <br><br>
> 
> <b>Corollary (Triangle inequality):</b><br>
>
> $$\forall~ v,w \in \mathbb R^n,~ ||v+w|| \leq ||v|| + ||w||$$
>
> <br>
> <i>Proof:</i><br>
>
> $$||v + w||^2 = \langle v+w, v+w \rangle$$
> $$=\langle v, v \rangle + 2 \langle v,w \rangle + \langle w,w \rangle$$
> $$=||v||^2 + 2 \langle v,w \rangle + ||w||^2 \leq ||v||^2 + 2 ||v|| \cdot ||w|| + ||w||^2$$
>
> (notice that the $\leq$ is applied using the Cauchy-Schwarz inequality)<br>
>
> $$ =(||v|| + ||w||)^2$$
>
> Therefore,
>
> $$||v+w|| \leq ||v|| + ||w||$$
>
> <div style="float:right">$\square$</div>
>
> <br>
> Geometric interpretation: the length of each side of a triangle is less than the sum of the lengths of the other two sides.
> </div>

In Aerie, the *Triangle inequality* and *Cauchy-Schwarz* are applied as follows:

We have $\Delta^{(\mathbb Z)} = ctr_p(S_1) + H \circledast S_2 -C$, so, applying the triangle inequality:

$$|\Delta^{\mathbb Z}(j,i)| \leq |ctr_p(S_1(j,i))| +|(H(\cdot,i)\circledast S_2(\cdot,i))_j| +|C(j,i)|$$

> <div style="font-size:85%;">
> Notice that here we're first computing the convolution of the two vectors $H(\cdot, i),~ S_2(\cdot,i)$, ie. $(H(\cdot,i)\circledast S_2(\cdot,i))$, and then selecting the coefficient $j$.
> </div>

and then the Cauchy-Schwarz:

$$|(H(\cdot, i) \circledast S_2(\cdot, i))_j| \leq \|H(\cdot, i) \|_2 \cdot \|S_2(\cdot, i) \|_2$$

Thus

$$|\Delta^{(\mathbb Z)}(j,i)| \leq |ctr_p(S_1(j,i))| + \| H(\cdot,i) \| \cdot \| S_2(\cdot,i) \| +|C(j,i)|$$

> <div style="font-size:85%;">
> Here notice that we're taking the absolute values ($| \ldots |$) for $\Delta^{(\mathbb Z)},~ ctr_p(S_1),~ C$ coefficients, but the Euclidean norm ($\| \ldots \|$) for $H$ and $S_2$ vectors. This is because we're taking the individual scalar coefficients of the first three (hence the $(j,i)$ notation), while $H$ and $S_2$ are vectors (hence the $(\cdot,i)$ notation as mentioned earlier).
> </div>

And at Aerie's Lemma A.6, it is proven that $|ctr_p(S_1(j,i))| + \|H(\cdot,i)\| \cdot \|S_2(\cdot, i)\| +|C(j,i)| < p/2$, therefore we have that

$$|\Delta^{(\mathbb Z)}(j,i)| < p/2$$


In this way, we can express the bound on each coefficient of $\Delta^{(\mathbb Z)}$ using the known bounds on $S_1, S_2, H, C$.
This bound is proven indirectly through the *randomized norm projection*, which probabilistically bounds $S_1$ and $e$. Combined with the coefficient bounds on $S_2$ and the algebraic norm equation, this establishes the integer norm bounds needed to derive the residual bound.

> Isn't great when you can apply Propositions and Corollaries you learned in the past to applied cases? I love the feeling.


With the bounds in place, we have $ctr_p(\Delta)=\Delta^{(\mathbb Z)}$, and then the original Falcon relation is equivalent to

$$\Delta^{(\mathbb{Z})} \equiv 0 \pmod q$$

To do this last check, Aerie uses the randomized *residual projection* technique.


### Randomized residual projection

Now that we've established $\operatorname{ctr}_p\Delta=\Delta^{(\mathbb Z)}$, we want to prove $\Delta^{(\mathbb Z)} \equiv 0 \pmod q$, which means that every coefficient of $\Delta^{(\mathbb Z)}$ is divisible by $q$. $\Delta^{(\mathbb Z)}$ has $nN^*$ coefficients ($N$ signatures batched, $N^*$ padded signatures, each with $n$ coefficients); but we want to do it without checking all $nN^*$ coefficients individually.


> <div style="font-size:85%">
> Here $N$ is the number of Falcon signatures being aggregated, and $N^*= max(2^{\lceil log_2 N \rceil}, 4096/n)$ is the next power of two of $N$ with a minimum padded size of $4096/n$.
> </div>

Aerie uses a randomized linear projection $\Pi_{res}$, which takes the $nN^*$ evaluations of the batched MLE $\Delta$ and compresses them into just 31 field elements:

$$\Pi_{res}: \mathbb{F}_p^{nN^*} \longrightarrow \mathbb{F}_p^{31}$$

<div style="text-align: center;margin:25px;">
<img src="img/posts/aerie/residual-projection.png" style="max-width:70%;" />
 <br><i style="font-size:80%;">Diagram from Aerie paper <a href="https://github.com/a16z/aerie/blob/main/paper/aerie.pdf" target="_blank">https://github.com/a16z/aerie/blob/main/paper/aerie.pdf</a></i>
</div>

The prover computes $y_{res}=\Pi_{res}(\Delta)\in\mathbb F_p^{31}$, where each output value is a random linear combination

$$y_{res,o} = \sum_{j,i} w_{o,j,i} \Delta(j,i) \pmod p$$

where the weights $w_{o,j,i}$ are determined by three successive layers of random matrices derived from a random seed chosen by the verifier after the prover commits to the witnes, so that the witness can not be chosen to exploit the projection.

$\Longrightarrow$ If every coefficient of $\Delta^{(\mathbb Z)}$ is divisible by $q$, then any integer linear combination of its coefficients is also divisible by $q$.
<!-- Conversely, if at least one coefficient is not divisible by $q$,the random linear combinations are unlikely to be divisible by $q$. Since we use 31 output rows, and 25 possible random coefficients (their values range from -12 to 12), the probability is $25^{-31}$; which across three layers used by Aerie, the probability is at most $3 \cdot 25^{-31}$.-->

Thus, the verifier checks the centered integer representatives of the 31 projected values:

$$q\mid\operatorname{ctr}_p(y_{res,o}),~~~~ \forall\,o<31$$

There is one subtlety: the projection operates over $\mathbb F_p$, so its results might wrap around modulo $p$, potentially changing their divisibility by $q$. Aerie bounds this probability, together with the probability that the random combinations fail to detect an invalid residual.

Finally, since the verifier doesn't know $\Delta$, Aerie uses SumCheck (+ Akita PCS) to prove that the 31 projected values were correctly computed from the committed witness. Thus, the randomized projection checks **divisibility**, while SumCheck checks **consistency with the witness**.



### Wrapup

When I first saw Aerie's [announcement](https://x.com/SuccinctJT/status/2111136119593082925) I was interested to see how they managed to batch efficiently the Falcon verifications, since as mentioned at the beginning of this post, in the past I worked on the Falcon verification inside zk circuits (plonky2). The paper turned out to be a pleasant read, making usage of proofs I've studied in the past (ie. Cauchy-Schwarz inequality) which is also a nice feeling, and improving on the projections I saw some months ago in the LaBRADOR scheme.

I'm still following in parallel all the hash-based schemes development, specially now that Ethereum ecosystem is doubling down on them (at least for the validators signatures). But while hash-based signatures are considered by part of the community (not everybody) the safest path, they are also a bit more 'boring' (without any aim to underestimate them); I find more interesting to study the lattice-based ones which have much more interesting mathematical objects behind them to dive into.

