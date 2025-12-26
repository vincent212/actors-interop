# actors-interop

FFI (Foreign Function Interface) interop layer between actors-cpp and actors-rust.

This allows C++ and Rust actors to communicate via direct function calls with C as the common ABI - faster than ZMQ for same-machine communication.

## Quick Start

```bash
# Generate code from message definitions
make generate

# Build C++ and Rust libraries
make all
```

## Key Design Principles

### Location Transparency

Actors don't know where other actors are. Messages go in `msg::` namespace (not `interop::`).
The sender shouldn't know if the recipient is in C++, Rust, or remote.

```cpp
// C++ actor - doesn't know it's crossing to Rust
publisher_.send(msg::Subscribe{"AAPL"});
```

```rust
// Rust actor - doesn't know it's crossing to C++
self.publisher.send(&Subscribe { topic: ... });
```

### send() vs fast_send()

| Method | Behavior | Use Case |
|--------|----------|----------|
| `send()` | Async - queues message, returns immediately | Fire-and-forget, high throughput |
| `fast_send()` | Sync - blocks until message is processed | Critical ordering, request-response |

---

# Developer Guide

## How to Add a New Interop Message

### Step 1: Define the Message in C Header

Edit `messages/interop_messages.h`:

```c
/* Choose a unique ID (1000+) */
INTEROP_MESSAGE(MyNewMessage, 1005)
typedef struct {
    int32_t id;
    double value;
    interop_string name;    /* use for strings */
    int32_t is_active;      /* use int32_t for bool */
    double prices[10];      /* fixed-size arrays supported */
} MyNewMessage;
```

**Rules:**
- Use `int32_t`, `int64_t`, `double` (not `int`, `long`, `float`)
- Use `int32_t` for booleans (1=true, 0=false)
- Use `interop_string` for strings (max 64 chars)
- Use fixed-size arrays: `type name[size]`
- Message IDs must be unique and >= 1000

### Step 2: Regenerate Code

```bash
cd ~/actors-interop
python3 codegen/generate.py messages/interop_messages.h generated
```

This generates:
- `generated/cpp/InteropMessages.hpp` - C++ message class with `to_c_struct()` / `from_c_struct()`
- `generated/rust/interop_messages.rs` - Rust struct with `to_c_struct()` / `from_c_struct()`
- Bridge function dispatch code

### Step 3: Rebuild

```bash
make
```

### Step 4: Use in Your Actor

**C++ actor sending to Rust:**
```cpp
#include "interop/RustActorIF.hpp"
#include "InteropMessages.hpp"

class MyActor : public actors::Actor {
    interop::RustActorIF rust_actor_{"rust_handler", "my_cpp_actor"};

    void some_method() {
        rust_actor_.send(msg::MyNewMessage{42, 3.14, "hello", true});
        // or synchronously:
        rust_actor_.fast_send(msg::MyNewMessage{42, 3.14, "hello", true});
    }
};
```

**Rust actor sending to C++:**
```rust
use actors_interop::{CppActorIF, MyNewMessage};

struct MyActor {
    cpp_actor: CppActorIF,
}

impl MyActor {
    fn some_method(&self) {
        self.cpp_actor.send(&MyNewMessage {
            id: 42,
            value: 3.14,
            name: "hello".to_string(),
            is_active: true,
        });
        // or synchronously:
        self.cpp_actor.fast_send(&msg);
    }
}
```

## How to Register an Actor for Cross-Language Calls

Actors must be registered so the other language can find them by name.

**Register a C++ actor (so Rust can call it):**
```cpp
#include "CppActorBridge.hpp"

// In your actor setup code:
MyActor* actor = new MyActor();
cpp_actor_register("my_actor", actor);

// Later, to unregister:
cpp_actor_unregister("my_actor");
```

**Register a Rust actor (so C++ can call it):**
```rust
use actors_interop::rust_actor_register_boxed;

let actor = Box::new(MyActor::new());
rust_actor_register_boxed("my_actor", actor);
```

## Pub/Sub Pattern

See `examples/` for complete pub/sub examples:

### C++ Subscribes to Rust Publisher

1. C++ sends `Subscribe` to Rust publisher
2. Rust stores subscriber name from `SenderContext`
3. Rust periodically sends `MarketUpdate` via `CppActorIF`
4. C++ receives updates in message handler

### Rust Subscribes to C++ Publisher

1. Rust sends `Subscribe` to C++ publisher
2. C++ stores subscriber name from message metadata
3. C++ periodically sends `MarketUpdate` via `rust_actor_send()`
4. Rust receives updates in `InteropActor::receive()`

## Type Mapping

| C Type | C++ Type | Rust Type |
|--------|----------|-----------|
| int32_t | int32_t | i32 |
| int64_t | int64_t | i64 |
| double | double | f64 |
| int32_t (bool) | bool | bool |
| interop_string | std::string | String |
| double[N] | std::array<double, N> | [f64; N] |
| char[N] | std::array<char, N> | [u8; N] |

## Quick Reference

| Task | Command/Code |
|------|--------------|
| Add message | Edit `messages/interop_messages.h` |
| Regenerate | `make generate` |
| Rebuild | `make` |
| C++ -> Rust async | `rust_actor_.send(msg)` |
| C++ -> Rust sync | `rust_actor_.fast_send(msg)` |
| Rust -> C++ async | `cpp_actor.send(&msg)` |
| Rust -> C++ sync | `cpp_actor.fast_send(&msg)` |
| Register C++ actor | `cpp_actor_register("name", actor_ptr)` |
| Register Rust actor | `rust_actor_register_boxed("name", boxed_actor)` |

## License

MIT License - see individual source files for copyright.
