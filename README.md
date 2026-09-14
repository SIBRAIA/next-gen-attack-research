# Cross-Layer WebView Trust Hijacking

[**English**](./01-cross-layer-webview-trust-hijacking.md) | [**العربية**](./01-cross-layer-webview-trust-hijacking-ar.md)

**Android WebView · JavaScript Bridge · Native Trust Boundaries · Banking Malware · Malvertising · Future AI-Adaptive Abuse**

This repository documents defensive security research into a cross-layer Android pattern in which web content rendered inside a `WebView` can cross into native Android functionality through JavaScript bridges, runtime script execution, DOM instrumentation, and permission-related workflows.

## Choose a language

- 🇬🇧 **English:** [Full research paper](./01-cross-layer-webview-trust-hijacking.md)
- 🇸🇦 **العربية:** [النسخة العربية الكاملة](./01-cross-layer-webview-trust-hijacking-ar.md)

Both editions use the same section structure, evidence model, Evidence IDs, historical timeline, and disclosure boundaries.

## Core Question

> **What can WebView content do inside an Android application if that content becomes untrusted?**

## Why This Matters

A legitimate website does not need to be compromised at the server for its customer interaction to become unsafe inside a hostile or improperly trusted Android client.

```text
Dynamic Web Content
        ↓
JavaScript Execution
        ↓
Runtime Instrumentation
        ↓
DOM / User Interaction Observation
        ↓
Web-to-Native Bridge
        ↓
Android Application Logic
        ↓
Data / Permission / Device Workflows
```

## Scope

This research concerns **Android applications and Android phones only**.

It does not claim server compromise, universal Android RCE, automatic permission granting, compromise through every advertisement, or attribution of the analyzed sample to a named malware family or actor.

## Evidence Model

Claims are separated into:

- **Sample-confirmed**
- **Platform-confirmed**
- **Threat-intelligence correlation**
- **Research assessment**
- **Not established**

## Historical Finding

The public record shows dynamic Android malware predating SpyNote and SpyMax:

- **2010:** remote-command / botnet-like control
- **2011:** post-install downloadable capability
- **2012:** targeted Android RAT architecture
- **2013:** updateable malicious scripts and commodity RAT tooling
- **2015–2016:** banking overlays and SpyNote
- **2019–2020:** SpyMax consolidation and code propagation
- **2021–2022:** legitimate-site WebView/session abuse and JavaScript/native bridges
- **2024–2026:** MaaS, malvertising, On-Device Fraud, Device Takeover, and international scaling

## Future Threat Model

The research includes a bounded forecast:

> **From static trust hijacking to AI-adaptive trust manipulation.**

This is explicitly labeled as a research forecast, not a capability demonstrated by the analyzed sample.

## Responsible Disclosure

Operational payloads, private implementation identifiers, target-specific mappings, and step-by-step exploitation instructions are intentionally excluded.

## Release

**v2.0 — Final Public Release**

Publication date: **14 September 2026**  
Evidence cut-off: **13 September 2026**

### Integrity

English paper SHA-256:

```text
d403356da2a09cde4468f3b150051d94354c217b4180fc0eeffc2a1310d60065
```

Arabic paper SHA-256:

```text
6cd74446c0c21b9168c2c63a53740610ab2007bfe6e9aabca1e0a38a2efd2355
```
