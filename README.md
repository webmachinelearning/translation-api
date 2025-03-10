# Explainer for the Translator and Language Detector APIs

_This proposal is an early design sketch by the Chrome built-in AI team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome._

Browsers are increasingly offering language translation to their users. Such translation capabilities can also be useful to web developers. This is especially the case when browser's built-in translation abilities cannot help, such as:

* translating user input or other interactive features;
* pages with complicated DOMs which trip up browser translation;
* providing in-page UI to start the translation; or
* translating content that is not in the DOM, e.g. spoken content.

To perform translation in such cases, web sites currently have to either call out to cloud APIs, or bring their own translation models and run them using technologies like WebAssembly and WebGPU. This proposal introduces a new JavaScript API for exposing a browser's existing language translation abilities to web pages, so that if present, they can serve as a simpler and less resource-intensive alternative.

An important supplement to translation is language detection. This can be combined with translation, e.g. taking user input in an unknown language and translating it to a specific target language. In a similar way, browsers today often already have language detection capabilities, and we want to offer them to web developers through a JavaScript API.

## Goals

Our goals are to:

* Help web developers perform real-time translations (e.g. of user input).
* Help web developers perform real-time language detection.
* Guide web developers to gracefully handle failure cases, e.g. translation not being available or possible.
* Harmonize well with existing browser and OS translation technology ([Brave](https://support.brave.com/hc/articles/8963107404813-How-do-I-use-Brave-Translate), [Chrome](https://support.google.com/chrome/answer/173424&co=GENIE.Platform%3DDesktop#zippy=%2Ctranslate-selected-text), [Edge](https://support.microsoft.com/topic/use-microsoft-translator-in-microsoft-edge-browser-4ad1c6cb-01a4-4227-be9d-a81e127fcb0b), [Firefox](https://support.mozilla.org/kb/website-translation), [Safari](https://9to5mac.com/2020/12/04/how-to-translate-websites-with-safari-mac/)), e.g. by allowing on-the-fly downloading of different languages instead of assuming all are present from the start.
* Allow a variety of implementation strategies, including on-device vs. cloud-based translation, while keeping these details abstracted from developers.
* Allow implementations to expose different capabilities for translation vs. language detection. For example, an implementation might be able to detect 30+ languages, but only be able to translate between 6.

The following are explicit non-goals:

* We do not intend to force every browser to ship language packs for every language combination, or even to support translation at all. It would be conforming to implement this API by always saying translation and language detection are unavailable, or to implement this API entirely by using cloud services instead of on-device translation.
* We do not intend to provide guarantees of translation and language detection quality, stability, or interoperability between browsers. These are left as quality-of-implementation issues, similar to the [shape detection API](https://wicg.github.io/shape-detection-api/). (See also a [discussion of interop](https://www.w3.org/reports/ai-web-impact/#interop) in the W3C "AI & the Web" document.)

The following are potential goals we are not yet certain of:

* Allow web developers to know whether translations are done on-device or using cloud services. This would allow them to guarantee that any user data they feed into this API does not leave the device, which can be important for privacy purposes. (Similarly, we might want to allow developers to request on-device-only translation, in case a browser offers both varieties.)
* Allow web developers to know some identifier for the translation and language detection models in use, separate from the browser version. This would allow them to allowlist or blocklist specific models to maintain a desired level of quality.

Both of these potential goals are potentially detrimental to interoperability, so we want to investigate more how important such functionality is to developers to find the right tradeoff.

## Examples

Note that in this API, languages are represented as [BCP 47](https://www.rfc-editor.org/info/bcp47) language tags, as already used by the existing JavaScript `Intl` API or the HTML `lang=""` attribute. Examples: `"ja"`, `"en"`, `"de-AT"`, `"zh-Hans-CN"`.

See [below](#language-tag-handling) for more on the details of how language tags are handled in this API, and the [appendix](#appendix-converting-between-language-tags-and-human-readable-strings) for some helper code that converts between language tags and human-readable strings.

### Translation

Here is the basic usage of the translator API, with no error handling:

```js
const translator = await Translator.create({
  sourceLanguage: "en",
  targetLanguage: "ja"
});

const text = await translator.translate("Hello, world!");
const readableStreamOfText = await translator.translateStreaming(`
  Four score and seven years ago our fathers brought forth, upon this...
`);
```

Note that the `create()` method call here might cause the download of a translation model or language pack. Later examples show how to get more insight into this process.

### Language detection

A similar simplified example of the language detector API:

```js
const detector = await LanguageDetector.create();

const results = await detector.detect(someUserText);
for (const result of results) {
  console.log(result.detectedLanguage, result.confidence);
}
```

Here `results` will be an array of `{ detectedLanguage, confidence }` objects, with the `detectedLanguage` field being a BCP 47 language tag and `confidence` beeing a number between 0 and 1. The array will be sorted by descending confidence, and the confidences will be normalized so that all confidences that the underlying model produces sum to 1, but very low confidences will be lumped together into an [`"und"`](https://www.rfc-editor.org/rfc/rfc5646.html#:~:text=*%20%20The%20'und'%20(Undetermined)%20primary,certain%20situations.) language.

The array will always contain at least 1 entry, although it could be for the undetermined (`"und"`) language.

For more details on the ways low-confidence results are excluded, see [the specification](https://webmachinelearning.github.io/translation-api/#note-language-detection-post-processing) and the discussion in [issue #39](https://github.com/webmachinelearning/translation-api/issues/39).

### Language detection with expected input languages

If there are certain languages you need to be able to detect for your use case, you can include them in the `expectedInputLanguages` option when creating a language detector:

```js
const detector = await LanguageDetector.create({ expectedInputLanguages: ["en", "ja"] });
```

This will allow the implementation to download additional resources like language detection models if necessary, and will ensure that the promise is rejected with a `"NotSupportedError"` `DOMException` if the browser is unable to detect the given input languages.

### Checking before creation, and a more realistic combined example

Both APIs provide the ability to know, before calling `create()`, what is possible with the implementation. This is done via `availability()` methods, which takes the same options as `create()`. They return a promise, which fulfills with one of the following values:

* `"unavailable"` means that the implementation does not support translation or language detection of the given language(s).
* `"downloadable"` means that the implementation supports translation or language detection of the given language(s), but it will have to download something (e.g., a machine learning model) as part of creating the associated object.
* `"downloading"` means that the implementation supports translation or language detection of the given language(s), but it will have to finish an ongoing download as part of creating the associated object.
* `"available"` means that the implementation supports translation or language detection of the given language(s), without performing any downloads.

Here is an example that adds capability checking to log more information and fall back to cloud services, as part of a language detection plus translation task:

```js
async function translateUnknownCustomerInput(textToTranslate, targetLanguage) {
  const detectorAvailability = await LanguageDetector.availability();

  // If there is no language detector, then assume the source language is the
  // same as the document language.
  let sourceLanguage = document.documentElement.lang;

  // Otherwise, let's detect the source language.
  if (detectorAvailability !== "unavailable") {
    if (detectorAvailability !== "available") {
      console.log("Language detection is available, but something will have to be downloaded. Hold tight!");
    }

    const detector = await LanguageDetector.create();
    const [bestResult] = await detector.detect(textToTranslate);

    if (bestResult.detectedLanguage ==== "und" || bestResult.confidence < 0.4) {
      // We'll just return the input text without translating. It's probably mostly punctuation
      // or something.
      return textToTranslate;
    }
    sourceLanguage = bestResult.detectedLanguage;
  }

  // Now we've figured out the source language. Let's translate it!
  const translatorAvailability = await Translator.availability({ sourceLanguage, targetLanguage });
  if (translatorAvailability === "unavailable") {
    console.warn("Translation is not available. Falling back to cloud API.");
    return await useSomeCloudAPIToTranslate(textToTranslate, { sourceLanguage, targetLanguage });
  }

  if (translatorAvailability !== "available") {
    console.log("Translation is available, but something will have to be downloaded. Hold tight!");
  }

  const translator = await Translator.create({ sourceLanguage, targetLanguage });
  return await translator.translate(textToTranslate);
}
```

### Download progress

For cases where using the API is only possible after a download, you can monitor the download progress (e.g. in order to show your users a progress bar) using code such as the following:

```js
const translator = await Translator.create({
  sourceLanguage,
  targetLanguage,
  monitor(m) {
    m.addEventListener("downloadprogress", e => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  }
});
```

If the download fails, then `downloadprogress` events will stop being emitted, and the promise returned by `create()` will be rejected with a "`NetworkError`" `DOMException`.

Note that in the case that multiple entities are downloaded (e.g., an `en` ↔ `ja` language pack and a `en` ↔ `ko` language pack to support the `ja` ↔ `ko` use case) web developers do not get the ability to monitor the individual downloads. All of them are bundled into the overall `downloadprogress` events, and the `create()` promise is not fulfilled until all downloads and loads are successful.

The event is a [`ProgressEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent) whose `loaded` property is between 0 and 1, and whose `total` property is always 1. (The exact number of total or downloaded bytes are not exposed; see the discussion in [webmachinelearning/writing-assistance-apis issue #15](https://github.com/webmachinelearning/writing-assistance-apis/issues/15).)

At least two events, with `e.loaded === 0` and `e.loaded === 1`, will always be fired. This is true even if creating the translator or language detector doesn't require any downloading.

<details>
<summary>What's up with this pattern?</summary>

This pattern is a little involved. Several alternatives have been considered. However, asking around the web standards community it seemed like this one was best, as it allows using standard event handlers and `ProgressEvent`s, and also ensures that once the promise is settled, the translator or language detector object is completely ready to use.

It is also nicely future-extensible by adding more events and properties to the `m` object.

Finally, note that there is a sort of precedent in the (never-shipped) [`FetchObserver` design](https://github.com/whatwg/fetch/issues/447#issuecomment-281731850).
</details>

### Too-large inputs

It's possible that the inputs given for translation or language detection might be too large for the underlying machine learning model to handle. Although there are often techniques that allow implementations to break up the inputs into smaller chunks, and combine the results, the APIs have some facilities to allow browsers to signal such too-large inputs.

Whenever any API call fails due to too-large input, it is rejected with a `QuotaExceededError`. This is a proposed new type of exception, which subclasses `DOMException`, and replaces the web platform's existing `"QuotaExceededError"` `DOMException`. See [whatwg/webidl#1465](https://github.com/whatwg/webidl/pull/1465) for this proposal. For our purposes, the important part is that it has the following properties:

* `requested`: how much "usage" the input consists of
* `quota`: how much "usage" was available (which will be less than `requested`)

The "usage" concept is specific to the implementation, and could be something like string length, or [language model tokens](https://arxiv.org/abs/2404.08335).

This allows detecting failures due to overlarge inputs and giving clear feedback to the user, with code such as the following:

```js
const detector = await LanguageDetector.create();

try {
  console.log(await detector.detect(potentiallyLargeInput));
} catch (e) {
  if (e.name === "QuotaExceededError") {
    console.error(`Input too large! You tried to detect the language of ${e.requested} tokens, but ${e.quota} is the max supported.`);

    // Or maybe:
    console.error(`Input too large! It's ${e.requested / e.quota}x as large as the maximum possible input size.`);
  }
}
```

In some cases, instead of providing errors after the fact, the developer needs to be able to communicate to the user how close they are to the limit. For this, they can use the `inputQuota` property and the `measureInputUsage()` method on the translator or language detector objects:

```js
const translator = await Translator.create({
  sourceLanguage: "en",
  targetLanguage: "jp"
});
meterEl.max = translator.inputQuota;

textbox.addEventListener("input", () => {
  meterEl.value = await translator.measureInputUsage(textbox.value);
  submitButton.disabled = meterEl.value > meterEl.max;
});

submitButton.addEventListener("click", () => {
  console.log(translator.translate(textbox.value));
});
```

Note that if an implementation does not have any limits, e.g. because it uses techniques to split up the input and process it a bit at a time, then `inputQuota` will be `+Infinity` and `measureInputUsage()` will always return 0.

Developers need to be cautious not to over-use this API, however, as it requires a round-trip to the underlying model. That is, the following code is bad, as it performs two round trips with the same input:

```js
// DO NOT DO THIS

const usage = await translator.measureInputUsage(input);
if (usage < translator.inputQuota) {
  console.log(await translator.translate(input));
} else {
  console.error(`Input too large!`);
}
```

If you're planning to call `translate()` anyway, then using a pattern like the one that opened this section, which catches `QuotaExceededError`s, is more efficient than using `measureInputUsage()` plus a conditional call to `translate()`.

### Destruction and aborting

The API comes equipped with a couple of `signal` options that accept `AbortSignal`s, to allow aborting the creation of the translator/language detector, or the translation/language detection operations themselves:

```js
const controller = new AbortController();
stopButton.onclick = () => controller.abort();

const languageDetector = await LanguageDetector.create({ signal: controller.signal });
await languageDetector.detect(document.body.textContent, { signal: controller.signal });
```

Destroying a translator or language detector will:

* Reject any ongoing calls to `detect()` or `translate()`.
* Error any `ReadableStream`s returned by `translateStreaming()`.
* And, most importantly, allow the user agent to unload the machine learning models from memory. (If no other APIs are using them.)

Allowing such destruction provides a way to free up the memory used by the model without waiting for garbage collection, since machine learning models can be quite large.

Aborting the creation process will reject the promise returned by `create()`, and will also stop signaling any ongoing download progress. (The browser may then abort the downloads, or may continue them. Either way, no further `downloadprogress` events will be fired.)

In all cases, the exception used for rejecting promises or erroring `ReadableStream`s will be an `"AbortError"` `DOMException`, or the given abort reason.

## Detailed design

### Language tag handling

If a browser supports translating from `ja` to `en`, does it also support translating from `ja` to `en-US`? What about `en-GB`? What about the (discouraged, but valid) `en-Latn`, i.e. English written in the usual Latin script? But translation to `en-Brai`, English written in the Braille script, is different entirely.

We're proposing that the API use the same model as JavaScript's `Intl` APIs, which tries to do [best-fit matching](https://tc39.es/ecma402/#sec-lookupmatchinglocalebybestfit) of the requested language tag to the available language tags. The specification contains [a more detailed example](https://webmachinelearning.github.io/translation-api/#example-language-arc-support).

### Multilingual text

For language detection of multilingual text, we return detected language confidences in proportion to the languages detected. The specification gives [an example](https://webmachinelearning.github.io/translation-api#example-multilingual-input) of how this works. See also the discussion in [issue #13](https://github.com/webmachinelearning/translation-api/issues/13).

A future option might be to instead have the API return back the splitting of the text into different-language segments. There is [some precedent](https://github.com/pemistahl/lingua-py?tab=readme-ov-file#116-detection-of-multiple-languages-in-mixed-language-texts) for this, but it does not seem to be common yet. This could be added without backward-compatibility problems by making it a non-default mode.

### Downloading

The current design envisions that `availability()` methods will _not_ cause downloads of language packs or other material like a language detection model. Whereas, the `create()` methods _can_ cause downloads. In all cases, whether or not creation will initiate a download can be detected beforehand by the corresponding `availability()` method.

After a developer has a `Translator` or `LanguageDetector` object, further calls are not expected to cause any downloads. (Although they might require internet access, if the implementation is not entirely on-device.)

This design means that the implementation must have all information about the capabilities of its translation and language detection models available beforehand, i.e. "shipped with the browser". (Either as part of the browser binary, or through some out-of-band update mechanism that eagerly pushes updates.)

## Privacy considerations

This proposal as-is has privacy issues, which we are actively thinking about how to address. They are all centered around how sites that use this API might be able to uniquely fingerprint the user.

The most obvious identifier in the current API design is the list of supported languages, and especially their availability status (`"unavailable"`, `"downloadable"`, `"downloading"`, and `"available"`). For example, as of the time of this writing [Firefox supports 9 languages](https://www.mozilla.org/firefox/features/translate/), which can each be [independently downloaded](https://support.mozilla.org/kb/website-translation#w_configure-installed-languages). With a naive implementation, this gives 9 bits of identifying information, which various sites can all correlate.

Some sort of mitigation may be necessary here. We believe this is adjacent to other areas that have seen similar mitigation, such as the [Local Font Access API](https://github.com/WICG/local-font-access/blob/main/README.md). Possible techniques are:

* Grouping language packs to reduce the number of bits, so that downloading one language also downloads others in its group.
* Partitioning download status by top-level site, introducing a fake download (which takes time but does not actually download anything) for the second-onward site to download a language pack.
* Only exposing a fixed set of languages to this API, e.g. based on the user's locale or the document's main language.

As a first step, we require that detecting the availability of translation/detection be done via individual calls to `Translator.availability()` and `LanguageDetector.availability()`. This allows browsers to implement possible mitigation techniques, such as detecting excessive calls to these methods and starting to return `"unavailable"`.

Another way in which this API might enhance the web's fingerprinting surface is if translation and language detection models are updated separately from browser versions. In that case, differing results from different versions of the model provide additional fingerprinting bits beyond those already provided by the browser's major version number. Mandating that older browser versions not receive updates or be able to download models from too far into the future might be a possible remediation for this.

Finally, we intend to prohibit (in the specification) any use of user-specific information in producing the results. For example, it would not be permissible to fine-tune the translation model based on information the user has entered into the browser in the past.

## Alternatives considered and under consideration

### Streaming input support

Although the API contains support for streaming output of a translation, via the `translateStreaming()` API, it doesn't support streaming input. Should it?

We believe it should not, for now. In general, translation works best with more context; feeding more input into the system over time can produce very different results. For example, translating "<span lang="ja">彼女の話を聞いて、驚いた</span>" to English would give "I was surprised to hear her story". But if you then streamed in another chunk, so that the full sentence was "<span lang="ja">彼女の話を聞いて、驚いたねこが逃げた</span>", the result changes completely to "Upon hearing her story, the surprised cat ran away." This doesn't fit well with how streaming APIs behave generally.

In other words, even if web developers are receiving a stream of input (e.g. over the network or from the user), they need to take special care in how they present such updating-over-time translations to the user. We shouldn't treat this as a usual stream-to-string or stream-to-stream API, because that will rarely be useful.

That said, we are aware of [research](https://arxiv.org/abs/2005.08595) on translation algorithms which are specialized for this kind of setting, and attempt to mitigate the above problem. It's possible we might want to support this sort of API in the future, if implementations are excited about implementing that research. This should be possible to fit into the existing API surface, possibly with some extra feature-detection API.

### Flattening the API and reducing async steps

The current design requires multiple async steps to do useful things:

```js
const translator = await Translator.create(options);
const text = await translator.translate(sourceText);

const detector = await LanguageDetector.create();
const results = await detector.detect(sourceText);
```

Should we simplify these down with convenience APIs that do both steps at once?

We're open to this idea, but we think the existing complexity is necessary to support the design wherein translation and language detection models might not be already downloaded. By separating the two stages, we allow web developers to perform the initial creation-and-possibly-downloading steps early in their page's lifecycle, in preparation for later, hopefully-quick calls to APIs like `translate()`.

Another possible simplification is to make the `availability()` APIs synchronous instead of asynchronous. This would be implementable by having the browser proactively load the capabilities information into the main thread's process, upon creation of the global object. We think this is not worthwhile, as it imposes a non-negligible cost on all global object creation, even when the APIs are not used.

### Allowing unknown source languages for translation

An earlier revision of this API including support for combining the language detection and translation steps into a single translation call, which did a best-guess on the source language. The idea was that this would possibly be more efficient than requiring the web developer to do two separate calls, and it could possibly even be done using a single model.

We abandoned this design when it became clear that existing browsers have very decoupled implementations of translation vs. language detection, using separate models for each. This includes supporting different languages for language detection vs. for translation. So even if the translation model supported an unknown-source-language mode, it might not support the same inputs as the language detection model, which would create a confusing developer experience and be hard to signal in the API.

## Stakeholder feedback

* W3C TAG: <https://github.com/w3ctag/design-reviews/issues/948>
* Browser engines:
  * Chromium: prototyping behind a flag
  * Gecko: <https://github.com/mozilla/standards-positions/issues/1015>
  * WebKit: <https://github.com/WebKit/standards-positions/issues/339>
* Web developers:
  * Some comments in [W3C TAG design review](https://github.com/w3ctag/design-reviews/issues/948)
  * Some comments in [WICG proposal](https://github.com/WICG/proposals/issues/147)

## Appendix: converting between language tags and human-readable strings

This code already works today and is not new to this API proposal. It is likely useful in conjunction with this API, for example when building user interfaces.

```js
function languageTagToHumanReadable(languageTag, targetLanguage) {
  const displayNames = new Intl.DisplayNames([targetLanguage], { type: "language" });
  return displayNames.of(languageTag);
}

languageTagToHumanReadable("ja", "en");      // "Japanese"
languageTagToHumanReadable("zh", "en");      // "Chinese"
languageTagToHumanReadable("zh-Hant", "en"); // "Traditional Chinese"
languageTagToHumanReadable("zh-TW", "en");   // "Chinese (Taiwan)"

languageTagToHumanReadable("en", "ja");      // "英語"
```
