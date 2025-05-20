# Cryptographic System
As part of my second-year cryptography module, I created an emulated stock
investment platform:

{{<button href="https://github.com/OscarTopliss/iss-coursework">}}
View Repository
{{</button>}}

The coursework focused on **system design**, in particular the selection,
justification and implementation of **cryptographic mechanisms**. The project
was built with a \textbf{server-client} model, using two seperate programs.

## System Design
### Considerations
I designed the system with a few key things in mind:
- I wanted the program to be able to handle **multiple user sessions
concurrently**.
- I needed strong **encryption at rest** and **in transit** to ensure the
**Confidentiality**, **Integrity** and **Non-Repudiation** of data.
### Encryption
#### In Transit
- I implemented **TLS 1.3** using OpenSSL and Python's built-in `ssl` library for
**encryption in transit**.
#### At Rest
- Any sensitive data was encrypted using **AES-256-GCM** before being stored
in the program's database. Encryption with GCM creates a **Message
Authentication Code** or **MAC** which can be used to verify the integrity of
the data, which allowed me to ensure that data hadn't been tampered with
since it was originally encrypted.
#### Passwords
- Paswords were verified using **Argon2id**, a hashing/password-based key
derivation function.
- I implemented Argon2id with the default safe parameters suggested in its
RFC (see [here](https://www.rfc-editor.org/rfc/rfc9106.pdf), page 12).
My password verification system had a number of security features:
##### Dictionary Attack Mitigations
- Argon2id is both **memory-hard** and **time-hard**. This means that running
the algorithm to hash a password with Argon2id requires requires either a
lot of memory, a lot of time, or both. In my program's case, the parameters
used to encrypt the password require 2GiB of memory, and encryption takes
approximately 0.5 seconds.
- The time and memory requirements mean that trying to find a password via
**brute force**, e.g. hashing every common password in a dictionary (known as a
"dictionary attack"), is infeasible.
  - Each individual hashing attempt takes too long.
  - The algorithm's high memory requirements mean that **paralelising** the
  process, e.g. running multiple instances to check more hashes at once, is
  expensive and difficult to scale.
##### Secret Salt
- To further increase password security, I made use of a secret salt.
- After the user entered their password, a **salt** would be randomly generated.
And combined with the plaintext password.
- Afterwards, a **secret salt** (or "pepper") would be taken from the program's
"HSM" (in this case, just a JSON file representing a real Hardware Security
Module) and combined with the password.
- The password is then hashed and the resulting bytes, alongside the public
salt, can be stored in the database.
- The use of any salt negates the risk of **rainbow-table attacks**, as
attacker's can't simply compare the hash to a set of pre-computed hash values.
- The use of a private salt further reduces the risk of **dictionary attacks**,
as without access to the private salt an attacker cannot verify that a password
matches a stolen hash.

##### Code Example
Excerpt from the `Database` class in [server-database.py](https://github.com/OscarTopliss/iss-coursework/blob/main/server_database.py).
Taken from the `validate_user_credentials` function responsible for checking
a user's password:
```Python
kdf = Argon2id(
                salt=salt,
                length=32,
                iterations=1,
                lanes=4,
                memory_cost=2 * 1024 * 1024, # 2 Gib
                ad=None,
                secret=pepper,
            )

            try:
                kdf.verify(
                    key_material = password.encode(),
                    expected_key = password_hash
                )
            except InvalidKey:
                return False
            else:
                return True
```
