# Fix #229: Watch companion app support (resign, provision, install)

Fixes altstoreio/AltStore#229 (tracked here for SideStore) — sideloaded apps containing an
Apple Watch companion app.

Paired PR: `SideStore/SideSign` branch `fix/issue-229-watch-companion` (submodule pinned here).

## What was broken

1. **Watch bundles were invisible to the resign pipeline.** `AppBundle` only enumerated
   `PlugIns/*.appex`; anything in `Watch/*.app` was never re-ID'd, never provisioned, never
   signed with its own profile. Installs failed at `installd` with
   `InvalidCompanionAppBundleIdentifier` (the watch app's `WKCompanionAppBundleIdentifier`
   still pointed at the un-mangled parent bundle ID).

2. **Profiles requested for watch bundles are not watchOS profiles.** The standard
   developerservices2 flow yields profiles with `platforms: [iOS, xrOS, visionOS]`.
   watchOS `installd` requires a **watchOS-platform** profile and rejects otherwise:
   `MIInstallerErrorDomain Code=13, 0xe8008015 — A valid provisioning profile for this
   executable was not found` — even when the watch's UDID is in the profile.

## What this branch fixes (part 1 — structural)

- `SideSign/AppBundle`: enumerate `Watch/*.app` (`watchApps`) and their nested extensions;
  `allNestedBundles` covers extensions + watch apps + watch-app extensions.
- `SideSign/AppBundleSigner`: prepare entitlements and embed profiles for watch bundles
  (zsign already deep-walks `Watch/`).
- `FetchProvisioningProfilesOperation`: register App IDs + profiles for all nested bundles,
  using **structural** ID remapping for watch bundles — `com.app.watchkitapp` →
  `{resignedParent}.watchkitapp` (parent-prefix rule; suffix-append here reproduces the
  installd error one level deeper).
- `ResignAppOperation`: rewrite watch app `CFBundleIdentifier`, set
  `WKCompanionAppBundleIdentifier` to the parent's final resigned ID, fix
  `WKAppBundleIdentifier` in WatchKit extensions.
- `PrepareAppExtensionBundleIDsOperation`: `useMainProfile` path covers watch bundles.

Verified: full app builds; resign of a real Watch-embedded IPA produces exactly the ID
remap installd demands; iPhone-side install of a Watch-embedded app now succeeds on device
(previously hard-failed).

## Device verdict (2026-09-19, iPhone Air iOS 27 + Watch Ultra 3 watchOS 26, free account)

With this branch (structural fix + watchOS-platform profiles via `DTDK_Platform=watchos`):

- iPhone install: ✅ works end to end.
- Watch profile: fetched with the watch device type, accepted by the portal.
- Watch-app "Install" (companion transfer / `appconduitd`): ❌ **`MIInstallerErrorDomain
  Code=111` — "authorized by a free provisioning profile, but apps validated by those are
  not allowed to be installed from this source."** This is a watchOS **policy gate on the
  transfer channel**, not a signing defect: the same signed bundle no longer produces
  `0xe8008015`. Companion transfer appears usable only for paid-team profiles.
- Conclusion for free accounts: the watch app must be installed over the **developer
  channel** (companion_proxy / RSD direct install — the path Xcode and
  [nab138/isideload PR #12](https://github.com/nab138/isideload/pull/12) use). For SideStore
  that would mean an idevice-gateway install path to the watch rather than handing off to
  the iOS Watch app.

This PR therefore fixes everything fixable in the resign/provisioning pipeline (and is
required groundwork for either channel); the final hop for free accounts needs the direct
install path. Paid-team accounts are expected to work with companion transfer as-is
(untested here).

**Update 2026-09-20: end-to-end success confirmed on a free account** using the developer
channel: the same device pair, signed via the recipe above and installed with
[nab138/isideload PR #12](https://github.com/nab138/isideload/pull/12)'s watch install path
(companion_proxy + direct install to the watch), runs on the wrist. Two auth fixes were also
needed to log in at all (GSA blocks the Xcode client identifier — isideload PR #14 — and
answers a share of well-formed requests with spurious 429s that must be retried — PR #19).
This confirms: the blocker for the Watch-app Install button is purely the Code=111 channel
policy; everything else in this PR's pipeline is correct and sufficient.

## What is still needed (part 2 — provisioning platform; recipe proven elsewhere)

[nab138/isideload PR #12](https://github.com/nab138/isideload/pull/12) validated free-account
watch installs on real hardware. The missing pieces, confirmed against our device logs:

1. Register the paired watch's UDID as a **watchOS device type** (UDID obtainable at runtime
   via `companion_proxy.get_device_registry()`).
2. Request the watch bundle's profile on the normal `…/QH65B2/ios/…` endpoint but add
   **`DTDK_Platform = "watchos"`** to the request body → Apple issues a true
   watchOS-platform profile.
3. Provision iOS and watchOS App IDs separately; skip app-group features on the watch App ID.
4. Install path: the Watch-app "Install" button (`appconduitd` companion transfer) is
   unproven for free accounts even with correct profiles; the proven route is
   `companion_proxy` port-forward to watch lockdownd (62078) → watch `installation_proxy` →
   stream the signed watch app via `com.apple.streaming_zip_conduit` (Xcode's own path;
   plain usbmuxd, no CoreDevice needed).
5. User-side prerequisites: watch Developer Mode ON; PPQ verification (Apple Account
   password prompt on/via the watch) must be able to complete online.

## Device evidence (iPhone Air iOS 27 + Watch Ultra 3 watchOS 26, free account)

- Before: `InvalidCompanionAppBundleIdentifier` at iPhone install time.
- After part 1: iPhone install OK; watch transfer completes; watch installd rejects with
  `0xe8008015`; captured profile shows `platforms` lacking `watchOS` — root cause of part 2.
- `Code=111 (free profile not allowed from this source)` observed once on an earlier
  signing attempt; superseded by the platform finding.

## Notes

- Each embedded watch app consumes one extra free-account App ID (10 per 7 days cap).
- Not yet wired: parts 2.1–2.4 above. This PR is submitted as the structural half with the
  provisioning half specified; happy to split/land incrementally.
