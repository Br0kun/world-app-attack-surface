# World App — Security Research

**Android Reverse Engineering · Rust · Bedrock · Offensive Security**

Independent security research into World App, focusing on application internals, native components, network communication, and potential security weaknesses.

This repository documents my technical investigations, reverse engineering work, code analysis, experiments, and findings.

## Research Areas

### Android Reverse Engineering
- APK analysis and decompilation
- JADX and Smali analysis
- Tracing application execution flows
- Following data between application components
- Investigating native library interactions

### Bedrock & Rust
- Analysis of Bedrock's Rust implementation
- Android-to-native communication through UniFFI
- Network request handling and URL validation
- Input flow and trust-boundary analysis
- Reviewing implementation logic and security assumptions

### Application & API Security
- Attack-surface analysis
- Access control and authorization
- API behavior and response validation
- Input handling and trust boundaries
- Identifying potential security weaknesses

## Current Research

### Custom Bundler — Potential SSRF Investigation

An investigation into World App's custom bundler functionality, tracing the configured RPC URL from the Android application through the native Bedrock layer to outbound JSON-RPC requests.

The research examined:

- Android configuration and URL handling.
- Native endpoint verification.
- Rust URL validation.
- HTTP client configuration and request sinks.
- Acceptance of private and special-use HTTPS destinations.

**Current assessment:** A permissive HTTPS validation behavior was reproduced through a unit test. However, remote attacker control and exploitable SSRF impact have not been established.

[Read the investigation](./README.md)

## Tools & Technologies

- **Languages:** Python, Java, Kotlin, Rust
- **Reverse Engineering:** JADX, Smali
- **Security Testing:** Burp Suite, ffuf
- **Environment:** Linux, Kali Linux
- **Source Analysis:** Git, command-line tools

## Research Approach

Research is based on examining implementations, tracing execution paths, testing observable behavior, and documenting reproducible evidence.

Findings are distinguished from hypotheses. Unconfirmed behavior is not presented as an exploitable vulnerability.

---

*Independent security research for educational purposes.*
