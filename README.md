# App Store Release — a Claude skill

A skill that lets Claude take an iOS app from a git push to a live App Store release, the way a person would: Xcode Cloud builds the app, and Claude works App Store Connect in your own Chrome.

It was written from a real release: a subscription app shipped with a 16-language listing. Every pitfall in it was hit and fixed along the way.

## What it covers

1. **Ground rules.** What Claude never does for you: passwords, banking, tax, agreements, API keys.
2. **Working App Store Connect through Chrome.**
   - Filling its React forms so the changes actually save.
   - Picking the right language before writing into it.
   - Long tasks without browser timeouts.
3. **Build setup.**
   - Xcode Cloud with XcodeGen.
   - Correct version numbers.
   - iPhone-only builds.
   - App Attest team IDs.
4. **Store languages.** Why the App Store's "Languages" line comes from the app binary, and how to list many languages without mirroring an English UI.
5. **App Store Connect, step by step.**
   - Agreements, app info, pricing and countries, privacy.
   - Subscriptions, including the review-screenshot size and the "submit with its group" rule.
   - The version page, localized listings, submission and release.
6. **Errors and fixes.** A table of the real error messages and what solved each one.

## Install

**Claude apps (claude.ai, desktop, mobile)**
1. Download this repo as a ZIP, or zip the `app-store-release` folder.
2. Turn on **Settings → Capabilities → Create and edit files**.
3. Go to **Customize → Skills → + → Create skill → Upload a skill**, and choose the ZIP.

**Claude Code**
```sh
git clone https://github.com/Mephisto1122/app-store-release-skill
cp -r app-store-release-skill/app-store-release ~/.claude/skills/
```

## Use

Ask in plain words, for example "submit version 1.2 to the App Store" or "add Hebrew and Spanish to my App Store listing". Claude picks up the skill by itself.

You'll need:
- the Claude in Chrome extension;
- Xcode Cloud set up for your app (no Mac needed during the release);
- App Store Connect open and signed in.

Claude stops and asks you to sign in, accept agreements, or upload files it can't upload itself.

## License

MIT
