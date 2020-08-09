---
title: "Introduction to Federated Learning"
date: 2020-06-07 13:49:16
---

## Introduction

This blog post is Part 1 of a two-part series on Federated Learning (FL). The series is intended to clearly yet concisely explain the key nuances in the nascent field of FL and demystify some of the terms that you may encounter. FL is at the intersection of numerous fields, not just machine learning (ML), but distributed optimization, cryptography, security, differential privacy, fairness, etc. which means that a lot of the material may be unfamiliar if you are coming just from a ML background.

Having recently had this problem myself, I wasn’t successful in finding a resource that did just this at the level or the depth that I wanted. So, this two-part series is intended to fill that gap assuming no prior knowledge of FL. In this first part, we will just introduce FL whilst in the second part we will look a bit deeper at FL and some of the other privacy-preserving techniques it is commonly associated with.

## So What is Federated Learning?

In the most broad or vanilla definition, FL is simply a ML setting where many clients collaboratively train a model under the orchestration of a central server while keeping the training data decentralized. That’s all.

The term was introduced in a paper by [McMahan et al (2016)] from Google and has taken off since then as a way to facilitate collaboration whilst ensuring privacy. The technology was initially introduced specifically for mobile and edge device applications where we have millions or potentially even billions of unreliable clients (reliability here refers to the likelihood of dropping out due to a variety of different reasons e.g. network issues, battery status, compute limitations, etc.) and we want to train a global model that we ship to all devices. Let's say this is for language modelling i.e. predicting the next word a user is going to type.

Previously, in order to do this, each mobile phone would have to upload their data to the cloud where a *central orchestrator* (e.g. Google) could train a model on all their users' typing data in one place. In practice, this would likely be across numerous data-centres and compute nodes but the key is that all the data is visible in its entirety to the orchestrator.

What FL did is turn this idea on its head. Instead of sending the data to the model, we send the model to the data.

> _'If the mountain will not come to Muhammad, then Muhammad must go to the mountain'_
> -- Francis Bacon, 1625

## How does it work?

The algorithm works as follows:

1. **Client Selection:** The central server orchestrating the training process samples from a set of clients meeting eligibility requirements
2. **Broadcast:** The selected clients download the current model weights and training program (assuming training is not being done from scratch)
3. **Client computation:** Each selected device locally computes an update to the model parameters
4. **Aggregation:** The central server collects all the model updates from the devices and aggregates them
5. **Model update:** The server locally updates the shared model based on the aggregated update

_Steps 2-5 are repeated until our model has converged_

This procedure is demonstrated in the figure below:

<img class="blog_image" src="/images/fl/fl.gif">

So instead of the orchestrator being able to see everyone's data, they just receive each user's update to the model parameters. The user data never leaves the device. This method is now in use by both Google and Apple for [GBoard] and [Siri] respectively and is termed **Cross-device FL**. For GBoard, this is responsible for predicting the next word a user is going to type (i.e. coming up with the 3 predictions displayed just above the keyboard), whilst for Siri the technology is used to better recognise and respond to a given user's voice.

## But I don't have access to millions of devices...

However, unless you are Google or Apple, chances are you don't have ready access to millions of devices which are happy to compute things for you using their data and send you their parameters :expressionless: But this doesn't mean FL is within the purview of just large tech conglomerates. We can still apply the same technique to a smaller number of clients, which is known as **Cross-silo FL**.

Where cross-device FL is characterised by a _very large number of unreliable devices_, cross-silo FL is characterised by a _small number of reliable organisations/entities_. Organisations which are actively collaborating with one another to train a global model from which they will all benefit. Cross-silo FL therefore essentially does away with a lot of the communication and computation bottlenecks that are present in cross-device FL. Computation can be performed on powerful GPUs rather than small mobile devices and communication is only between a handful of devices, typically just 2-100. This extra flexibility allows for a whole host of new opportunities.

This is what makes FL truly exciting. It is the prospect of finally removing the lock on the vast troves of confidential data that have been siloed for legal, regulatory or privacy reasons. ML techniques require *heaps* of data to be able to produce good predictions, but instead, in many cases, what we've had are small, fragmented and sequestered datasets. These have been a big barrier to the adoption (or at least full leverage) of ML methods in areas like Healthcare and Finance in particular.

In healthcare, FL adoption should see an improvement in patient outcomes for a plethora of conditions as a result of hospitals and research centres being able to share this data access with one another. Whilst in finance, better predictions on creditworthiness should result in easier and cheaper access to credit, and so on so forth for insurance premiums, fraud detection, etc. for people across the world. This could be particularly impactful for the [1.7bn adults who are still unbanked].

However, the applications don't stop at mobile, healthcare and finance. It is often said that we are currently living in the _[Wild West Era of Data]_. As we start to come out of this age with regulations like GDPR, techniques like FL will become increasingly important and commonplace in protecting all sorts of user data. Keeping data decentralised doesn't just protect it from an opportunistic orchestrator but also helps protect it from malicious adversaries who regularly hack into all sorts of sensitive databases. Just take a look at this [interactive visualisation] to see the scale and frequency of these data breaches.

## Why now?

There are numerous angles from which we can look at this question. From what we've discussed so far, FL has not presented a material hurdle from a technical standpoint. In fact, the idea is remarkably simple. Why then has it taken this long to be adopted? While, in practice FL is combined with other crucial privacy-preserving technologies (which we will discuss in the next blog post), these technologies also predate the original 2016 FL paper by some years. Yes, researchers are advancing the state-of-the-art with these all the time but these advancements have not really had a material impact on outright feasibility since the deep learning boom of the past decade. This is *particularly* true for cross-silo FL given its relaxed requirements.

I think it's likely that without the determined efforts from the likes of Google and Apple with cross-device FL, the possibility of cross-silo FL would not have come to light (or at least as soon), even despite its relaxed requirements compared to cross-device FL. But what took so long for cross-device FL?

- **Neural Networks:** Whilst the application of FL is not limited to neural networks, this is certainly its greatest area of research and interest due to the particular data-hungriness of neural networks - even more so than other ML methods. Since neural networks only started to become a popular ML method after the deep learning boom began in 2012, FL thereby didn't really have a reason to exist until this event happened i.e. until neural networks became commonplace. 

- **Compute:** Running compute-intensive neural network models on mobile phones has only recently become a feasible option. As computing power has increased for both CPUs and GPUs, this has also made its way into mobile phones and facilitated this transition to FL. With the recent adoption of so called _[AI chips]_, specifically designed for AI operations on mobile, this trend will only accelerate

- **Expectations:** Consumer expectations of data privacy have been increasing steadily over recent years - particularly as a result of data breaches and scandals such as that of Cambridge Analytica. This coupled with regulations like GDPR coming into force, technology companies are increasingly prioritising consumer privacy where previously they might have taken a more lax approach

## Next time...

In the next blog post, we will look at some of the other privacy-preserving technologies regularly used alongside Federated Learning as well as some important nuances and how you can use some of these technologies yourself.

[mcmahan et al (2016)]: https://arxiv.org/abs/1602.05629
[siri]: https://www.technologyreview.com/2019/12/11/131629/apple-ai-personalizes-siri-federated-learning/
[gboard]: https://ai.googleblog.com/2017/04/federated-learning-collaborative.html
[1.7bn adults who are still unbanked]: https://globalfindex.worldbank.org/sites/globalfindex/files/chapters/2017%20Findex%20full%20report_chapter2.pdf
[wild west era of data]: https://www.theatlantic.com/sponsored/pwc-2019/internets-wild-west-days-are-coming-close/3064/
[interactive visualisation]: https://www.informationisbeautiful.net/visualizations/worlds-biggest-data-breaches-hacks/
[ai chips]: https://www2.deloitte.com/us/en/insights/industry/technology/technology-media-and-telecom-predictions/2020/ai-chips.html
