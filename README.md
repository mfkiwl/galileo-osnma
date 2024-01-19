# galileo-osnma

[![Crates.io][crates-badge]][crates-url]
[![Rust](https://github.com/daniestevez/galileo-osnma/actions/workflows/rust.yml/badge.svg)](https://github.com/daniestevez/galileo-osnma/actions/workflows/rust.yml)

[crates-badge]: https://img.shields.io/crates/v/galileo-osnma.svg
[crates-url]: https://crates.io/crates/galileo-osnma

galileo-osnma is a Rust implementation of the Galileo OSNMA (Open Service
Navigation Message Authentication) protocol. This protocol is used by the
Galileo GNSS to sign cryptographically the navigation message data transmitted
by its satellites, in order to prevent spoofing. Briefly speaking, galileo-osnma
can process the navigation message data and OSNMA cryptographic data and check
all the cryptographic signatures against an ECDSA public key and/or Merkle tree,
in order to check the authenticity of the navigation data.

galileo-osnma does not require the Rust Standard library (it can be built with
`no_std`), allocates all its data statically on the stack, and has a relatively
small memory footprint for the data (~76 KiB if Slow MAC is used and data for 36
satellites in parallel is stored, and ~8.5 KiB if Slow MAC is not used and data
for only 12 satellites in parallel is stored). This makes it possible to use the
library in some embedded microcontrollers. A demo of galileo-osnma running in a
[Longan nano](https://longan.sipeed.com/en/) GD32VF103 board is provided in the
[osnma-longan-nano](https://github.com/daniestevez/galileo-osnma/tree/main/osnma-longan-nano)
crate. This is a RISC-V microcontroller with 128 KiB of flash and 32 KiB of RAM
that is similar to the popular STM32F103 ARM Cortex-M3 microcontroller.

## Documentation

The documentation for galileo-osnma is hosted in
[docs.rs](https://docs.rs/galileo-osnma/).

The following reference documents from the Galileo system are relevant:

* [Galileo OS SIS ICD v2.1](https://www.gsc-europa.eu/sites/default/files/sites/all/files/Galileo_OS_SIS_ICD_v2.1.pdf)

* [Galileo OSNMA SIS ICD v1.1](https://www.gsc-europa.eu/sites/default/files/sites/all/files/Galileo_OSNMA_SIS_ICD_v1.1.pdf).

* [Galileo OSNMA Receiver Guidelines v1.1](https://www.gsc-europa.eu/sites/default/files/sites/all/files/Galileo_OSNMA_Receiver_Guidelines_v1.1.pdf)

## Quick start using Galmon

galileo-osnma comes with a binary application that can read Galileo INAV pages
using the [Galmon](https://github.com/berthubert/galmon) [transport
protocol](https://github.com/berthubert/galmon#internals). This is located in
the `galmon-osnma` folder.

A quick way to see this working is to use the Galmon Galileo navigation data
feed, which streams from 86.82.68.237, TCP port 10000. From the `galmon-osnma`
folder, we can run
```
nc 86.82.68.237 10000 | \
    RUST_LOG=info cargo run --release -- --pubkey osnma-pubkey.pem --pkid N
```
to see galileo-osnma processing the OSNMA and navigation data streamed by Galmon.
The [env_logger](https://docs.rs/env_logger/latest/env_logger/) documentation describes
how the logging information produced by this application can be configured.

The file `osnma-pubkey.pem` should contain the Galileo OSNMA public key, and the
number `N` should be its associated Public Key ID (PKID). See the section below
for how to obtain this data.

Note that Galmon aggregates data from many receivers around the world and
packets occasionally arrive out-of-order in the stream. This is not the main
expected use case for galileo-osnma. Therefore, when running this, there can be
some small problems with data or timestamps inconsistencies.

Alternatively, you can use one of the tools of Galmon with your own GNSS
receiver. For instance, an uBlox receiver can be used as
```
ubxtool --wait --port /dev/ttyACM0 --station 1 --stdout --galileo \
    | RUST_LOG=info cargo run --release -- --pubkey osnma-pubkey.pem --pkid N
```

## Obtaining the Galileo OSNMA public key and Merkle tree root

The OSNMA ECDSA public key and/or the Merkle tree root need to be obtained to
run `galmon-osnma` and other example applications, as well as to make full use
of the library. The current ECDSA public key is needed to validate OSNMA
cryptographic data (more precisely, TESLA root keys) transmitted in the
signal-in-space. The Merkle tree root is needed to validate ECDSA public keys
broadcast in the signal-in-space. These keys are transmitted only every 6
hours (at 00:00, 06:00, 12:00, and 18:00 GST).

The `galmon-osnma` application can be run using either the ECDSA public key
(using the `--pubkey` and `--pkid` arguments), the Merkle tree root (using the
`--merkle-root` argument), or both. If only the ECDSA public key is given, the
application will not be able to use new public keys that are broadcast in the
signal-in-space for a public key renewal or revocation. If only the Merkle tree
root is given, it will be necessary to wait until the current ECDSA public key
is broadcast in the signal-in-space.

The public key and the Merkle tree root can be
downloaded from the [European GNSS Service Centre](https://www.gsc-europa.eu/),
under [GSC Products > OSNMA_PUBLICKEY](https://www.gsc-europa.eu/gsc-products/OSNMA/PKI).
It is necessary to register an account to obtain these files.

The public key is downloaded as an x509 certificate. The Public Key ID is included
in the filename, and it is also listed elsewhere in the GSC Products website.
The current certificate file is `OSNMA_PublicKey_20240115100000_newPKID_1.crt`,
and the corresponding Public Key ID is `1`. The key in PEM format, as required by
`galmon-osnma` can be extracted with
```
openssl x509 -in OSNMA_PublicKey_20240115100000_newPKID_1.crt -noout -pubkey > osnma-pubkey.pem
```

The Merkle tree information is downloaded in an XML file. Only the tree root is
needed. This corresponds to the following entry in the XML file:
```xml
<TreeNode><j>4</j><i>0</i><lengthInBits>256</lengthInBits><x_ji>...</x_ji></TreeNode>
```
The tree root is given as a 256-bit hexadecimal number in place of the `...`. This
256-bit hexadecimal format is the one that is directly used by the `galmon-osnma`
`--merkle-root` argument. The tree root is also listed in other parts of the GSC Products
website.

The public key is also given as 264-bit compressed point in hexadecimal, both in
an XML file containing the public key, and in the Merkle tree XML file, as well
as in other parts of the GSC Products website. This format can be converted to
PEM with the [ecdsa](https://pypi.org/project/ecdsa/) Python package as follows:
```
ECDSA_PUBKEY="..." python -c 'import ecdsa; import os; vk = ecdsa.VerifyingKey.from_string(bytes.fromhex(os.environ["ECDSA_PUBKEY"]), curve=ecdsa.NIST256p); print(str(vk.to_pem(), encoding="ascii"), end="")'
```
Here `...` should be replaced by the 264-bit hexadecimal representation of the
public key.

## Development status

galileo-osnma has been usable since its first release during the public test
phase of OSNMA, and then updated for the OSNMA ICD changes done in the service
phase. Currently it is in-line with the OSNMA SIS ICD v1.1. galileo-osnma can
authenticate all the types of navigation data currently supported by OSNMA using
the ECDSA public keys and Merkle tree. There are some features of the OSNMA
protocol and some roadmap features that are not implemented yet. These are
listed below.

Supported features:

* Verification of DSM-KROOT using ECDSA P-256.
* Experimental support for verification of DSM-KROOT using ECDSA P-521.  This is
  untested, because there are no test vectors or signal-in-space using ECDSA
  P-521. Currently the `galmon-osnma` application assumes that the key that is
  loaded is a P-256 key. The `p521` feature, which is enabled by default, is
  used to enable P-521 support. It is disabled in the `osnma-longan-nano` demo,
  since otherwise the firmware size is too large for the target microcontroller.
* Verification of DSM-PKR against the Merkle tree root.
* Verification of TESLA keys using the TESLA root key or another previously
  authenticated key in the chain.
* Verification of the MACSEQ and ADKD fields of the MACK message using the MAC
  look-up table. This includes checking the flexible ADKDs.
* Verification of navigation data for ADKD=0, ADKD=4 and ADKD=12 using all the
  tags in the MACK messages.
* Retrieval of DSM messages using OSNMA data.
* Retrieval of MACK messages using OSNMA data.
* Navigation data retrieval using INAV words.
* Storage of the current ECDSA public key and potentially the next ECDSA public
  key, in order to support key renewal or revocation scenarios seamlessly.
* Storage of the current TESLA key and potentially a TESLA key for the next
  chain, in order to support chain renewal or revocation scenarios seamlessly.
* Storage and classification of MACK messages and navigation data.
* Tag accumulation. 80 bit worth of tags are required to consider a piece
  of navigation data as authenticated.
* Non-nominal scenarios (renewals, revocations, alerts), according to the values
  of the NMA status and CPKS fields in the NMA header.

Unsupported features:

* Warm start, by loading a previously authenticated TESLA key.

Roadmap features. These are not features of OSNMA itself, but will add to the
functionality and usability of galileo-osnma:

* C API
* Python API

## Minimum Supported Rust Version

Rust **1.70** or higher.

Minimum supported Rust version can be changed in the future, but it will be done
with a minor version bump.

## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
