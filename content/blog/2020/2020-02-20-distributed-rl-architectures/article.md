Date: 2020-02-20
Slug: distributed-architectures-reinforcement-learning
Summary: History of distributed architectures for Reinforcement Learning
Title: History of distributed architectures for Reinforcement Learning
Tags: reinforcement-learning

Many recent successes of Reinforcement Learning are fueled by a massive increase
in the scale of experiments. This article traces the evolution of the
distributed architectures used to support these models and discusses how
algorithm families (eg on-policy, off-policy) affect implementation choices at
datacenter scale.

## In principle, there was Supervised Learning

In Supervised Learning, the training data distribution is stationary.  Gradient
Descent in SL is an (almost) "embarassingly parallel" problem in which multiple
workers can progress independently with few communication bottlenecks. _Data
parallelism_, where different workers train in parallel on different chunks of
the training dataset is a great fit for Stochastic Gradient Descent.

The archetype for  this approach is **DistBelief**. [@distbelief] Multiple
workers sample batches of training data, compute gradients, and send them to a
global parameter server which applies them to the shared instance of the model.
Since the training data distribution is stationary, the parameter server can
simply average incoming gradients before applying them (ie the gradient and sum
operators commute).  Individual workers can also take advantage of *model
parallelism* and break the model down in multiple parts to fit in limited GPU
memory. The parameter server itself can also be sharded to cope with the traffic
generated by many workers.

Reinforcement Learning doesn't enjoy the same luxury. For one, models need to go
out and fetch their own data from the environment. Even more worringly, the
training data itself depends on the policy, and therefore can't be reused
willy-nilly (this is true even in off-policy algorithms). As a result, each RL
breakthrough relies on a delicate interplay of algorithms, distributed
architectures and plain hacks.

In the next sections, we'll follow the history of distributed architectures for
the two main families of RL algorithms: state-based and policy-based. We'll see
how they recycle and iterate on the same ideas and have somewhat converged over
the past few years. Basic knowledge of RL fundamentals and vocabulary is
assumed.

## Architectures for Q-Learning algorithms

Q-Learning algorithms are value-based off-policy algorithms especially suited
for discrete action spaces.

Google's 2015 paper about **Gorila** describes the archetypical architecture for
distributed Q-Learning at scale. [@gorila] _Gorila_ follows the same blueprint
as DeepBelief, with independent agents that collect samples from the
environment, update their parameters, and communicate gradients back to a global
parameter server. In the "bundled" configuration analyzed in the paper, each
worker also has its dedicated _replay memory_ to store state transitions and
periodically downloads an updated copy of the Q model from the parameter server.
As sketched out in the paper, workers are designed to run on different machines
within a datacenter, and coordinate over the network through the parameter
server.

<img src="{attach}graphics-1.svg"
     style="max-width: 80%; transform: scale(1);"
     class="img-center" />

DeepMind's **Ape-X** architecture from 2018 is another approach for distributed
Q-Learning where the focus is not on parallelizing learning (as in _Gorila_),
but data collection. [@apex] Again, we have independent workers that periodically sync
with a global copy of the Q-Network. Compared to _Gorila_, however, workers
share experiences rather than gradients. This is achieved through a _global_
replay buffer which receives experience samples from all independent workers.

<img src="{attach}graphics-2.svg"
     style="max-width: 80%; transform: scale(1);"
     class="img-center" />

Since Ape-X takes advantage of *Prioritized Experience Replay*, a single GPU
worker has enough capacity to keep up with the experiences produced by hundreds
of CPU workers. The insight behind sharing experiences instead of gradients is
that they remain useful for longer because they're not as strongly dependent on
the current Q function.

## Architectures for Actor-Critic algorithms

Actor-critic algorithms are a family of on-policy algorithms that combine a
policy-based (_actor_) and value-based (_critic_) models with the goal of
improving the performance of vanilla policy gradients (mainly by reducing variance).

The landmark architecture for Actor-Critic methods is **A3C**, which came out in
2016. [@a3c] A3C also uses asynchronous actor-learners, but implements them as
CPU threads within a single physical host, therefore eliminating network
communication overhead. Each worker maintains a local copy of the value and
policy networks, and periodically synchronizes them with the global copy.

<img src="{attach}graphics-3.svg"
     style="max-width: 80%; transform: scale(1);"
     class="img-center" />

The algorithmic insight behind A3C is to take advantage of parallelism to reduce
the adverse effects of correlation in the training data. In other words,
aggregating values from multiple concurrent actors effectively achieves a
similar result as random sampling from the replay buffer in Q-Learning.

A3C doesn't scale well beyond 16 concurrent actor threads because the
local policy gradients quickly become outdated as the updates from other threads
are included asynchronously in the global model. On-policy algorithms like
Actor-Critic don't cope well with this _policy lag_ problem.

Pratically, it turns out that A3C is little more than a thought exercise. A
non-distributed version of A3C, **A2C**, obtains the same performance with
simpler code. [@a2c] A2C achieves a higher throughput than A3C by (synchronously)
batching together updates from multiple environments and running gradient
descent on a GPU. In effect, the regularization and exploration benefits in A3C
should not be attributed to async-induced noise, but to the presence of multiple
independent exploration policies.

DeepMind's **IMPALA** from 2018 takes A2C further and converges towards the
multiple-actor, single-learner model proposed in Ape-X. [@impala] In contrast to
A3C, IMPALA workers communicate experiences, and not gradients, just like Ape-X.
However, Ape-X runs on-policy algorithms which can learn from any state
transition, whereas IMPALA is an Actor-Critic that can only learn from on-policy
experience.  In other words, experience collected by workers under out-of-date
policies corrupts the policy gradient (_policy lag_). IMPALA uses a novel
importance sampling algorithm (_V-trace_) to work around this problem without
throwing away samples.

<img src="{attach}graphics-4.svg"
     style="max-width: 80%; transform: scale(1);"
     class="img-center" />

Another success story of large-scale RL based on Actor-Critic methods was
reported by OpenAI when their **OpenAI Five** agent defeated the Dota 2 world
champions. [@openai2019dota] Like IMPALA, actors submit samples to
a pool of GPU optimizers, each of which maintains a local experience buffer and
samples it to produce policy gradients for the PPO algorithm. Gradients are
aggregated across the GPU pool and applied synchronously.

Both the Dota game engine and the OpenAI Five model are significantly more
complex than the Atari games used in other experiments. Under these conditions,
it's more efficient to create a dedicated pool of GPU machines to run the
forward pass on the latest policy, rather than leaving this responsibility in
the actors. Coupled with the synchronous policy updates, this likely explains
why the paper doesn't make any mention of policy-lag corrections.

## Takeaways

From the architecture point of view, the main difference between state-based and
policy-based algorithms is that the former are generally off-policy, whereas the
latter usually need on-policy samples. Nevertheless, the most recent designs for
both families have converged on a similar design:

- Both Ape-X and IMPALA share experiences instead of gradients to reduce
  communication overhead. Q-Learning adapts naturally to this change, but
  Actor-Critics need correction factors.

- Likewise, distributed learning has fallen out of favor given the rapid
  increase in GPU capacity. In recent designs, each GPU learner can service
  hundreds of actor CPUs and reduce communication overhead.

If we take for granted that interactions with the environment (ie actors) need
to be parallelized (that's the whole point of this exercise in cloud madness),
the real choice is whether learning should be as well. The recent consensus
seems to be _no, GPUs are enough for learning_.

Probably the most enduring lesson from this review is to be careful about
recognizing where accidental complexity and implementation details begin to
obscure novel ideas. Case in point is A3C's supposedly better exploration
behavior due to asynchrony, which later turned out to be inferior to synchronous
methods in both performance and cost.
