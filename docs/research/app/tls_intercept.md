# Static TLS interception assessment

## Bottom line

A normal Android Wi-Fi proxy plus a user-installed mitmproxy CA is
not sufficient. The release build excludes user CAs. The Thingclips
API client also contains active OkHttp SPKI-pinning code.

Interception without changing the APK is plausible in an authorized
test environment, but only if both trust checks are handled:

1. Put the mitmproxy CA in the Android **system** trust store on a
   rooted test device/emulator.
2. Verify whether the SDK loaded current pins. If it did, use a
   runtime pinning bypass/instrumentation on the test process. A
   stock, non-rooted phone with only a user CA should fail.

There is one useful qualification: the pins bundled in this APK are
stale in September 2026. The pin builder rejects an entire hostname
when any entry for it is expired. Thus, on a clean install before a
successful remote pin refresh, `createPinner()` should return `null`.
In that state a system-trusted mitmproxy CA may be enough. Do not rely
on this for an existing/authenticated install because cached pins can
replace the bundled list and the SDK can refresh them remotely.

## Manifest and Android trust

- Package: `com.olight.omall`, version `3.2.5` (`239`).
- Minimum SDK 24; target SDK 36.
- Application class is protected/wrapped as
  `com.Proxy.ShellApplication`.
- The application is not marked debuggable.
- `android:networkSecurityConfig` resolves to resource
  `0x7f180009`, named `network_security_config_release`.
- The release config has `cleartextTrafficPermitted="false"` and
  trusts only `<certificates src="system"/>`.
- Its `debug-overrides` also contains only the system store.
- The APK contains a debug config that trusts `user` and `system`,
  but the manifest does not select it.
- The manifest says `usesCleartextTraffic="true"`, but the selected
  network security config is more specific and disables cleartext.

Therefore a user-added CA is excluded on Android 7 and later. A CA
installed as a system root satisfies platform trust but does not, by
itself, satisfy OkHttp certificate pinning.

## Target API call

The target is constructed in `classes6.dex` by the obfuscated method:

`com.thingclips.sdk.home.oo000o.OooO0o(long)`

It builds:

- API name: `m.life.my.group.mesh.list`
- API version: `3.1`
- POST data: `gid=<long>`
- `sessionRequire=true`

The response dispatcher in the same class parses `result` as a
`MeshListResponseBean`. Its lists include `mesh`, `sigmesh`, `beacon`,
and `bleMesh` objects.

The actual HTTP URL is region-selected, not hard-coded beside the API
name. Decrypting the bundled `assets/thing_domains_v1/regions` with
the APK's own AES-CTR routine gives these mobile API bases:

- AY/CN: `https://a1.tuyacn.com`
- AZ/US: `https://a1-us.iotbing.com`
- EU, the bundled default: `https://a1.tuyaeu.com`
- IN: `https://a1-in.iotbing.com`

Capture/filter all four if the account region is uncertain. The
request's API selector will contain
`m.life.my.group.mesh.list`, normally as a signed Thingclips request
parameter rather than a REST path.

## Pinning implementation

The main client is `ThingSmartNetWork` in `classes7.dex`.

- Its static default `ThingSmartNetWorkConfig` sets
  `isSupportSSLPinning=true`.
- `addHttpBuilder()` calls `ThingCertificatePinner.createPinner()`.
- A non-null result is passed to
  `OkHttpClient.Builder.certificatePinner()`.
- No custom hostname verifier is set on this main client, so OkHttp's
  normal hostname verifier remains in force.
- A custom `ThingOKHttpDNS` is installed. This should not inherently
  prevent use of an explicit HTTP proxy, though QUIC/UDP traffic may
  need to be blocked so the client falls back to proxied TCP TLS.
- TLS failure code `50516` reaches
  `handlingCertificateErrors()`. Pinning is downgraded only when the
  stored pin set is considered expired; valid pins are not bypassed.
- The SDK can refresh pins and rebuild the live OkHttp client through
  `CertPinRefresher`.

The decrypted bundled `pins` asset covers exactly the four API hosts
above. Each has three `sha256/...` SPKI pins. The first pin for every
host expired in 2024 or 2025, while two backup pins expire in 2031.
However, `putCersListToCertificatePinner()` requires every cert entry
for a host to pass `checkExpireTimeAllowed()`. One expired entry makes
it omit that whole host. With only the bundled list, all four hosts
are omitted and the method returns no pinner. A refreshed cache may
contain a corrected list and restore pinning.

Other bundled libraries contain TLS helpers and hostname verifiers,
but the target call flows through the main Thingclips OkHttp client
described above.

## Practical authorized test sequence

1. Prefer a disposable rooted emulator/test handset, not a personal
   production device.
2. Install the mitmproxy CA as a **system** CA and configure Android's
   explicit HTTP proxy to the mitmproxy listener.
3. Block UDP/443 during the test to discourage QUIC bypass, while
   leaving TCP/443 available.
4. Start with a clean app data set. Log in and watch the four regional
   API hosts for the target API selector.
5. If TLS succeeds, the stale bundled-pin behavior likely left the
   pinner empty. Preserve the authenticated response in mitmproxy.
6. If platform trust succeeds but OkHttp reports peer/pin failure,
   the app loaded refreshed cached pins. A runtime OkHttp pinning
   bypass is then required; system-CA installation alone cannot work.
7. If no request reaches the proxy, check for QUIC first, then confirm
   Android proxy routing. The custom DNS is less likely to be the
   blocker than UDP/QUIC or certificate rejection.

No APK rewriting/re-signing is needed for the clean-install/stale-pin
case or for a runtime-instrumented test. Plain mitmproxy on an
unmodified, stock, non-rooted phone is not plausible.

## Confidence and limits

This is a static assessment only. It did not use ADB or execute the
app. Static evidence establishes the selected CA policy, the pinning
call sites, bundled pin/region data, and the target request builder.
Runtime state determines whether a newer pin cache already exists and
whether HTTP-over-QUIC is selected for this particular request.
