
# Diffie-Hellman Key Exchange

A script that demonstrates how a [Diffie-Hellmman Key Exchange](https://ee.stanford.edu/~hellman/publications/24.pdf) works. We will be showing an example between a Diffie-Hellman key exchange between the users Mark and Janet.

First, we select two random large primes. Typically, these are several hundred digits, but for the sake of readability, we'll use easy to follow examples. These primes are publicly transmitted.


```python
alpha = 31
q = 51
```

Next, we define Mark and Janet's private keys. These values are never transmitted.


```python
mark_private = 24
janet_private = 39
```

Next, we calculate the public keys from private keys. Mark transmits his public key to Janet and Janet transmits hers to Mark.


```python
def generate_public_key(alpha, q, private_key):
    return (alpha ** private_key) % q
```


```python
%%time
mark_public = generate_public_key(alpha, q, mark_private)
janet_public = generate_public_key(alpha, q, janet_private)
```

    CPU times: user 7 µs, sys: 1 µs, total: 8 µs
    Wall time: 12.2 µs


A quick observation of their public keys shows:


```python
print("Mark's Public Key:", mark_public)
print("Janet's Public Key:", janet_public)
```

    Mark's Public Key: 16
    Janet's Public Key: 40


Last, both Mark and Janet calculate the secret key for their communication. These keys are never transmitted.


```python
def generate_communication_key(public_key, private_key, q):
    return (public_key ** private_key) % q
```


```python
%%time
mark_communication = generate_communication_key(janet_public, mark_private, q)
janet_communication = generate_communication_key(mark_public, janet_private, q)
```

    CPU times: user 8 µs, sys: 1 µs, total: 9 µs
    Wall time: 11.4 µs


A quick observation of their communication keys shows they're the same values.


```python
print("Mark's Communication Key:", mark_communication)
print("Janet's Communication Key:", janet_communication)
```

    Mark's Communication Key: 16
    Janet's Communication Key: 16


No matter what `p`, `alpha` and private keys you select, the communication keys will always be the same value. Mark and Janet can then use any cryptography communication they choose without the problem of securely sharing a key. Also note how fast and cheap it was for a computer to calculate these values.

## Cracking Diffie-Hellman

Suppose an outside user - Jim - wanted to intercept the messages between Mark and Janet. Mark and Janet have publicly shared the variables: `alpha`, `q`, `mark_public` and `janet_public` to produce a secret communication key.

If Jim wishes to find this secret communication key, hey would need to calculate at least one private key given the two public keys. He do this by taking the log (base `alpha`) of either public key. However, because the public keys are modular exponentials, he must add a constant `d * q` where `d` could be any integer.


```python
def naive_solution(value, min=-1):
    n = min + 1
    while n < q:
        n += 1
        if ((alpha ** n) % q) == value:
            return n
```

There exist more efficent ways to calculate discrete logs, namely the baby step giant step algorithm.


```python
from math import ceil, sqrt

def baby_step_giant_step(value, min=-1):
    order = ceil(sqrt(q - 1))  # phi(q) is q-1 if q is prime

    # Baby step.
    baby_steps = {pow(alpha, i, q): i for i in range(order)}

    # Precompute via Fermat's Little Theorem
    c = pow(alpha, order * (q - 2), q)

    # Search for an equivalence in the table. Giant step.
    for j in range(order):
        gamma = (value * pow(c, j, q)) % q
        if gamma in baby_steps:
            candidate = j * order + baby_steps[gamma]
            if candidate > min:
                return j * order + baby_steps[gamma]
```


```python
def get_communication_key_from_public_keys(public_key1, public_key2, alpha, q, actual_communication_key, discrete_log):
    """
    We pass in the actual communication key to "test" if we have found the communication key
    from the public keys. In the real world, the only way you'd test if you found
    the real communication key is if you attempt to decrypt encrypted communications 
    using your communication key candidate, which would increase the time to crack a key.
    """
    d = 0
    communication_key_candidate1 = communication_key_candidate2 = None
    private_key_candidate1 = private_key_candidate2 = -1
    
    while not (
        communication_key_candidate1 == actual_communication_key or 
        communication_key_candidate2 == actual_communication_key
    ):
        private_key_candidate1 = discrete_log(public_key2, private_key_candidate1)
        private_key_candidate2 = discrete_log(public_key1, private_key_candidate2)
        d += private_key_candidate1 + private_key_candidate2
        communication_key_candidate1 = public_key1 ** private_key_candidate1 % q
        communication_key_candidate2 = public_key2 ** private_key_candidate2 % q
        
    return d
```


```python
%%time
minimum_tries = get_communication_key_from_public_keys(mark_public, janet_public, alpha, q, mark_communication,
                                                       discrete_log=naive_solution)
```

    CPU times: user 17 µs, sys: 3 µs, total: 20 µs
    Wall time: 23.6 µs



```python
print("Jim Discrete Logs Calculated:", minimum_tries)
```

    Jim Discrete Logs Calculated: 15



```python
%%time
minimum_tries = get_communication_key_from_public_keys(mark_public, janet_public, alpha, q, mark_communication,
                                                       discrete_log=baby_step_giant_step)
```

    CPU times: user 39 µs, sys: 6 µs, total: 45 µs
    Wall time: 49.4 µs



```python
print("Jim Discrete Logs Calculated:", minimum_tries)
```

    Jim Discrete Logs Calculated: 15


Note that Jim had to try 108 different values of `d` before he was able to crack the communication key between Mark and Janet. Also note it took him a relatively long time to crack it. This becomes especially true with larger values of `p`, `alpha` and private keys. Take for instance the following:


```python
alpha = 6577
q = 32416190071

mark_private = 23565
janet_private = 17629
```


```python
%%time
mark_public = generate_public_key(alpha, q, mark_private)
janet_public = generate_public_key(alpha, q, janet_private)
```

    CPU times: user 13.6 ms, sys: 4.41 ms, total: 18 ms
    Wall time: 17.8 ms



```python
print("Mark's Public Key:", mark_public)
print("Janet's Public Key:", janet_public)
```

    Mark's Public Key: 18827759766
    Janet's Public Key: 16109188564



```python
%%time
mark_communication = generate_communication_key(janet_public, mark_private, q)
janet_communication = generate_communication_key(mark_public, janet_private, q)
```

    CPU times: user 51.1 ms, sys: 0 ns, total: 51.1 ms
    Wall time: 50.9 ms



```python
print("Mark's Communication Key:", mark_communication)
print("Janet's Communication Key:", janet_communication)
```

    Mark's Communication Key: 3614890724
    Janet's Communication Key: 3614890724



```python
%%time
get_communication_key_from_public_keys(mark_public, janet_public, alpha, q, mark_communication,
                                                       discrete_log=naive_solution)
```

    CPU times: user 51.8 s, sys: 0 ns, total: 51.8 s
    Wall time: 51.8 s
    41194

```python
%%time
get_communication_key_from_public_keys(mark_public, janet_public, alpha, q, mark_communication,
                                                       discrete_log=baby_step_giant_step)
```

    CPU times: user 764 ms, sys: 19.8 ms, total: 784 ms
    Wall time: 783 ms
    41194


Clearly, Jim is going to have some trouble trying to crack the communication key in a timely manner for very large `q`.

You can read more about the Diffie-Hellman Key Exchange in my Fall 2018 paper: ["A Brief Analysis and Proof of Public-Private Key Cryptography Systems"](#).
