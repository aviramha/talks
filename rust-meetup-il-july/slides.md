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

# Microdosing Rust <mdi-language-rust/> at Your Workplace

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
- Real life experience
- Native Extensions
    - PyO3
    - napi-rs
    - neon
- Closing notes
- Questions


---

# Why Rust?

<v-clicks>


- Safety 🔒
- Performance 💪
- Community 🌐
- Love 😍 🥰

</v-clicks>

<!--
Talk about how choosing a language is many times based on preference and not neccesarily objective decision.
Point the objective benefits of the language.
-->

---

# Challenges

- Team needs to learn the language
  - Rust has a steep learning curve
  - Functional programming isn't mainstream
- Continuing to deliver value
  - Business values > technical stack
  - Can't spend too much time on refactoring and breaking

---


# Let's RWIR*! 🚧

Microservices - I can re-write only one service at a time using Rust!

<v-click>
<img src="/simply.jpg" style="height: 60%; width: 50%; margin-left: auto; margin-right: auto;" class="center"/>
</v-click>

*Rewrite it in Rust
<!--
Talk about the assumption that microservices help you introduce Rust. It does help, but a good microservice architecture will have very similar infrastructure used across the microservices.
-->
---

# Microservices, no? 😌

Lets take the "average", bare minimum microservice requirements for production:
- Logging 🗒 - https://github.com/rust-lang/log/blob/master/rfcs/0296-structured-logging.md
- Tracing 🕵️‍♀️
- Metrics 📈

<!--
Those are the bare minimum for a real production service, and each one can be implemented very differently across frameworks:
 Python
 Rust
 Go
and some of the implementation are different, for example using structured logging.
-->

---


# Let's take it a step further 🤓

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
- PyO3 (Python) <mdi-language-python/> 🐍
- j4rs (Java) - <mdi-language-java/>  ☕️
- rutie - <mdi-language-ruby/> 

---

# PyO3
<!-- This is NOT a note because it precedes the content of the slide -->
Basic example from PyO3 project[^1]
```rs
use pyo3::prelude::*;

/// Formats the sum of two numbers as string.
#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

/// A Python module implemented in Rust. The name of this function must match
/// the `lib.name` setting in the `Cargo.toml`, else Python will not be able to
/// import the module.
#[pymodule]
fn string_sum(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)?;
    Ok(())
}
```

[^1]: https://github.com/PyO3/pyo3
<!--> 
Notice how the function content is Rusty and the only way we know it's interoparable from Python is the macro.
<!-->
---

# PyO3..

```
$ python
>>> import string_sum
>>> string_sum.sum_as_string(5, 20)
'25'
```


---

# PyO3 - Concurrency

```rs
fn search_sequential(contents: &str, needle: &str) -> usize {
    contents.lines().map(|line| count_line(line, needle)).sum()
}

#[pyfunction]
fn search_sequential_allow_threads(py: Python, contents: &str, needle: &str) -> usize {
    py.allow_threads(|| search_sequential(contents, needle))
}

```

---

# PyO3 - Build & Distribute

>Build and publish crates with pyo3, rust-cpython and cffi bindings as well as rust binaries as python packages.
  This project is meant as a zero configuration replacement for setuptools-rust and milksnake. It supports building wheels for python 3.5+ on windows, linux, mac and freebsd, can upload them to pypi and has basic pypy support."[^2]

```
maturin develop
maturin build
maturin publish
```

[^2]: [maturin](https://github.com/PyO3/maturin)
---

# PyO3 - Example library - orjson

>orjson is a fast, correct JSON library for Python. It benchmarks as the fastest Python library for JSON and is more correct than the standard json library or other third-party libraries. It serializes dataclass, datetime, numpy, and UUID instances natively.[^3]

#### twitter.json serialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            0.41 |                  2419.7 |                 1    |
| ujson      |                            1.8  |                   555.2 |                 4.36 |
| rapidjson  |                            1.26 |                   795   |                 3.05 |
| simplejson |                            2.27 |                   440.6 |                 5.5  |
| json       |                            1.83 |                   548.2 |                 4.42 |


[^3]: [orjson](https://github.com/ijl/orjson)

---

# PyO3 - orjson, deserialization


#### twitter.json deserialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            0.85 |                  1173   |                 1    |
| ujson      |                            1.88 |                   532.1 |                 2.2  |
| rapidjson  |                            2.7  |                   371   |                 3.16 |
| simplejson |                            2.16 |                   463.1 |                 2.53 |
| json       |                            2.33 |                   429.7 |                 2.73 |

--- 

# PyO3 - More crates

- [fastuuid](https://github.com/thedrow/fastuuid/) - UUID generation
- [ormsgpack](https://github.com/aviramha/ormsgpack) - msgpack serialization
- [cryptography](https://github.com/pyca/cryptography) - Python's most popular cryptography library (partially in Rust)

--- 

# PyO3 - Case Study @ BioCatch

<v-clicks>

- Start very small - rfernet
- Go a bit bigger - ormsgpack
- And even bigger - data processor/encoder/decoder

</v-clicks>

---

# NAPI!
>Node-API (formerly N-API) is an API for building native Addons.[^4]


```rust
use napi_derive::napi;

#[napi]
fn fibonacci(n: u32) -> u32 {
  match n {
    1 | 2 => 1,
    _ => fibonacci(n - 1) + fibonacci(n - 2),
  }
}
```

```js
import { fibonacci } from './index.js'

// output: 5
console.log(fibonacci(5))
```

[^4]: https://nodejs.org/api/n-api.html#node-api

---

# NAPI - Example libraries

- [Polars](https://github.com/pola-rs/polars)
- [next-swc - Next speedy web compiler](https://github.com/vercel/next.js/tree/canary/packages/next-swc)
- [ElectronMail](https://github.com/vladimiry/ElectronMail/blob/master/src/electron-main/database/serialization/compression-native/Cargo.toml) - created a compression library

---

# Neon
>Neon is a library and toolchain for embedding Rust in your Node.js apps and libraries.[^5]


[^5]: https://neon-bindings.com/


---
class: 'text-center'
layout: center
---

# Closing notes

- Happy to help - aviram@metalbear.co

---
class: 'text-center'
layout: center
---

# Questions