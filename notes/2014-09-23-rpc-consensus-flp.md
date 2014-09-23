# Remote procedure calls (RPC)

* You want a procedure call to run on a remote machine
* You have a server that has an API, and locally on a client you have a stub routine that look like
  the same API but instead send messages over the network to the server, waits for a message, and
  receives it.
* RPCs differentiate distributed systems from "normal" (read: local) ones

## high-level process is more complicated

* normally a method like `bool add_user` would just run some local routine and return a result
* Now, you must...
  * set up connections
  * bind to server
  * think about auth
  * etc...
* but specifically, instead of just a true/false return value, you now need
  * true
  * false
  * "I don't know!"
    * this can occur for so many reasons (network, remote machine failure, etc) and you don't have
      reliable ways to know

### A solution: At most once semantics

This is a pretty tricky issue.

* network failures
  * request lost: retransmit request on failure
  * if response lost or if server takes longer than timeout to respond: don't retransmit unless
    idempotent.
    * dealing with repeated requests: **replay cache**. Have I seen the ID of this request before?
      don't reexecute the procedure
      * But do you do this in memory? what if the server crashes and reboots? now the replay cache
        is erroneous
        * perhaps assume that if the server crashes then timeout all the users, but not always
          viable.
        * keep it on disk? now it's expensive to keep a replay cache **Cookies** can help: server
        * gives client a cookie which has the time of boot, client
          includes cookie with RPC, and if the server crashes and reboots, boot times don't match
          and reject the request
      * RPC needs to have a unique call ID to do the above, to avoid replaying methods that user
        might actually want to do twice
  * transmission errors are really hard to deal with - what if ID is mangled to something else?
  * probablistic stuff is the best we can do - checksum requests

## Parameter passing

Tricky with RPCs

* System details
  * data representation: big/little endian, data type size
* no shared memory
  * can't pass pointers, messes with garbage collections
* how to pass unions? what fields are active at any point in time? need *discriminated unions*
  (later)

## Interface definition languages

* Specify call, argument, return values, types, and RPC system figures out how to send data back
  and forth
* Output:
  * **marshal** (serialize) a native data structure into something that can be sent over the wire
  * stub routines locally

## Existent C++ RPC systems

* XML or JSON over HTTP (not designed for the purpose)
* Cereal - C++11 structure serilaizer
  * not a complete RPC system but good at serializing C++ data structures
* Google protobufs, Apache Thrift (same boat)
  * pros
    * Very compact encoding - small ints are smaller than big ints
    * defensively coded - can deal with malicious input
    * compatible with evolving message formats
  * cons
    * not a full RPC library. doesn't generate local stub code, etc
    * Complex encoding - need to deal with differently sized ints, etc.
    * not C++11. Misses existing structures like unions without opening up big cans of worms
* Apache Avro - self-describing messages that contain a schema.
  * not great for small messages
* *Hot trend*: Cap'n Proto, Google FlatBuffers
  * store things in pre-marshalled formats in memory, so it's very fast
  * cons
    * less mature
    * nondeterministic wire format (dangerous for crypto purposes)
    * complex to allocate memory for native stuff on the fly whenever you need to access data
* XDR - used by internet standards such as NFS
  * dirt simple, very short (1 page of ASCII text standard)
  * has all the features you need
  * cons
    * big endian (most machines are little endian)
    * binary, but rounds everything to multiples of 4 bytes, so it can be wasteful

## Case study: XDR encoding

* C-like, but you can have things like strings, variable-length arrays, larger integers (hyper =
  64-bit int), just a bunch of bytes (opaque type), optional data (represented as pointer notation,
  but they're not pointers - you don't have shared memory)
* boolean (bool) is 4 bytes, for instance - not compact
* variable length data types start with a 4-byte length, then the data, then padding for the rest
  of the bytes to be multiple of 4 bytes
* **union types**: the one type that really doesn't look like C

```
union type switch (simple_type which) {
  case value_A:
    type_A varA;
  case value_B:
    type_B varB;
  default:
    void;
};
```

# Consensus in asynchronous systems

A mind-bending topic. One of the simplest functions you can write: identity function.

* This is much harder on a distributed, asynchronous system.
* **Asynchronous system** - not like asynchronous IO
  * set of agents exchanging messages
  * no bound on message delays
  * no bound of relative execution and speed of agents
  * for convenience, internalevents like timeouts are special messages, so the network controls all
    timing
* Can't distinguish failed agent from a slow network. This introduces complexity
* **The consensus problem**
  * distributed version of the identity function
  * problem description in laymans terms
    * You give a set of agents an input value
    * goal is to select and output one of these values
    * they will exchange a bunch of messages
    * and at one point they'll select one, and all agents output it
    * once an agent outputs a value, it can't change its mind.
    * how do establish consensus and consistency about what value gets outputted?
  * properties
    * safety
      * agreement - all outputs producd the same value
      * validity - the output value is one of the agents inputs
    * liveness
      * termination - eventually non-failed agents output a value
    * fault-tolerance
      * if an agent fails at any point, you can survive

## FLP impossibility result

This result drives many problems of systems that require consensus

* **FLP impossibility result**: no deterministic asynchronous system can have all three of these.
  :( But why?
  * **Bivalent states** - network can affect which value agents choose in a consensus protocol
    * state of the above system where you might output more than one value
      * for example, agent #2's messages are extremely slow. The other agents think it has
        failed, but it wakes up! Doesn't know what to do now since the other agents terminated.
  * **Univalent states** - network cannot affect choice - only one output remains possible
  * **i-valent states** - univalent state with output value i
  * Any state must be univalent, bivalent, or non-terminating (stuck)
  * Nodes can produce outputs only in a univalent state
  * If execution starts in a bivalent states but eventually terminates, it must have transitioned
    into a univalent state at some point.
  * Proof overview
    * The transition to a univalent state occurs due to some message m that seals the fate of
      what the output must be
    * There must be a deciding message
    * But if the network delays a deciding message, other messages get delivered before m, but
      now m is no longer a deciding message (it's been neutralized)
    * Process:
      1. There are bivalent starting configurations
      2. Network can neutralize any deciding message
      3. So, it can be bivalent forever if it always neturalizes the deciding message

* Some intuition
  * If it's a fault tolerant system, you have to be able to output without talking to agents that
    may have gone down. So, in that case, skipping an agent produces a bivalent starting
    configuration, since the network failing can affect the set of possible outputs
  * Lost a bit during the proof description lol

* Coping with the FLP result
  * Several systems guarantee fault-tolerance and safety, but just tend to terminate in practice (but
    don't guarantee it)
    * sometimes liveness is more important, though - for instance, avoiding lag in a first person
      shooter videogame, or guaranteeing fast responses in a distributed system like Amazon AWS
  * Others may make agents non-deterministic
