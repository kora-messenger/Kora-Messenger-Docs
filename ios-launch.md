# Kora Messenger — iOS Launch Guide

The `ios/` project is fully scaffolded (Flutter 3.47, bundle ID `com.kora.messenger`,
matching Android). `ios-build.yml` compiles the unsigned `.app` on every relevant push.

## What's already done

- Xcode project + workspace (Runner), Swift Package Manager plugin integration
- All 10 permission strings in `Info.plist` (mic, camera, contacts, photos, Face ID, speech, Bluetooth, local network)
- Background modes: audio, voip, remote-notification
- `PrivacyInfo.xcprivacy` (Apple privacy manifest)
- Custom native plugins registered: OnDeviceTranslation, VoiceVectorPlugin, RealtimeDspPlugin
- Asset catalog converted to PNG (Apple requirement)
- CI: `Build Kora Messenger iOS` workflow (macOS runner, unsigned build, artifact upload)

## Blocked on Apple Developer Program ($99/yr)

1. **APNs key** — Apple → Certificates/Identifiers → Keys → create APNs Auth Key,
   download `.p8`, upload to Firebase Console (Project settings → Cloud Messaging → APNs).
   This is the only hard blocker for push notifications on iOS.
2. **GoogleService-Info.plist** — Firebase Console → add iOS app with bundle ID
   `com.kora.messenger` → download plist → place at `ios/Runner/GoogleService-Info.plist`
   (committed). Without it the app compiles but crashes at launch (firebase_core).
3. **Signing** — add signing certs + provisioning profile, change CI to
   `flutter build ipa` and ship via TestFlight (altool/App Store Connect API key).

## App Store review checklist (before submitting)

- koramessenger.com live with privacy policy + terms (Play requires this too)
- In-app account deletion (Apple requires it — already built for Android parity)
- E2EE security documentation ready — Apple scrutinizes encryption claims
  (`ITSAppUsesNonExemptEncryption` is set in Info.plist)
- iPad: verify wide-screen layouts on chat list + chat screens
