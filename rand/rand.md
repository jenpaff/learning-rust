# rand

## Basics 

### What is randomness? 
- random data means a sequence of random bytes, where for each byte each of the 256 possible values are equally likely

### TRNG vs PRNG
- true random generators observe natural processes (e.g. atomic decay, thermal noise) which is considered truly random
- pseudo-random generators have the following properties: 
    - address the challenge we face in generating randomness by reliably producing articifial random bits from a few true random bits
    - how it works: 
        - regularly update entropy pool (memory buffer) which is the source of entropy
        - on update it mixes pool's bits together to remove statistical bias
        - runs a DRBG to achieve determinstic results
        - PRNG ensures that DRBG never receives the same input twice
    - functions
        - init: initialise entropy pool
        - refresh(R): update entropy pool using some data R sourced from RNG, often called re-seeding
        - next(N): returns N pseudorandom bits and updates entropy pool

Cryptographically-secure PRNG's: 
- infeasible to brute-force and guess the initial state
- infeasible to brute-force and predict the next output value
- seed value needs to be large enough (usually 256bits)
- protect from side-channel attacks (i.e. prevent from reading internal state)

### Entropy 
- measure of the amount of unknown information in some piece of data
- e.g. unbiased boolean or coin flip has 1 bit of entropy (if biased, less entropy)

### Simple example 

```
extern crate rand;
// import commonly used items from the prelude:
use rand::prelude::*;

fn main() {
    // We can use random() immediately. It can produce values of many common types:
    let x: u8 = random();
    println!("{}", x);

    if random() { // generates a boolean
        println!("Heads!");
    }

    // If we want to be a bit more explicit (and a little more efficient) we can
    // make a handle to the thread-local generator:
    let mut rng = thread_rng();
    if rng.gen() { // random bool
        let x: f64 = rng.gen(); // random number in range [0, 1)
        let y = rng.gen_range(-10.0..10.0);
        println!("x is: {}", x);
        println!("y is: {}", y);
    }

    println!("Dice roll: {}", rng.gen_range(1..=6));
    println!("Number from 0 to 9: {}", rng.gen_range(0..10));
    
    // Sometimes it's useful to use distributions directly:
    let distr = rand::distributions::Uniform::new_inclusive(1, 100);
    let mut nums = [0i32; 3];
    for x in &mut nums {
        *x = rng.sample(distr);
    }
    println!("Some numbers: {:?}", nums);
}
```

The rand libaray automatically initialises a secure, thread-local generator on demand. `thread_rng` returns a handle to a generator. To specify a seed for `thread_rng()` you need to specify an RNG then use a method like seed_from_u64 or from_seed.

Note that seed_from_u64 is not suitable for cryptographic uses since a single u64 cannot provide sufficient entropy to securely seed an RNG. All cryptographic RNGs accept a more appropriate seed via from_seed.

## PRNGs in Rust

### Basic PRNGs
i.e. non-cryptographic PRNGs
- goals: simplicity, quality, memory usuage, performance
- advantages: small state size, fast initialization
- list of examples: https://rust-random.github.io/book/guide-rngs.html
- used in games or scientific simulations

### Cryptographically secure PRNGs
- goal: security

| name | full name |  performance | initialization | memory | security (predictability) | forward secrecy |
|------|-----------|--------------|--------------|----------|----------------|-------------------------|
| [`StdRng`] | (unspecified) | 1.5 GB/s | fast | 136 bytes | widely trusted | no |
| [`ChaCha20Rng`] | ChaCha20 | 1.8 GB/s | fast | 136 bytes | [rigorously analysed](https://tools.ietf.org/html/rfc7539#section-1) | no |
| [`ChaCha8Rng`] | ChaCha8 | 2.2 GB/s | fast | 136 bytes | small security margin | no |
| [`Hc128Rng`] | HC-128 | 2.1 GB/s | slow | 4176 bytes | [recommended by eSTREAM](http://www.ecrypt.eu.org/stream/) | no |
| [`IsaacRng`] | ISAAC | 1.1 GB/s | slow | 2072 bytes | [unknown](https://burtleburtle.net/bob/rand/isaacafa.html) | unknown |
| [`Isaac64Rng`] | ISAAC-64 | 2.2 GB/s | slow | 4136 bytes| unknown | unknown |

### Decision parameters
- performance (mostly for PRNGs): initialisation time 
- quality 
- period: 
    - The period or cycle length of a PRNG is the number of values that can be generated after which it starts repeating the same random number stream. 
    - In general: we recommend a period of at least 2^128. (Alternatively, a PRNG with shorter period of at least 264 and support for multiple streams may be sufficient) 
    - Avoid reusing values: On today's hardware, a fast RNG with a cycle length of only 264 can be used sequentially for centuries before cycling. However, when multiple RNGs are used in parallel (each with a unique seed), there is a significant chance of overlap between the sequences generated
    - Collisions and the birthday paradox: For a generator with outputs of equal size to its state, it is recommended not to use more than √P outputs to avoid collisions
- security
    - predictability
        - backward secrecy (prediction resistance): given some previous output, can i predict the next output value?
            - updating the state should be irreversible
        - forward secrecy (backtracking resistance): in case state is revealed, it's infeasible to reconstruct previous states
            - entropy pool should be refreshed regularly
    - state and seeding
        - security of CSPRNG relies on the seed being with a secure random key, therefore use `getrandom` crate which interfaces the OS's secure random interface. `SeedableRng::from_entropy` is a wrapper around getrandom for convenience. Alternatively, using a user-space CSPRNG such as `ThreadRng` for seeding should be sufficient.

## Seeding

Using a fresh seed (direct from the OS):
```
extern crate rand;
extern crate rand_chacha;
use rand::prelude::*;
use rand_chacha::ChaCha20Rng;

fn main() {
    let mut rng = ChaCha20Rng::from_entropy();
    println!("{}", rng.gen_range(0..100));
}
```

## Parallel RNGs
- use `thread_rng` in each worker thread, this is seeded automatically on each thread where it is used
- use `thread_rng` (or another master RNG) to seed a custom RNG on each worker thread
- use a custom RNG per work unit, not per worker thread
- use a single master seed

## Error handling
- `thread_rng` seeds itself via OsRng on first use and periodically thereafter, thus can potentially fail, though unlikely. If initial seeding fails, a panic will result

Sources: 
- https://rust-random.github.io/book/
- Book: Serious Cryptography