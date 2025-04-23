# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

> 01.  What information does this feature expose,
>      and for what purposes?

This feature exposes two main pieces of information:

- The availability information for each `(sourceLanguage, targetLanguage)` translation pair, or possible language detection result, so that web developers know what translations and detections are possible and whether they will require the user to download a potentially-large language pack.

  (This information has to be probed for each individual pair or possible language, and the browser can say that a language is unavailable even if it is, for privacy reasons.)

- The actual results of translations and language detections, which can be dependent on the AI models in use.

The privacy implications of both of these are discussed, in general terms, [in the _Writing Assistance APIs_ specification](https://webmachinelearning.github.io/writing-assistance-apis/#privacy), which was written to cover all APIs with similar concerns.

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to implement the intended functionality?

We believe so. It's possible that we could remove the exposure of the download status information. However, it would almost certainly be inferrable via timing side-channels. (I.e., if downloading a language pack is required, then the web developer can observe the first translation taking longer.)

> 03.  Do the features in your specification expose personal information,
>      personally-identifiable information (PII), or information derived from
>      either?

No. Although it's imaginable that the translation or language detection models could be fine-tuned on PII to give more accurate-to-this-user translations, we intend to disallow this in the specification.

> 04.  How do the features in your specification deal with sensitive information?

We do not deal with sensitive information.

> 05.  Do the features in your specification introduce state
>      that persists across browsing sessions?

Yes. The downloading of language packs and translation or language detection models persists across browsing sessions.

> 06.  Do the features in your specification expose information about the
>      underlying platform to origins?

Possibly. If a browser does not bundle its own models, but instead uses the operating system's functionality, it is possible for a web developer to infer information about such operating system functionality.

> 07.  Does this specification allow an origin to send data to the underlying
>      platform?

Possibly. Again, in the scenario where translation is done via operating system functionality, such data would pass through OS libraries.

> 08.  Do features in this specification enable access to device sensors?

No.

> 09.  Do features in this specification enable new script execution/loading
>      mechanisms?

No.

> 10.  Do features in this specification allow an origin to access other devices?

No.

> 11.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No.

> 12.  What temporary identifiers do the features in this specification create or
>      expose to the web?

None.

> 13.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

We use permissions policy to disallow the usage of these features by default in third-party (cross-origin) contexts. However, the top-level site can delegate to cross-origin iframes.

Otherwise, some of the possible [anti-fingerprinting mitigations](https://webmachinelearning.github.io/writing-assistance-apis/#privacy-availability) involve partitioning information across sites, which is kind of like distinguishing between first- and third-party contexts.

> 14.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

We are not yet sure. It is likely that some behavior will be different to deal with the [anti-fingerprinting considerations](./README.md#privacy-considerations). For example, if storage partitioning infrastructure is used, then this would be automatic.

Another possible area of discussion here is whether cloud-based translation APIs make sense in such modes, or whether they should be disabled.

> 15.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

Yes:

* [Privacy considerations](https://webmachinelearning.github.io/translation-api/#privacy) (delegates to [the corresponding section in _Writing Assistance APIs_](https://webmachinelearning.github.io/writing-assistance-apis/#privacy))
* [Security considerations](https://webmachinelearning.github.io/translation-api/#security) (delegates to [the corresponding section in _Writing Assistance APIs_](https://webmachinelearning.github.io/writing-assistance-apis/#security))

> 16.  Do features in your specification enable origins to downgrade default
>      security protections?

No.

> 17.  What happens when a document that uses your feature is kept alive in BFCache
>      (instead of getting destroyed) after navigation, and potentially gets reused
>      on future navigations back to the document?

Ideally, nothing special should happen. In particular, `Translator` and `LanguageDetector` objects should still be usable without interruption after navigating back. We'll need to add web platform tests to confirm this, as it's easy to imagine implementation architectures in which keeping these objects alive while the `Document` is in the back/forward cache is difficult.

(For such implementations, failing to bfcache `Document`s with active `Translator` or `LanguageDetector` objects would a simple way of being spec-compliant.)

> 18.  What happens when a document that uses your feature gets disconnected?

As with the previous question, nothing special should happen: the objects should still stay alive and be usable.

As with the previous question, it's easy to imagine implementations where this is difficult to implement. We may need to add a check in the specification to prevent such usage, if prototyping shows that the difficulty is significant.

> 19.  What should this questionnaire have asked?

Seems fine.
