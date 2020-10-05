---
title: "A Closer Look At Federated Learning"
date: 2020-08-10 17:49:16
---

This blog post is Part 2 of a two-part series on Federated Learning (FL). Part 1 was more of a high-level introduction which just introduced the concept and its applications, which you can [read here](https://amin-nejad.github.io/2020/06/07/introduction-to-federated-learning/). In this post, we will delve a little deeper into FL and introduce some privacy-preserving technologies commonly related to FL and why they are used.

## So what is the problem with FL?

So far, as we've described it in the previous post, FL seems like a great way to collaborate with other parties to jointly train a model without actually sharing your data. So what is the problem then? The core problem is that the central orchestrator to whom each party sends their model parameter updates is actually able to infer a surprising amount of information about the underlying data. For instance, in cross-device language modelling, it has been demonstrated that it's possible to learn individual words that a user has typed by inspecting that user’s most recent update [[1]](#ref_1). Further research has shown that this even extends to rare words/numbers typed only once e.g. sensitive data such as bank card details or social security numbers [[2]](#ref_2). Not only is this possible but memorization in fact actually peaks when the test set loss is lowest! Naturally this poses significant privacy concerns despite the user's data having never left the user's device.

In FL scenarios, we often think of the central orchestrator as being *honest-but-curious* (you can think of this as being one rung below *malicious*). An honest-but-curious adversary is one who will not deviate from the defined protocol but will attempt to learn all possible information that they can from the messages they receive. In most cases this is a reasonable assumption and typical of how we can expect the likes of Google and Apple to act i.e. they won't do anything they shouldn't do (or haven't said they will do) but if we send them any kind of data, they ***will*** use it. Clearly, FL in this paradigm is not a suitable privacy preserving framework. Therefore, it is almost always paired with one or more of the techniques listed below:

- Differential Privacy
- Secure Multi-Party Computation
- Homomorphic Encryption
- Trusted Execution Environments

Let's take a look at them and what they bring to the table.

## Differential Privacy

Differential privacy (DP), simply put, is a framework for measuring the leakage of information from a given model. Specifically, it provides a measure of how likely a single sample is to be identified as being present in the dataset used for training a model.

When talking about DP, we generally mean $$\epsilon$$-DP [[3]](#ref_3), where the level of privacy is is reflected by the value of $$\epsilon$$ (epsilon). We can formally define this as follows:

Let's say we have a dataset of loans $$D_n$$ and we are performing some function on our dataset $$f(D_n)$$ e.g. a machine learning model which predicts whether a loan will default or not. We can define our dataset missing one sample (loan) as $$D_{n-1}$$. For our data to be completely differentially private, we want:

$$ f(D_n) = f(D_{n-1})$$

or as close as possible as we can get it. This means that our function will output the _exact_ same value regardless of whether that missing sample is in the dataset or not. The formal definition is concerned with the probability distributions of these functions such that:

$$ e^\epsilon = \frac{P(f(D_n))}{P(f(D_{n-1}))} $$

Therefore, the smaller $$\epsilon$$ is, the more private our data is with a lower bound of 0 i.e. peak privacy.

> *DP is a technique that adds sufficient noise to our data **before we start training***

Confusingly however, we don't just refer to DP as the _framework_ for measuring information leakage, but often also as the _technique_ for minimising information leakage. Therefore in a rather tautologous way, we can say we are applying DP to achieve DP. The foremost technique for doing this is by applying noise to our data. In this sense, DP is a technique that adds sufficient noise to our data _before we start training_ such that an adversary who has access to a model trained on our data cannot infer anything about a specific sample in our dataset from the model. Intuitively we can see that the more noise we add to our data, the more our samples start to become indistinguishable from one another.

Therefore, as you will notice, DP is different from the other techniques we will discuss in that its provision of privacy comes at the expense of performance. The more differentially private our model is, the worse its performance will be. So, DP is not a simple technique we can just add to our pipeline and forget about. It comes with a delicate trade-off that needs to be managed and balanced appropriately. Unless we have a _very_ large dataset, its unlikely we will be able to achieve complete DP (i.e. $$\epsilon$$ of 0) without ending up with a model that has useless predictions!

An interesting area of research in DP is specifying different levels of privacy for different features in the dataset. High dimensional data from users leads to large amounts of noise being required to preserve differential privacy. However if users do not equally care about protecting their data from all possible inferences (which is a rather likely scenario), the noise on some features can be relaxed. This is formalised in what’s known as the _Pufferfish framework of privacy_ [[4]](#ref_4).

## Secure Multi-Party Computation

Broadly speaking, Secure Multi-Party Computation (SMPC) allows multiple parties to collectively perform some computation and receive the resulting output without ever exposing any party’s sensitive input.

In the specific case of FL, where DP addresses the issue of sensitive information being inferred from our model parameters by adding noise to our data, Secure Multi-party Computation (SMPC) allows the inferring of information but instead prevents it from being attributed to a single party. It essentially makes each party's model parameter updates useless until they are averaged with one another. So, after the averaging, even though the orchestrator can add infer information from the model parameters, they can't know which party's data it comes from.

This can be a little hard to digest so let's demonstrate this with a simple, yet classic example. Let's say Alice, Bob and Charlie want to compute their average salary without revealing their own salary to the others. Alice adds a random number to her salary and passes it on to Bob. Bob then adds his own salary to the number passed to him by Alice and passes it on to Charlie. Charlie does the same and passes it back to Alice. At this point no-one can infer anything about anyone else's salary. All Alice has to do is to subtract the random number she added at the beginning and divide by 3 to obtain the average salary without any individual salary having been exposed. In practice, SMPC is more complex than this in order to prevent collusion by a subset of the parties or various other adversarial attacks. Generally, most SMPC algorithms rely on what is known as Shamir's Secret Sharing (SSS) [[5]](#ref_5) which we will introduce below.

### Shamir's Secret Sharing

The basic principle behind SSS can be illustrated easily by considering how many points are needed to define a polynomial of degree $$k$$. 2 points are sufficient to uniquely define a line, 3 points to define a parabola, 4 points to define a cubic curve and so on so forth. Thus, it takes $$k$$ points to define a polynomial of degree $$k-1$$. In SSS, we split a secret $$S$$ (e.g. an important number) into $$n$$ _shares_, where $$n$$ is equal to the number of parties, but only $$k$$ shares are required to reconstruct the secret, where $$k < n$$. In order to do this, we therefore need to use a polynomial of degree $$k-1$$. Let's demonstrate with another example.

Let's say $$S=10$$, $$n=3$$ and for simplicity, let's say $$k=n$$, i.e. we only want the secret to be deciphered with the co-operation of _all_ parties (In cross-device FL, this ability to deconstruct the secret with only a subset of the parties is very important since clients are unreliable and may regularly drop out of training). We therefore need to construct a second order polynomial of the form $$ax^2 + bx + c$$. We encode $$S$$ as the intercept of our polynomial i.e. equal to $$c$$ and come up with random numbers for $$a$$ and $$b$$:

$$f(x) = 2x^2 + 5x + 10$$

We construct $$k$$ random points $$D_k = (x, f(x))$$ on our polynomial by inputting random numbers as values of $$x$$ (just not 0, as that would give away the secret) which we share with each party respectively.

$$D_0 = (-3, 13), D_1 = (1, 17), D_2 = (4, 62)$$

These parties can now collaborate with one another to interpolate the polynomial, find the intercept and thus reconstruct the secret. One way this can be done is using [Lagrange basis polynomials](https://en.wikipedia.org/wiki/Lagrange_polynomial):

$$
\begin{align*}
f(x) = \sum_{j=1}^{k} y_jl_j = y_0l_0 + y_1l_1 + y_2l_2
 \end{align*}
$$

Don't worry if you're not super familiar with this method. We're not going to show how it works here anyway, we'll just use the `scipy` function in python instead. If you want a numeric walkthrough, take a look [here](https://medium.com/@apogiatzis/shamirs-secret-sharing-a-numeric-example-walkthrough-a59b288c34c4).

<script src="https://gist.github.com/amin-nejad/09e172e8848f659a07c5987f6da7b1df.js"></script>

And voila, providing those three points, we can interpolate the function and thereby reconstruct the secret $$S = 10$$ by taking the value of the intercept. You can play around with the code and verify it for yourself [here](https://repl.it/@aminnejad/LagrangeBasisPolynomial#main.py).

Now how can we directly adapt this for SMPC in the context of FL? This is where things can get a bit complicated. One simple way could be that each party adds a random number to their parameter updates after they have completed a local round of training. They encode this random number as a secret and split it into shares which they share with all the other parties. Each party thus ends up with a single share from every other party. If we make sure that each party only receives a share of the same $$x$$ value, they can sum these shares that they have received to form a *total share* which they send to the central orchestrator along with their encrypted parameter update. The central orchestrator averages all the parameter updates and then reconstructs the totalled shares using the same Lagrange basis polynomial technique to get the *totalled secret*. If they now subtract this totalled secret from the encrypted average parameter update, they will get the unencrypted average parameter update without having learnt anything about any individual update.

The implementation described above is very simple but should give you an idea of how SSS could work. In reality, the algorithms in this space are a lot more complex but are still underpinned by this same idea at its core. One of the most popular ones is called SPDZ [[6]](#ref_6) which you can read more about in this great [blog post by Morten Dahl](https://mortendahl.github.io/2017/09/03/the-spdz-protocol-part1/).


## Homomorphic Encryption

The idea behind Homomorphic Encryption (HE) is remarkably simple. Let's say we have an encryption function $$E(x)$$. For this encryption function to be homomorphic, an equation such as the following would need to be satisfied:

$$E(a) + E(b) = E(a+b)$$

What this means is that we can perform computation on variables ***while they are encrypted***. If you take anything from this section, let it be that because that's all HE is. Implementing this however is not nearly as simple as the high level explanation. This is completely different from almost all other encryption schemes which render the *ciphertext* (encrypted variables) completely unusable. There have been numerous implementations of *somewhat* (partial) HE algorithms over the years which allow a limited set of operations on the ciphertext, but the first Fully Homomorphic Encryption (FHE) algorithm (which allows arbitrary computation) only came in 2009 [[7]](#ref_7).

This was something of a watershed moment in the field and there are now many FHE algorithms to choose from, however the reality is that FHE is still very expensive computationally speaking. Too expensive to make it feasible or practical for the purposes of FL. However, it turns out that we don't actually really need FHE for FL. Neural Networks can be implemented with surprisingly few operations. As long as we can perform addition, multiplication and some kind of activation e.g. ReLU (a bit harder but can be done), we have what we need. In fact there is a great [blog post](https://iamtrask.github.io/2017/03/17/safe-ai/) by [Andrew Trask](https://twitter.com/iamtrask) which takes you through a basic neural network implemention in Python using partial HE, so we won't go into much more detail here - if you want to learn more of the details, just read that. Just to be clear, it is the *parameters* of the neural network which we are encrypting, *not* the data (though that is also possible but doesn't have as much of a use case).

So now we know that we can perform this trade-off where we if we limit the type of computations that we can perform, we can still have an HE scheme that allows us to train an encrypted neural network. But how do we apply this to FL? Well, we can use an HE algorithm to encrypt our model parameters at all times and perform training, aggregation and all the rest of it while in this encrypted state. But that's pretty costly, even using one of the partial HE algorithms, this is still significantly slower than training without encryption. It turns out that if we are happy to relax our security requirements a little, we can just use HE as a replacement for SMPC i.e. for the parameter aggregation step of the process.

Each client trains their model locally on their own data as normal. When it comes to sharing the model parameters, instead of using SMPC, we can use an HE algorithm to encrypt the model parameters. If for instance, for simplicity, all clients agree on a certain encryption key, the central orchestrator can simply perform an arithmetic average of all the clients' parameter updates, apply the update and send back the new global parameters all in the encrypted state. Each client can then decrypt the parameters locally, perform another round of training, then encrypt again to send to the central orchestrator. This can also be done with multiple encryption keys but this is a bit more complicated.

> *HE requires little communication but expensive computation whereas SMPC uses cheap computation but extensive communication*


When used in this way, HE is very comparable to SMPC - they are essentially two different solutions for the same problem (hiding the individual parameter updates from the central orchestrator). Practically speaking, where they differ is in terms of the trade-off between computation and communication. HE requires little communication but expensive computation whereas SMPC uses cheap computation but a significant amount of communication between parties. The choice as to which is the best option will depend on the particular details of the use case.

## Trusted Execution Environments

This final method that we will introduce for increasing privacy falls into a completely separate bracket. Rather than approaching the issue of privacy from a data or algorithmic perspective, Trusted Execution Environments (TEEs) approach the issue of privacy from a hardware / low level software perspective. A TEE can be thought of as a ***secure enclave*** within a processor where code can be executed completely inaccessible, unbeknownst and in parallel to the main processor. Anything stored or computed in the TEE is cryptographically secure and its state and execution remain secret. Crucially also, the TEE can prove to a remote party that a piece of execution was done on a TEE and therefore not visible to anyone else, including any administrators of the computer in which the TEE resides.

Most [Apple products](https://support.apple.com/en-gb/guide/security/sec59b0b31ff/web) and some other mobile phones have already had TEEs built in for some time (e.g. [since the iPhone 5S](https://www.forbes.com/sites/quora/2013/09/18/what-is-apples-new-secure-enclave-and-why-is-it-important/)) - primarily for storing and using sensitive information such as biometric data, for Face ID or Touch ID. However with Intel and ARM having implemented there own versions called [SGX](https://software.intel.com/sgx) and [TrustZone](https://developer.arm.com/ip-products/security-ip/trustzone) respectively, there's a reasonable chance your laptop or desktop might have this included too.

Indeed TEEs are becoming increasingly common for performing various kinds of computation privately and naturally researchers and practitioners are attempting to extend this to the FL use case, however it's still early days. Theoretically, I can see two use cases for TEEs in the FL context:


1. **Protecting the model parameters or IP from the client**

    So far we have been assuming that all clients are working together collaboratively to train a model from which they will all benefit. This is generally how FL is presented but it is important to note that this is not the only paradigm under which FL can be useful. In some cases, a third party may wish to train a model on the data from multiple clients and retain the intellectual property (IP) of the model i.e. prevent anyone else seeing the model they are training. This use case necessarily requires training for every client to be done within a TEE. The problem however is that current secure enclaves are limited in terms of memory and provide access only to CPU resources, not GPUs which are commonly used for training machine learning models. This is an active area of research but unfortunately appears to be infeasible at the moment.

2. **Preventing inference on the client's parameter updates from the central orchestrator**

    This use case again addresses the same problem as that solved by SMPC and HE i.e. preventing the central orchestrator from seeing any individual client's parameter updates. In this use case, the central orchestrator simply acts entirely within a TEE and therefore anyone in the role of the central orchestrator will not have any visibility on incoming parameter updates. The only computation done here is a simple average of parameter updates, meaning we are not constrained by the limited compute offered by the TEE environment. There is a good [blog post](https://blog.openmined.org/pysyft-pytorch-intel-sgx/) from OpenMined on this use case.

Whilst the second use case described is currently feasible, there are still additional concerns aside from limited memory and compute when it comes to using TEEs in practice. For example, we may want to make sure that the runtime and memory access patterns of the TEE do not reveal information about the data upon which it is computing - this is still an open problem. Furthermore, the TEE typically only proves that a particular binary is running, it does not provide a means for proving that that binary has the desired privacy properties. In essence, use of TEEs in the FL context is still an up-and-coming area of research but one that definitely shows promise. 

## Bringing it all together

Achieving all the desired privacy properties for federated learning would ideally likely require incorporating all of the above (and possibly others) into an end-to-end system. None of the techniques described above are mutually exclusive and all solve slightly different privacy issues (with a lot of overlap). The idea being that each of these techniques adds a layer of privacy which degrades as gracefully as possible - where one layer fails for whatever reason (adversarial attacks are not within the scope of this post) - and falls back on the layer before it i.e. multiple things need to go wrong for privacy to be breached. However it is important to note that in many circumstances, some of these privacy requirements can be relaxed and we can make do with just one or two without leaking any private information. Ideally we would have them all but there are always trade-offs that need to be made.

## Other problems with FL

Thus far we have been focusing entirely on the privacy aspect of FL - and for good reason, this is the main reason it exists! But I didn't want to finish without just touching on some of the other issues FL introduces very briefly:

- One of the fundamental problems with FL is the presence of non-IID (independent and identically distributed) data (since data is coming from multiple different sources and distributions). Most models assume data is IID and therefore it is not clear how performant our models can be when this assumption is violated. There are a number of different ways we can try and deal with this such as data augmentation, hyperparameter optimization, model personalisation, etc. but essentially this is still an open question.

- Optimization algorithms i.e. variants of stochastic gradient descent (SGD) can't easily be adapted to the FL context where data is partitioned. Currently there is no consensus on how to incorporate momentum and variance-reduction techniques into the Federated Averaging algorithm (described in Part 1) or whether momentum even accelerates the convergence rate in the federated learning setting like it does locally when incorporated into popular optimization algorithms such as Adam

- It is well known that communication is one of the primary bottlenecks in FL, particularly cross-device FL. Alongside algorithmic advancements to communication efficiency for techniques such as MPC, gradient compression or quantization is also being explored as a way to reduce the communication overhead. One issue with these methods however is where they intersect with DP or SMPC as there is currently no way of marrying these techniques

If you are interested in a more comprehensive assessment of the current state of FL, take a look at [this paper](https://arxiv.org/abs/1912.04977) [[8]](#ref_8).

## Tools

Finally, if you want to get your hands dirty with FL, don't fret, the following libraries will help you get started:

- [PySyft](https://github.com/OpenMined/PySyft): whilst not yet ready for production usage, PySyft is by far and away the leading open-source library for FL. Works on top of both `pytorch` and `tensorflow`.
- [FATE](https://github.com/FederatedAI/FATE): currently the most popular library that facilitates FL for a variety of different machine learning models, not *just* neural networks
- [TF Federated](https://github.com/tensorflow/federated): the official `tensorflow` library for FL

## References

#### 1. [Bonawitz, et al. (2017)](https://eprint.iacr.org/2017/281.pdf) {#ref_1}

#### 2. [Carlini, et al. (2019)](https://arxiv.org/pdf/1802.08232.pdf) {#ref_2}

#### 3. [Dwork (2006)](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/dwork.pdf) {#ref_3}

#### 4. [Kifer and Machanavajjhala (2014)](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.436.2576&rep=rep1&type=pdf) {#ref_4}

#### 5. [Shamir (1979)](https://cs.jhu.edu/~sdoshi/crypto/papers/shamirturing.pdf) {#ref_5}

#### 6. [Damgård, et al. (2012)](https://eprint.iacr.org/2011/535.pdf) {#ref_6}

#### 7. [Gentry (2009)](https://www.cs.cmu.edu/~odonnell/hits09/gentry-homomorphic-encryption.pdf) {#ref_7}

#### 8. [Kairouz, McMahan et al. (2019)](https://arxiv.org/abs/1912.04977) {#ref_8}