---
caip: 154
title: Restrict Web3 Provider API Injection
description: Wallet guidance for restricting Web3 Provider API access to secure contexts for improved privacy and security for wallet users
author: Yan Zhu (@diracdeltas), Brian R. Bondy (@bbondy), Andrea  Brancaleoni (@thypon), Kyle Den Hartog (@kdenhartog)
status: Draft
type: Standard
created: 2022-10-19
updated: 2022-10-19
---

## Abstract

Historically the web platform has had a notion of “powerful” APIs like [geolocation][w3c-geolocation] and [camera and microphone usage][w3c-mediastreams], which are subject to additional security restrictions such as those defined by [secure contexts][w3c-secure-contexts]. Since the Web3 Provider APIs allow dApp websites to request access to sensitive user data and to request use of user funds, new Web3 Provider APIs generally should align to the security considerations of secure context in order to better protect the users data and users funds managed via the web.

## Motivation

Wallets are oftentimes maintaining security and safety of users' funds that can be equivalent to large portions of money. For this reason, it's a good idea to restrict access to the Web3 Provider APIs to align it with other powerful APIs on the web platform. This will assist in reducing the surface area that attacks can be conducted to access users funds or data. Additionally, by adding in restrictions we're reducing the surface area that malicious web pages could fingerprint the user via the Web3 Provider APIs providing some additional privacy benefits. An example of a specific attack that's avoided by this is one where a malicious advertisement is loaded on a legitimate dApp that attempts to interact with a users wallet to maliciously request the user to access funds. With this CAIP implemented the advertisement frame would be considered a third party iframe and therefore would not have the Web3 Provider API injected into it's sub frame because it's not a secure context.

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Restrictions for providers

The provider objects, e.g. `window.ethereum` or a [Solana wallet adapter][solana-wallet-adapters], are expected to only inject the Web3 provider APIs in secure context when conforming with this standard. The following restrictions are REQUIRED for conformant wallets:

- Provider objects MAY be accessible in private (incognito) windows.
- The origin MUST be a [potentially trustworthy origin][w3c-secure-context-trustworthy-origin] to have access to the Web3 provider APIs. This can be checked using `window.isSecureContext`, including inside iframes.
    - Secure contexts include sites that are served from HTTPS but also HTTP `localhost`. 
    - The browser implementation MAY optionally also support configured [potentially trustworthy origins] that would normally not be considered trustworthy if the user configures their browser to do so. See the [Development Environments section of Secure Contexts][w3c-secure-context-dev-env] for additional details. For example, in Chromium based browsers this is done via the `chrome://flags/#unsafely-treat-insecure-origin-as-secure` flag.
- By default the Web3 Provider APIs MUST NOT be exposed to third-party iframes.
- Web3 provider APIs MUST be `undefined` in an iframe where `window.isSecureContext` returns `false` in that iframe.
- If the iframe is a third party to the top-level secure origin, it SHOULD be blocked. It MAY be unblocked if the iframe uses the `allow="ethereum"` or `allow="solana"` attribute. In order to achieve this implementers are REQUIRED to extend the Permissions API.
- If the iframe is first-party to the top-level origin AND the `sandbox` attribute is set on the iframe, the provider object MUST be blocked. If the sandbox attribute is set to `sandbox="allow-same-origin"` it MUST be injected for a first party frame.
    - Note `"allow-same-origin"` does nothing if the iframe is third-party. The case of the third party iframe is dictated by the usage of the `allow` attribute and the Permissions API as defined in the rule above.

## Rationale

By limiting the capabilities of where the Web3 provider APIs are being injected we can reduce the surface area of where attacks can be executed.Given the sensitivity of data that's passed to the Web3 Provider APIs some basic levels of authentication and confidentiality should be met in order to ensure that request data is not being intercepted or tampered with. While there has been attempts to [limit request access in the ethereum wallet][eip-2255] interface, there's not been limitations that have been set to where these Web3 Provider APIs are expected to be or not be injected. Since the secure contexts web platform API is a well developed boundary that's been recommended by W3C and the fact that the Web3 provider APIs are extending the traditional web platform APIs, no other alternative solutions have been considered in order to extend current established prior art.


## Backwards Compatibility

Wallet extensions SHOULD consider adding a "developer mode" toggle via a UX so that dApp developers have the capability to disable the insecure context (http) check for the http://localhost:<any-port> origin only in the event that [localhost][w3c-secure-context-localhost] does not return `true` for secure context</a>. This will allow dApp developers to be able to continue to host dApps on the localhost origin if a browser environment has chosen to not already consider localhost a secure context. Most major browser providers do consider localhost a secure context already. If such a toggle is made available, it MUST be set to disabled by default.

## Test Cases

### Required Test Cases

- Top level `http://a.com` -> blocked (insecure/top level)
- Top level `https://a.com` -> allowed
- Top level `https://a.com` with `<iframe src="http://a.com/">` -> blocked (insecure first party iframe)
- Top level `http://a.com` with `<iframe src="https://a.com/">` -> blocked (insecure top level window)
- Top level `https://a.com` with `<iframe src="https://a.com">` -> allowed
- Top level `https://a.com` with `<iframe src="https://b.com">` -> blocked (third-party iframe without sufficient privileges)
- Top level `https://b.com` with `<iframe src="http://a.com/">` with `<iframe src="https://b.com">` -> blocked (insecure iframe)
- Top level `https://b.com` with `<iframe src="https://a.com">` with `<iframe src="https://b.com">` -> blocked (third-party iframe without sufficient privileges)
- Top level `https://a.com` with `<iframe src="https://sub.a.com">` -> blocked (third-party iframe without sufficient privileges)
- Top level `https://a.com` with `<iframe src="https://a.com" sandbox>` -> blocked (sandbox attribute without "allow-same-origin")
- Top level `https://a.com` with `<iframe src="https://a.com" sandbox="allow-same-origin allow-scripts">` -> allowed (but note this case is discouraged in https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe#attr-sandbox because it’d allow the iframe to remove its own sandbox attribute)
- Top level `data://foo with <iframe src="data://bar">` -> blocked (insecure top level scheme)
- Top level `file://foo with <iframe src="file://bar">` -> blocked (third-party iframe)
- Top level `https://a.com` with `<iframe src="https://b.com" sandbox="allow-same-origin allow-scripts">` -> blocked (third-party iframe without sufficient privileges)


## Reference Implementation

Test suite link needs to be created and linked here still.

## Security Considerations

### User Enables Developer Mode 

Oftentimes developers require the ability to develop dApps locally in order to test their website and develop while hosting their dApp on http://localhost. In this case localhost would be blocked and compatibility issues would arise when developing a dApp locally. In order to increase compatibility for dApp developers a toggle to disable the check for the localhost can be considered. If this were to be extended beyond the localhost origin it could be used as a means to convince users to enable developer mode in order to subvert the guards put in place by this CAIP. Therefore, implementations should be cautious when extending this developer toggle beyond the scope of the localhost origin.

## Privacy Considerations

### Web3 Provider Fingerprinting

Due to the nature of how the provider object is injected by default into most webpages, there's a risk that a malicious web page may utilize the provider object to fingerprint the user more precisely as a Web3 user. Implementers of this CAIP should consider the risks of injecting the Web3 provider APIs into pages by default in order to consider what privacy characteristics they wish to enable for their users.
## Links

TBD - add better citation section rather than just having markdown reference links

[w3c-mediastreams]: https://www.w3.org/TR/mediacapture-streams/
[w3c-geolocation]: https://www.w3.org/TR/geolocation/
[w3c-secure-context]: https://www.w3.org/TR/secure-contexts/
[w3c-secure-context-localhost]: https://www.w3.org/TR/secure-contexts/#localhost
[w3c-secure-context-dev-env]: https://www.w3.org/TR/secure-contexts/#development-environments
[w3c-secure-context-trustworthy-origin]: https://www.w3.org/TR/secure-contexts/#is-origin-trustworthy
[eip-2255]: https://eips.ethereum.org/EIPS/eip-2255
[solana-wallet-adapters]: https://github.com/solana-labs/wallet-adapter
## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
