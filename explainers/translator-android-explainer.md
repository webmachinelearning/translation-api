# Explainer: Translator API on Android

## Problem Description
Browsers are increasingly exposing on-device AI capabilities through standards-track Web APIs like the Translator API. On desktop platforms, translation is typically powered by packaged Neural Machine Translation (NMT) engines and background component updater pipelines. However, bringing these capabilities to mobile browsers (such as Chrome on Android) presents key platform constraints:

* **Binary Size Limits:** Porting desktop translation engines directly into mobile binaries adds 10–20+ MB to the base application package (APK), far exceeding mobile binary growth constraints (e.g., Chrome on Android's strict binary addition limits).
* **Model Distribution & Updating:** Mobile operating systems do not support desktop-style background component updater architectures (e.g., Chrome Component Updater) to download and update large model dependencies on demand.
* **Resource Latency & Memory:** Mobile hardware operates under strict memory, thermal, and battery limits, making heavy standalone model runtimes impractical.

As a result, mobile web applications are currently forced to rely on cloud translation endpoints, missing out on zero-latency, privacy-preserving, and offline-capable built-in web translation.

---

## Proposal: OS-Backed Mobile On-Device Translation
We propose extending the standard Translator API to Android by leveraging platform-native Neural Machine Translation (NMT) capabilities (such as Android's system ML services and Play Feature Delivery) as an execution backend. 

By abstracting the platform execution layer behind the standard W3C Web Machine Learning Translator API interface, the browser can:
1. **Eliminate Base Binary Overhead:** Offload model delivery to dynamic feature modules or OS-provided NMT libraries rather than bundling heavy translation models in the base APK.
2. **Maintain Full API Parity:** Allow web developers to write standard JavaScript using `Translator.create()`, `translator.translate()`, and `translator.translateStreaming()`, without needing mobile-specific branches or custom SDKs.
3. **Preserve Privacy & Offline Performance:** Perform fast, local translation entirely on-device without transmitting user text to cloud endpoints.

---

## API Surface & Specification Integration
The JavaScript API surface on Android strictly follows the main [Translator API Specification](https://webmachinelearning.github.io/translation-api/):

```javascript
// Check capability availability for a language arc on Android
const availability = await Translator.availability({
  sourceLanguage: "en",
  targetLanguage: "es"
});

if (availability !== "unavailable") {
  // Create translator (triggers Play Feature Delivery / system model fetch if "downloadable")
  const translator = await Translator.create({
    sourceLanguage: "en",
    targetLanguage: "es",
    monitor(m) {
      m.addEventListener("downloadprogress", e => {
        console.log(`Download progress: ${e.loaded * 100}%`);
      });
    }
  });

  // Single string translation
  const translatedText = await translator.translate("Hello world!");

  // Streaming translation output
  const stream = translator.translateStreaming("Longer text stream...");
  const reader = stream.getReader();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    console.log("Chunk:", value);
  }
}
```

### Availability & Download Lifecycle
On Android, `Translator.availability()` evaluates language arc support by mapping requested BCP 47 language tags (using best-fit matching) directly to system NMT capabilities:
* **`"available"`:** The required NMT language pack is already cached on-device by the OS/provider.
* **`"downloadable"`:** The OS or Play Feature Delivery supports the language pair, but the language pack must be downloaded prior to first use.
* **`"downloading"`:** An existing download for the requested language pair is currently in progress.
* **`"unavailable"`:** The requested language pair is not supported by the mobile NMT engine, or the device lacks required system capabilities.

---

## Provider Routing, Quotas, and Fallback

### Input Quota & Usage Measurement
To account for mobile resource limits, implementations on Android integrate with standard specification usage guardrails:
* **`translator.inputQuota`:** Exposes the maximum input quota available for translation operations (or `+Infinity` if unconstrained).
* **`translator.measureInputUsage(input)`:** Allows developers to inspect token/character usage before dispatching large inputs, preventing unexpected `QuotaExceededError` exceptions.

### Graceful Degradation & Fallback
When `Translator.create()` is invoked on Android:
* **Primary Route:** The browser bridges Web API calls to system NMT services, retrieving language models via dynamic delivery as needed.
* **Graceful Degradation:** On devices lacking required system services or for unsupported language pairs, `availability()` returns `"unavailable"`. Web applications can detect this state early and fall back to cloud-based translation endpoints.

---

## Privacy & Security Considerations
This mobile implementation adheres to all security and privacy guidelines defined in the main specification:
* **Data Locality:** Input and output text strings are processed locally within the OS or on-device model sandbox, ensuring user text never leaves the device.
* **Permissions Policy Integration:** Access to the API is gated by the policy-controlled feature `"translator"`, defaulting to `'self'`. Cross-origin `<iframe>` contexts require explicit delegation (`allow="translator"`).

---

## Future Work: Scaling Built-in AI Web APIs to Mobile
Bringing the Translator API to Android establishes an architectural blueprint for mobile-native execution of client-side AI capabilities. 

By validating OS-backed service routing and dynamic feature module delivery, this work lays the foundation for extending other standards-track Built-in AI Web APIs to Android in future iterations, including:
* **Language Detector API:** Offloading language detection to native OS libraries to complement text translation.
* **Summarizer API:** Leveraging on-device mobile AI execution engines for low-latency text summarization.
* **Prompt / LanguageModel API & Writing Assistance APIs:** Exploring system LLM runtimes to power tasks like writing, rewriting, and structured prompting on mobile devices without inflating application packages.

---

## Conclusion
Extending the Translator API to Android via platform-native NMT services unlocks fast, offline, and privacy-preserving translation on mobile devices while satisfying strict APK binary size constraints. Crucially, it creates a scalable design pattern for bringing the broader suite of Built-in AI Web APIs to mobile web ecosystems.