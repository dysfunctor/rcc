# Rust Circuit Compiler (RCC)

A proof-of-concept circuit building and witness generation library.

## Compilation Pipeline

The high-level flow is as follows:

```
                                           ┌─────────────────┐
                                       ┌──►│ circuit_config  │
                                       │   └─────────────────┘
                                       │
                                       │
┌──────────────┐ rustc   ┌───────────┐ │   ┌─────────────────┐  rustc   ┌──────────┐    ┌───────────┐
│  circuit.rs  ├─────────┤  binary   ├─┴──►│ runtime.rs      ├─────────►│ runtime  ├───►│ witnesses │
└──────────────┘         └───────────┘     └─────────────────┘          └─────▲────┘    └───────────┘
                                                                              │
                                                                              │
                                                                              │
                                                                     ┌────────┴────────┐
                                                                     │ input           │
                                                                     └─────────────────┘
```

To compile the demo circuit specified in `src/bin/circuit.rs` file, run

```
cargo --release --bin circuit
```

To compile and run witness generation binary (generated at `src/bin/circuit_runtime.rs`) file with input 999, run

```
cargo --release --bin circuit_runtime 999
```

## Circuit Component

A circuit component is a function that
- Takes input a mutable reference to a composer
  - The composer exposes interfaces for circuit building
- Takes arbitrary input, including data structures over `Wire`
- Gives arbitrary outputs, including data structures over `Wire`

A circuit component can call other circuit components. However, recursive calls
are not allowed.

## Example circuit

An example circuit is given in `src/bin/circuit.rs`.

```rust
const N: usize = 10;
const M: usize = 10;

#[component(e)]
fn mul_seq(e: &mut BaseComposer, a: Wire, b: Wire) -> Wire {
    let mut v = vec![e.mul(a, b)];
    for i in 0..M {
        v.push(e.mul(
                *v.get(i).unwrap(),
                *v.get(i).unwrap()
        ));
    }
    *v.get(M).unwrap()
}

#[component(e)]
fn gen(e: &mut BaseComposer, val: Wire) -> (Vec<Wire>, Vec<Wire>) {
    let (a, b): (Vec<Wire>, Vec<Wire>) = (0..N).map(|i| {
        (
            e.add_const(val, F::from(i as u32)),
            e.sub_const(val, F::from(i as u32))
        )
    }).unzip();

    (a, b)
}

#[component(e)]
pub fn my_circuit(e: &mut BaseComposer) {
    let val = e.new_wire();
    e.arg_read(val, 1);

    let (a, b) = gen(e, val);
    let c: Vec<Wire> = a.iter().zip(b.iter()).map(|(ai, bi)| {
        mul_seq(e, *ai, *bi)
    }).collect();
    let sum = e.sum(c);
    e.log(sum);
}
```

A functionally equivalent circuit in `circom` is given in `example_circom/example.circom`:

```circom
pragma circom 2.1.2;

template MulSeq(M) {
    signal input a;
    signal input b;

    signal c[M+1];

    signal output d;

    c[0] <== a * b;

    for (var i = 0; i < M; i++) {
        c[i+1] <== c[i] ** 2;
    }

    d <== c[M];
}

template Gen(N) {
    signal input val;
    signal output a[N];
    signal output b[N];

    for (var i = 0; i < N; i++) {
        a[i] <== val + i;
        b[i] <== val - i;
    }
}

template Main(N, M) {
    signal input val;
    signal a[N];
    signal b[N];

    (a, b) <== Gen(N)(val);

    signal c[N];
    signal output d;

    for (var i = 0; i < N; i++) {
        c[i] <== MulSeq(M)(a[i], b[i]);
    }

    var sum;
    for (var i = 0; i < N; i++) {
        sum += c[i];
    }

    d <== sum;
    log(d);
}

component main = Main(1000, 1000);
```
