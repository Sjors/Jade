Draft Taproot anti-exfiltration
==============================

This experimental firmware implements the single-signer BIP-EXFIL protocol via
``sign_psbt``. ``get_version_info`` reports ``JADE_TAPROOT_ANTI_EXFIL: true``.
Hosts must require that capability before sending an anti-exfil request; released
firmware does not recognize these fields and could sign normally.

The provisional input key is ``fc 02 61 65 <subtype> <32-byte output key>``.
Subtype 0 carries the host commitment, subtype 1 the full compressed signer
nonce, and subtype 2 the host entropy. See the BIP draft for the exact tagged
hashes and encoding. BIP assignment will change the domains and test vectors.

The first invocation carries only commitments and returns public nonces without
signatures. After retaining those nonces, the host adds entropy to a second
invocation. The signer recomputes and checks the original commitment and nonce,
adds the host scalar, and verifies the BIP340 signature before releasing it.
Nothing secret is retained across calls. Retry the identical session; never
restart automatically with fresh entropy after a failure.

Before using a signature, the host must independently select the output key and
BIP341 message, verify the BIP340 signature and ``sig.r == x(R0 + rho*G)``, and
reject changed transaction, UTXO, transcript, sighash, or extra-signature data.
Transport participation alone is not anti-exfil verification. HWI enforces these
host checks around the native PSBT adapter.

This first implementation requires every input to be a BIP86 key-path input
owned by this device and carrying a request for the same round. PSBT v0 and v2,
SIGHASH_DEFAULT and SIGHASH_ALL are supported. Script paths, merkle roots,
pre-existing signatures, mixed rounds, and fields outside input maps are rejected.
The actual tweaked signing public key must match the requested output key.
Each invocation retains the ordinary transaction and fee review.

``python3 test_anti_exfil.py`` compiles the actual bundled libsecp dependency
natively and checks the BIP's shared vectors and transcript rejection. HWI's
``jade_test_suite(..., anti_exfil=True)`` additionally exercises the real QEMU
firmware, reconnects between rounds, submits altered transcripts directly to
firmware, verifies both inputs, and broadcasts and mines the result.
