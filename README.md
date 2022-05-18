# PlonkUp 
![Build Status](https://github.com/dusk-network/plonkup/workflows/Continuous%20integration/badge.svg)
[![Repository](https://img.shields.io/badge/github-plonkup-blueviolet?logo=github)](https://github.com/dusk-network/plonkup)
[![Documentation](https://img.shields.io/badge/docs-plonkup-blue?logo=rust)](https://docs.rs/plonkup/)


_This is a pure Rust implementation of the PlonkUp proving system over BLS12-381. More details on this proving system can be found in this [paper](https://eprint.iacr.org/2022/086.pdf)._


This library contains a modularised implementation of KZG10 as the default
polynomial commitment scheme.

**DISCLAIMER**: This library is currently unstable and still needs to go through
an exhaustive security analysis. Use at your own risk.

## Usage

```rust
use dusk_plonk::prelude::*;
use rand_core::OsRng;

// Implement a circuit that checks:
// 1) a + b = c where C is a PI
// 2) a <= 2^6
// 3) b <= 2^5
// 4) a * b = d where D is a PI
// 5) JubJub::GENERATOR * e(JubJubScalar) = f where F is a Public Input
#[derive(Debug, Default)]
pub struct TestCircuit {
    a: BlsScalar,
    b: BlsScalar,
    c: BlsScalar,
    d: BlsScalar,
    e: JubJubScalar,
    f: JubJubAffine,
}

impl Circuit for TestCircuit {
    const CIRCUIT_ID: [u8; 32] = [0xff; 32];
    fn gadget(
        &mut self,
        composer: &mut TurboComposer,
    ) -> Result<(), Error> {
        let a = composer.append_witness(self.a);
        let b = composer.append_witness(self.b);

        // Make first constraint a + b = c
        let constraint = Constraint::new()
            .left(1)
            .right(1)
            .public(-self.c)
            .a(a)
            .b(b);

        composer.append_gate(constraint);

        // Check that a and b are in range
        composer.component_range(a, 1 << 6);
        composer.component_range(b, 1 << 5);

        // Make second constraint a * b = d
        let constraint = Constraint::new()
            .mult(1)
            .public(-self.d)
            .a(a)
            .b(b);

        composer.append_gate(constraint);

        let e = composer.append_witness(self.e);
        let scalar_mul_result = composer
            .component_mul_generator(e, dusk_jubjub::GENERATOR_EXTENDED);

        // Apply the constraint
        composer.assert_equal_public_point(scalar_mul_result, self.f);
        Ok(())
    }

    fn public_inputs(&self) -> Vec<PublicInputValue> {
        vec![self.c.into(), self.d.into(), self.f.into()]
    }

    fn padded_gates(&self) -> usize {
        1 << 11
    }
}

// Now let's use the Circuit we've just implemented!

let pp = PublicParameters::setup(1 << 12, &mut OsRng).unwrap();
// Initialize the circuit
let mut circuit = TestCircuit::default();
// Compile/preproces the circuit
let (pk, vd) = circuit.compile(&pp).unwrap();

// Prover POV
let proof = {
    let mut circuit = TestCircuit {
        a: BlsScalar::from(20u64),
        b: BlsScalar::from(5u64),
        c: BlsScalar::from(25u64),
        d: BlsScalar::from(100u64),
        e: JubJubScalar::from(2u64),
        f: JubJubAffine::from(
            dusk_jubjub::GENERATOR_EXTENDED * JubJubScalar::from(2u64),
        ),
    };
    circuit.prove(&pp, &pk, b"Test", &mut OsRng).unwrap()
};

// Verifier POV
let public_inputs: Vec<PublicInputValue> = vec![
    BlsScalar::from(25u64).into(),
    BlsScalar::from(100u64).into(),
    JubJubAffine::from(
        dusk_jubjub::GENERATOR_EXTENDED * JubJubScalar::from(2u64),
    )
    .into(),
];
TestCircuit::verify(
    &pp,
    &vd,
    &proof,
    &public_inputs,
    b"Test",
).unwrap();
```

## Performance

Benchmarks taken on `Apple M1`
For a circuit-size of `2^16` constraints/gates:

- Proving time: `17.392s`
- Verification time: `10.475ms`. **(This time will not vary depending on the circuit-size.)**

For more results, please run `cargo bench` to get a full report of benchmarks in respect of constraint numbers.

## Acknowledgements

- Reference implementation AztecProtocol/Barretenberg
- FFT Module and KZG10 Module were taken and modified from zexe/zcash and scipr-lab, respectively.

## Licensing

This code is licensed under Mozilla Public License Version 2.0 (MPL-2.0). Please see [LICENSE](https://github.com/dusk-network/plonkup/blob/main/LICENSE) for further info.

## About

Implementation designed by the [dusk](https://dusk.network) team.

## Contributing

- If you want to contribute to this repository/project please, check [CONTRIBUTING.md](https://github.com/dusk-network/plonkup/blob/main/CONTRIBUTING.md)
- If you want to report a bug or request a new feature addition, please open an issue on this repository.
