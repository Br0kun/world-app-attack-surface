# Following the URL: Investigating a Potential SSRF in World App's Custom Bundler

**Target:** World App (`com.worldcoin`)  
**Version:** 4.0.4602  
**Component:** Custom Bundler / Bedrock Rust networking layer  
**Research type:** Android reverse engineering and native code analysis  
**Status:** Potential SSRF-related weakness — exploitability not established
---

<iframe src="https://giphy.com/embed/5uDNFrOi6VwrkFKBaj" width="480" height="480" style="" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/waiting-meh-disapproving-5uDNFrOi6VwrkFKBaj">via GIPHY</a></p>


## 01 — The Question Behind the Feature

Every application has boundaries. Some are obvious: authentication, permissions, API endpoints. Others are hidden inside features that appear completely ordinary.

This investigation started with one such feature: World App's custom bundler configuration.

The idea was straightforward. If an application allows a configurable network endpoint, where does that endpoint actually go? Is it merely stored as a preference, or does it eventually become the destination of a network request?

More importantly, what restrictions are applied before the application connects to it?

I wanted to investigate whether this functionality could expose an SSRF attack surface.

Rather than starting with assumptions about exploitability, I decided to trace the URL from its origin in the Android application all the way to the native networking implementation.

The path I wanted to understand was:

`Configuration → Android → Native Bridge → Rust → HTTP Request`

The interesting part would not be finding a URL field. It would be establishing what the application actually does with it.

---

## 02 — Starting on Android: Where Does the URL Come From?

I started by examining the Android application's wallet relayer settings.

The first relevant component was:

`WalletTransactionRelayerStoreImpl`

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

At this point, I had identified the configuration mechanism. But configuration alone tells us very little about security.

A value can be stored without ever reaching a sensitive operation.

The next step was to follow the URL into the application logic.

### 2.1 From Settings to Native Verification

The investigation led through the wallet relayer settings implementation and its validation coroutines.

The relevant components included:

- `WalletRelayerSettingViewModel`
- `nn1/f0`
- `nn1/g0`
- `ux0/z`

The important discovery was inside the validation path.

The implementation eventually calls:

```smali
invoke-static {p1, v0},
    Luniffi/bedrock/BedrockKt;->verifyBundlerRpcEntrypoint(
        Ljava/lang/String;
        Lkotlin/coroutines/Continuation;
    )Ljava/lang/Object;
```

This call is significant.

The application does not simply perform a local syntax check and accept the URL. It invokes the native Bedrock implementation to verify the RPC endpoint.

That means the URL crosses the Android/native boundary and enters functionality responsible for communicating with the configured endpoint.

I now had a concrete path to investigate.

The next question was no longer where the URL was stored.

It was what happened after Bedrock received it.

---

## 03 — Entering Bedrock: Examining the Trust Boundary

I moved to the corresponding Rust implementation:

```text
bedrock/src/transactions/custom_bundler.rs
```

The first function that deserved attention was:

```rust
validate_rpc_url(url: &str)
```

This function determines whether a supplied RPC URL satisfies the implementation's validation rules.

The behavior differs depending on the URL scheme.

### 3.1 HTTP Restrictions

For HTTP URLs, the examined implementation permits only specific loopback destinations:

| Host | Result |
|---|---|
| `127.0.0.1` | Accepted |
| `localhost` | Accepted |
| `::1` | Accepted |
| `[::1]` | Accepted |
| Other HTTP destinations | Rejected |

This is an explicit restriction on non-loopback HTTP destinations.

HTTPS, however, follows a different validation path.

### 3.2 The HTTPS Validation Gap

The examined implementation accepts parseable HTTPS URLs without explicitly rejecting:

- Private IPv4 addresses.
- Loopback addresses.
- Link-local addresses.
- Other reserved or special-use destinations.

This caught my attention.

If the intention is to restrict the bundler to publicly reachable endpoints, the validator does not explicitly enforce that policy for HTTPS destinations.

But I did not want to stop at reading the source code.

A suspicious-looking condition is a starting point, not proof.

I wanted to test the actual behavior.

---

## 04 — Testing the Validator Instead of Guessing

I added a focused unit test:

```rust
test_https_private_destinations_are_currently_accepted
```

The objective was simple: determine whether the validator rejects HTTPS URLs pointing to private or special-use IP addresses.

The test included:

```text
https://10.0.0.1
https://192.168.1.1
https://169.254.169.254
https://[::1]
```

The result:

```text
1 passed; 507 filtered out
```

The test passed.

This confirmed that the examined validator accepts these URL values.

It was a useful result because it moved the investigation from source-code interpretation to reproducible behavior.

However, there is an important distinction:

**The test demonstrated URL acceptance, not successful network access.**

It did not establish that the device could establish a connection, complete a TLS handshake, or communicate with an internal service.

I had confirmed a permissive validation boundary.

I had not demonstrated SSRF.

---

## 05 — Finding the Actual Network Request

A validator becomes more interesting when its output reaches a network sink.

I therefore continued examining the functions responsible for constructing the HTTP client and sending requests.

### 5.1 HTTP Client Construction

The relevant function was:

```rust
build_bundler_client()
```

The examined client configuration included:

- A 15-second timeout.
- Redirects disabled through `Policy::none()`.
- Gzip response support.
- TLS configuration through the project's TLS client builder.

I found no indication in the examined configuration that TLS certificate verification was disabled.

The redirect policy was also worth noting.

Disabling redirects prevents an HTTP response from automatically redirecting the client to another destination.

However, this does not prevent a request from being sent to a private IP address when that address is supplied as the original URL.

The initial destination still matters.

### 5.2 The Request Sink

The next function was:

```rust
post_json_rpc_to_url()
```

This function performs the outbound JSON-RPC POST request using the supplied URL.

The custom bundler implementation uses this request mechanism for several operations.

The relevant JSON-RPC methods include:

| Method | Purpose |
|---|---|
| `eth_supportedEntryPoints` | Verify supported EntryPoint contracts |
| `eth_estimateUserOperationGas` | Estimate UserOperation gas |
| `eth_sendUserOperation` | Submit a UserOperation |

These operations establish that the custom URL is used in actual network communication rather than being an isolated configuration value.

At this stage, the investigation had established several connected facts:

1. The Android application stores a configurable bundler URL.
2. Android invokes native verification using that URL.
3. Bedrock applies URL validation.
4. The HTTPS validation path accepts private and special-use destinations.
5. Bedrock constructs an HTTP client.
6. The supplied URL is used for outbound JSON-RPC requests.

This was enough to establish a genuine attack surface worth investigating.

But an attack surface is not yet an attack.

---

## 06 — The Critical Question: Who Controls the Destination?

This was the point where the investigation needed to become more demanding.

A possible SSRF vulnerability requires more than a network sink accepting an unusual URL.

I needed to establish whether an attacker could influence the destination used by another user's application.

The distinction is fundamental.

A user configuring their own RPC endpoint is not automatically equivalent to an attacker controlling that endpoint remotely.

I had already traced the URL through the Android application and into Bedrock. I had also demonstrated that private HTTPS destinations passed the examined validator.

But I could not establish a remote mechanism that allowed an attacker to modify another user's configured URL.

Without this link, the attacker-controlled request path remained incomplete.

I also did not demonstrate that a private destination could be reached successfully or that sensitive information could be obtained through the behavior.

This left the investigation with a confirmed implementation behavior but an unproven exploitation scenario.

I decided not to fill that gap with assumptions.

---

## 07 — Separating Evidence from the Hypothesis

After examining the Android and Rust implementations, I separated the confirmed observations from the questions that remained unanswered.

| Observation | Evidence |
|---|---|
| Android stores a custom bundler URL | Confirmed |
| Android invokes native endpoint verification | Confirmed |
| Native code performs outbound JSON-RPC requests | Confirmed |
| Private HTTPS destinations pass the examined validator | Confirmed by unit test |
| HTTP redirects are disabled | Confirmed in source |
| A remote attacker can control another user's URL | Not established |
| Internal service access can be achieved | Not demonstrated |
| Sensitive information can be extracted | Not demonstrated |
| An exploitable SSRF vulnerability exists | Not established |

The distinction matters because security research should describe what the evidence actually demonstrates.

The permissive validator is a legitimate observation.

The possibility of SSRF is a reasonable hypothesis.

But the available evidence does not justify presenting the hypothesis as a confirmed vulnerability.

---

## 08 — What Could Make This More Interesting?

Although the SSRF investigation reached an evidentiary limit, the custom bundler feature exposes other security-relevant components worth examining.

The next research direction could move away from URL validation and toward transaction integrity.

Potential questions include:

### 8.1 UserOperation Integrity

Does the application verify that the UserOperation returned or prepared through the bundler is consistent with the operation the user intended to sign?

### 8.2 RPC Response Validation

Are RPC responses validated beyond basic parsing and deserialization?

Could an unexpected response influence transaction preparation?

### 8.3 Chain and EntryPoint Consistency

Are chain ID, sender, nonce, and EntryPoint values checked against locally expected values before signing or submission?

### 8.4 Sponsorship and Gas Estimation

Can an untrusted bundler response influence sponsorship data, gas parameters, or other transaction fields without adequate validation?

These are separate research hypotheses, not established vulnerabilities. Each would require its own code trace and reproducible evidence.

---

## 09 — Final Assessment

The investigation successfully mapped the custom bundler URL from the Android configuration layer through the native Bedrock implementation and identified a permissive HTTPS validation behavior.

The local unit test confirmed that private and special-use HTTPS destinations are accepted by the examined validator.

The native implementation also uses the configured URL for outbound JSON-RPC communication.

However, the investigation did not establish remote attacker control over another user's configuration, successful access to internal services, or meaningful security impact.

For that reason, the result should be classified as:

**Potential SSRF-related attack surface — exploitability not established.**

This is not a confirmed vulnerability report.

It is a documented security investigation with reproducible validation evidence and clearly identified limitations.

---

## 10 — Research Notes and References

**Primary source**

```text
bedrock/src/transactions/custom_bundler.rs
```

**Important Rust functions**

- `validate_rpc_url`
- `build_bundler_client`
- `post_json_rpc_to_url`
- `verify_bundler_rpc_entrypoint`
- `prepare_bundler_sponsored_operation`
- `send_user_operation_to_url`

**Android components examined**

- `WalletTransactionRelayerStoreImpl`
- `WalletRelayerSettingViewModel`
- `ux0/z`
- `y00/j`

**Development history**

- Pull request #460 — Prepare transfers with a custom bundler.
- Commit `8e5a76b` — Custom bundler integration.

**Research principle:**

Follow the input. Identify the trust boundary. Find the sink. Test the behavior. Establish attacker control. Prove the impact.

And if the evidence stops, say exactly where it stops.
