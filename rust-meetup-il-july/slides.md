---
# try also 'default' to start simple
theme: academic
# background: backgrounds/julian-hochgesang-SiflIx5IlRI-unsplash.jpg
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  Talk about introducing Rust to the workplace.
# persist drawings in exports and build
drawings:
  persist: false
author: Aviram Hassan
authorUrl: https://github.com/aviramha/
hideInToc: ture
---

# Microdosing Rust <mdi-language-rust/> at your workplace

How to start using Rust at your workplace at ease 🦀

---
layout: center
class: text-center
---

# About Me 🤓 

Hey! Nice to meet you 👋🏽. I am Aviram Hassan (he/him).

I am CEO of [MetalBear](https://metalbear.co)

Author @ [ormsgpack](https://github.com/aviramha/ormsgpack)

Contributor @ [PyO3](pyo3.rs)

Past - Backend Team Leader @ Biocatch, Solution Architect @ SAM, IDF


---

# What are we going to talk about?

- Why would you want to use Rust
- Why is it hard to introduce Rust to your workplace
- Microservices?
- Native Extensions
    - PyO3
    - napi-rs
- Closing notes
- Questions


---

# Why Rust?

<v-clicks>

- Love 😍 🥰
- Safety 🔒
- Performance 💪
- Commmunity 🌐

</v-clicks>

<!--
Talk about how choosing a language is many times based on preference and not neccesarily objective decision.
Point the objective benefits of the language.
-->
---
layout: center
class: text-center
---

# Let's RWIR! 🚧

Microservices - I can re-write only one service at a time using Rust!

<v-click>
<img src="/simply.jpg" style="height: 60%; width: 50%; margin-left: auto; margin-right: auto;" class="center"/>
</v-click>

<!--
Talk about the assumption that microservices help you introduce Rust. It does help, but a good microservice architecture will have very similar infrastructure used across the microservices.
-->
---

# Microservices, no? 😌

Lets take the "average", bare minimum microservice requirements for production:
- Logging 🗒
- Tracing 🕵️‍♀️
- Monitoring 📈

<!--
Those are the bare minimum for a real production service, and each one can be implemented very differently across frameworks:
 Python
 Rust
 Go
and some of the implementation are different, for example using structured logging.
-->

---


# Let's sizzle it a bit 🔥

Those were **just bare minimum** requirements for production. Without actual logic!
Few examples of more dependencies:
- Database SDK (Redis, PostgreSQL, Cassandra, MongoDB, ..insert_more)
- Machine Learning (TensorFlow, PyTorch, Keras)
- Storage (S3, GCS, Azure Blob Storage)
- Managed services (Queues, emails, etc)

No stable Azure SDK!

<!--
You need to find out that your microservice depedencies exist in the Rust eco system and that those work properly.
Also, implementation might be different
-->

---

# Okay, so what else?

Rust has an amazing ecosystem of interoparbility libraries for integrating with other languages.

This lets you *stay* in your current primary language, while introducing Rust piece by piece.

Libraries:
- napi-rs (Node) <mdi-nodejs/>
- PyO3 (Python) 
- j4rs
- rutie

---
---