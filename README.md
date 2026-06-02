# Explainer for the Incognito Static Storage Quota

This proposal is a design sketch by the Chrome Privacy team to describe the problem of quota-based Incognito detection and to solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Participate
- https://github.com/explainers-by-googlers/static-storage-quota/issues
- [Chromium Bug Tracker](https://issues.chromium.org/issues/464484739)

## Introduction

Websites can often detect if a user is browsing in Incognito mode. A common method involves comparing the output of `navigator.storage.estimate()` with other metrics. Due to different storage limitations (disk-based in Regular vs. RAM-based in Incognito), this API often yields different results between two modes, creating a fingerprintable difference. This proposal aims to enhance the effectiveness of the existing `kStaticStorageQuota` feature flag, which currently does not prevent this Incognito detection method on all devices.

## Goals

The primary goal is to improve user privacy by reducing the ability of websites to detect the use of Incognito mode through the `navigator.storage.estimate()` API. Currently, the Incognito can be detected with code as simple as this:

```js
// Example of how a site might try to detect Incognito. According to an internal
// analysis of Incognito detection methods, most examples boil down to the following:
async function checkIncognito() {
    const { quota, usage } = await navigator.storage.estimate();
    jsHeapStorage = performance.memory.jsHeapSizeLimit;
    const THRESHOLD = 1.0 // or similar value, varies across detection scripts
    const isLikelyIncognito = quota < THRESHOLD * jsHeapStorage;
    if (isLikelyIncognito) {
      console.log("Incognito mode possibly detected");
    } else {
      console.log("Regular mode likely");
    }
}
```

## Non-goals

- Addressing all possible Incognito mode detection vectors.
- Changing the underlying storage mechanisms of Incognito mode.
- Modifying other web APIs.

## Use cases

### Use case 1:  Private Browsing without Detection

A user wishes to browse the web in Incognito mode to keep their activity private from other users of the same device. They visit various websites, some of which might employ scripts to detect Incognito mode. If detected, the website might restrict access, alter the user experience, or log the user's attempt to browse privately. The user expects Incognito mode to not be easily detectable by the websites they visit.


## [Potential Solution]

The proposed solution modifies the behavior of `navigator.storage.estimate()` only when in Incognito mode AND when the `kStaticStorageQuota` feature flag (`predictable-reported-quota` in `chrome://flags`) is enabled.

In `kStaticStorageQuota release`, in order to reduce a risk of a web breakage, a fix to prevent overestimating the quota was introduced, clipping the bucketed value to fit into available storage size. In case of Incognito, that meant clipping to the available quota (~15% of device's RAM size), effectively not changing the behavior at all. 

The change will make Incognito mode, under this flag, always report a static quota of **10GiB + current usage**, mirroring the behavior of Regular mode.

This change is implemented under the `kIncognitoStaticStorageQuota` in crrev.com/c/7207198.

### How this solution would solve the use cases

[If there are a suite of interacting APIs, show how they work together to solve the use cases described.]

#### Use case 1

By making `navigator.storage.estimate()` return a similar large, static value (10GiB + usage) in both Regular and Incognito modes (when the feature flag is enabled), we remove the discrepancy that websites exploit. This makes it more difficult for scripts to distinguish between the browsing modes based on this API's output, allowing the user to browse privately without being as easily detected. While it is still possible to detect the Incognito mode, that is the most widespread and simple method available.

```js
async function checkIncognito() {
    const { quota, usage } = await navigator.storage.estimate();
    jsHeapStorage = performance.memory.jsHeapSizeLimit;
    const isLikelyIncognito = quota < THRESHOLD * jsHeapStorage; // not reliable anymore
    if (isLikelyIncognito) {
      console.log("Incognito mode possibly detected");
    } else {
      console.log("Regular mode likely");
    }
}
```

## Detailed design discussion

### Breakage risk

A risk of web breakage in Incognito mode persists. Websites might attempt to use the full 10GiB of storage, which will not actually be available. However, the `navigator.storage.estimate()` API is supposed to be an *estimate* only, hence Chrome is not strictly obliged to guarantee the reported quota.

## Considered alternatives

### [Alternative 1]: Adjusting RAM-based calculatoin

Trying to scale the RAM-based calculation in Incognito to mimic disk space is fraught with complexity and would vary wildly between devices. This approach was deemed less robust than using a fixed static value.

### [Alternative 2]: Use encrypted storage for Incognito mode

This would require significantly more engineering work than the suggested fix.

## Security and Privacy Considerations

This feature is designed to enhance user privacy by making Incognito mode detection harder. It does not introduce new security concerns. The main consideration is the potential for site breakage, as discussed in the "Breakage risk" section. This will be monitored during the Finch experiment.

## Stakeholder Feedback / Opposition

- Internal Teams: The proposal originates from internal analysis of Incognito detection methods.
- External Stakeholders: Feedback will be gathered during the Finch experiment phase and if the feature proceeds towards public rollout.

No significant opposition is known at this stage.
