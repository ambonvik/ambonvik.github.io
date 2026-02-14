---
title: "Cimba is loose"
date: 2026-02-12  13:32:20 +0300
description: "Cimba is released as public beta on GitHub and has already collected 
more than 60 stars there."
excerpt: "A first look at Cimba's main advantages and early traction"
mathjax: true
header:
  overlay_image: /assets/images/cimba_logo_banner.jpg
  teaser: /assets/images/cimba_logo_teaser.jpg
---
My [discrete event simulation library Cimba 3.0](https://github.com/ambonvik/cimba), 
written in C, was released as a public beta on GitHub on December 27, 2025. Following 
discussions both on [Hacker News](https://news.ycombinator.com/item?id=46872818) and [Reddit](https://www.reddit.com/r/OperationsResearch/comments/1r0aiir/cimba_open_source_discrete_event_simulation/), 
Cimba has already collected more than 60 GitHub stars in its first few weeks in public.

The headline advantage of Cimba is its speed compared to similar tools. SimPy 
with its interpreted Python base language is the most direct comparison. As one 
perhaps would expect from a library of compiled C and assembly code, 
[Cimba runs about 45 times faster than a comparable model in SimPy](https://cimba.readthedocs.io/en/latest/background.html#benchmarking-cimba-against-simpy). 
A speedup of this magnitude is typical when going from Python to C, but as far as I 
have been able to find out, no such thing existed for process-oriented discrete 
event simulation before now.

This speed advantage translates directly to increased resolution in experiments. For 
example, if one is able to run 10 replications of a particular scenario in SimPy within a 
certain budget for time and computing resources, Cimba can run 450. 

All else equal, this will tighten the confidence intervals on the result by a factor of 
nearly 8. With few samples ($n < 30$), one needs to use the $t$-distribution to construct 
a confidence interval. For $n = 10$, this gives a critical value of $t^*$ = 2.26 for a 
95 % confidence level, while a larger sample can use the critical value 
$Z_{\alpha/2}$ = 1.96. Together with the increased degrees of freedom, this gives a 
confidence interval width ratio of

$$ r = \frac{\frac{2.262}{\sqrt{10}}}{\frac{1.960}{\sqrt{450}}} = 7.7$$

The other main advantage is one of model expressivity. Cimba is based on simulated 
processes as stackful asymmetric coroutines. That makes it possible for a simulated 
process to yield and resume control from any level of its call stack. The processes 
are first-class citizens of the language and can be created at runtime, assigned to 
variables, passed as arguments, returned by functions, and so on. Processes can create 
and destroy other processes as needed, and handle other processes as if they were 
passive objects in the model.

With Cimba, one can build models where, e.g., queuing customers are opinionated agents 
that balk, renege, and jockey about in the queues. These ornery customer-agents can be 
created by the arrival process, served by the service process, and destroyed by the 
departure process as any passive object would. One example can be found in the 
[Cimba tutorial](https://cimba.readthedocs.io/en/latest/tutorial.html#agents-balking-reneging-and-jockeying-in-queues). 

In addition, early users have pointed out the powerful logging features and the 
native integration with C debuggers as major advantages to understand in detail what is 
going on in a model.

It is well known that object-oriented programming was invented with Simula67 as a 
natural way of organizing a simulation model. One might think that building simulation 
models with C as the base language, without explicit support for object-orientation in 
the language itself might be a disadvantage. In fact, it is not. 

Object-orientation is a design pattern, not a language feature. Cimba uses object-oriented 
design wherever that is the most natural way of structuring the design. For example, the 
processes are subclasses of the coroutine class, and the user model can derive 
subclasses of processes where needed. 

At the same time, it does not force the user to apply object-oriented design where it is 
*not* appropriate. Cimba's pseudo-random number generators are not objects, just functions 
to be called. Conceptually, the entropy is part of the simulated universe, not a 
thing one carries around. This avoids writing unnecessary "object-oriented" boilerplate 
code when it is not the right tool for the job.

In some areas, Cimba is even approaching a functional programming paradigm. Our 
processes can pass an explicit demand function as a waiting criterion to order a 
wakeup call whenever that function evaluates as true. This is exposed to the user in 
the condition variable class, and is an implicit part of all other resource and queue 
classes (where the demand function is pre-packaged internally).

Combined with a similar feature for custom priority queue ordering functions, this gives 
great flexibility to the user model, while providing simple pre-packaged resources and 
queues for the typical cases.

I'm biased, of course, but I think Cimba turned out pretty good. You can find the code on 
[GitHub](https://github.com/ambonvik/cimba) and the documentation on 
[ReadTheDocs](https://cimba.readthedocs.io/en/latest/index.html).

In the next post, we will discuss how the 45x speed advantage translates into 
statistical power and experimental resolution.