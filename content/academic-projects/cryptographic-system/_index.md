---
title: "Cryptographic System"
weight: 2
---
# Cryptographic System
As part of my second-year cryptography module, I created an emulated stock
investment platform:

{{<button href="https://github.com/OscarTopliss/iss-coursework">}}
View Repository
{{</button>}}

The coursework focused on **system design**, in particular the selection,
justification and implementation of **cryptographic mechanisms**. The project
was built with a **server-client** model, using two separate programs.

## System Design

---
### Considerations
I designed the system with a few key things in mind:
- I wanted the program to be able to handle **multiple user sessions
concurrently**.
- I needed strong **encryption at rest** and **in transit** to ensure the
**Confidentiality**, **Integrity** and **Non-Repudiation** of data.

---
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
the algorithm to hash a password with Argon2id requires either a
lot of memory, a lot of time, or both. In my program's case, the parameters
used to encrypt the password require 2GiB of memory, and encryption takes
approximately 0.5 seconds.
- The time and memory requirements mean that trying to find a password via
**brute force**, e.g. hashing every common password in a dictionary (known as a
"dictionary attack"), is infeasible.
  - Each individual hashing attempt takes too long.
  - The algorithm's high memory requirements mean that **parallelising** the
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

---
## Program Architecture
### Client/server Architecture
As a security measure, I aimed to make the client-side program as simple as 
possible. I built it to essentially establish a TLS-wrapped socket connection
to the server, send the user's responses to the server, and display any 
resulting messages from the server in a basic CLI.

### Multi-Processing and Database Architecture
My architecture for user sessions was somewhat complicated, but incredibly
stable:
- User sessions were **continuous** - instead of storing information about the
user session on the client side, I opted for user sessions to use a continuous 
socket connection.
- When a client establishes a socket connection, the server created a new session
handler object and begins its execution on one of five worker processes in a 
pool. This meant that the program could handle up to 5 simultaneous user 
connections.
- As the program stores data with a database, simultaneous user connections
presented a potential issue with **simultaneous database accesses**. To address
this, I assigned one worker process to host a database process, and gave the
session handler processes access to a **thread safe queue**. They would then add
database request objects to this queue, which would be executed in-order by the
database process.
  - Each database request object would contain a `Connection` object (from the 
Python `multiprocessing` library), which acts essentially as a socket between
processes. This enables the database object to return the result of a database
query to the relevant session handler object.

Below: The server process's startup logic, including initialising database
and session handler processes.
```python
with context.wrap_socket(sock, server_side=True) as ssock:
                ...
                process_num = 5
                database_queue = Queue()
                socket_worker_pool = [
                    Process(
                        target = ClientSession.handle_sessions,
                        args = (ssock, database_queue,)
                    )
                    for x in range(process_num)
                ]
                database_worker = Process(
                    target = Database.start_database,
                    args = (database_queue,)
                )
                database_worker.start()

                for worker in socket_worker_pool:
                    worker.daemon = True
                    worker.start()
                
                print("Listener started.")

                while True:
                    sleep(10)
```

### State Machine Sessions
- The user sessions themeselves were implemented using a form of **state 
machine**. The state of the user session was stored in a variable called
`sessionState`, which referenced an enum class called `SessionState`.
- The state could be updated by handling valid user input, and would otherwise
remain unchanged, leaving the user where they were in the menu if valid input
wasn't provided.
Below: the `SessionState` enum class from `server.py`
```python
class SessionState(Enum):
        START_MENU = 1
        LOGIN_MENU_USERNAME = 2
        LOGIN_MENU_PASSWORD = 3
        CREATE_NEW_USER_USERNAME = 4
        CREATE_NEW_USER_PASSWORD = 5
        CREATE_USER_SUCCESSFUL = 6
        CLIENT_MENU = 7
        LOGIN_SUCCESSFUL = 8
        ADVISOR_MENU = 9
```

---
## Conclusions/Lessons Learnt
- I love cryptography, and being able to implement so many interesting
cryptographic systems in this project was brilliant fun.
- This was the first time I implemented user sessions with a state machine,
which I found was easier to handle than traditional procedural logic, but felt
overly verbose at times.
- I'm now a huge fan of the `cryptography` library, I loved working with it and
it has sensible defaults, alongside rather good documentation with really
useful code examples.

After completing this project, I feel much more confident with:
- Socket-based programming.
- Multiprocessing, and multi-process system design.
- Cryptographic system design and implementation.
- OOP in Python