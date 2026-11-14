# Play Store submission — Somno

This is the working checklist for getting Somno onto Google Play: the signing details, the
permission declarations, the Data safety answers, and the deletion route. It is the document
`listing/LISTING.md` points at for §2 and §4, and the one `scripts/check-release.ts` reads.

Everything here is written against the source. Where a value has to match something in the app, the
file is named so the two can be checked against each other. Where a step can only be done by the
account owner in a console, it says so.

The marketing copy, screenshots and content-rating questionnaire live in `listing/LISTING.md`.
This file covers what Play audits.

---

## 1. App identity and contact

| Field | Value | Source of truth |
| --- | --- | --- |
| App name | `Somno: Sleep Debt & Alertness` | `listing/LISTING.md` |
| Package name | `com.somno.app` | `app.json` → `expo.android.package` |
| Version name / code | `1.0.0` / `1` | `app.json` → `expo.version`, `expo.android.versionCode` |
| Category | Health & Fitness | `listing/LISTING.md` |
| Content rating | Everyone / PEGI 3 | `listing/LISTING.md` |
| Minimum age | 16 | `MIN_AGE` in `src/engine/debt.ts`; also `legal/privacy.md` and `legal/terms.md` |
| Support email | `support@somno.app` | `SUPPORT_EMAIL` in `src/lib/legal.ts` |
| Privacy policy | `https://somno.app/privacy` | `LEGAL_URLS.privacy` in `src/lib/legal.ts` |
| Terms | `https://somno.app/terms` | `LEGAL_URLS.terms` in `src/lib/legal.ts` |
| Account deletion URL | `https://somno.app/delete-account` | `LEGAL_URLS.deleteAccount` in `src/lib/legal.ts` |

The three legal URLs must resolve to published pages before the listing is submitted. The text for
them is in `legal/`. A dead link rejects the listing outright.

---

## 2. Signing, keys and SHA-1

Somno is built for release with **EAS Build**, which supplies its own upload key. Do not publish a
local `./gradlew bundleRelease` artefact: `expo prebuild` generates a release build type that signs
with the *debug* keystore in some configurations, and Play rejects a debug-signed upload. `npm run
check:release` reports this rather than patching the generated project, because `android/` is
recreated by the next prebuild.

### Upload key

```bash
eas credentials -p android      # choose the production profile
```

Keep the keystore and its passwords out of the repository. Nothing under `.env`, no `*.keystore`, no
`*.jks` and no `android/` directory is committed — see `.gitignore`.

### Google Sign-In SHA-1

Google sign-in fails on a released build, while working in development, unless the SHA-1 of the key
that actually signs the installed APK/AAB is registered. Two places need it:

1. **Google Cloud Console → APIs & Services → Credentials → the Android OAuth client.**
   Add the SHA-1 for package `com.somno.app`.
2. **Supabase → Authentication → Providers → Google → Authorized Client IDs.**
   Add the Android client ID alongside the web and iOS client IDs.

Get the fingerprint from the key EAS signs with:

```bash
eas credentials -p android      # prints the SHA-1 / SHA-256 fingerprint
```

If Play App Signing is enabled, Google re-signs the uploaded bundle. In that case the fingerprint
Play reports under **Release → Setup → App signing** is the one the installed app carries, and that
is the value the Android OAuth client needs — not the upload key's. Register both if you are unsure;
an extra correct fingerprint is harmless, a missing one breaks sign-in.

The **web** client ID is the one Supabase validates, so `EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID` must be
the web client even on Android. See `supabase/SETUP.md §2`.

### Verifying the merged manifest

Permission declarations are merged from the app and from `modules/smart-wake-alarm`. After a
prebuild, confirm the final manifest contains what the listing declares:

```bash
npx expo prebuild -p android
cd android && ./gradlew :app:processReleaseManifest
# inspect android/app/build/intermediates/merged_manifests/release/AndroidManifest.xml
```

---

## 3. Permission declarations

The app declares only what its features use. Play asks for a justification for each sensitive
permission; the answers are:

| Permission | Why it is needed | Feature |
| --- | --- | --- |
| `CAMERA` | Front camera frames are measured in memory for the optional face scan. No photo is saved or uploaded. | Face scan |
| `POST_NOTIFICATIONS` | Alarm and check-in reminders. | Smart alarm |
| `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` | The wake alarm must fire at the chosen minute, not within a window. | Smart alarm |
| `RECEIVE_BOOT_COMPLETED` | Re-arms scheduled alarms after a reboot. | Smart alarm |
| `VIBRATE` | Alarm and haptic feedback. | Smart alarm |
| `FOREGROUND_SERVICE` | Plays the alarm and shows the dismiss/snooze screen while the phone is locked. | Smart alarm |
| `WAKE_LOCK` | Keeps the alarm activity alive long enough to ring and be dismissed. | Smart alarm |
| `USE_FULL_SCREEN_INTENT` | Shows the alarm over the lock screen. | Smart alarm |

Deliberately **not** requested, and blocked in `app.json`:

- `SYSTEM_ALERT_WINDOW` — no floating overlays.
- `RECORD_AUDIO` — the app never records audio; the camera plugin is configured with
  `microphonePermission: false` and `recordAudioAndroid: false`.
- `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` — exports use the system file picker; the app
  does not read the shared storage tree.

There is no `ACCESS_FINE_LOCATION`, no contacts, no SMS, no call log, no `QUERY_ALL_PACKAGES`.

The exact-alarm permission is declared through the smart-wake-alarm module and its config plugin
(`plugins/withSmartWakeAlarm.js`), and the alarm receivers are in
`modules/smart-wake-alarm/android/src/main/AndroidManifest.xml`.

---

## 4. Data safety

This section is the Play Console **Data safety** form, answered. It must agree word for word with
`legal/privacy.md`, because reviewers read both. If one changes, change the other in the same
commit.

### Summary answers

| Console question | Answer |
| --- | --- |
| Does your app collect or share any of the required user data types? | **Yes** — but only when the user creates an optional account |
| Is all of the user data encrypted in transit? | **Yes** — HTTPS for every request |
| Does your app provide a way for users to request that their data be deleted? | **Yes** — in-app and a web route |
| Is any collected data shared with third parties? | **No** — Supabase is a processor, not a recipient for its own purposes |
| Is any data used for advertising or tracking? | **No** — no ad SDK, no analytics SDK, no crash reporter |
| Is any data used to train AI models? | **No** |

The app is fully functional without an account. If the user never signs in, **nothing leaves the
device**, and the correct "collection" answer for that user is none. Play asks about the app as a
whole, so the form is completed for the signed-in path, and the listing and policy both state that
signing in is optional.

### Data types collected

| Play category | Data type | Collected? | Shared? | Optional? | Purpose | Where it goes |
| --- | --- | --- | --- | --- | --- | --- |
| Personal info | Email address | Yes | No | Yes — only on account creation | Account management, authentication | Supabase Auth |
| Personal info | Name | Yes | No | Yes — only if Google provides one | Account display | Supabase |
| Health and fitness | Health info (sleep logs, check-ins, sleep debt, alertness scores) | Yes | No | Yes — only when signed in | App functionality, sync across devices | Supabase database |
| Health and fitness | Fitness info (reaction-time / PVT results) | Yes | No | Yes — only when signed in | App functionality, sync across devices | Supabase database |
| App activity | Other user-generated content (alarms, alarm events, consent log) | Yes | No | Yes — only when signed in | App functionality, sync across devices | Supabase database |

### Data types **not** collected

| Play category | Data type | Why not |
| --- | --- | --- |
| Photos and videos | Photos | The face scan measures camera frames in memory and discards them. If a recent-scan image is cached on the device, it never syncs and is deleted with app data. |
| Audio | Voice or sound recordings | The microphone is not requested or used. |
| Location | Approximate or precise location | No location permission is requested. |
| Financial info | Payment info, purchase history | There is no billing or paywall in this release. |
| Messages | Emails, SMS, in-app messages | No messaging feature. |
| Contacts | Contacts | No contacts permission. |
| Calendar | Calendar events | No calendar access. |
| Web browsing | History | The app does not read browser history. |
| App info and performance | Crash logs, diagnostics | Faults are logged locally and never sent automatically. |
| Device or other IDs | Advertising ID, device ID | No advertising or analytics SDK is present. |

### Special notes for the reviewer

- **Face data.** The camera frames are processed on-device and are not retained, uploaded, or used
  for facial recognition. The app stores a numerical "facial baseline" (running average and spread
  of brightness/redness/eye-region measurements) so later scans can be scored against the user's own
  normal. This is not a face template and cannot identify a person. Mark the scan as processed
  **ephemerally**, and do not declare "Photos" as collected.
- **No background transmission.** Data is pushed only when the user takes an action (a check-in, a
  log, a settings change) and on explicit restore. There is no background telemetry.
- **Android backup is off.** `allowBackup: false` in `app.json`, so the OS does not copy app data to
  the user's Google account.
- **Supabase region.** Data is held in the region chosen for the Supabase project. State that region
  in the privacy policy before publishing.

### Data deletion

Two routes, both required by Play for apps with accounts:

1. **In-app:** Settings → Data & privacy → Delete. Confirmation is required. This deletes every row
   belonging to the account with `delete_own_account()` (see the end of `supabase/schema.sql`) and
   wipes local storage, cancelling scheduled alarms at the system level.
2. **Web:** `https://somno.app/delete-account` — for users who have already uninstalled the app.
   Requests are completed within 30 days, normally within one working day.

Deletion is immediate, not a queued review. The policy text is in `legal/delete-account.md`.

Play also asks what data is deleted and what is retained. Answer: **all** account-associated data is
deleted; nothing is retained for analytics or legal hold in this release.

---

## 5. Content rating questionnaire

Answered in `listing/LISTING.md`. The short version: no violence, sexuality, profanity, controlled
substances, gambling, or user-generated content sharing; the app does collect health data when an
account is created. Expected rating: Everyone / PEGI 3.

---

## 6. Pre-submission checklist

- [ ] `npm run typecheck` passes.
- [ ] `npm test` passes (on Linux/CI; the `export` suite shells out to `unzip`).
- [ ] `npm run check:release` reports no failures.
- [ ] `legal/privacy.md`, `legal/terms.md` and `legal/delete-account.md` are published at the three
      `LEGAL_URLS` addresses.
- [ ] The operator name in `legal/privacy.md` is replaced with the real legal entity.
- [ ] `support@somno.app` reaches a mailbox someone reads.
- [ ] The upload key's SHA-1 is in the Android OAuth client and Supabase's authorized client IDs.
- [ ] The production AAB is built with EAS, not a local debug-signed bundle.
- [ ] The Data safety answers above are entered in the Console.
- [ ] The content rating questionnaire is completed.
- [ ] At least two phone screenshots are uploaded (eight are provided in `listing/play/`).
