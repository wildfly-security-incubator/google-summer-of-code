# Google Summer of Code 2026

## WildFly Elytron Native PEM KeyStore for Kubernetes TLS Secrets

**Contributor:** Charlie Zhang  
**Organization:** JBoss Community  
**Project:** WildFly Elytron Native PEM KeyStore for Kubernetes TLS Secrets  
**Primary Project:** [WildFly Elytron](https://github.com/wildfly-security/wildfly-elytron)  
**Mentors:** Darran Lofthouse and Diana Krepinska

---

## Project Overview

Kubernetes TLS Secrets normally expose TLS credentials as two PEM files:

- `tls.crt` containing the certificate chain
- `tls.key` containing the private key

WildFly Elytron already had utility support for loading some combined PEM material, but it did not provide a registered native `PEM` KeyStore type or a direct way to load certificate and private-key material from separate files.

My Google Summer of Code 2026 project focused on adding native PEM KeyStore support to WildFly Elytron using Java's standard `KeyStoreSpi` mechanism.

The goal was to allow PEM-encoded TLS credentials, including Kubernetes TLS Secrets, to be loaded directly into an Elytron-compatible Java `KeyStore` without requiring conversion to formats such as JKS or PKCS12.

I also developed a WildFly feature pack that integrates this functionality with Kubernetes TLS Secrets and WildFly's existing Elytron TLS infrastructure.

Alongside my main project, I completed additional work on ACME External Account Binding support in WildFly.

---

## Project Goals

The main goals of the project were to:

- Add a native `PEM` KeyStore type to WildFly Elytron
- Support Kubernetes-style separate certificate and private-key files
- Support combined PEM files
- Integrate with Java's standard `KeyStore` API
- Register the implementation through Elytron security providers
- Preserve certificate-chain ordering
- Validate that private keys match their leaf certificates
- Support common RSA and EC private-key formats
- Reject malformed or invalid PEM material safely
- Maintain compatibility with Elytron's existing PEM loading API
- Integrate the KeyStore with WildFly and Kubernetes TLS Secrets
- Provide testing and documentation for the complete workflow

---

# Direct GSoC Project Work

## 1. Native PEM KeyStore Design and Proposal

Before implementing the feature, I worked on the WildFly feature proposal defining the architecture and expected behaviour of the native PEM KeyStore.

**Status:** Open

- [ELY-3051](https://issues.redhat.com/browse/ELY-3051)
- [WildFly Proposals Issue #835](https://github.com/wildfly/wildfly-proposals/issues/835)
- [WildFly Proposals PR #836](https://github.com/wildfly/wildfly-proposals/pull/836)

The proposal covers:

- The motivation for native PEM support
- Kubernetes TLS Secret integration
- The proposed `KeyStoreSpi` design
- Separate certificate and private-key files
- Provider registration
- Expected configuration behaviour
- Security considerations
- Testing requirements
- Integration through a dedicated Kubernetes TLS Secrets feature pack

During review, the proposal was updated to reflect the dedicated `kubernetes-tls-secrets-feature-pack` as the WildFly integration layer.

---

## 2. Baseline Kubernetes TLS PEM Test Coverage

Before introducing the new KeyStore SPI, I added regression coverage for Elytron's existing PEM loading functionality.

**Status:** Merged on 22 July 2026

- [ELY-3057](https://issues.redhat.com/browse/ELY-3057)
- [WildFly Elytron PR #2480](https://github.com/wildfly-security/wildfly-elytron/pull/2480)

The test creates:

- An RSA private key
- A service certificate
- A CA certificate
- Combined PEM content representing Kubernetes-style TLS material

It verifies:

- Successful KeyStore creation
- Correct key-entry detection
- Correct private-key recovery
- A two-certificate chain
- Correct leaf-to-CA certificate ordering

This provided a baseline for the behaviour that needed to remain compatible when introducing the native PEM KeyStore.

---

## 3. Native PEM KeyStore Implementation

The primary implementation of the project adds a native, read-only `PEM` KeyStore to WildFly Elytron.

**Status:** Submitted and under review

- [ELY-3051](https://issues.redhat.com/browse/ELY-3051)
- [WildFly Elytron PR #2494](https://github.com/wildfly-security/wildfly-elytron/pull/2494)

### Main Components

The implementation introduces:

- `PemKeyStoreSpi`
- `PemKeyStoreLoadParameter`
- `PemKeyStoreUtil`
- Elytron provider registration for the `PEM` KeyStore type
- Updates to existing PEM and KeyStore utilities
- Extensive automated test coverage

### Native `PEM` KeyStore

The implementation registers a new KeyStore type with Elytron:

```java
KeyStore keyStore = KeyStore.getInstance("PEM", provider);
```

This allows PEM credentials to be handled through Java's standard KeyStore APIs.

The KeyStore is intentionally read-only because PEM files mounted through systems such as Kubernetes are external credential sources rather than mutable KeyStore databases.

Mutation and store operations are therefore rejected.

### Separate Certificate and Private-Key Files

A new `PemKeyStoreLoadParameter` allows certificate and private-key material to be supplied through separate paths.

This directly supports the normal Kubernetes TLS Secret layout:

```text
tls.crt
tls.key
```

The implementation is not tied to these filenames, so other PEM-based layouts can also use the same functionality.

### Combined PEM Support

The native KeyStore also supports combined PEM input containing both certificates and private-key material.

This keeps the implementation useful outside Kubernetes and maintains compatibility with existing Elytron PEM workflows.

### Private-Key Support

The implementation supports several common private-key encodings:

- RSA PKCS#1
- RSA PKCS#8
- EC PKCS#8
- EC SEC1

### Certificate Handling

The implementation:

- Parses X.509 certificates
- Preserves certificate-chain ordering
- Identifies the leaf certificate
- Verifies that the supplied private key matches the leaf certificate

This helps prevent invalid TLS configurations from being accepted silently.

### Validation and Error Handling

The implementation rejects invalid input such as:

- Malformed PEM material
- Missing certificate files
- Missing private-key files
- Multiple private keys
- Mismatched certificate and private-key pairs
- Invalid entry ordering
- Unsupported protection parameters
- Unsupported mutation operations
- Unsupported store operations

Errors are also designed to avoid exposing private-key material.

### Atomic Loading

Loading and reloading is designed to be atomic.

If a new load attempt fails, previously valid KeyStore material remains available instead of leaving the KeyStore in a partially updated state.

### Backwards Compatibility

The existing API:

```java
KeyStoreUtil.loadPemAsKeyStore(...)
```

is retained as a compatibility path.

Its implementation is integrated with the new PEM KeyStore utilities so existing Elytron users are not required to immediately migrate to the new native KeyStore API.

---

## 4. Kubernetes TLS Secrets Feature Pack Integration

The second major deliverable integrates the native Elytron PEM KeyStore into WildFly through a dedicated Kubernetes TLS Secrets feature pack.

**Status:** Submitted and under review

- [ELY-3051](https://issues.redhat.com/browse/ELY-3051)
- [Kubernetes TLS Secrets Feature Pack PR #1](https://github.com/wildfly-security-incubator/kubernetes-tls-secrets-feature-pack/pull/1)

This work demonstrates how the native KeyStore can be consumed in a real WildFly deployment.

### Subsystem

The feature pack introduces a `tls-secrets` subsystem with resources such as:

```text
/subsystem=tls-secrets/key-store=<name>
```

The subsystem can load mounted PEM certificate and private-key files and expose them as a KeyStore capability that can be consumed by Elytron.

### Runtime Integration

The implementation includes runtime services that:

- Load certificate and private-key material
- Create the native Elytron PEM KeyStore
- Expose the KeyStore through WildFly capabilities
- Integrate with Elytron key managers
- Integrate with Elytron server SSL contexts
- Handle service lifecycle correctly
- Prevent partially initialized KeyStores from being published

### Runtime Updates

The subsystem supports runtime operations including:

- Adding KeyStore resources
- Removing KeyStore resources
- Updating configuration
- Targeted service restarts
- Rollback when updates fail

### Expression Support

Configuration supports WildFly expressions for values such as:

- Certificate paths
- Private-key paths
- Aliases

### Kubernetes Integration

The feature pack includes handling for Kubernetes-mounted TLS Secret layouts and projected-secret filesystem behaviour.

This allows credentials mounted into a WildFly container to be consumed without first converting them to JKS or PKCS12.

---

# Testing and Validation

Testing was a major part of the project.

## Native PEM KeyStore Testing

The Elytron PEM KeyStore tests cover areas including:

- Provider-based `PEM` KeyStore lookup
- Combined PEM loading
- Separate certificate and private-key files
- Custom aliases
- RSA PKCS#1 keys
- RSA PKCS#8 keys
- EC PKCS#8 keys
- EC SEC1 keys
- Certificate-chain ordering
- Private-key and certificate matching
- Malformed PEM input
- Missing files
- Empty input
- Multiple private keys
- Invalid PEM entry placement
- Mismatched key pairs
- Unsupported protection parameters
- Reading before initial load
- Failed reload behaviour
- Read-only mutation rejection
- Unsupported store operations
- Errors that avoid exposing sensitive private-key material

## Feature Pack Testing

The Kubernetes TLS Secrets feature pack includes tests for:

- Loading Kubernetes-style TLS Secrets
- Certificate and key path handling
- Kubernetes projected-secret symlinks
- Missing files
- Malformed credentials
- Mismatched certificate and private-key material
- Subsystem parsing
- Management operations
- Capability conflicts
- Runtime service lifecycle
- Configuration updates
- Rollback behaviour
- RSA TLS configurations
- EC TLS configurations
- SEC1 EC private keys
- Self-signed certificates
- CA-issued certificates
- End-to-end loopback TLS handshakes

At the time of the final project audit, the feature-pack test suite recorded:

```text
42 tests
0 failures
0 errors
2 skipped
```

---

# Documentation

The project includes documentation at several levels.

## Design Documentation

The WildFly proposal documents:

- Project motivation
- Architecture
- Security considerations
- Expected configuration
- Testing requirements
- Integration strategy

See:

- [WildFly Proposals PR #836](https://github.com/wildfly/wildfly-proposals/pull/836)

## Feature Pack Documentation

The Kubernetes TLS Secrets feature pack includes:

- `README.md`
- `docs/configuration.md`
- `docs/development.md`
- `docs/kubernetes.md`

These documents cover:

- Feature-pack usage
- Subsystem configuration
- Kubernetes deployment
- TLS Secret layout
- Development setup
- Known limitations
- Credential rotation behaviour

---

# Additional WildFly Work During GSoC

Alongside my official PEM KeyStore project, I also worked on adding ACME External Account Binding configuration and integration to WildFly.

External Account Binding is defined by the ACME protocol and is required by some certificate authorities when creating an ACME account.

The underlying Elytron ACME client work was originally implemented before my GSoC acceptance. During the GSoC period, I completed the WildFly design, subsystem integration, and administrator documentation needed to expose the feature to users.

---

## 1. ACME External Account Binding Proposal

**Status:** Open

- [WFCORE-7565](https://issues.redhat.com/browse/WFCORE-7565)
- [WildFly Proposals Issue #834](https://github.com/wildfly/wildfly-proposals/issues/834)
- [WildFly Proposals PR #831](https://github.com/wildfly/wildfly-proposals/pull/831)

The proposal defines how External Account Binding should be represented in WildFly's Elytron subsystem.

The design includes:

- An `external-account-binding` configuration object
- A CA-provided key identifier
- Secure storage of the HMAC key through a credential reference
- Validation requirements
- CLI configuration
- XML configuration
- Preview stability requirements

The proposal was revised during review to improve the management model and credential handling.

---

## 2. WildFly Core ACME EAB Integration

**Status:** Submitted and under review

- [WFCORE-7565](https://issues.redhat.com/browse/WFCORE-7565)
- [WildFly Core PR #6791](https://github.com/wildfly/wildfly-core/pull/6791)

This work integrates External Account Binding into the WildFly Elytron subsystem.

The implementation includes:

- Optional `external-account-binding` configuration
- Required EAB key identifier
- Credential-reference support for the HMAC key
- Secure credential resolution
- Passing the EAB credentials to `AcmeAccount`
- Preview schema support
- XML parser integration
- Management-model validation
- Transformation handling for older model versions
- Credential-reference validation and update handling

Testing includes:

- ACME account creation with EAB
- Mock ACME request validation
- Management-model validation
- XML and schema coverage
- Credential-resolution failures
- Validation that secret HMAC material is not exposed in error messages

---

## 3. ACME External Account Binding Documentation

**Status:** Submitted and under review

- [WFLY-21978](https://issues.redhat.com/browse/WFLY-21978)
- [WildFly PR #20113](https://github.com/wildfly/wildfly/pull/20113)

The documentation explains:

- What External Account Binding is
- When a certificate authority may require it
- How to configure the key identifier
- How to store the HMAC key using a credential store
- How to configure EAB through the WildFly CLI
- The preview stability requirement

---

# Contributions

## Direct GSoC Project Work

| Work | Project | Issue | Pull Request | Status |
| --- | --- | --- | --- | --- |
| Design native PEM KeyStore support for Kubernetes TLS Secrets | WildFly Proposals | [ELY-3051](https://issues.redhat.com/browse/ELY-3051) / [GitHub #835](https://github.com/wildfly/wildfly-proposals/issues/835) | [wildfly/wildfly-proposals#836](https://github.com/wildfly/wildfly-proposals/pull/836) | Open |
| Add baseline Kubernetes-style PEM KeyStore test coverage | WildFly Elytron | [ELY-3057](https://issues.redhat.com/browse/ELY-3057) / [ELY-3051](https://issues.redhat.com/browse/ELY-3051) | [wildfly-security/wildfly-elytron#2480](https://github.com/wildfly-security/wildfly-elytron/pull/2480) | Merged |
| Implement the native read-only PEM KeyStore SPI | WildFly Elytron | [ELY-3051](https://issues.redhat.com/browse/ELY-3051) | [wildfly-security/wildfly-elytron#2494](https://github.com/wildfly-security/wildfly-elytron/pull/2494) | Under review |
| Integrate PEM KeyStores with Kubernetes TLS Secrets through a WildFly feature pack | Kubernetes TLS Secrets Feature Pack | [ELY-3051](https://issues.redhat.com/browse/ELY-3051) | [wildfly-security-incubator/kubernetes-tls-secrets-feature-pack#1](https://github.com/wildfly-security-incubator/kubernetes-tls-secrets-feature-pack/pull/1) | Under review |

## Additional WildFly Work During GSoC

| Work | Project | Issue | Pull Request | Status |
| --- | --- | --- | --- | --- |
| Design WildFly ACME External Account Binding configuration | WildFly Proposals | [WFCORE-7565](https://issues.redhat.com/browse/WFCORE-7565) / [GitHub #834](https://github.com/wildfly/wildfly-proposals/issues/834) | [wildfly/wildfly-proposals#831](https://github.com/wildfly/wildfly-proposals/pull/831) | Open |
| Integrate ACME External Account Binding into the Elytron subsystem | WildFly Core | [WFCORE-7565](https://issues.redhat.com/browse/WFCORE-7565) | [wildfly/wildfly-core#6791](https://github.com/wildfly/wildfly-core/pull/6791) | Under review |
| Document ACME External Account Binding configuration | WildFly | [WFLY-21978](https://issues.redhat.com/browse/WFLY-21978) | [wildfly/wildfly#20113](https://github.com/wildfly/wildfly/pull/20113) | Under review |

---

# Current State

As of 20 August 2026:

- The baseline Kubernetes PEM regression test has been merged into WildFly Elytron.
- The native PEM KeyStore implementation has been completed and submitted upstream for review.
- The Kubernetes TLS Secrets feature-pack implementation has been completed and submitted for review.
- The native PEM KeyStore proposal remains open.
- The ACME EAB proposal remains open.
- The WildFly Core ACME EAB integration has been submitted and is under review.
- The WildFly ACME EAB documentation has been submitted and is under review.

The main implementation work for my GSoC project is therefore complete, with several upstream contributions still going through the normal WildFly review process.

---

# Remaining Work

The main remaining work is upstream review and any changes requested by the WildFly and Elytron maintainers.

For the PEM KeyStore work, this includes:

- Completing review of the native Elytron PEM KeyStore PR
- Completing review of the Kubernetes TLS Secrets feature pack
- Completing review of the WildFly feature proposal
- Addressing any additional maintainer feedback

For the additional ACME EAB work, the remaining work is also primarily upstream review.

---

# Challenges and Learnings

One of the main technical challenges was mapping PEM-based credentials cleanly onto Java's standard KeyStore model.

Kubernetes provides certificates and private keys as separate files, while traditional Java KeyStores such as JKS and PKCS12 normally package key material together.

Supporting this through `KeyStoreSpi` required careful API design around:

- Separate certificate and key paths
- Combined PEM streams
- Key aliases
- Private-key formats
- Certificate chains
- Validation
- Reload behaviour
- Read-only semantics

Another challenge was ensuring invalid or mismatched TLS credentials fail safely without exposing private-key material through error messages.

The feature-pack work also required integrating the KeyStore into WildFly's runtime model. This involved subsystem design, capability registration, MSC services, runtime updates, rollback, and integration with Elytron's key-manager and SSL-context infrastructure.

The ACME EAB work gave me additional experience implementing functionality that spans multiple WildFly repositories and layers, from protocol support through management configuration to administrator documentation.

During GSoC I gained practical experience with:

- Java security APIs
- `KeyStore` and `KeyStoreSpi`
- PEM parsing
- X.509 certificates
- RSA cryptography
- EC cryptography
- PKCS#1
- PKCS#8
- SEC1
- TLS credential handling
- Kubernetes TLS Secrets
- WildFly Elytron
- WildFly subsystems
- WildFly capabilities
- MSC services
- Galleon feature packs
- ACME
- External Account Binding
- RFC-based protocol implementation
- Secure credential references
- Compatibility in established Java APIs
- Automated testing of security-sensitive code
- Contributing production changes to large open-source projects
- Responding to maintainer review and revising designs based on feedback

---

# Source Code

The source code for this project is contributed directly to the upstream WildFly projects rather than duplicated in this repository.

## Native PEM KeyStore

- [WildFly Proposals #836 — Native PEM KeyStore design](https://github.com/wildfly/wildfly-proposals/pull/836)
- [WildFly Elytron #2480 — Kubernetes TLS PEM baseline tests](https://github.com/wildfly-security/wildfly-elytron/pull/2480)
- [WildFly Elytron #2494 — Native PEM KeyStore implementation](https://github.com/wildfly-security/wildfly-elytron/pull/2494)
- [Kubernetes TLS Secrets Feature Pack #1 — WildFly integration](https://github.com/wildfly-security-incubator/kubernetes-tls-secrets-feature-pack/pull/1)

## Additional ACME External Account Binding Work

- [WildFly Proposals #831 — EAB configuration proposal](https://github.com/wildfly/wildfly-proposals/pull/831)
- [WildFly Core #6791 — EAB subsystem integration](https://github.com/wildfly/wildfly-core/pull/6791)
- [WildFly #20113 — EAB administrator documentation](https://github.com/wildfly/wildfly/pull/20113)

### Pre-GSoC Foundation

The underlying Elytron ACME External Account Binding client support was originally implemented before my GSoC acceptance and is therefore not counted as GSoC-period implementation work:

- [WildFly Elytron #2428 — ACME External Account Binding client support](https://github.com/wildfly-security/wildfly-elytron/pull/2428)

The post-acceptance WildFly proposal, WildFly Core integration, and documentation listed above build on this underlying support.

---

# Acknowledgements

Thank you to my mentors **Darran Lofthouse** and **Diana Krepinska** for their guidance and feedback throughout Google Summer of Code.

Thank you to the **JBoss Community**, the **WildFly Elytron** maintainers, the wider **WildFly** community, and **Google Summer of Code** for the opportunity to contribute.
