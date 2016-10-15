---
layout: post
title: Defensive RNG design 
is-note: true 
permalink: /blog/:year-:month-:day-:title
published: true
---
<p>The design goals of a strong system CSPRNG is to ensure that it will securely provide numbers that are computationally indistinguishable from random.</p>

<p>The purpose of this is to allow the secure use of random numbers in cryptography and other security critical contexts. Thus it must be infeasible for an adversary to extract enough information about the private entropy pool to predict its outputs, and it must likewise be infeasible to influence its outputs.</p>

<p>Given strong cryptographic mixing algorithms and a sufficient amount of secret entropy, it is computationally infeasible for any adversary to predict either the output or determine the seed (entropy pool contents). The big crypto plumbing problems around CSPRNGs are mostly about entropy collection and estimation, not what algorithms to use.</p>

<p>If the entropy pool has too little unpredictable entropy, an adversary can bruteforce its contents after seeing the outputs. This might require access to multiple outputs if the pool is large in size, to extract information about the entire pool. If the adversary also controls an entropy source used by the CSPRNG, whose output only is mixed with low entropy sources and/or with the main pool when it is also low on (unknown) entropy, then the adversary can even control the output of the CSPRNG.</p>

<p>How to make a defensive RNG:</p>

<p>To first get our entropy, we run a number of specialized entropy collection programs (some of which may run as services). They all have unique ID:s, and some are trusted (such as the OS kernel module). They estimate the entropy of their own sources (which should all be independent from each other), they attempt to extract entropic data from them, and then serve it to the CSPRNG input buffers. The CSPRNG service records the entropy estimates, their sources, timing and other data.</p> 

<p>When the individual buffers' estimated entropy reach a threshold value (128 bits by default, can be unique per service), the CSPRNG considers them "ready" to extract entropy from. This is to ratchet up the internal entropy enough so the inputs can't be bruteforced if an adversary polled the CSPRNG before and after seeding, when the initial entropy was low.</p>

<p>When a number of untrusted entropy collectors all have reached their own thresholds, they get mixed together into an untrusted entropy pool....??? 
The same thing is done for the trusted collectors, except that for the trusted pool the CSPRNG will also attempt to estimate the cumulative entropy.</p>

<p>Mixing and pools... 
Ring pools, trusted and untrusted, tampering resistance... Metadata included in entropy mixing to prevent collisions... Parallelism... 
Entropy estimates, trusted and untrusted... </p>

<p>The "final" output from the pools after entropy extraction is a CSPRNG root seed (minimum of 256 bits, maximum of 1024), for use by generators. This root seed may be updated occasionally from the entropy pools through a mixing algorithm (entropy extractor).</p>

<p>It is however never used directly - whenever a program needs entropy, then a new generator is instantiated for it by the CSPRNG, and a generator seed is derived for that generator from the CSPRNG root seed. Every request gets a unique ID based on the process asking, time and number of received requests, etc, mixed in like a salt.</p>

<p>A possible generator seed derivation method is to encrypt the root seed using an HMAC of the root seed and of a generator instance counter + metadata of the requestee.</p>

<p>Generators must allow halting (on for example VM cloning) to have their seeds refreshed before continuing. If the generator algorithms (such as a stream cipher algorithm) has a suggested maximum output size before rekeying becomes necessary, that limit shall be respected and the generator must be halted and refreshed / rekeyed before continuing. For smooth performance, a replacement generator may be instantiated silently in advance by the CSPRNG service to replace it automatically. </p>

<p>For when there is plenty of one-off requests from different processes, there could be a special generator designed to handle that usecase more efficiently (instead of instantiating a new streaming generator). This generator will handle its own threading and seed refreshing, and will also refresh when prompted by the CSPRNG service. </p>