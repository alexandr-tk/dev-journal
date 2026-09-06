---
project: rquickshare
tags: [rust, nearby-share, networking, open-source]
status: PR Submitted - Awaiting Review
---

# Abstracting Transport Layer for Offline BLE Support

**The Result:** [PR \#420](https://github.com/Martichou/rquickshare/pull/420)

## 1. Workflow

- **Task:** Decouple the core I/O logic from `TcpStream` and update the protobuf definitions to support future offline Bluetooth (BLE) features.
- **Reasoning:**
    - The current application relies strictly on mDNS and TCP for local network sharing.
    - My university's network security blocks peer-to-peer connections, rendering the app unusable on campus.
    - To resolve this, I need to implement the offline BLE Quick Share functionality, but the existing codebase is hardcoded exclusively for TCP streams.

## 2. Context

As a daily Linux user, I rely heavily on `rquickshare` to transfer files to my Android device. Because the campus Wi-Fi blocks the necessary discovery protocols, I decided to build out the offline Bluetooth Low Energy (BLE) and Wi-Fi Direct pathways myself.

This is my first major Rust project. Naturally, my initial assumptions about memory management and stream handling required some adjustment. Before I could begin working with the Linux Bluetooth daemon (`bluer`), I realized the application's foundation was too rigidly tied to Wi-Fi. A major refactoring phase was required first to make the network stack transport-agnostic.

## 3. Plan for Solving

Currently, modules like `inbound`, `outbound`, and `manager` explicitly expect a `TcpStream`. Passing a different stream type (such as a Bluetooth socket) would immediately cause type mismatches. My plan for this architectural stage is:

1. **Protobuf Update:** Pull the latest `wire_format.proto` file from the upstream Google Nearby repository.
2. **Fix Compilation:** Resolve the resulting struct initialization errors by implementing explicit default fallbacks.
3. **Validation:** Write a round-trip unit test (encode -> decode) to ensure data integrity is maintained with the new fields.
4. **Logic Isolation:** Extract the read and write logic into generic `write_frame` and `read_frame` functions to remove the `TcpStream` dependency.
5. **Generic Wrapper:** Create a generic connection wrapper, implement it across the core modules, and verify functionality with unit tests.

## 4. Solving the Issue

### Steps 1 & 2: Updating Protobufs and Enforcing Initialization

First, I integrated the latest `.proto` file to ensure the project had the correct data structures for offline BLE visibility flags and endpoint data.

Running `cargo check` surfaced numerous missing parameter errors during struct initialization. Coming from other languages, my first instinct was to pass `None` or leave the new fields uninitialized. However, Rust’s strict type system requires exhaustive initialization. I went through the codebase and implemented explicit defaults, ensuring the new protobuf fields fell back to zeroes or nulls so the existing Wi-Fi logic remained intact.

### Step 3: The Round-Trip Test

To ensure these fallback values wouldn't inadvertently corrupt the payload during a real transfer, I implemented a validation test.

The logic was straightforward: instantiate a dummy Quick Share event, encode it into a byte buffer using the new protobuf definitions, and decode it back to an event. Asserting that the original and decoded events were identical confirmed that the serialization logic remained sound.

### Step 4: Isolating Read and Write Logic

Next, I needed to untangle the core read/write operations from their TCP-specific methods and isolate them into standalone, generic functions. This phase exposed me to several core Rust paradigms:

**Error 1: Sized Traits at Compile Time**
My initial approach was to pass the trait directly (e.g., `stream: AsyncWrite`). The compiler flagged that the size of `dyn AsyncWrite` cannot be known at compile time. [Trait objects](https://doc.rust-lang.org/reference/types/trait-object.html) need indirection, such as a reference or `Box<dyn AsyncWrite>`; boxing is not the only option. I chose static dispatch through generics.

**Error 2: Move Semantics and Borrowing**
When attempting to pass the generic stream by value, I encountered move semantics errors. The stream was being consumed by the `write_frame` function, preventing the caller from using it to read the subsequent response. I corrected this by adjusting the signature to accept a mutable reference (`&mut W`).

**Error 3: The `Unpin` Requirement**
With the references corrected, the async runtime enforced the `Unpin` trait bound. The async read/write helpers used here require `Unpin` when borrowing the stream through `&mut R` or `&mut W`. [`Unpin`](https://doc.rust-lang.org/std/marker/trait.Unpin.html) means the type does not rely on remaining pinned in memory; it is not a guarantee that the value will stay at one address. Adding the bound resolved the compiler constraints.

The final isolated logic successfully abstracted the stream:

```rust
pub async fn write_frame<W: AsyncWrite + Unpin>(writer: &mut W, frame: &[u8]) -> Result<()> {
    // Logic to write the frame length header, followed by the frame bytes
}

pub async fn read_frame<R: AsyncRead + Unpin>(reader: &mut R) -> Result<Vec<u8>> {
    // Logic to read the exact length of the incoming bytes, then decode them
}
```

### Step 5: Creating the Generic Wrapper

With the functions genericized, I needed to replace the hardcoded `TcpStream` instances throughout the codebase. I created a generic wrapper struct for streams satisfying the listed trait bounds:

```rust
pub struct Connection<S>
where
    S: AsyncRead + AsyncWrite + Unpin + Send + 'static
{
    stream: S,
}
```

This ensured that whenever the application initializes a connection, it wraps the stream in this generic interface. I refactored the `inbound`, `outbound`, and `manager` files to utilize this new wrapper.

## 5. Testing the Architecture

To confirm the newly abstracted architecture functioned correctly, I wrote a unit test specifically for the stream wrapper. Instead of opening a real network socket, I simulated a read/write cycle using an in-memory byte buffer.

The in-memory round-trip test passed for `write_frame` and `read_frame`, showing that those helpers can work without a TCP socket. It did not test a Bluetooth transport or an end-to-end offline transfer.

## 6. Review & Next Steps

After verifying that `cargo test` and `cargo check` passed cleanly, I pushed the branch and submitted the PR. Next step: generating and broadcasting the 17-byte Quick Share BLE payload via the Linux `bluer` stack.
