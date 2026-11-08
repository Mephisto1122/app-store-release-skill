---
name: "app-store-release"
description: "Ship an iOS app to the App Store end to end: Xcode Cloud builds, App Store Connect in Chrome, subscriptions, listings in many languages, review, release."
---

# Shipping an iOS app to the App Store, A to Z

Use this when the person wants an iOS app built, put on TestFlight, listed, submitted for review or released. App Store Connect (ASC) work is done in the person's Chrome (Claude in Chrome). Builds come from Xcode Cloud, triggered by a git push. Nothing here needs the person's Mac to be driven by computer use.

## 0. Ground rules (always)

- **Never** type passwords or sign in for the person. If ASC or developer.apple.com shows a login page, ask them to sign in in that tab and say when.
- **Never** enter bank/IBAN, tax forms, or government IDs, and **never** accept agreements (Paid Apps Agreement, Small Business Program declarations, EU trader declarations). Open the page, explain what to choose and why, and let the person do it.
- **Never** create, paste or rotate API keys (.p8 etc.). Server secrets go only into the server's secret store, set by the person.
- Submitting for review and pressing **Release** are actions the person must have asked for in chat. Pulling a version from review, deleting a version, removing a build: confirm first.
- Keep release on **Manual** unless the person says otherwise.

## 1. Chrome technique for App Store Connect

Load the tools once: tabs_context_mcp, navigate, computer, find, get_page_text, javascript_tool, form_input, browser_batch, file_upload.

- Start with tabs_context_mcp. Reuse the person's ASC tab if they point to it; otherwise open a new one.
- ASC is a slow React app. After navigate, wait 4–8 s; if get_page_text returns nothing or a spinner, wait again or re-navigate to `https://appstoreconnect.apple.com/apps` and click into the app.
- **Reading fields:** `document.getElementById(id).value`. Version page ids: `promotionalText`, `description`, `whatsNew`, `keywords`, `supportUrl`, `marketingUrl`, `versionString`, `copyright`, `notes` (review notes). App Information ids: `name`, `subtitle`.
- If a JS result is refused as "cookie/query string data", return smaller pieces (e.g. `.slice(0,800)`), or lengths only.
- **Writing React fields:** use the native setter and fire events, or the UI won't register the change:
  ```js
  const setT = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype,'value').set;
  const setI = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set;
  (el.tagName==='TEXTAREA'?setT:setI).call(el, v);
  ['input','change','blur'].forEach(t=>el.dispatchEvent(new Event(t,{bubbles:true})));
  ```
  Then click **Save** and confirm it shows a check mark ("Saved").
- Native `<select>`s: use form_input with the option **value** (e.g. `ONE_MONTH`), not the label.
- Menus (like the localization language picker): click the opener, then click the leaf element whose text equals the item exactly. Before filling, assert the picker now shows the intended language, so text never lands in the wrong localization.
- JS calls time out at ~45 s. For loops over many items, start an async loop that pushes results to `window.__log` and return at once; read `window.__log` after waiting.
- **File uploads:** file_upload only accepts files the person gave this session. Files Claude made (resized screenshots etc.) cannot be uploaded by Claude: send the file to the person and tell them exactly where to click to choose it.
- Use browser_batch for predictable sequences (open menu → pick → wait → fill → Save → zoom on the Save button to confirm).

## 2. Project setup that avoids the classic build failures

- **Xcode Cloud + XcodeGen:** the repo has no .xcodeproj, so add `ci_scripts/ci_post_clone.sh` (chmod +x):
  ```sh
  #!/bin/sh
  set -e
  cd "$CI_PRIMARY_REPOSITORY_PATH"
  command -v xcodegen >/dev/null 2>&1 || brew install xcodegen
  xcodegen generate
  ```
- In project.yml Info properties set `CFBundleShortVersionString: $(MARKETING_VERSION)` and `CFBundleVersion: $(CURRENT_PROJECT_VERSION)`, or every build says 1.0.
- iPhone-only app: set `TARGETED_DEVICE_FAMILY: "1"` **at target level** (XcodeGen's iOS preset sets "1,2" there), or ASC demands 13-inch iPad screenshots.
- Bump `MARKETING_VERSION` for each new App Store version (1.0.1 → 1.0.2). Xcode Cloud sets the build number itself. A build for an already-released version fails (closed train).
- Xcode Cloud workflow: Build + Archive with App Store Connect distribution, post-action TestFlight internal group. Every push to the branch builds; check TestFlight → Builds for "Complete".
- Don't upload from the person's Xcode Organizer when Xcode Cloud is set up; mismatched versions fail there.
- **App Attest:** the server must verify against the team that owns the App ID (developer.apple.com → Identifiers → the app → "App ID Prefix"), not whatever DEVELOPMENT_TEAM says locally. A mismatch shows as `rpid / app id mismatch` and every install stays unverified.

## 3. Languages the store shows

The App Store's "Languages" line comes from the **binary**, not from ASC text. To list N languages: add `CFBundleLocalizations` and a `<lang>.lproj/InfoPlist.strings` per language (translate the permission prompts, e.g. NSLocationWhenInUseUsageDescription). Use `pt-BR`, `zh-Hans`. If the UI stays English, force left-to-right (`UIView.appearance().semanticContentAttribute = .forceLeftToRight` and `.environment(\.layoutDirection, .leftToRight)`) so Hebrew/Arabic phones don't mirror the screens. It shows on the store after that version is released.

## 4. App Store Connect, in order

1. **Agreements, Tax, Banking** (Business): the person completes them. Paid Apps Agreement must be Active before anything paid sells.
2. **App Information**: name (≤30), subtitle (≤30), category, content rights, age rating, privacy policy URL. Name changes need a new version.
3. **Pricing and Availability**: price tier, countries. EU needs a trader declaration (DSA): without it, leave EU countries off. Note any country the person wants excluded (e.g. China needs extra licences for some apps).
4. **App Privacy**: answer the data-collection questions to match the app honestly.
5. **In-App Purchases / Subscriptions** (if any):
   - Subscription: create the group (with a group localization), then the product: reference name, product ID, duration, price (pick the base country price; check the equalised prices), localization (display name + description), review screenshot.
   - **Review screenshot size must be exact**, e.g. 1320×2868 for a 6.9" iPhone. A phone screenshot scaled down (e.g. 1184×2576) is rejected; resize with PIL and hand the file to the person to upload.
   - App Store Server Notifications: production and sandbox URLs on the app's App Information page.
   - The **first** subscription must go to review **with an app version**, and **its subscription group must be added to the same submission** ("Your auto-renewable subscription must be submitted with its subscription group").
   - Join the Small Business Program (15% instead of 30%) at developer.apple.com/app-store/small-business-program/enroll; the person makes the declarations.
6. **Version page** (iOS App → + → version number):
   - Promotional text (≤170, changeable any time), description (≤4000), keywords (≤100, comma-separated, no spaces), support URL, marketing URL, copyright, What's New.
   - Subscription apps: the description must state price and period, auto-renewal, how to cancel, and link Terms of Use (EULA) and Privacy Policy; the paywall in the app must show the same.
   - Screenshots: 6.9" iPhone set (1320×2868 / 2868×1320).
   - Build: + → pick the build → Done → Save. To swap, Delete the old build row first.
   - App Review Information: contact, notes explaining how to use the app (no account? say so), demo account only if needed.
   - Release: Manual.
7. **Localizations**: language picker on the version page ("English (U.S.)" ▾) → pick a language under "Not Localized" → fill promo, description, keywords, What's New, support URL → Save. Then App Information → same picker → name + subtitle → Save. Re-load and read back every language to confirm it persisted. Keep the texts in the repo (e.g. tools/appstore/listing.json) and check length limits with a script before entering.
8. **Submit**: version page **Add for Review** (adds to the draft; choose the existing draft), also Add for Review on each new IAP/subscription and the subscription group. Open **Draft Submissions** → all items listed → **Submit for Review**. Expect "N Items Submitted". Review is usually within 48 h.
9. **After approval**: the person says release → version page **Release This Version**. It appears on the store within about an hour (up to 24 h). Then raise any server-side minimum app version only after the update is visible in the store (and ideally a day or two later).

## 5. Errors seen and their fixes

| Message | Fix |
|---|---|
| "…xcodeproj does not exist" on Xcode Cloud | add ci_post_clone.sh running xcodegen |
| "Preparing build (version 1.0)" fails / wrong version | Info.plist version keys from build settings; bump MARKETING_VERSION |
| "You must upload a screenshot for 13-inch iPad" | target-level TARGETED_DEVICE_FAMILY "1" |
| "The dimensions of one or more screenshots are wrong" | exact device size (1320×2868); person uploads |
| "auto-renewable subscription must be submitted with its subscription group" | Add for Review on the group page into the same draft |
| Version removed from review, IAPs gone from draft | re-add them to the draft before resubmitting |
| App Attest `rpid` failures | server team ID = App ID prefix of the bundle |
| ASC shows a login page or endless spinner | person signs in; re-navigate via /apps |
| JS refused as query-string data | read in slices / lengths only |

## 6. Finishing

Tell the person, briefly: what is submitted or live, what they must do themselves (sign-ins, agreements, uploads of files Claude made, pressing Release if they prefer), and the one next step.