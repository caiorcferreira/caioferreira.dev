---
author: "Caio Ferreira"
title: "Notes on Dos and Dont's of Machine Learning in Computer Security"
date: "2023-06-22"
tags: ["ml", "security", "supervised-learning", "ai"]
cover:
  image: /images/notes-on-do-and-donts-of-ml-in-security.png
  hidden: true
---

Following the subject from my last post, [Reflections about Supervised Learning on Security](https://caioferreira.dev/posts/reflections-supervised-ml/reflections-supervised-learning-in-security/), I put down some more thoughts about the implementation of learning-based systems in the Security domain.

This is my extension to the problems and recommendations presented on the paper [Dos and Don'ts of Machine Learning in Computer Security](https://mlsec.org/docs/2022-sec.pdf) (Quiring, et al, 2022). I encourage you to also read the paper, as it's excellent and provide a lot of insights about how to better build machine learning models.

## Machine learning workflow

### Data collection

> Pitfalls: Sampling Bias, Label Inaccuracy

As with any other domain, Security is also heavily affected by bad data quality used for training. However, it's often worse in Security because data acquisition of adversary activity is usually hard and the method's used to acquire it will bring a bias.

Let's take for example a honeypot implemented with a vulnerable Apache Server. Even though there are lots of bad actors in the wild, if you have a medium size environment, the volume of data they will produce attacking your honeypot will not come near to the volume of packets in your production network. Also, the TTPs you are going to see will come from threat actors that are used to leverage Apache Server in their kill chain, possibly leaving other adversaries, that may be focusing on other types of vectors, producing different kill chains, out of the dataset.

On top of that, we are usually dealing with lots of examples. So, if we have label inaccuracies, dealing with them is a lot harder. We can't just apply common techniques like using the mode to fill in, because in the Security domain, the adversary is actively trying to mimic the distribution of the benign cases. Therefore, we have very little space for noise, as it would blur even more the distinction between the outcomes.

Besides the recommendations presented in the article, I would add two more:

- User open and disseminate datasets whenever possible. Dataset sharing is still uncommon on the Security community. We already have some network and host datasets, but there are many more assets nowadays (logs for cloud, kubernetes, CDN, CI/CD, etc), and the technologies on networks and hosts are constantly evolving, posing the need for these datasets to be always updated.
- Use a model design that depends less on adversary data, such as we discussed in the previous article. This will reduce the dependency on the adversary behavior and increase the sources of data available to use.

### Model design and implementation

> Pitfalls: Data Snooping, Spurious Correlations, Biased Parameter Selection

Developing machine learning models is no easy task. Feature engineering, hyperparameters optimization, data preparation and appropriate splitting for validation (to avoid snooping). These are just some of the challenges when taking on this endeavor.

Using tools like AutoML may help reduce the burden, however it won't take care of everything. In the end, you still need to understand your data characteristics and how it's related to your problem.

But, some of these relations and characteristics tend to repeat. Time relations, for example, are extremely common and important in a lot of Security problems. Therefore, I suggest that every Security team doing machine learning on its own to think about how they can extract such aspects into reusable components.

This has the added benefit to scale the impact of the Security members that are more focused on implementing machine learning. Our domain has many different areas and enabling other teams and specialists to more easily and correctly implement models, even as proofs of concept, expands the possibilities of what the enterprise can achieve.

In the end, the best strategy to best tackle all these challenges and be more prepared to handle pitfalls it to avoid jumping to complex and sexy algorithms from the start, such as neural networks.

Using explanation techniques, as the article suggests, can help you catch spurious correlation, but even better is to use simpler and more understandable methods. I have been seen great results with simple probabilistic methods, such as with Histogram Based Outlier Score. Using a simple algorithm, we can immediately see what is driving its decision and catch faster these types of pitfalls.

### Performance evaluation

> Pitfalls: Inappropriate Baseline, Inappropriate Performance Measures, Base Rate Fallacy

As I mentioned before, most times when building a learning-based system in Security, we end up with imbalanced and noisy datasets. This demands a special care when choosing performance metrics to best reflect our model. Building on the previous suggestion, having common components that use by default metrics more fit to most problems in Security, such as precision, recall, and MCC, would reduce the change of human error.

However, even after computing correct performance metrics, we still need good baselines to compare them against. The article suggest goods options such as using simple methods or automated machine learning. I would add that comparing it with vendor products can also be a good experiments to fully understand the impact of replacing the third party with the model.

### Operation

> Pitfalls: Lab-Only Evaluation, Inappropriate Threat Model

Although the article identifies Lab-Only Evaluation as a common pitfall in Security model, I think this is one of the topic where in Security we have the most options to address.

First, we could use adversary emulation frameworks and popular attack toolkits to launch real world attacks against a test environment and study the model response. Another option would be to set up a honeypot and evaluate how well the model performed by comparing to a human analyses of the evidences afterward.

This type of experiment would also feed back into our threat model. Given the hype of AI in latest months, more knowledge is beings shared and produced about security best practices and process for machine learning, however we can still consider the topic in its infancy.

One special vector that we are still starting to discuss is supply-chain attacks to models, specially with the popularization of transfer-learning for more complex algorithms such as LLMs.

[Splunk showed](https://www.splunk.com/en_us/blog/security/paws-in-the-pickle-jar-risk-vulnerability-in-the-model-sharing-ecosystem.html) that more than 80% of HuggingFace's models use pickle-serialized code, which is vulnerable to arbitrary code execution and code injection, although it's not possible to say if any of them is malicious.

But, besides low-level vulnerabilities like this, a transfer-learning based model can also inherit the biases (intentionally placed or not) from the original model. That is, if the original model is vulnerable to an adversarial example, there is a high risk that your new model is going to also be.

This opens the possible for two types of attacks:

1. Malicious models: base models shared with adversarial examples trained, such as a large batch of images with a negative label having a red square, making the model learn that any image with a red square should be classified as negative, therefore implanting a bypass. This type of attack would be extremely difficult to detect.
2. Cross model generalization: as [Szegedy, et all points in their paper](https://arxiv.org/pdf/1312.6199.pdf), different models may be susceptible to the same adversarial examples, even when having different hyperparameters. In this case, the models would not even be so different, therefore an adversary could search for adversarial examples against the original model and just use them in the target system, with a high chance of success.

## Conclusion

In conclusion, implementing learning-based systems in the security domain presents challenges that require careful consideration. Addressing issues related to data collection, model design and implementation, performance evaluation, and operational considerations is crucial. By implementing these recommendations and reflecting on these problems, organizations can enhance their capabilities in security.
