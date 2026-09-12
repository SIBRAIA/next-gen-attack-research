# Cross-Layer WebView Trust Hijacking

## When Web Content Crosses the Android Native Trust Boundary

### Part I — WebView Native Bridge Abuse

## Executive Summary

Android `WebView` is often treated as a presentation layer embedded inside an application.

That assumption becomes dangerous when JavaScript running inside the page is allowed to invoke native Android functionality through:

```java
addJavascriptInterface(...)
```

In the sample analyzed for this research, the WebView is not used only for rendering remote content.

It establishes a direct execution path between web content and the native Android runtime:

```text
Web Content
    ↓
JavaScript
    ↓
Native Bridge
    ↓
Android Runtime
    ↓
Application Capabilities
```

The core issue is not a standalone WebView vulnerability.

It is a failure of the trust boundary between web-controlled content and the native application layer.

---

## 1. The Entry Point

The WebView enables JavaScript:

```java
webView.getSettings().setJavaScriptEnabled(true);
```

At the same time, a native Java object is exposed to the JavaScript context:

```java
webView.addJavascriptInterface(
    new WebAppInterface(this),
    "Android"
);
```

This creates an object accessible from the page as:

```javascript
window.Android
```

The existence of a JavaScript bridge is not automatically a vulnerability.

The security impact depends primarily on two questions:

```text
Which native capabilities are exposed?

Which web origins are allowed to access them?
```

When JavaScript execution and a powerful native bridge coexist inside the same WebView, the boundary between web and native code becomes security-critical.

---

## 2. The Native Bridge Is Capability-Rich

The exposed interface contains multiple methods annotated with:

```java
@JavascriptInterface
```

Representative capabilities include:

```text
sendInputData(...)
sendToServer(...)
requestPermission(...)
RequestData(...)
```

JavaScript can therefore pass information from the web execution context directly into native Java code.

A simplified example from the observed design:

```java
@JavascriptInterface
public void sendInputData(String data) {
    ...
    dbHandler.addNewCourseW(String.valueOf(0), data);
    RequestData(data);
}
```

Additional methods forward information through native application components:

```java
@JavascriptInterface
public void sendToServer(String data) {
    callSendMethod(...);
}
```

and:

```java
@JavascriptInterface
public void RequestData(String data) {
    callSendMethod(...);
}
```

The resulting trust path is no longer limited to the DOM:

```text
JavaScript
    ↓
Native Bridge
    ↓
Application Logic
    ↓
Storage / Processing / Network
```

---

## 3. DOM Input Collection

The injected JavaScript monitors multiple classes of page elements:

```text
input
textarea
button
a
select
```

It also observes events including:

```text
focus
input
keydown
keyup
blur
resize
click
```

When an input field becomes active, contextual information can be collected, including:

```text
Field Name
Page URL
Input Value
Event Type
```

A simplified form of the observed logic:

```javascript
document.addEventListener('input', function(event) {
    let element = event.target;

    if (['input', 'textarea'].includes(
        element.tagName.toLowerCase()
    )) {
        let inputValue = element.value.trim();
        ...
    }
});
```

The collected value can then cross into the native layer through:

```javascript
window.Android.sendInputData(...)
```

The full trust path becomes:

```text
User Input
    ↓
DOM Event
    ↓
Injected JavaScript
    ↓
window.Android
    ↓
Native Java Code
    ↓
Storage / Processing / Network
```

This is an important architectural transition.

The browser-like environment is no longer an isolated presentation layer. User-controlled DOM state can become native application input.

---

## 4. Web Content Can Initiate Native Permission Flows

The bridge also exposes a method equivalent to:

```java
@JavascriptInterface
public void requestPermission(String request)
```

The associated native logic handles multiple categories of Android permissions and capabilities, including:

```text
CAMERA
LOCATION
AUDIO
CALL_LOG
CONTACTS
FILES
SMS
CALL_PHONE
```

Additional flows are associated with:

```text
OVERLAY
ALLFILES
ACCESSIBILITY
BATTERY
```

JavaScript does not directly grant these permissions.

Android still controls the final permission and settings interfaces.

The security concern is different:

> Web-controlled execution can initiate native workflows associated with sensitive Android capabilities.

This changes the role of the WebView from a content renderer into a trigger surface for privileged native operations.

---

## 5. `JavascriptInterface` Is Not the Vulnerability

It would be inaccurate to reduce the issue to:

```text
addJavascriptInterface = vulnerability
```

A JavaScript bridge can be used safely when strong controls exist around it.

For example:

```text
Trusted Content Only
        +
Strict Origin Control
        +
Minimal Native Methods
        +
No Sensitive Native Actions
```

The risk emerges when several capabilities are combined:

```text
JavaScript Enabled
        +
Native Bridge
        +
Sensitive Native Methods
        +
Dynamic Navigation
        +
Injected JavaScript
        +
Weak Origin Isolation
```

The resulting architecture effectively creates:

```text
Web Content
     │
     │ Trust Boundary Crossing
     ▼
Application Runtime
```

That boundary is the central security issue.

---

## 6. Navigation Trust

The analyzed WebView accepts URLs based primarily on scheme checks such as:

```text
https://
http://
```

before passing the value to:

```java
webView.loadUrl(uri);
```

A scheme check does not establish origin trust.

For example:

```text
https://trusted.example
```

and:

```text
https://untrusted.example
```

both satisfy:

```java
uri.startsWith("https://")
```

This does not prove that an attacker can control the loaded URL.

It does mean that the security model cannot rely on HTTPS alone when a privileged JavaScript bridge is exposed.

A WebView with native capabilities requires an explicit origin policy.

---

## 7. Runtime JavaScript Injection

The sample also contains a path that executes JavaScript supplied through runtime data:

```java
if (!RAW_DATA_INJECT_JAVASC.isEmpty()) {
    webView.evaluateJavascript(
        decodeHtmlBase64ToUtf8(
            RAW_DATA_INJECT_JAVASC
        ),
        null
    );
}
```

Architecturally, this creates another trust path:

```text
Runtime Data
    ↓
Decode
    ↓
evaluateJavascript()
    ↓
Current WebView Context
```

`evaluateJavascript()` is not inherently unsafe.

The critical questions are:

```text
Where did the script originate?

Who can modify it?

Which page will execute it?

Which native bridge is exposed at that moment?
```

The combination of runtime script injection and a native bridge significantly increases the importance of provenance and origin validation.

---

## 8. Expanded WebView Attack Surface

Several WebView capabilities are enabled simultaneously:

```java
setAllowUniversalAccessFromFileURLs(true);
setJavaScriptCanOpenWindowsAutomatically(true);
setSupportMultipleWindows(true);
setAllowContentAccess(true);
setAllowFileAccess(true);
setJavaScriptEnabled(true);
```

None of these settings alone proves exploitability.

The concern is composition.

A permissive WebView becomes more sensitive when it also exposes native application capabilities.

The security model must therefore be evaluated as a system rather than as individual configuration flags.

---

## 9. Threat Model

The relevant defensive threat model is:

```text
Untrusted / Compromised Web Content
              ↓
        JavaScript Execution
              ↓
       window.Android Bridge
              ↓
        Native Java Methods
              ↓
 ┌────────────┼────────────┐
 ↓            ↓            ↓
Data       Permissions    Services
 ↓            ↓            ↓
Storage      Android      Native
Network       UI           Runtime
```

This is no longer purely a web security problem.

It is also not purely an Android native security problem.

It is a cross-layer trust problem.

---

## 10. Conditions Required for Exploitation

The existence of this architecture does not prove that the application is compromised.

A practical exploitation path would require control over one or more parts of the web execution context, such as:

```text
Attacker-Controlled Page
Compromised Trusted Domain
Unsafe Redirect
Injected Third-Party Script
XSS Inside a Trusted Origin
Remote JavaScript Configuration
Navigation to an Untrusted Origin
```

If one of these conditions occurs while:

```javascript
window.Android
```

remains available, a web-layer compromise may gain access to native application capabilities.

That is where impact can cross layers.

---

## 11. Architectural Classification

A useful description for this pattern is:

### Cross-Layer WebView Trust Hijacking

or:

### Web-to-Native Trust Boundary Failure

It should not automatically be classified as:

```text
Android RCE
```

The analyzed architecture does not demonstrate a breakout from the Android application sandbox.

The trust failure exists inside the application's own security design.

That distinction matters.

---

## 12. Security Impact

Depending on the capabilities exposed by the native bridge, this architecture can enable:

```text
Sensitive Input Collection
Native Data Forwarding
Permission Flow Initiation
Interaction With Privileged Services
Access to Application State
Cross-Layer Command Execution
```

The effective risk can be modeled as:

```text
Loaded Origin
        ×
Bridge Capabilities
        ×
Application Permissions
        ×
Backend Trust
```

A compromise of the web layer becomes increasingly serious as those native capabilities increase.

---

## 13. Defensive Architecture

The correct mitigation is not simply to disable JavaScript.

The real objective is to restore the trust boundary.

### 13.1 Enforce an Origin Allowlist

Native-capable WebViews should accept only explicitly trusted origins.

For example:

```text
https://app.example.com
https://auth.example.com
```

Not:

```text
https://*
```

Scheme validation is not origin validation.

---

### 13.2 Do Not Expose the Bridge Globally

The native bridge should not remain available across arbitrary navigation.

The security model should resemble:

```text
Trusted Origin
      ↓
Bridge Enabled

Untrusted Origin
      ↓
Bridge Removed
```

Native capability exposure should follow origin state.

---

### 13.3 Minimize the Bridge

Avoid broad interfaces such as:

```text
window.Android.requestPermission(...)
window.Android.sendToServer(...)
window.Android.RequestData(...)
```

Prefer a narrow policy boundary:

```text
Web Request
   ↓
Strict Schema Validation
   ↓
Native Policy Engine
   ↓
Allow / Deny
```

The bridge should expose limited capabilities rather than general-purpose control.

---

### 13.4 Treat DOM Data as Untrusted

Data originating from:

```text
window.*
document.*
DOM Events
JavaScript Callbacks
```

should remain untrusted even when the current page belongs to an expected domain.

A trusted origin does not automatically make every script or DOM value trustworthy.

---

### 13.5 Separate Permission Decisions From Web Content

Web content should not directly decide:

```text
Request CAMERA
Request SMS
Request ACCESSIBILITY
```

Sensitive native operations should pass through a policy layer that evaluates:

```text
Explicit User Action
Application State
Current Trusted Origin
Feature Context
Risk Policy
```

This prevents the WebView from becoming a permission orchestration layer.

---

### 13.6 Disable Unused WebView Capabilities

If the application does not require:

```text
File Access
Content Access
Multiple Windows
Universal File URL Access
```

they should remain disabled.

The safer default is:

```text
Disabled by Default
Enabled Only When Required
```

Every additional capability increases the number of assumptions the security model must protect.

---

## 14. Detection Perspective

From a defensive review perspective, the following APIs are useful signals:

```text
setJavaScriptEnabled(true)

addJavascriptInterface(...)

evaluateJavascript(...)

loadUrl(dynamicValue)

@JavascriptInterface

setAllowUniversalAccessFromFileURLs(true)
```

Individually, these APIs generate many legitimate matches.

The stronger detection pattern is relational:

```text
WebView
   ↓
Dynamic Content
   ↓
JavaScript Execution
   ↓
Native Bridge
   ↓
Sensitive Native Capability
```

Security tooling should identify the trust path rather than simply flag isolated API usage.

---

## 15. Conclusion

A WebView is not dangerous merely because it executes JavaScript.

A `JavascriptInterface` is not dangerous merely because it exists.

The critical failure occurs when:

```text
Web Trust
    ↓
Native Trust
```

can be crossed without a strong policy boundary.

At that point, compromise of the web execution layer can gain effects far beyond the page itself.

The important security question is not:

> Does this application use WebView?

It is:

> What can WebView content do inside the application if that content becomes untrusted?

That is the real trust boundary.

---

## Next

### Part II — Remote-Controlled Dynamic Code Loading in Android

The next part moves below the WebView layer and into the application runtime:

```text
Remote Payload
      ↓
DEX Bytes
      ↓
ByteBuffer
      ↓
InMemoryDexClassLoader
      ↓
Runtime Class Loading
      ↓
Application Runtime
```

Part II examines the distinction between:

```text
Dynamic Code Loading
Mitigation Avoidance
Restriction Bypass
Runtime ClassLoader Mutation
```

and how related techniques appear in modern Android threat activity.