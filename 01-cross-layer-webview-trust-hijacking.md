# Cross-Layer WebView Trust Hijacking
[**English**](./01-cross-layer-webview-trust-hijacking.md) | [**العربية**](./01-cross-layer-webview-trust-hijacking-ar.md)

## Legitimate-Site Instrumentation, Banking Fraud, Third-Party Content, and Native Android Trust Abuse

**Security Research & Responsible Disclosure Reference**  
**Version:** 2.0 — Final Public Release
**Date:** 14 September 2026  
**Scope:** Android WebView · JavaScript Bridge · DOM Instrumentation · Native Permission Workflows · Banking Malware Correlation · Malvertising  
**Classification:** Defensive Security Research · Technical Reference for Responsible Disclosure
**Review basis:** Analyzed sample · Official Android documentation · Cited public threat intelligence  
**Evidence current through:** 13 September 2026

---

## Executive Summary


> **Scope boundary:** This publication concerns **Android applications and Android phones only**. It analyzes client-side WebView and Android-native trust boundaries. It does not claim compromise of a company's server, website backend, cloud infrastructure, iOS application, desktop application, or unrelated platform.

This research documents a cross-layer Android security pattern in which web content rendered inside a `WebView` is allowed to cross into native Android functionality through a JavaScript bridge.

The analyzed sample combines:

```text
Remote / Dynamic Web Content
        ↓
JavaScript Execution
        ↓
Runtime JavaScript Injection
        ↓
DOM Input / Click Observation
        ↓
JavaScript ↔ Native Bridge
        ↓
Android Application Logic
        ↓
Storage / Network / Permission Workflows
```

The core issue is not a single vulnerable API. It is the composition of:

```text
Dynamic Navigation
    +
JavaScript Enabled
    +
addJavascriptInterface(...)
    +
Sensitive Native Methods
    +
Runtime JavaScript Injection
    +
Weak Origin Isolation
    +
Permission-Oriented Native Workflows
```

This creates a **Web-to-Native trust-boundary failure**.

A legitimate website does not need to be compromised for the user-facing session to become unsafe. If a legitimate banking, authentication, payment, messaging, or account website is opened inside a hostile or improperly trusted WebView, the host application can instrument the page locally, observe user interaction, and move selected data into native code.

This paper describes the pattern as:

> **Cross-Layer WebView Trust Hijacking**

or:

> **Client-side legitimate-site instrumentation through a bridge-enabled Android WebView**

It should **not** automatically be described as server compromise, website RCE, or compromise of the legitimate website for all users.

---

## Evidence Model and Terminology


> **Evidence traceability:** The claim-to-evidence mapping and evidence classifications are provided in **Appendix A — Evidence & Claim Matrix** and **Appendix B — Evidence Boundaries**. Sample-specific identifiers and source locations are retained privately.

This publication deliberately separates four evidence classes:

| Evidence class | Meaning |
|---|---|
| **Sample-confirmed** | Directly observable in the analyzed Android source/sample. |
| **Platform-confirmed** | Security behavior documented by Android / Google. |
| **Threat-intelligence correlation** | Independently documented behavior in public malware research. |
| **Research assessment** | An interpretation derived from the combined evidence and explicitly labeled as such. |

The label **Cross-Layer WebView Trust Hijacking** is a research term used in this publication to describe the observed trust-boundary pattern. It is **not** presented as an official Android vulnerability class, CVE name, or industry-standard taxonomy.

The sample supports the following architectural conclusion:

> **A WebView that can load dynamic content, inject JavaScript, expose native methods, observe DOM interaction, and initiate native workflows creates a security boundary whose failure can move attacker-controlled web state into Android application trust.**

### Sample context and attribution boundary

The analyzed source contains behavior consistent with offensive Android tooling, including DOM-input collection, native data forwarding, permission-oriented workflows, and socket-backed transmission code. This paper analyzes those capabilities as a security pattern.

The source snapshot alone does **not** establish:

```text
a specific threat actor
a specific campaign
a deployment date
a victim population
a relationship to any named malware family
```

Those questions require independent attribution evidence. Malware-family comparisons later in this paper are therefore treated as **correlation**, not identity or authorship.


---


## Public Technical Disclosure Policy

The public edition intentionally uses **genericized code fragments and platform API names** rather than unpublished implementation code.

It focuses on the parts defenders need to understand:

```text
the WebView trust boundary
the dangerous API combinations
the data-flow pattern
the permission-orchestration pattern
the Android phone impact
the detection opportunities
the mitigations
```

It does not publish the full source, private implementation identifiers, operator infrastructure, target-specific mappings, or provenance fingerprints.

This is deliberate: the goal is to document the technique and enable remediation without redistributing a non-public offensive implementation.


---

# 1. Core Finding

The analyzed WebView enables JavaScript and exposes a Java object to the page:

```java
webView.getSettings().setJavaScriptEnabled(true);

webView.addJavascriptInterface(
    new NativeBridge(),
    "Android"
);
```

Conceptually, JavaScript receives a native object similar to:

```javascript
window.Android
```

The existence of `addJavascriptInterface()` is not by itself a vulnerability.

The security impact depends on:

```text
Which pages can reach the bridge?
Which frames can reach the bridge?
Which native methods are exposed?
Can navigation leave a trusted origin?
Can scripts be injected at runtime?
Can DOM activity initiate sensitive native behavior?
Can resulting data leave the device?
```

In the analyzed design, these concerns intersect.

---

# 2. Dynamic Navigation Is Not Origin Validation

The sample accepts network URLs including:

```text
https://
http://
```

and passes them to:

```java
webView.loadUrl(uri);
```

A scheme check is not an origin policy.

All of these satisfy a basic HTTPS prefix check:

```text
https://trusted.example
https://partner.example
https://third-party.example
https://attacker-controlled.example
```

A bridge-enabled WebView should make security decisions using at least:

```text
scheme
host
port
navigation state
frame origin
```

and should not keep a sensitive native bridge exposed across arbitrary navigation.

The code path **accepts and attempts to load** both `http://` and `https://` URLs. Whether a cleartext HTTP navigation succeeds on a particular device/build also depends on the application manifest, Network Security Configuration, target SDK, and platform policy. The sample therefore proves acceptance of HTTP input at the application layer, not universal cleartext reachability on every Android configuration.

---

# 3. Security-Sensitive Native Bridge

The analyzed interface contains multiple `@JavascriptInterface` methods.

Representative capabilities include methods equivalent to:

```text
a bridge data-receive method
a native data-forwarding method
a permission-workflow bridge method
a native data-processing method
```

A reduced representation is:

```java
class NativeBridge {

    @JavascriptInterface
    public void receiveWebData(String value) {
        // Web-originating data enters native code.
    }

    @JavascriptInterface
    public void requestNativeAction(String request) {
        // Web-originating input influences native behavior.
    }
}
```

The effective trust path becomes:

```text
Web JavaScript
      ↓
Native Bridge
      ↓
Application Logic
      ↓
Storage / Network / Android UI
```

The meaningful security property is:

> **Web-controlled state becomes native application input.**

---

# 4. Runtime JavaScript Injection

The sample contains a path equivalent to:

```java
webView.evaluateJavascript(runtimeScript, null);
```

Architecturally:

```text
Runtime Data
    ↓
Decode / Transform
    ↓
evaluateJavascript(...)
    ↓
Currently Loaded Web Document
```

`evaluateJavascript()` is a legitimate Android API.

The risk depends on provenance:

```text
Who controls the script?
Who can modify it?
Which page receives it?
Which native bridge is exposed at that moment?
Which authenticated user session is active?
```

When runtime script execution and a capability-rich native bridge coexist, origin and provenance controls become security-critical.

In the analyzed snapshot, `runtime script data` is consumed by the WebView injection path, but this source tree does not establish where that value is populated at runtime. The paper therefore treats **runtime JavaScript execution** as confirmed while leaving the **provenance of that runtime script** unestablished unless additional runtime evidence is available.

---

# 5. DOM Input Instrumentation

The analyzed JavaScript observes common interactive elements including:

```text
input
textarea
button
a
select
```

and events including:

```text
focus / focusin
input
keydown
keyup
blur
click
resize
```

A deliberately reduced representation is:

```javascript
document.addEventListener("input", event => {
    if (event.target.matches("input, textarea")) {
        // Observe field state.
    }
});
```

The analyzed implementation can pass selected field information into the native layer through the JavaScript bridge.

The trust chain is therefore:

```text
User Input
    ↓
DOM Event
    ↓
Injected JavaScript
    ↓
Native Bridge
    ↓
Native Processing / Storage / Network
```

---

# 6. Native Permission Workflows

Associated native logic references capability classes such as:

```text
Camera
Location
Microphone
Call Log
Contacts
SMS
Phone
File Access
Overlay
All Files Access
Accessibility
Battery Optimization
```

This does **not** mean JavaScript directly grants Android permissions. Android still controls permission dialogs and protected Settings surfaces. Actual grantability also depends on factors such as manifest declarations, Android version, target SDK, application role, special-app-access rules, and distribution policy.

The security problem is:

> **Web-controlled execution can initiate or orchestrate native workflows associated with sensitive Android capabilities.**

This can transform a WebView from a renderer into a **permission-orchestration surface**: the user trusts the web experience, while the application uses that interaction to move the user toward security-sensitive native capabilities.

---

## Exploitation Preconditions and Reachability

The presence of this architecture does not mean every loaded website is automatically exploited.

A practical abuse path requires one or more of the following conditions:

```text
Attacker-controlled page
Compromised trusted origin
Unsafe redirect / navigation control
Third-party script or frame able to execute JavaScript
Compromised runtime-script source
Application-controlled injection logic that targets the loaded page
```

and, at the same time:

```text
The native bridge remains exposed
        +
A callable native method produces a security-relevant effect
```

For permission-related outcomes, Android may still require explicit user interaction or a protected Settings flow. The finding is therefore about **who can initiate and shape the permission journey**, not about silently granting every Android permission.

Likewise, advertising content rendered in a **separate WebView owned by an ad SDK** does not automatically inherit a bridge registered on another WebView. The risk described here applies when third-party content executes in the same bridge-enabled WebView or otherwise reaches the same exposed interface.


---

# 7. Third-Party Frames and Advertising Content

Android's WebView security guidance warns that `addJavascriptInterface()` injects the Java object into **every frame of the WebView, including iframes**.

Therefore:

```text
Host Page
   │
   ├── First-party script
   │
   └── Third-party iframe / embedded content
                     ↓
              Native Bridge
```

The Same-Origin Policy still protects many DOM interactions between different origins, but:

> **DOM origin isolation is not the same thing as native-bridge isolation.**

This distinction matters when pages include advertising, analytics, widgets, embedded payment components, or other third-party frames.

References:

- Android Developers — https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges
- Android Developers — https://developer.android.com/develop/ui/views/layout/webapps/webview

---

# 8. Advertising and Third-Party Distribution Context

## What the analyzed sample proves

The sample does **not** prove distribution through AdMob, Meta Ads, Google Ads, TikTok Ads, or another advertising network.

No ad-network distribution claim should be made solely from this sample.

## Architectural risk

If third-party HTML/JavaScript advertising content is rendered inside the **same bridge-enabled WebView**, it enters a high-risk trust environment.

```text
Application
    ↓
Bridge-enabled WebView
    ↓
Page containing third-party ad / iframe
    ↓
Third-party JavaScript
    ↓
Native bridge exposure
```

Whether a particular advertisement can exploit the bridge depends on the actual frame model, bridge exposure, navigation rules, Android version, and application code.

## Real-world correlation

Public threat intelligence confirms that malicious advertising is already used to distribute Android financial malware:

- **2024 — PWA/WebAPK banking phishing:** ESET documented social-media advertisements leading victims to fake banking / Google Play pages and malicious PWA or WebAPK installation flows.
- **2025 — Crocodilus:** ThreatFabric documented Facebook advertising delivering an Android banking-trojan dropper and later geographic expansion.
- **2026 — StreamRat:** ThreatFabric documented Meta and TikTok advertisements impersonating a free TV-streaming service, reaching an estimated ~570,000 potential victims before leading to an Android banking RAT using Accessibility, overlays, keylogging, and remote control.

These campaigns do not prove that every advertisement can reach a WebView bridge. They demonstrate that advertising is already being used at scale to establish the trust and installation path required by modern Android financial malware.

References:

- https://www.welivesecurity.com/en/eset-research/be-careful-what-you-pwish-for-phishing-in-pwa-applications/
- https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global
- https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat

---

# 9. Legitimate Banking Websites Inside a Hostile WebView

Yes — **at the client/WebView layer**.

No — not automatically at the bank's server.

A legitimate bank can have:

```text
Valid HTTPS
Valid TLS certificate
Uncompromised servers
Correct domain
Secure backend
```

while its website is displayed inside an untrusted Android host application.

The resulting model is:

```text
Legitimate Bank Server
        ↓ HTTPS
Hostile / Unsafe Android WebView
        ↓
Injected Local JavaScript
        ↓
DOM / Cookie / Session Observation
        ↓
Native Application
```

TLS protects the network connection. It does not prevent the Android application that owns the WebView from instrumenting its own rendering context.

---

# 10. Direct Banking-Malware Correlation

## 10.1 Brokewell — 2024

ThreatFabric documented Brokewell loading the **legitimate website** inside its own WebView.

The malware overrides `onPageFinished()`, waits for login activity, then retrieves session cookies and sends them to its command-and-control infrastructure.

This demonstrates the same strategic concept:

> **The legitimate website does not need to be compromised. The hostile Android host controls the client-side WebView context.**

ThreatFabric also reported rapid active development, Accessibility logging, overlays, remote control, and risk to customers of financial institutions.

Reference:

- https://www.threatfabric.com/blogs/brokewell-do-not-go-broke-by-new-banking-malware

## 10.2 Sturnus — 2025

ThreatFabric documented a banking trojan whose HTML overlay engine launches a WebView configured with:

```text
JavaScript
DOM storage
JavaScript bridge
```

The bridge intercepts and forwards victim-entered data to C2.

Sturnus also combines this with Accessibility-based keylogging and remote control.

This is a close public analogue to the:

```text
WebView
→ JavaScript
→ Native Bridge
→ Data Exfiltration
```

path described in this research.

Reference:

- https://www.threatfabric.com/blogs/sturnus-banking-trojan-bypassing-whatsapp-telegram-and-signal

## 10.3 Crocodilus — 2025

Crocodilus combines:

```text
Accessibility abuse
Overlay phishing
Credential interception
Remote control
Hidden / black-screen operation
```

ThreatFabric later documented global expansion and malicious social-media advertising.

References:

- https://www.threatfabric.com/blogs/exposing-crocodilus-new-device-takeover-malware-targeting-android-devices
- https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global

## 10.4 PlayPraetor — 2025

Cleafy documented a large-scale Android RAT / banking-fraud operation infecting more than 11,000 devices in less than three months.

The operation targeted a globally distributed list of nearly 200 banking applications and crypto wallets and used a multi-tenant MaaS infrastructure.

Reference:

- https://www.cleafy.com/cleafy-labs/playpraetors-evolving-threat-how-chinese-speaking-actors-globally-scale-an-android-rat

## 10.5 TrickMo — 2026

ThreatFabric documented active 2026 TrickMo distribution targeting banking, fintech, wallet, and authenticator applications.

Capabilities include:

```text
Fullscreen WebView credential overlays
Keylogging
Accessibility-assisted device control
SMS / notification interception
Screen streaming
On-device network pivoting
```

Campaign tags showed parallel activity involving customers in France, Italy, and Austria.

Reference:

- https://www.threatfabric.com/blogs/new-trickmo-variant-device-take-over-malware-targeting-banking-fintech-wallet-auth-app

## 10.6 StreamRat — 2026

ThreatFabric reported a large advertising-driven campaign using Meta and TikTok ads.

The malware combines:

```text
Credential overlays
UI-tree collection
Keylogging
Accessibility abuse
VNC / remote control
Hidden-screen operation
```

This demonstrates the modern convergence of:

```text
Mass advertising
        +
Social engineering
        +
Permission acquisition
        +
Device takeover
        +
Financial credential theft
```

Reference:

- https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat

---

# 11. Historical Lineage: From Remote C2 to Dynamic Android Fraud Platforms

The historical record suggests that the shift toward **dynamic Android malware** began substantially earlier than SpyNote or SpyMax.

For this paper, "dynamic" does not mean only that a device can be remotely controlled. It describes a progression in which attacker behavior increasingly moves **outside the fixed APK** and into remote commands, downloadable capability, updateable scripts, dynamically supplied web content, and later adaptive decision layers.

A useful historical model is:

```text
Static malicious APK
        ↓
Remote command execution
        ↓
Downloadable / replaceable capability
        ↓
Script / behavior updates
        ↓
Commodity RAT control panels
        ↓
Overlay + Accessibility + WebView abuse
        ↓
Dynamic Web-to-Native fraud workflows
        ↓
Device Takeover / On-Device Fraud
        ↓
Potential AI-adaptive orchestration
```

The dates below represent the **earliest public evidence located in this review**, not proof that no private or earlier implementation existed.

## 11.1 2010 — Geinimi and the Android botnet transition

In December **2010**, Lookout described **Geinimi** as the first Android malware observed in the wild with botnet-like capabilities.

Public analysis documented that it could communicate with remote infrastructure and had code paths capable of receiving commands from a remote server.

Reported capabilities included:

```text
device / location collection
remote-server communication
download of additional applications
application install prompts
remote-command potential
```

This is historically important because it marks an early shift from:

```text
malicious app = fixed local behavior
```

toward:

```text
malicious app
    ↓
remote infrastructure
    ↓
attacker-selected behavior
```

Academic Android-botnet datasets later place Geinimi in **2010** and identify that period as the beginning of documented Android botnet activity.

References:

- SecurityWeek — Sophisticated New Android Trojan “Geinimi” Spreading in China  
  https://www.securityweek.com/sophisticated-new-android-trojan-geinimi-spreading-china/
- University of New Brunswick — Android Botnet Dataset  
  https://www.unb.ca/cic/datasets/android-botnet.html

### Historical interpretation

Geinimi should not be described as a modern RAT equivalent to later SpyNote/SpyMax platforms.

Its importance is architectural:

> **Remote infrastructure was already becoming part of Android malware behavior by 2010.**

That predates SpyNote's public appearance by roughly six years and SpyMax's reported development by roughly nine years.

---

## 11.2 2011 — Downloadable capability becomes operational

The 2011 **DroidDream** generation provides another important step.

Public reporting on DroidDream and DroidDream Light documented malware capable of communicating with command-and-control infrastructure and downloading additional software after installation.

The original DroidDream variants also used privilege-escalation techniques to install additional code with reduced user involvement on vulnerable devices, while later variants retained download capability but could require user confirmation.

The significance is the architectural separation of:

```text
initial application
        ↓
remote infrastructure
        ↓
additional capability
```

Once capability can be obtained after installation, the APK is no longer necessarily the complete attack.

This principle becomes central to later modular RAT and banking-malware ecosystems.

Reference:

- Ars Technica — DroidDream Light  
  https://arstechnica.com/gadgets/2011/06/droiddream-light-a-malware-nightmare-booted-from-android-market-encumbered-boobies-out-of-android-marketdroiddream-light-an-malware-nightware-booted-from-android-market/

---

## 11.3 2012 — Luckycat shows targeted Android RAT development inside APT infrastructure

The **Luckycat** investigation is an important early piece of evidence in this lineage.

In **2012**, Trend Micro discovered two Android applications on infrastructure associated with the Luckycat targeted-attack campaign.

The applications were still in development / proof-of-concept stage, but could:

```text
receive commands from a remote C2
execute attacker-directed actions
collect device information
browse directories
upload files
download files
support a remote-shell capability under development
```

Trend Micro's research explicitly described them as resembling Remote Access Trojans.

This is a major historical point because it demonstrates that by 2012, attackers involved in targeted operations were already designing Android malware around a **remote-command architecture** rather than a fixed payload.

Reference:

- Trend Micro — Adding Android and Mac OS X Malware to the APT Toolbox  
  https://documents.trendmicro.com/assets/wp/wp_adding-android-and-mac-osx-malware-to-the-apt-toolbox.pdf

### Why 2012 matters

This research treats **2012 as an important transition year**.

By this point the public record supports an Android attack architecture containing:

```text
persistent remote C2
        +
attacker-selected commands
        +
file upload / download
        +
planned remote shell
```

That is conceptually much closer to later RAT platforms than simple credential-stealing or premium-SMS malware.

---

## 11.4 Late 2012–2013 — AndroRAT commoditizes remote Android control

**AndroRAT** appeared on underground forums in late **2012** and became publicly discussed in 2013.

It provided a user-friendly remote-control panel and could perform actions including:

```text
monitor / initiate calls
monitor / send SMS
obtain GPS coordinates
activate camera / microphone
access device files
```

By mid-2013, APK binders were being sold that automated the process of inserting AndroRAT into legitimate Android applications.

That matters because the ecosystem moved from:

```text
specialist malware development
```

toward:

```text
reusable RAT
    +
control panel
    +
automated binding / repackaging
    +
lower operator skill requirement
```

This is the beginning of a pattern that later becomes central to Android crimeware-as-a-service.

Reference:

- SecurityWeek — Hackers Sell APK Binders for Google Android Remote Access Tool  
  https://www.securityweek.com/hackers-sell-apk-binders-google-android-remote-access-tool/

---

## 11.5 2013 — updateable scripts make the dynamic model explicit

Trend Micro documented **ANDROIDOS_KSAPP** in 2013 as an Android backdoor family capable of receiving and executing commands from a remote malicious user.

More importantly for this research, Trend Micro reported that the malware could **automatically update its script to evade detection**.

This represents a conceptual jump:

```text
remote command
        ↓
remote behavior adjustment
        ↓
script update
```

The malicious behavior is no longer defined entirely by the version initially installed.

This is an early publicly documented example of the design principle:

> **Keep the installed client stable while changing behavior remotely.**

Reference:

- Trend Micro — Malicious and High-Risk Android Apps Hit 1 Million: Where Do We Go from Here?  
  https://www.trendmicro.com/vinfo/us/security/news/mobile-safety/malicious-and-high-risk-android-apps-hit-1-million-where-do-we-go-from-here

---

## 11.6 2013–2014 — SandroRAT / DroidJack commercialize the RAT model

The **SandroRAT → DroidJack** lineage demonstrates another transition: commercialization.

Public reporting traces:

```text
2013 — Sandroid / SandroRAT lineage
        ↓
2014 — DroidJack
```

DroidJack was sold commercially and provided broad remote-control capabilities without requiring root access.

Reported functions included:

```text
install APKs
copy files
read messages
monitor calls
access contacts
record microphone / camera
retrieve location
execute remote commands
```

This represents the maturation of the Android RAT into a packaged product rather than merely a malware sample.

Reference:

- SecurityWeek — Developers of Android RAT DroidJack Traced to India  
  https://www.securityweek.com/developers-android-rat-droidjack-traced-india/

---

## 11.7 2015–2016 — banking overlays and SpyNote converge with remote control

By **2015**, CERT Polska documented **GMBot**, an Android banking trojan using application overlays and explicitly compared the technique to banking webinjects.

By **2016**, Mandiant documented European campaigns using Android view overlays to imitate legitimate applications and capture banking credentials.

At approximately the same historical point, **SpyNote** became publicly visible.

Public reporting in 2016 described leaked SpyNote builder/tooling with capabilities such as access to:

```text
SMS
microphone
camera
contacts
location
device data
remote-control functions
```

This period is significant because two previously developing lines begin to coexist in the ecosystem:

```text
Remote Administration
        +
Credential / Interface Deception
```

References:

- CERT Polska — GMBot: Android “poor man's webinjects”  
  https://cert.pl/posts/2015/10/lang_plgmbot-androidowa-ubozsza-wersja-webinjectowlang_pllang_engmbot-android-poor-mans-webinjectslang_en-2/
- Mandiant / Google Cloud — Android Overlay Malware Spreading via SMS Phishing in Europe  
  https://cloud.google.com/blog/topics/threat-intelligence/latest-android-overlay-malware-spreading-in-europe/
- Palo Alto Networks — SpyNote reported in 2016  
  https://www.paloaltonetworks.com/blog/2016/07/palo-alto-networks-news-of-the-week-july-30-2016/
- MITRE ATT&CK — SpyNote RAT  
  https://attack.mitre.org/software/S0305/

---

## 11.8 SpyNote and SpyMax — important lineage points, not the origin of dynamic Android control

The available evidence supports the following chronology:

```text
SpyNote publicly visible by 2016
        ↓
SpyMax reported as built around 2019
        ↓
SpyMax source leak reported around 2020
```

Public malware-family naming is inconsistent.

Some industry sources use SpyNote and SpyMax as related or overlapping names, while other reports distinguish development branches and later forks.

Therefore this paper does **not** claim:

```text
SpyNote created SpyMax
SpyMax created SpyNote
every SpyNote sample == every SpyMax sample
```

The public evidence supports the following statement:

> **SpyNote predates the publicly documented SpyMax development date, while SpyMax later became an important consolidation and source-lineage point in the modular Android RAT ecosystem.**

References:

- Group-IB — From SpyMax to Craxs RAT  
  https://www.group-ib.com/blog/craxs-rat-malware/
- UNODC — Developments in Cyber-Enabled Fraud and Technological Innovation (2024)  
  https://www.unodc.org/roseap/uploads/documents/Publications/2024/TOC_Convergence_Report_2024.pdf
- ERNW / Insinuator — SpyMax Technical Analysis  
  https://insinuator.net/2022/09/spymax-the-android-rat-and-it-works-like-that/

---

## 11.9 Why SpyMax remains historically important

Group-IB reports that SpyMax was built around **2019** and that its implementation code was leaked in **2020**, after which it was reused and customized by additional actors.

Independent technical analysis documents a broad feature set including:

```text
remote control
broad Android permissions
Accessibility-based keylogging
camera / microphone access
location
SMS / contacts / files
dynamic loading of additional components
remote commands
```

Its importance is therefore not invention.

It is **consolidation and propagation**.

A reusable architecture plus leaked implementation code can accelerate:

```text
forking
customization
feature transfer
operator adoption
derivative malware families
```

This helps explain the historical significance of the SpyMax → later-derivative ecosystem.

---

## 11.10 2021 — S.O.V.A. demonstrates legitimate-site WebView/session abuse

The public record becomes directly relevant to this paper's WebView thesis in **2021**.

ThreatFabric documented **S.O.V.A.** opening a legitimate URL inside a malware-owned WebView and using Android's `CookieManager` to retrieve authenticated session cookies after login.

The model is:

```text
Legitimate Website
        ↓
Malware-Owned WebView
        ↓
Authenticated Session
        ↓
Session Cookie Extraction
```

S.O.V.A. also used Accessibility Services, overlays, keylogging, and WebView injections.

Reference:

- ThreatFabric — S.O.V.A.  
  https://www.threatfabric.com/blogs/sova-new-trojan-with-fowl-intentions

This is one of the clearest public predecessors to the legitimate-site instrumentation model discussed in this paper.

---

## 11.11 2022 — Xenomorph makes WebView + JavaScript + native bridge explicit in banking malware

In **2022**, ThreatFabric documented **Xenomorph** creating banking-overlay WebViews with JavaScript enabled and a native JavaScript interface registered through `addJavascriptInterface(...)`.

It also used Accessibility Services and dynamically supplied overlay content.

This demonstrates that by 2022 the public Android banking-malware ecosystem contained the combination:

```text
WebView
    +
JavaScript
    +
Native Bridge
    +
Dynamic Overlay Content
    +
Accessibility
```

Reference:

- ThreatFabric — Xenomorph  
  https://www.threatfabric.com/blogs/xenomorph-a-newly-hatched-banking-trojan

This is substantially closer to the cross-layer architecture analyzed in this research than the early RAT generations.

---

## 11.12 2023–2024 — leaked and cracked RAT tooling becomes a hostile supply chain

The Android RAT ecosystem also developed a tooling-supply-chain problem.

CYFIRMA reported in August 2023 that some cracked CraxsRAT copies were backdoored and that cracked builders distributed for Windows had appeared with pre-existing malware or ransomware.

Broadcom independently noted that CraxsRAT had been cracked and leaked for free, widening access to the tool.

This creates a second dynamic ecosystem:

```text
Private / Commercial RAT
        ↓
Cracking / Reverse Engineering
        ↓
Leaked Builders / Code
        ↓
Backdoored Cracked Tooling
        ↓
Threat Actors Become Targets
        ↓
Credential / Source / Infrastructure Exposure
```

This can accelerate both code propagation and counter-compromise inside criminal ecosystems.

The cited evidence supports the existence of backdoored cracked tooling and wider code distribution, but it does not establish actor-specific attribution for downstream compromises.

References:

- CYFIRMA — Unmasking EVLF DEV / CraxsRAT  
  https://www.cyfirma.com/research/unmasking-evlf-dev-the-creator-of-cypherrat-and-craxsrat/
- Broadcom — CraxsRAT  
  https://www.broadcom.com/support/security-center/protection-bulletin/craxsrat
- Group-IB — From SpyMax to Craxs RAT  
  https://www.group-ib.com/blog/craxs-rat-malware/

---

## 11.13 2024–2026 — convergence, MaaS, malvertising, and international scaling

The reviewed evidence indicates that **2024–2026 represents an acceleration and convergence phase**, not the origin of the underlying techniques.

### 2024 — legitimate-site abuse and active refinement

Public reporting documented:

```text
Brokewell legitimate-site WebView/session abuse
rapid Device Takeover development
PWA/WebAPK banking phishing promoted through social media
Medusa On-Device Fraud capability
ToxicPanda Account Takeover / On-Device Fraud
```

**Assessment:** 2024 shows mature combinations of trusted web interaction, session abuse, overlays, remote control, and native privilege.

### 2025 — specialization and scalable fraud platforms

Public reporting documented:

```text
Crocodilus geographic expansion
social-media malvertising
PlayPraetor MaaS scaling
Sturnus WebView JavaScript bridges
Accessibility logging
hidden remote control
operator infrastructure specialization
```

**Assessment:** 2025 shows the transition from individual techniques to integrated fraud platforms.

### 2026 — international acquisition and Device Takeover

Public reporting documented:

```text
TrickMo parallel banking / wallet campaigns
StreamRat Meta / TikTok acquisition
credential overlays
keylogging
Accessibility abuse
screen streaming
hidden-screen operation
Device Takeover
```

**Assessment:** By 2026, the components developed across the previous decade can be assembled into industrialized international fraud operations.

---

## 11.14 Historical Model

The resulting timeline is:

```text
2010
Geinimi
remote C2 / botnet-like Android control
        ↓
2011
DroidDream
downloadable post-install capability
        ↓
2012
Luckycat Android RAT
targeted C2 + upload/download + remote-shell development
        ↓
late 2012–2013
AndroRAT
commodity remote-control panel + binders
        ↓
2013
KSAPP
remote commands + automatic script updates
        ↓
2013–2014
SandroRAT / DroidJack
commercial RAT maturation
        ↓
2015–2016
banking overlays + SpyNote
        ↓
2019–2020
SpyMax consolidation + implementation leak
        ↓
2021
S.O.V.A.
legitimate-site WebView/session abuse
        ↓
2022
Xenomorph
WebView + JavaScript + native bridge + dynamic overlays
        ↓
2023–2024
cracked / leaked RAT supply-chain expansion
        ↓
2024–2026
MaaS + malvertising + ODF + Device Takeover
        ↓
Next-stage threat model
AI-adaptive trust manipulation
```

---

## 11.15 What the timeline does — and does not — prove

The evidence supports a long architectural evolution.

It does **not** prove that one developer or one hidden controller designed the entire progression.

A plausible alternative is normal adversarial evolution:

```text
successful technique
        ↓
copying / reuse
        ↓
tool leakage
        ↓
commercialization
        ↓
competition
        ↓
feature convergence
```

Private tooling may also have existed before its first public discovery.

For this reason, the correct historical language is:

> **earliest publicly documented evidence**

not:

> **first implementation ever**

Likewise, the early appearance of these architectural ideas does not by itself prove that their developers predicted the exact attack landscape of 2026.

What it does show is that several principles associated with modern adaptive malware were recognized remarkably early:

```text
keep remote control outside the APK
separate capability from initial installation
change behavior after deployment
reduce operator skill through reusable tooling
reuse trusted interfaces
move decisions toward remote infrastructure
```

Those principles later became foundational to modern Android fraud ecosystems.

---

## 11.16 Historical Conclusion

The historical record supports the following conclusion:

> **Dynamic Android malware predates SpyNote and SpyMax by several years. Public evidence shows remote-command / botnet-like Android control by 2010, downloadable post-install capability by 2011, targeted Android RAT development with C2-directed upload/download and remote-shell functionality by 2012, and automatically updateable malicious scripting by 2013. AndroRAT, SandroRAT and DroidJack then helped commoditize remote control. SpyNote became publicly visible by 2016, while SpyMax emerged later as an important consolidation and code-propagation point rather than the origin of dynamic Android control. By 2021–2022, WebView/session abuse and JavaScript/native-bridge banking overlays were public, and 2024–2026 represents convergence, commercialization, malvertising-driven acquisition, and international scaling rather than invention.**

This timeline traces the architectural transition itself rather than beginning only with later product names such as SpyNote or SpyMax.

---

# 12. Impact Assessment: Trust Conversion and Permission Acquisition

No universal benchmark ranks all Android attack techniques by a single measure of impact or effectiveness. The relevant assessment is therefore architectural and evidence-based.

The research assessment is:

> **Cross-layer trust conversion — combining trusted web content, WebView/native bridges, social engineering, Accessibility or overlay workflows, and native permission prompts — can support high-impact permission-acquisition and device-takeover workflows in modern Android financial malware.**

The technique does not rely on one technical bypass.

It combines multiple forms of trust:

```text
Trusted Brand
    +
Legitimate or convincing Web Content
    +
User Interaction
    +
Android Native UI
    +
Permission Workflow
    +
Accessibility / Overlay / Remote Control
```

The user may believe they are interacting with:

```text
their bank
their payment provider
their browser
a system update
a security check
a trusted application
```

while the malware progressively converts that trust into native application capability and user-approved security access.

Google explicitly warns that granting Accessibility permission to a malicious application can allow an attacker to gain control of the device and steal sensitive/private data such as banking information.

Reference:

- https://blog.google/security/whats-new-in-android-security-privacy-2025/

---

# 13. The Trust-Conversion Model

**Trust Conversion** is an analytical model used in this paper; it is not presented as an established Android, OWASP, MITRE ATT&CK, or academic taxonomy.

The most useful way to understand this class of attack is not “permission theft”.

Android still requires the user or system to authorize many protected capabilities.

The attacker instead performs **trust conversion**:

```text
Brand Trust
    ↓
Web Trust
    ↓
Interaction Trust
    ↓
Native UI Trust
    ↓
Permission Approval
    ↓
Device Capability
```

The technical advantage is psychological and architectural at the same time.

A permission prompt that appears unexpectedly is suspicious.

A permission prompt that appears to be part of a trusted banking, fraud-prevention, account-recovery, update, or verification journey may be more persuasive.

This helps explain why Accessibility, Overlay, notification access, screen sharing, and related workflows remain attractive to modern financial malware.

---

# 14. Brand Trust as an Attack Surface Against Customers

A company does not need to have its server compromised for attackers to weaponize its customer trust.

Attackers can reproduce or abuse:

```text
Brand identity
Legitimate URLs
Login flows
Account-recovery journeys
Customer-support language
Security warnings
Application screenshots
Payment / verification flows
```

The resulting path can be:

```text
Company Trust
      ↓
Hostile Client Environment
      ↓
Customer Interaction
      ↓
Credential / Session / Permission Capture
```

Therefore:

> **A company may remain technically uncompromised while its customer relationship is being actively weaponized.**

A customer-facing journey that assumes the client device is trustworthy can become a usable surface for fraud against the company's own customers.

This does **not** mean a company is responsible for arbitrary malware installed on a customer's device.

It means organizations should not assume that:

```text
HTTPS
MFA
A secure backend
A trusted brand
```

are sufficient when the client device or rendering environment is hostile.

---

# 15. Implications for Financial Institutions

Modern On-Device Fraud is designed to make fraudulent activity originate from the victim's legitimate device.

This can weaken controls based only on:

```text
Device identity
IP reputation
Known browser
Valid session
Correct password
Correct OTP
```

because malware may operate after authentication or inside an already trusted user environment.

A resilient defensive model includes:

```text
Device integrity
Session integrity
Behavioral analysis
Transaction context
Risk-based authentication
Out-of-band confirmation
Trusted app / browser binding
Anti-overlay / anti-tamper controls
Runtime application protection
Fraud telemetry
```

ThreatFabric's Brokewell research specifically notes the challenge that Device Takeover creates for fraud systems that rely heavily on device identification or fingerprinting.

---

# 16. What the Analyzed Sample Proves

The sample supports these conclusions:

- JavaScript is enabled inside a WebView.
- A native JavaScript bridge is exposed.
- Dynamic network content can be loaded.
- Runtime JavaScript injection exists.
- DOM input and interaction events are observed.
- Web-originating data can cross into native code.
- Native data-handling / transmission paths exist.
- Web interaction can influence permission-related native workflows.
- Sensitive Android capability classes are referenced.
- The boundary between web content and native application logic is weak.

---

# 17. What the Sample Does Not Prove

The sample alone does **not** prove:

```text
Compromise of a bank server
Compromise of a legitimate website backend
Remote code execution on the website server
Persistent modification of the legitimate website
Self-propagation between websites
Automatic infection through every advertisement
Compromise of users outside the hostile application
Automatic Android permission granting without user/system interaction
A universal Android sandbox escape
```

These distinctions should remain explicit in responsible disclosure.

---

# 18. Severity and Triage Guidance

This paper does **not** assign a universal CVSS score or a universal product severity. The same architectural pattern can range from a design smell to a critical product vulnerability depending on reachability and demonstrated impact.

### Architecture review priority

Treat the pattern as **high-priority for security review** when these conditions coexist:

```text
Untrusted / dynamic content
        +
Legacy bridge exposure
        +
Sensitive native methods
```

### Product-specific severity

A **High** or **Critical** application-specific rating may be justified only when authorized testing demonstrates a reliable path to outcomes such as:

```text
Account takeover
Credential/session theft
Unauthorized financial transaction
High-impact data exfiltration
Sensitive permission acquisition
Device takeover
```

The assessment should account for required user interaction, origin control, frame reachability, authentication state, Android version, permission prerequisites, and whether the impact can be reproduced.

Do not assign a Critical rating solely because `addJavascriptInterface()` exists.

---

# 19. Responsible-Disclosure Summary

The Android application exposes security-sensitive native methods through a WebView JavaScript bridge while the same WebView can load dynamic network content and execute injected JavaScript.

The analyzed implementation also instruments DOM interaction and can move selected web-originating data into native application logic. Permission-related native workflows are reachable from the same Web-to-Native trust path.

Android documents that legacy `addJavascriptInterface()` objects are injected into every frame of the WebView, including iframes, making origin and third-party content controls security-critical.

If a legitimate website is navigated to inside the affected WebView, the host application can instrument that page locally without requiring compromise of the website's server.

The finding is classified here as a **Web-to-Native trust-boundary failure / Cross-Layer WebView Trust Hijacking**, not as demonstrated server-side compromise.

Public threat intelligence shows close real-world analogues in Android financial malware, including Brokewell's legitimate-site WebView session theft, Sturnus' JavaScript-bridge credential collection, and later banking-trojan campaigns combining overlays, Accessibility abuse, malvertising, and Device Takeover.

---

# 20. Defensive Architecture

## 20.1 Separate trusted and untrusted WebViews

```text
Trusted internal WebView
      ↓
Minimal native capability

External / third-party content
      ↓
System browser or isolated WebView
      ↓
No sensitive bridge
```

## 20.2 Enforce exact origin allowlists

Validate:

```text
scheme
host
port
redirect destination
navigation state
```

Do not use broad patterns such as `https://*`.

## 20.3 Remove bridge capability when trust changes

```text
Trusted Origin
    ↓
Bridge enabled

Untrusted Origin
    ↓
Bridge unavailable
```

## 20.4 Minimize native methods

Where a Web-to-Native messaging channel is necessary, Android's current guidance recommends origin-scoped messaging such as `WebViewCompat.addWebMessageListener(...)`, which uses `allowedOriginRules` and provides sender-origin metadata. This is materially different from treating legacy `addJavascriptInterface()` as if it were origin-aware.

Origin scoping is **hardening, not a complete trust solution**. If attacker-controlled JavaScript executes inside an already allowed origin—for example through a same-origin script compromise or XSS—the origin rule may still be satisfied. Narrow capabilities, payload/schema validation where appropriate, native authorization, user intent, and web-content integrity remain necessary. OWASP MASTG explicitly makes this distinction in its current WebView bridge guidance and demonstrations, and separately recommends restricting the native functionality exposed through WebView bridges.

Prefer:

```text
Narrow API
    ↓
Strict schema validation
    ↓
Native policy engine
    ↓
Allow / Deny
```

rather than general-purpose web-to-native command surfaces.

## 20.5 Keep permission decisions out of web content

Sensitive Android permission workflows should require:

```text
Explicit user intent
Trusted native UI
Current trusted origin
Valid application state
Feature-specific justification
Risk-policy approval
```

A DOM click should not be sufficient authority for Accessibility, Overlay, SMS, microphone, or similarly sensitive workflows.

## 20.6 Disable unnecessary WebView capabilities

Android recommends disabling risky file-access settings where they are unnecessary:

```java
setAllowUniversalAccessFromFileURLs(false);
setAllowFileAccess(false);
setAllowContentAccess(false);
```

and using `WebViewAssetLoader` for local application content.

Reference:

- https://developer.android.com/privacy-and-security/risks/webview-unsafe-file-inclusion

## 20.7 Do not treat HTTPS as client-integrity protection

HTTPS protects traffic in transit.

It does not protect the page from the application that owns its rendering environment.

## 20.8 Add client-side fraud controls

For financial and high-risk services, consider:

```text
Application attestation
Device-risk signals
Runtime integrity checks
Transaction signing / binding
Behavioral fraud detection
Step-up authentication
Independent transaction confirmation
Session anomaly detection
```

---

# 21. Detection Perspective

High-signal APIs include:

```text
setJavaScriptEnabled(true)
addJavascriptInterface(...)
evaluateJavascript(...)
loadUrl(dynamicValue)
@JavascriptInterface
setAllowUniversalAccessFromFileURLs(true)
```

Individually, these APIs have legitimate uses.

Detection is more informative when these APIs are analyzed relationally:

```text
Dynamic Web Content
      ↓
JavaScript Execution
      ↓
Native Bridge
      ↓
DOM / Runtime Injection
      ↓
Sensitive Native Capability
      ↓
Storage / Network / Permission Workflow
```

Security tooling should reconstruct the trust path rather than report isolated APIs.

---

# 22. Research Assessment

Based on the analyzed sample and public threat intelligence:

1. **The Web-to-Native pattern is technically real and security-relevant.**
2. **Legitimate websites can be instrumented locally inside a hostile WebView without server compromise.**
3. **The same strategic model has direct analogues in banking malware such as Brokewell and Sturnus.**
4. **Advertising and social-media campaigns are proven distribution channels for modern Android financial malware.**
5. **Accessibility, overlays, WebViews, keylogging, and remote control are increasingly combined rather than used independently.**
6. **The dynamic Android lineage predates SpyNote and SpyMax: remote-command / botnet-like control is publicly documented by 2010, downloadable post-install capability by 2011, targeted RAT architecture by 2012, and automatic script updating by 2013; later SpyNote/SpyMax generations represent consolidation and commoditization rather than the origin of dynamic Android control.**
7. **The cross-layer WebView-to-Native pattern can support high-impact permission-acquisition and device-takeover workflows in modern Android financial malware. “Trust Conversion” is the analytical model used in this paper to explain that progression; the reviewed evidence does not establish a universal ranking among Android attack techniques.**
8. **The defensive problem extends beyond protecting company servers: organizations must also protect customer trust, sessions, transactions, and client-side execution environments.**
9. **A plausible next-stage threat model is AI-adaptive trust manipulation, where behavioral telemetry informs the timing or content of hostile client-side interaction; this is a research forecast, not a capability established by the analyzed sample.**


## 22.1 Future Threat Model — From Static Trust Hijacking to AI-Adaptive Trust Manipulation

### Status: Research Assessment / Forecast

The analyzed sample does **not** demonstrate an AI agent controlling the attack loop.

However, the observable primitives in this research can support a future adaptive architecture if they are connected to an AI decision layer.

The relevant distinction is between **emotion recognition** and **behavioral inference**.

Events such as:

```text
focus / focusin
input
keydown / keyup
blur
click
resize
navigation state
```

do not reliably reveal a person's internal emotional state.

They can, however, act as **behavioral proxies** for observable conditions such as:

```text
hesitation
repeated input correction
abandonment
navigation reversal
interaction speed
workflow stage
response to prompts
```

A future adaptive attack model could therefore resemble:

```text
Legitimate / convincing web experience
        ↓
Interaction telemetry
        ↓
Behavioral state estimation
        ↓
AI / agent decision layer
        ↓
Selection of the next interaction strategy
        ↓
Client-side content adaptation or native workflow timing
        ↓
Outcome observation
        ↓
Policy adaptation
```

The security significance is not that an AI model can "read emotions."

It is that the attacker may no longer need to encode every decision as a fixed rule.

A decision layer could potentially determine whether to:

```text
continue showing the normal page
delay a suspicious action
change explanatory content
alter the timing of a native workflow
stop the interaction when resistance is detected
```

This would convert a static social-engineering sequence into a **closed-loop adaptive interaction system**.

### Why this forecast is technically credible

Public reporting already documents several adjacent developments:

1. **AI-assisted social engineering is operational.**  
   Mandiant / Google Threat Intelligence Group reports adversaries using LLMs for increasingly personalized social engineering.

2. **LLMs are being queried during malware execution.**  
   M-Trends 2026 and Google's AI risk research describe malware families such as PROMPTFLUX and PROMPTSTEAL querying language models during execution for dynamic code or command generation.

3. **Agentic attack behavior is emerging.**  
   Google's 2026 AI risk assessment describes a transition during 2025 from AI as a productivity tool toward adaptive tools and agents operating with less human oversight.

4. **Modern Android financial malware already supplies the non-AI control primitives.**  
   Public campaigns document combinations of WebViews, overlays, Accessibility, keylogging, remote control, dynamic C2 instructions, and device-takeover workflows.

The remaining step is the integration of those telemetry and control primitives with a real-time decision model.

### What is observed vs forecast

```text
OBSERVED / PUBLICLY DOCUMENTED

WebView instrumentation                     ✓
Native JavaScript bridges                   ✓
DOM / credential collection                 ✓
Accessibility abuse                         ✓
Overlay attacks                             ✓
Remote device control                       ✓
Dynamic C2 instructions                     ✓
AI-assisted social engineering              ✓
Malware querying LLMs during execution      ✓
Agentic / adaptive offensive tooling        ✓


NOT ESTABLISHED BY THIS RESEARCH

DOM telemetry
      ↓
AI behavioral model
      ↓
autonomous live WebView adaptation
      ↓
permission-workflow optimization
      ↓
closed-loop Android financial fraud
                                                ?
```

Accordingly, the following should be treated as a **future threat model**, not a statement that such a complete Android banking-malware chain has already been publicly demonstrated at scale.

### Defensive implication

Static client-side protections are increasingly insufficient when an attacker can vary behavior in response to the environment or the user.

Organizations should assume that future hostile clients may optimize their interaction strategy dynamically and should therefore prioritize controls that do not depend solely on the appearance or sequence of the client UI.

Relevant defensive principles include:

```text
independent transaction confirmation
server-side behavioral fraud analytics
device and application integrity signals
session anomaly detection
strong origin separation
minimal Web-to-Native capability
runtime detection of Accessibility / overlay abuse
high-risk action binding to trusted application state
```

The research assessment is therefore:

> **The likely next evolution is not simply AI-generated phishing content. It is AI-assisted orchestration of the customer interaction itself — using observable behavior to decide when and how to manipulate a hostile client environment.**

This forecast is consistent with the wider movement from static malware toward adaptive and agentic offensive tooling, but the complete Android-specific closed loop described above remains **not yet established by the evidence reviewed for this publication**.


---

# 23. References

## Android / Platform

1. [Android Developers — WebView: Native Bridges](https://developer.android.com/privacy-and-security/risks/insecure-webview-native-bridges)
2. [Android Developers — Build Web Apps in WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview)
3. [Android Developers — WebViews: Unsafe File Inclusion](https://developer.android.com/privacy-and-security/risks/webview-unsafe-file-inclusion)
4. [Android Developers — WebSettings API](https://developer.android.com/reference/android/webkit/WebSettings)
5. [Android Developers — Access Native APIs with JavaScript Bridge](https://developer.android.com/develop/ui/views/layout/webapps/native-api-access-jsbridge)
6. [Android Developers — WebView API Reference](https://developer.android.com/reference/android/webkit/WebView)
7. [AndroidX WebKit — WebViewCompat](https://developer.android.com/reference/androidx/webkit/WebViewCompat)
8. [Android Developers — Security Checklist](https://developer.android.com/privacy-and-security/security-tips)
9. [Google Security Blog — What's New in Android Security and Privacy in 2025](https://blog.google/security/whats-new-in-android-security-privacy-2025/)
10. [Google Security Blog — Keeping Google Play & Android App Ecosystems Safe in 2025](https://blog.google/security/keeping-google-play-android-app-ecosystem-safe-2025/)
## Early Android Dynamic-Control History

11. [SecurityWeek — Sophisticated New Android Trojan “Geinimi” Spreading in China](https://www.securityweek.com/sophisticated-new-android-trojan-geinimi-spreading-china/)
12. [University of New Brunswick — Android Botnet Dataset](https://www.unb.ca/cic/datasets/android-botnet.html)
13. [Ars Technica — DroidDream Light](https://arstechnica.com/gadgets/2011/06/droiddream-light-a-malware-nightmare-booted-from-android-market-encumbered-boobies-out-of-android-marketdroiddream-light-an-malware-nightware-booted-from-android-market/)
14. [Trend Micro — Adding Android and Mac OS X Malware to the APT Toolbox](https://documents.trendmicro.com/assets/wp/wp_adding-android-and-mac-osx-malware-to-the-apt-toolbox.pdf)
15. [SecurityWeek — Hackers Sell APK Binders for Google Android Remote Access Tool](https://www.securityweek.com/hackers-sell-apk-binders-google-android-remote-access-tool/)
16. [Trend Micro — Malicious and High-Risk Android Apps Hit 1 Million](https://www.trendmicro.com/vinfo/us/security/news/mobile-safety/malicious-and-high-risk-android-apps-hit-1-million-where-do-we-go-from-here)
17. [SecurityWeek — Developers of Android RAT DroidJack Traced to India](https://www.securityweek.com/developers-android-rat-droidjack-traced-india/)
## Banking / RAT Historical Lineage

18. [CERT Polska — GMBot: Android “poor man's webinjects”](https://cert.pl/posts/2015/10/lang_plgmbot-androidowa-ubozsza-wersja-webinjectowlang_pllang_engmbot-android-poor-mans-webinjectslang_en-2/)
19. [Mandiant / Google Cloud — Android Overlay Malware Spreading via SMS Phishing in Europe](https://cloud.google.com/blog/topics/threat-intelligence/latest-android-overlay-malware-spreading-in-europe/)
20. [Palo Alto Networks — SpyNote reported in 2016](https://www.paloaltonetworks.com/blog/2016/07/palo-alto-networks-news-of-the-week-july-30-2016/)
21. [MITRE ATT&CK — SpyNote RAT (S0305)](https://attack.mitre.org/software/S0305/)
22. [Group-IB — Craxs RAT, from SpyMax lineage to banking-fraud tooling](https://www.group-ib.com/blog/craxs-rat-malware/)
23. [UNODC — Developments in Cyber-Enabled Fraud and Technological Innovation (2024)](https://www.unodc.org/roseap/uploads/documents/Publications/2024/TOC_Convergence_Report_2024.pdf)
24. [ERNW / Insinuator — SpyMax Technical Analysis](https://insinuator.net/2022/09/spymax-the-android-rat-and-it-works-like-that/)
25. [ThreatFabric — S.O.V.A.](https://www.threatfabric.com/blogs/sova-new-trojan-with-fowl-intentions)
26. [ThreatFabric — Xenomorph](https://www.threatfabric.com/blogs/xenomorph-a-newly-hatched-banking-trojan)
27. [CYFIRMA — Unmasking EVLF DEV / CraxsRAT](https://www.cyfirma.com/research/unmasking-evlf-dev-the-creator-of-cypherrat-and-craxsrat/)
28. [Broadcom — CraxsRAT](https://www.broadcom.com/support/security-center/protection-bulletin/craxsrat)
## Modern Android Financial Malware / Threat Intelligence

29. [ThreatFabric — Brokewell](https://www.threatfabric.com/blogs/brokewell-do-not-go-broke-by-new-banking-malware)
30. [ThreatFabric — Sturnus](https://www.threatfabric.com/blogs/sturnus-banking-trojan-bypassing-whatsapp-telegram-and-signal)
31. [ThreatFabric — Exposing Crocodilus](https://www.threatfabric.com/blogs/exposing-crocodilus-new-device-takeover-malware-targeting-android-devices)
32. [ThreatFabric — Crocodilus: Evolving Fast, Going Global](https://www.threatfabric.com/blogs/crocodilus-mobile-malware-evolving-fast-going-global)
33. [ThreatFabric — New TrickMo Variant](https://www.threatfabric.com/blogs/new-trickmo-variant-device-take-over-malware-targeting-banking-fintech-wallet-auth-app)
34. [ThreatFabric — StreamRat: From Meta Ads to Full Device Takeover](https://www.threatfabric.com/blogs/from-meta-ads-to-full-device-takeover-uncovering-streamrat)
35. [Cleafy — PlayPraetor](https://www.cleafy.com/cleafy-labs/playpraetors-evolving-threat-how-chinese-speaking-actors-globally-scale-an-android-rat)
36. [Cleafy — ToxicPanda](https://www.cleafy.com/cleafy-labs/toxicpanda-a-new-banking-trojan-from-asia-hit-europe-and-latam)
37. [Cleafy — Medusa Reborn](https://www.cleafy.com/cleafy-labs/medusa-reborn-a-new-compact-variant-discovered)
38. [ESET — Phishing in PWA / WebAPK Applications](https://www.welivesecurity.com/en/eset-research/be-careful-what-you-pwish-for-phishing-in-pwa-applications/)
39. [ESET — Threat Report H2 2024](https://www.welivesecurity.com/en/eset-research/eset-threat-report-h2-2024/)
## AI / Adaptive Offensive Tooling

40. [Google Cloud / Mandiant — AI Risk and Resilience (2026)](https://cloud.google.com/security/resources/ai-risk-and-resilience)
41. [Google Cloud / Mandiant — M-Trends 2026 Executive Edition](https://cloud.google.com/security/resources/m-trends-executive-edition)
42. [Google Cloud / GTIG — AI Threat Tracker: Advances in Threat Actor Usage of AI Tools](https://cloud.google.com/blog/topics/threat-intelligence/threat-actor-usage-of-ai-tools)
## Mobile Security Testing / Defensive Standards

43. [OWASP MAS — MASTG-KNOW-0018: WebViews](https://mas.owasp.org/MASTG/knowledge/android/MASVS-PLATFORM/MASTG-KNOW-0018/)
44. [OWASP MAS — MASTG-TEST-0334: Native Code Exposed Through WebViews](https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0334/)
45. [OWASP MAS — MASTG-BEST-0035: Prefer Origin Scoped Messaging Over Legacy JavaScript Bridges](https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0035/)
46. [OWASP MAS — MASTG-TEST-0252: References to Local File Access in WebViews](https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0252/)
47. [OWASP MAS — MASTG-BEST-0058: Restrict Native Functionality Exposed Through WebView Bridges](https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0058/)
---

# 24. Final Disclosure Boundary

This publication intentionally omits operational payloads, campaign-control logic, target-specific element mappings, and step-by-step exploitation instructions.

The final review cross-checked privately retained artifact-integrity records, the principal static code-path mappings, the native data-send path, current Android bridge guidance, and the principal historical/threat-intelligence claims cited in the paper.

Claims in this paper are intentionally bounded by the evidence model above. Correlation with another malware family does not imply shared authorship, shared infrastructure, or direct code lineage unless a cited source establishes it.

The purpose is to document:

```text
the trust-boundary failure
the observable security impact
the relationship to real-world financial malware
the conditions required for exploitation
the defensive architecture required to prevent it
```

The central lesson is:

> **A legitimate website can remain secure at the server while its trusted customer interaction is hijacked inside a hostile mobile client.**

The corresponding defensive principle is:

> **Do not allow web trust to become native trust without an explicit, origin-aware, capability-minimized policy boundary.**

---


## Research Handling and Publication Boundary

This publication is designed to document defensive security findings without redistributing the underlying non-public artifact or asserting unverified authorship.

It intentionally excludes:

```text
private source archives
full unpublished implementation code
personal identifiers
operator identities
private infrastructure details
target-specific exploitation instructions
unverified attribution claims
```

Artifact-integrity records and non-public provenance details are retained separately for authorized review and responsible disclosure.

This publication is a technical research document, not legal advice and not an attribution report.


---

# Appendix A — Evidence & Claim Matrix

This appendix is the audit layer for the publication. It maps each material claim to the exact evidence class supporting it.

## A.1 Evidence classifications

| Label | Meaning |
|---|---|
| **CONFIRMED — SAMPLE** | Directly observable in the analyzed source/sample. |
| **CONFIRMED — PLATFORM** | Behavior explicitly documented by Android / Google. |
| **CORRELATED — THREAT INTEL** | Independently documented in public malware or fraud research; supports similarity or historical context, not common authorship. |
| **ASSESSMENT** | Research interpretation derived from multiple evidence classes. |
| **NOT ESTABLISHED** | A claim for which the reviewed evidence is insufficient. |

## A.2 Research Material Handling

The technical findings in this publication were derived from a **non-public Android security sample** reviewed for defensive research.

The underlying sample, source archive, implementation-specific class names, package names, developer identifiers, internal naming conventions, source hashes, and other provenance indicators are **not published**.

This publication intentionally exposes only the minimum technical material necessary to describe the security pattern:

```text
public Android APIs
security-relevant configuration patterns
genericized Web-to-Native data flows
genericized permission-workflow patterns
defensive detection logic
mitigation guidance
```

Implementation-specific identifiers have been removed or generalized because they are not necessary to understand or remediate the security issue and may reveal unrelated provenance or attribution information.

The publication does **not** attribute the analyzed sample to a particular malware family, developer, operator, research group, or historical lineage.

Any historical discussion of public malware families in this paper is based only on independently cited public reporting and is included for threat-model comparison, not for attribution of the analyzed sample.

Detailed source references, file fingerprints, exact class/method identifiers, and chain-of-custody information are retained privately for authorized security review and responsible disclosure where necessary.


## A.3 Claim Matrix

| ID | Public claim | Public evidence basis | Classification |
|---|---|---|---|
| **E-01** | A WebView can be configured to load dynamic network URLs while remaining inside the application. | Observed in the non-public sample; implemented through standard Android WebView URL-loading APIs. | **CONFIRMED — SAMPLE** |
| **E-02** | JavaScript execution and a native JavaScript bridge can coexist in the same WebView. | Observed using standard Android `setJavaScriptEnabled(...)` and `addJavascriptInterface(...)` APIs. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-03** | Broad file/content access settings increase the sensitivity of a bridge-enabled WebView. | Observed in the sample; Android documents the risks of permissive WebView file/content access. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-04** | Runtime JavaScript can be executed inside the currently loaded WebView document. | Observed using standard Android `evaluateJavascript(...)`. | **CONFIRMED — SAMPLE** |
| **E-05** | Page lifecycle events can be used to trigger page instrumentation after navigation completes. | Observed in the sample through WebView page-completion handling. | **CONFIRMED — SAMPLE** |
| **E-06** | Injected JavaScript can observe common DOM input and interaction events. | Observed in the sample through listeners attached to input-oriented DOM events. | **CONFIRMED — SAMPLE** |
| **E-07** | DOM-derived values can cross from JavaScript into native Android code through a bridge. | Observed in the sample; Android documents this behavior for annotated bridge methods. | **CONFIRMED — SAMPLE + PLATFORM** |
| **E-08** | Web-originating data can enter a native application processing / send path. | A static native processing and send path is present in the sample. Runtime network reachability is not claimed from source review alone. | **CONFIRMED — SAMPLE (STATIC PATH)** |
| **E-09** | Web interaction can initiate native permission or capability workflows. | Observed in the sample through a Web-to-Native permission/capability path. Final authorization remains controlled by Android/user interaction. | **CONFIRMED — SAMPLE** |
| **E-10** | Sensitive Android capabilities can be placed behind Web-to-Native workflows. | The sample references categories including camera, location, microphone/audio, call logs, contacts, SMS/phone, file access, overlay, all-files access, Accessibility, and battery-related settings. | **CONFIRMED — SAMPLE** |
| **E-11** | Legacy `addJavascriptInterface()` lacks origin-based access control and can be exposed to frames/iframes. | Android platform documentation. | **CONFIRMED — PLATFORM** |
| **E-12** | A legitimate website can be instrumented locally inside the application-owned WebView without proving server compromise. | Supported by the observed WebView + runtime script + DOM + bridge composition; independently correlated with public banking-malware research. | **CONFIRMED — SAMPLE CAPABILITY + CORRELATED — THREAT INTEL** |
| **E-13** | Similar WebView/native-bridge data-collection patterns exist in real Android banking malware. | Public reporting on families such as Xenomorph and Sturnus. | **CORRELATED — THREAT INTEL** |
| **E-14** | Malvertising and social advertising are real acquisition channels for modern Android financial malware. | Public reporting from ESET, ThreatFabric and others. | **CORRELATED — THREAT INTEL** |
| **E-15** | Cracked RAT tooling can itself become a hostile supply chain. | Public reporting on backdoored cracked tooling and leaked builders. | **CORRELATED — THREAT INTEL** |
| **E-16** | Dynamic Android control predates SpyNote and SpyMax. | Public evidence documents Geinimi remote-command/botnet behavior in 2010, DroidDream post-install download capability in 2011, Luckycat Android RAT development in 2012, and KSAPP script updating in 2013. | **CORRELATED — THREAT INTEL / HISTORICAL ASSESSMENT** |
| **E-17** | SpyNote/SpyMax are important consolidation points but are not established as the origin of dynamic Android malware. | SpyNote is public by 2016; Group-IB places SpyMax around 2019 with a later implementation leak, while dynamic Android control is documented years earlier. | **CORRELATED — THREAT INTEL / ASSESSMENT** |
| **E-18** | 2024–2026 is best described as a recent convergence and scaling phase, not the birth of the underlying techniques. | Comparative assessment across cited public campaigns. | **ASSESSMENT** |
| **E-19** | The cross-layer WebView-to-Native pattern can support high-impact Android fraud effects; “Trust Conversion” is an analytical framing rather than an official taxonomy or globally rankable technique. | Combined sample/platform/threat-intelligence assessment. | **ASSESSMENT** |
| **E-20** | The research does not establish website-server compromise, universal Android RCE, automatic permission granting, or self-propagation. | No such path is demonstrated by the reviewed evidence. | **NOT ESTABLISHED** |
| **E-21** | AI-adaptive WebView manipulation based on behavioral telemetry is a plausible future threat model, but the complete closed loop is not established by the analyzed sample. | Supported as a forecast by the sample's observable telemetry/control primitives plus public evidence of runtime LLM malware and adaptive offensive AI. | **ASSESSMENT / NOT ESTABLISHED AS OBSERVED SAMPLE BEHAVIOR** |


## A.4 Platform evidence for iframe/origin behavior

Android's current JavaScript-bridge guidance states that legacy `addJavascriptInterface()`:

- is available by default to every frame in the WebView, including `iframe` content;
- lacks origin-based access control;
- does not provide a safe way to identify the URL of the specific frame invoking the interface.

Primary platform reference:

- Android Developers — **Access native APIs with JavaScript bridge**  
  https://developer.android.com/develop/ui/views/layout/webapps/native-api-access-jsbridge

Additional platform references:

- Android Developers — **WebView API reference**  
  https://developer.android.com/reference/android/webkit/WebView
- Android Developers — **Build web apps in WebView**  
  https://developer.android.com/develop/ui/views/layout/webapps/webview
- Android Developers — **Security checklist**  
  https://developer.android.com/privacy-and-security/security-tips
- OWASP MAS — **MASTG-TEST-0334: Native Code Exposed Through WebViews**  
  https://mas.owasp.org/MASTG/tests/android/MASVS-PLATFORM/MASTG-TEST-0334/
- OWASP MAS — **MASTG-BEST-0035: Prefer Origin Scoped Messaging Over Legacy JavaScript Bridges**  
  https://mas.owasp.org/MASTG/best-practices/MASTG-BEST-0035/

---

# Appendix B — Evidence Boundaries

This appendix defines what may and may not be stated from the reviewed evidence.

## B.1 Proven / directly supported

The following statements are supported by direct sample or platform evidence:

```text
✓ The analyzed WebView can load network URLs.
✓ JavaScript is enabled.
✓ A native object is exposed through addJavascriptInterface().
✓ Runtime JavaScript execution/injection exists.
✓ Page lifecycle logic activates script/instrumentation behavior.
✓ DOM input and interaction events are observed.
✓ Field-derived data can cross into native Java code.
✓ Native code contains a subsequent processing / data-send path (static source path).
✓ Web interaction can initiate native permission-related workflows.
✓ The permission logic references multiple sensitive Android capabilities.
✓ Legacy addJavascriptInterface() is exposed to all frames, including iframes.
✓ Legacy addJavascriptInterface() lacks origin-based access control.
```

## B.2 Supported by independent real-world correlation

The following are **not attributed to this sample solely because they appear in other malware**. They are used as external validation that related techniques exist operationally:

```text
✓ Legitimate-site WebView/session abuse exists in banking malware.
✓ WebView + JavaScript + native bridge data collection exists in banking malware.
✓ Accessibility + overlays + keylogging + remote control are combined in modern fraud tooling.
✓ Social-media advertising / malvertising is used to acquire victims for Android financial malware.
✓ Cracked RAT ecosystems can distribute backdoored tooling and accelerate code reuse.
```

Representative public families/campaigns discussed in this paper include:

```text
S.O.V.A.
Xenomorph
Brokewell
Crocodilus
PlayPraetor
Sturnus
TrickMo
StreamRat
SpyNote / SpyMax / CraxsRAT lineage
```

Similarity does **not** prove:

```text
shared authorship
shared infrastructure
shared implementation code
direct lineage
the same operator
the same campaign
```

unless an external source independently establishes it.

## B.3 Not proven by the reviewed evidence

The reviewed evidence does **not** establish any of the following:

```text
✗ Compromise of the legitimate website's server
✗ Server-side remote code execution
✗ Modification of the legitimate website for other visitors
✗ Universal Android sandbox escape
✗ Automatic Android permission granting
✗ Infection through every advertisement
✗ Self-propagation from one website to another
✗ Compromise of users who never execute the hostile application
✗ A unique origin date for the exact technique
✗ A single-author origin for the full 2010–2026 evolutionary lineage
✗ A demonstrated end-to-end AI loop that autonomously converts DOM behavior into live permission-workflow optimization
✗ Remote provenance/control of `runtime script data` from this source snapshot alone
```

## B.4 Claims requiring additional validation

The following questions remain unresolved without additional evidence:

| Question | Evidence required |
|---|---|
| Can a specific target website be instrumented in a reproducible test? | Controlled reproduction against an authorized test origin with network and WebView traces. |
| Can the bridge be reached from a particular third-party advertising iframe? | Authorized frame-level test showing execution in the same bridge-enabled WebView. |
| Can a specific permission workflow be completed through the web-triggered journey? | Device/version-specific reproduction documenting each user/system interaction. |
| Does a particular backend receive the collected data? | Controlled packet capture, server-side logs, or authenticated protocol analysis. |
| Are two malware families directly related? | Code similarity, infrastructure overlap, developer artifacts, source lineage, or credible external attribution. |

## B.5 Interpretation Boundaries

The evidence supports three distinct conclusions:

1. **Sample capability:** the analyzed sample contains a Web-to-Native trust path capable of local page instrumentation, transfer of web-derived data into native code, and initiation of sensitive native workflows.
2. **Independent correlation:** public threat intelligence documents close operational analogues in Android financial malware.
3. **Client-side trust failure:** a legitimate website can remain uncompromised at the server while its user interaction is manipulated inside a hostile client environment.

These conclusions do not establish server compromise, universal advertising-to-device control, automatic Accessibility grants, or invention of the pattern by any single malware family.

Interpretation therefore separates four layers:

- **Capability** — what the architecture or reviewed source permits.
- **Reachability** — what can be reached under specific runtime conditions.
- **Real-world correlation** — what independent campaigns demonstrate in practice.
- **Attribution** — who developed, operated, or originated a specific implementation.

---

# Appendix C — Reproduction and Disclosure Checklist

Application-specific validation derived from this research requires the following evidence before exploitability or severity can be assigned:

```text
[ ] Exact application version / build
[ ] Android OS version and WebView version
[ ] Exact URL/origin loaded
[ ] Whether navigation is user-controlled, remote-controlled, or fixed
[ ] Whether the bridge is present on the relevant page/frame
[ ] Exact exposed @JavascriptInterface method
[ ] Exact native security effect reached
[ ] Required user interaction
[ ] Required Android permission / Settings confirmation
[ ] Network destination, if data transfer is claimed
[ ] Reproduction video / logs / packet trace where authorized
[ ] Expected behavior vs observed behavior
[ ] Mitigation validation after patch
```

Severity assessment should distinguish:

```text
Architecture exists
        ≠
Exploit path is reachable
        ≠
Sensitive impact is reproducible
        ≠
Critical severity is justified
```

This distinction should remain explicit in responsible disclosure.

