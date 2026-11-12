# Following the URL: Investigating a Potential SSRF Across World App and Bedrock

**Target:** World App (`com.worldcoin`)  
**Version:** 4.0.4602  
**Components:** World App Android application / Bedrock native Rust library  
**Research type:** Android reverse engineering, native code analysis, Rust source analysis and network security  
**Status:** Potential SSRF-related weakness — exploitability not established

---

<p align="center">
  <img src="./gif.gif" width="350" alt="Waiting">
</p>

## 01 — The Question Behind the Feature

Every application has boundaries. Some are obvious: authentication, permissions, API endpoints. Others are hidden inside features that appear completely ordinary.

This investigation started with one such feature: World App's custom bundler configuration.

The feature allows the application to work with a configurable RPC endpoint during wallet transaction operations. On the surface, this looks like a simple configuration option. But behind that setting is a path involving multiple components: Android application logic, a native interface, and a Rust networking implementation.

I wanted to understand what actually happens after a URL is entered.

Where is it stored? Which component validates it? Does the value remain inside the Android application, or does it cross into native code? And, most importantly, what happens when the native implementation uses it as a network destination?

The investigation focused on a possible Server-Side Request Forgery (SSRF) attack surface.

Rather than looking at the URL validator in isolation, I followed the value across the application architecture, from the Android configuration layer into Bedrock and its outbound networking operations.

The execution path under investigation was:

```text
World App Settings
       |
       v
Android Configuration
       |
       v
Wallet Relayer Logic
       |
       v
UniFFI / Bedrock Bridge
       |
       v
Bedrock Rust Implementation
       |
       v
URL Validation
       |
       v
HTTP Client
       |
       v
Outbound JSON-RPC Request
```

The important part was not simply finding a configurable URL.

It was establishing how that value moves between components, where the trust boundary exists, and whether the final network behavior introduces a security concern.

---

## 02 — Understanding the Architecture: World App and Bedrock

Before examining the URL itself, I needed to understand the relationship between the Android application and Bedrock.

Bedrock is not simply another Android package or a collection of unrelated Rust utilities. It is a foundational library used by World App's crypto wallet functionality.

The project provides native implementations written in Rust and exposes functionality to platform applications through foreign-language bindings, including Kotlin and Swift.

This architecture matters because some operations that appear to originate in the Android application are actually implemented in native Rust code.

For this investigation, that distinction was particularly important.

The Android application handles the user-facing configuration and application-level transaction flow. Bedrock provides the underlying implementation responsible for relevant wallet operations, including the custom bundler networking functionality.

The two layers therefore need to be examined together.

### 2.1 The Android Layer

The Android application contains the configuration and application logic responsible for obtaining and passing the custom bundler URL.

This is where I began tracing the value.

### 2.2 The Native Layer: Bedrock

Bedrock contains the Rust implementation responsible for validating and using the RPC endpoint.

The relevant implementation is located at:

```text
bedrock/src/transactions/custom_bundler.rs
```

This file became the central point of the investigation because it contains the URL validation logic, HTTP client construction, and request-handling functions used by the custom bundler functionality.

### 2.3 The Bridge Between Them

The connection between these layers is provided through UniFFI-generated bindings.

During the Android investigation, I identified a call into the Bedrock Kotlin binding:

```smali
invoke-static {p1, v0},
    Luniffi/bedrock/BedrockKt;->verifyBundlerRpcEntrypoint(
        Ljava/lang/String;
        Lkotlin/coroutines/Continuation;
    )Ljava/lang/Object;
```

This was an important discovery.

It demonstrated that the URL does not remain exclusively within Android's application-level validation logic. It is passed into a native Bedrock operation.

The investigation therefore needed to follow both sides of this boundary.

---

## 03 — Starting on Android: Where Does the URL Come From?

I started by examining the Android application's wallet relayer settings.

The first relevant component was:

```text
WalletTransactionRelayerStoreImpl
```

This implementation manages the custom bundler URL through the preference key:

```text
key_wallet_transaction_relayer_url
```

Its methods provide the basic configuration behavior:

- Reading the stored URL.
- Returning the configured default when no custom value exists.
- Checking whether the custom relayer functionality is enabled.
- Writing a new URL to preferences.

The application also contains a default endpoint:

```text
https://app-backend.toolsforhumanity.com/v2/rpc/worldchain
```

At this point, I had identified the configuration mechanism.

However, finding a stored URL is not enough to establish a security issue.

A value can exist in application preferences without ever reaching a sensitive operation.

I needed to understand how the application consumed it.

### 3.1 Following the Settings Validation Flow

The investigation led through the wallet relayer settings implementation and its validation coroutines.

The relevant components included:

- `WalletRelayerSettingViewModel`
- `nn1/f0`
- `nn1/g0`
- `ux0/z`

The settings flow performs more than a basic syntax check.

The URL is processed through the validation logic, which eventually reaches the Bedrock binding.

The important call was:

```smali
Luniffi/bedrock/BedrockKt;->verifyBundlerRpcEntrypoint(...)
```

This means the application invokes native endpoint verification using the supplied URL.

The significance is that the URL reaches a component responsible for communicating with the configured RPC endpoint.

It is not simply checked using a local URL parser.

### 3.2 Why This Matters

This changed the direction of the investigation.

Initially, the interesting question was whether the application accepted an unusual URL.

After identifying the native call, the more important question became:

**What does Bedrock do with the URL after receiving it?**

That required examining the Rust implementation.

---

## 04 — Entering Bedrock: Examining the Rust Implementation

I moved from the Android application into the Bedrock repository.

The primary file was:

```text
bedrock/src/transactions/custom_bundler.rs
```

This file contains the implementation relevant to the custom bundler's URL handling and network communication.

Rather than treating it as an isolated validator, I examined the main functions involved in the request lifecycle:

```rust
validate_rpc_url(url: &str)
```

```rust
build_bundler_client()
```

```rust
post_json_rpc_to_url()
```

Alongside these, I examined the functions responsible for endpoint verification, sponsored operation preparation, and submission.

The objective was to establish the relationship between the URL validation logic and the eventual HTTP request.

### 4.1 URL Validation

The first function that deserved attention was:

```rust
validate_rpc_url(url: &str)
```

This function parses the supplied URL and applies restrictions based on its scheme.

The behavior differs between HTTP and HTTPS.

### 4.2 HTTP Restrictions

For HTTP URLs, the examined implementation permits only specific loopback destinations:

| Host | Result |
|---|---|
| `127.0.0.1` | Accepted |
| `localhost` | Accepted |
| `::1` | Accepted |
| `[::1]` | Accepted |
| Other HTTP destinations | Rejected |

This is an explicit restriction on non-loopback HTTP destinations.

The implementation therefore distinguishes between local HTTP endpoints and other HTTP destinations.

HTTPS, however, follows a different validation path.

### 4.3 The HTTPS Validation Gap

The examined implementation accepts parseable HTTPS URLs without explicitly rejecting:

- Private IPv4 addresses.
- Loopback addresses.
- Link-local addresses.
- Other reserved or special-use destinations.

This caught my attention.

If the intended security policy is to restrict the custom bundler to publicly reachable endpoints, the examined validator does not explicitly enforce that restriction for HTTPS destinations.

The distinction is important.

The validator applies a specific host restriction to HTTP, but the corresponding HTTPS path does not contain an equivalent explicit private-address restriction.

At this point, I had identified a potential validation weakness.

But source-code inspection alone was not enough.

I wanted to reproduce the behavior.

---

## 05 — Testing the Validator Instead of Guessing

I added a focused unit test:

```rust
test_https_private_destinations_are_currently_accepted
```

The objective was to determine whether the URL validator rejects HTTPS URLs pointing to private or special-use IP addresses.

The test included:

```text
https://10.0.0.1
https://192.168.1.1
https://169.254.169.254
https://[::1]
```

These destinations were selected to test different categories of addresses:

- Private IPv4 space.
- Another private IPv4 range.
- IPv4 link-local space.
- IPv6 loopback.

The result was:

```text
1 passed; 507 filtered out
```

The test passed.

This confirmed that the examined validator accepts these URL values.

It was a useful result because it moved the investigation from source-code interpretation to reproducible behavior.

However, there is an important distinction:

**The test demonstrated URL acceptance, not successful network access.**

It did not establish that the device could establish a connection to those destinations, complete a TLS handshake, or communicate with an internal service.

The result demonstrated the behavior of the validation function.

It did not demonstrate the complete network-level exploitation scenario.

I had confirmed a permissive validation boundary.

I had not demonstrated SSRF.

---

## 06 — From Validation to Networking: The Bedrock Request Path

Finding a permissive validator becomes more interesting when its accepted input reaches an actual network sink.

I therefore continued examining the functions responsible for constructing the HTTP client and sending requests.

This was the most important part of the Bedrock analysis.

### 6.1 HTTP Client Construction

The relevant function was:

```rust
build_bundler_client()
```

This function constructs the HTTP client used by the custom bundler operations.

The examined configuration included:

- A 15-second timeout.
- Redirects disabled through `Policy::none()`.
- Gzip response support.
- TLS configuration through the project's TLS client builder.

I found no indication in the examined configuration that TLS certificate verification was disabled.

This matters because accepting a URL and accepting an unverified TLS connection are two different behaviors.

The investigation did not establish a TLS verification bypass.

### 6.2 Redirect Handling

The client also disables automatic redirects:

```rust
redirect(Policy::none())
```

This prevents the HTTP client from automatically following a redirect response to another destination.

That is a relevant restriction, but it does not eliminate the concern raised by the URL validation behavior.

If a private HTTPS destination is supplied as the original URL and accepted by validation, disabling redirects does not prevent the initial request from being directed there.

The original destination still matters.

### 6.3 The Actual Request Sink

The next important function was:

```rust
post_json_rpc_to_url()
```

This function performs the outbound JSON-RPC POST request using the supplied URL.

This established that the custom bundler URL is used as an actual network destination.

It is not merely stored, displayed, or passed through a configuration layer.

The request mechanism is used by multiple operations within the custom bundler implementation.

### 6.4 JSON-RPC Operations

The relevant methods include:

| JSON-RPC method | Purpose |
|---|---|
| `eth_supportedEntryPoints` | Verify supported EntryPoint contracts |
| `eth_estimateUserOperationGas` | Estimate UserOperation gas |
| `eth_sendUserOperation` | Submit a UserOperation |

These operations provide a clearer picture of why the configured endpoint matters.

The URL participates in the transaction preparation and submission workflow.

The native implementation therefore contains both:

1. A URL validation boundary.
2. An outbound network request mechanism.

That relationship is what made the behavior worth investigating.

### 6.5 What Was Established

At this stage, the investigation had established the following:

1. Android stores a configurable bundler URL.
2. Android invokes native endpoint verification using that URL.
3. The URL crosses the Android-to-Bedrock boundary.
4. Bedrock applies URL validation.
5. The HTTPS validation path accepts private and special-use destinations.
6. Bedrock constructs an HTTP client.
7. The supplied URL is used for outbound JSON-RPC requests.

This was enough to establish a genuine attack surface worth investigating.

But an attack surface is not yet an attack.

---

## 07 — The Trust Boundary: Who Controls the Destination?

This was the point where the investigation needed to become more demanding.

A possible SSRF vulnerability requires more than a network sink accepting an unusual URL.

I needed to establish whether an attacker could influence the destination used by another user's application.

The distinction is fundamental.

A user configuring their own RPC endpoint is not automatically equivalent to an attacker controlling that endpoint remotely.

The investigation demonstrated that the application supports a configurable URL and that Bedrock accepts certain private HTTPS destinations.

However, I could not establish a remote mechanism that allowed an attacker to modify another user's configured URL.

The Android analysis showed the configuration and validation flow, but it did not establish that an untrusted remote party could set the preference for another user.

This left an important link missing from the potential attack chain.

### 7.1 The Incomplete Attack Chain

The observed behavior can be represented as:

```text
Custom URL
    |
    v
Android Configuration
    |
    v
Bedrock Validation
    |
    v
HTTPS Private Destination Accepted
    |
    v
Outbound JSON-RPC Request
```

The missing part is the attacker-control mechanism:

```text
Remote Attacker
    |
    v
Control Another User's URL
    |
    v
Trigger Request in Victim's Application
```

I did not establish this path.

Without evidence that an attacker can influence another user's destination, the behavior cannot automatically be classified as a remotely exploitable SSRF vulnerability.

### 7.2 Network Reachability and Impact

There were also additional questions that remained unanswered:

- Can the application successfully connect to the accepted private destinations?
- Can the target device reach the relevant internal service?
- Does the request produce an observable security impact?
- Can sensitive information be retrieved or exposed?
- Is there an attacker-controlled mechanism that makes the behavior exploitable?

I did not demonstrate these conditions.

The result was a confirmed implementation behavior with an unproven exploitation scenario.

I decided not to fill that gap with assumptions.

---

## 08 — Separating Evidence from the Hypothesis

After examining both the Android application and Bedrock, I separated the confirmed observations from the questions that remained unanswered.

| Observation | Evidence |
|---|---|
| Android stores a custom bundler URL | Confirmed |
| Android invokes native endpoint verification | Confirmed |
| URL crosses into the Bedrock native implementation | Confirmed |
| Bedrock contains custom bundler URL validation | Confirmed |
| Private HTTPS destinations pass the examined validator | Confirmed by unit test |
| Bedrock constructs an HTTP client for bundler requests | Confirmed in source |
| Automatic HTTP redirects are disabled | Confirmed in source |
| The supplied URL is used for outbound JSON-RPC requests | Confirmed in source |
| A remote attacker can control another user's URL | Not established |
| Internal service access can be achieved | Not demonstrated |
| Sensitive information can be extracted | Not demonstrated |
| An exploitable SSRF vulnerability exists | Not established |

The distinction matters because security research should describe what the evidence actually demonstrates.

The permissive validator is a legitimate observation.

The URL's use in outbound network communication is also established.

The possibility of SSRF is a reasonable hypothesis.

But the available evidence does not justify presenting the hypothesis as a confirmed vulnerability.

---

## 09 — Beyond SSRF: Other Security Questions in the Custom Bundler

Although the SSRF investigation reached an evidentiary limit, the custom bundler feature exposes other security-relevant components worth examining.

The next research direction could move away from URL validation and toward transaction integrity.

This is especially interesting because Bedrock does more than send arbitrary HTTP requests. It participates in wallet transaction preparation and submission.

The interaction between an application, a configurable RPC endpoint, and transaction construction creates additional questions.

These are separate research hypotheses, not established vulnerabilities.

### 9.1 UserOperation Integrity

Does the application verify that the UserOperation returned or prepared through the bundler is consistent with the operation the user intended to sign?

Could an unexpected response influence fields that are later used in signing or submission?

### 9.2 RPC Response Validation

Are RPC responses validated beyond basic parsing and deserialization?

Could malformed, unexpected, or inconsistent responses influence transaction preparation?

### 9.3 Chain and EntryPoint Consistency

Are chain ID, sender, nonce, and EntryPoint values checked against locally expected values before signing or submission?

Are these values trusted from the configured bundler, or independently validated by the application and native implementation?

### 9.4 Sponsorship and Gas Estimation

Can an untrusted bundler response influence sponsorship data, gas parameters, or other transaction fields without adequate validation?

How are the results of `eth_estimateUserOperationGas` incorporated into the prepared operation?

### 9.5 Transaction Preparation and Submission

Are the values used during estimation consistent with the values eventually submitted through `eth_sendUserOperation`?

Could a difference between preparation and submission introduce an unexpected behavior?

These questions would require separate execution traces, source analysis, and reproducible tests.

They should not be treated as findings until the relevant behavior and impact are demonstrated.

---

## 10 — Final Assessment

This investigation examined the custom bundler functionality across two connected layers of World App.

On the Android side, I traced the custom URL through the wallet relayer configuration and settings validation flow.

I identified the native Bedrock binding responsible for endpoint verification, establishing that the URL crosses the Android-to-native boundary.

On the Bedrock side, I examined the Rust implementation responsible for URL validation, HTTP client construction, and outbound JSON-RPC communication.

The investigation identified a permissive HTTPS validation behavior. A focused unit test confirmed that private and special-use HTTPS destinations are accepted by the examined validator.

I also established that the configured URL is used for actual outbound JSON-RPC operations, including endpoint verification, gas estimation, and UserOperation submission.

However, the investigation did not establish remote attacker control over another user's configuration, successful access to internal services, or meaningful security impact.

For that reason, the result should be classified as:

**Potential SSRF-related attack surface — exploitability not established.**

This is not a confirmed vulnerability report.

It is a documented security investigation that maps an Android-to-native execution path, identifies a permissive validation behavior in Bedrock, reproduces that behavior through a unit test, and clearly records the limitations of the current evidence.

---

## 11 — Research Notes and References

### Primary Repository

Worldcoin Bedrock:

https://github.com/worldcoin/bedrock

Bedrock is described in its repository as the foundational library powering World App's crypto wallet. It provides Kotlin and Swift bindings for native application integration.

### Primary Source File

```text
bedrock/src/transactions/custom_bundler.rs
```

### Important Rust Functions

```rust
validate_rpc_url
build_bundler_client
post_json_rpc_to_url
verify_bundler_rpc_entrypoint
prepare_bundler_sponsored_operation
send_user_operation_to_url
```

### Android Components Examined

```text
WalletTransactionRelayerStoreImpl
WalletRelayerSettingViewModel
ux0/z
y00/j
```

### Native Integration

```text
Luniffi/bedrock/BedrockKt
```

Relevant binding:

```text
verifyBundlerRpcEntrypoint
```

### Development History

- Pull request #275 — Verify bundler RPC EntryPoint.
- Pull request #460 — Prepare transfers with a custom bundler.
- Commit `8e5a76b` — Custom bundler integration.

The Bedrock changelog records custom bundler support in version 0.7.3, alongside the related prepared-transfer changes. The endpoint verification functionality predates that integration.

### Research Principle

Follow the input.

Identify the trust boundary.

Understand the native implementation.

Find the network sink.

Test the behavior.

Establish attacker control.

Prove the impact.

And if the evidence stops, say exactly where it stops.
