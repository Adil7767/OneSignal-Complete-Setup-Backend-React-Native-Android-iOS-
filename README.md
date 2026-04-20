# OneSignal Complete Setup (Backend + React Native Android/iOS)

Use this as a copy-paste checklist to set up OneSignal quickly in a new app without missing steps.

This guide covers:

- OneSignal dashboard setup (Android + iOS)
- Frontend (React Native / Expo) integration
- Backend device registration + send flow
- Environment variables
- Testing and troubleshooting

---

## 1) OneSignal Dashboard Setup

Create (or open) your OneSignal app, then configure each platform.

### 1.1 Android (FCM)

In OneSignal -> Settings -> Android (Firebase):

- Connect Firebase project used by your Android app.
- Use FCM HTTP v1 (recommended).
- Upload Firebase service account JSON (or follow OneSignal Firebase connect flow).
- Ensure Android package name exactly matches app package (example: `com.myapp`).

### 1.2 iOS (APNs)

In OneSignal -> Settings -> Apple iOS (APNs):

- Auth type: `.p8 Auth Key (Recommended)`
- Upload APNs key file: `AuthKey_<KEY_ID>.p8`
- Fill:
  - Key ID: from APNs key
  - Team ID: from Apple Developer membership
  - App Bundle ID: must exactly match iOS bundle identifier (example: `com.myapp`)

If needed, see `IOS_ONESIGNAL_SETUP.md` for Apple-specific details.

---

## 2) Required Environment Variables

## 2.1 Frontend (`myapp-app/.env`)

```dotenv
ONESIGNAL_APP_ID=<ONESIGNAL_APP_UUID>
ONESIGNAL_REST_API_KEY=<optional_on_frontend_not_required_for_sdk>
```

Notes:

- Frontend SDK only needs `ONESIGNAL_APP_ID`.
- Keep REST API keys on backend only for sending server-originated notifications.

## 2.2 Backend (`myapp-backend/.env.*`)

```dotenv
# Push notifications
ONESIGNAL_APP_ID=<ONESIGNAL_APP_UUID>
ONESIGNAL_REST_API_KEY=<ONESIGNAL_REST_API_KEY>

# Optional: OneSignal Email channel (if used)
ONESIGNAL_EMAIL_APP_ID=<ONESIGNAL_EMAIL_APP_UUID>
ONESIGNAL_EMAIL_REST_API_KEY=<ONESIGNAL_EMAIL_REST_API_KEY>
```

---

## 3) Frontend Integration (React Native / Expo)

Current project references:

- Init/listeners: `lib/notifications/pushNotifications.ts`
- Backend sync helper: `lib/notifications/syncOneSignalPlayerToBackend.ts`
- App bootstrap: `app/_layout.tsx`
- Auth-linked sync/login: `api/hooks/useAuth.ts`
- Device API client: `api/services/notifications.ts`

### 3.1 Install packages

```bash
yarn add react-native-onesignal onesignal-expo-plugin
```

### 3.2 Add Expo plugin

In `app.config.js` plugins:

```js
[
  'onesignal-expo-plugin',
  { mode: 'production' }
]
```

### 3.3 Initialize OneSignal at app startup

- Call `initializeOneSignal()` once in root layout.
- Attach listeners:
  - Notification foreground/click listeners
  - Push subscription change listener (`pushSubscription` id updates)

### 3.4 Request permission + fetch player/subscription id

Current flow uses:

- `registerForPushNotificationsAsync()`:
  - requests permission
  - gets `OneSignal.User.pushSubscription.getIdAsync()`
  - retries briefly if not ready
  - stores in AsyncStorage key `onesignal_player_id`

### 3.5 Link OneSignal user to app user after login

On successful auth:

- `setOneSignalExternalUser(String(user.id))` (`OneSignal.login`)

On logout:

- `clearOneSignalExternalUser()` (`OneSignal.logout`)

### 3.6 Sync player id to backend

Current helper:

- `syncOneSignalPlayerToBackend(playerId)`
  - skips if empty id
  - skips if not authenticated
  - POSTs to backend `/devices`

---

## 4) Backend Integration

## 4.1 Device registration endpoints

Routes:

- `POST /devices` -> register/update playerId for current user
- `GET /devices` -> list user devices
- `DELETE /devices/:playerId` -> remove one
- `DELETE /devices` -> remove all user devices

Files:

- Routes: `myapp-backend/src/routes/deviceRoutes.js`
- Controller: `myapp-backend/src/controllers/deviceController.js`
- Mounted at: `myapp-backend/src/routes/index.js` (path `/devices`)

Device model stores:

- `userId`
- `playerId`
- `platform`
- timestamps / lastSeenAt

## 4.2 Sending notifications from backend

Current sender:

- `myapp-backend/src/services/notificationService.js`

It uses:

- `ONESIGNAL_APP_ID`
- `ONESIGNAL_REST_API_KEY`
- OneSignal endpoint `https://onesignal.com/api/v1/notifications`
- payload field `include_player_ids`

---

## 5) iOS Native Requirements

### 5.1 Bundle ID consistency

Must match everywhere:

- Apple App ID
- Xcode target bundle id
- OneSignal iOS APNs config Bundle ID
- Expo config iOS bundle identifier

### 5.2 Notification Service Extension (recommended for rich push)

Project includes:

- `ios/OneSignalNotificationServiceExtension/...`

Keep extension target healthy after prebuild/native changes.

### 5.3 Permission strings

Do not edit only `ios/.../Info.plist` directly if using Expo prebuild.
Set permission strings in `app.config.js` (`ios.infoPlist` and relevant plugins), because prebuild regenerates native files.

---

## 6) Android Native Requirements

### 6.1 Package name consistency

Must match everywhere:

- `app.config.js` android package
- `android/app/build.gradle` namespace + applicationId
- OneSignal Android/Firebase settings
- `google-services.json` package name client entry

### 6.2 Firebase / FCM setup

- Ensure `google-services.json` is valid for current package.
- Ensure OneSignal Android platform is connected to same Firebase project.

---

## 7) Verification Checklist (Do This Every Time)

### 7.1 Frontend logs (dev)

Filter by:

- `[myapp/push]`

Expected sequence (real device):

- requesting permission
- got playerId
- POST /devices
- device upserted OK

### 7.2 Backend checks

- `POST /devices` returns success
- DB has `playerId` for logged-in user

### 7.3 OneSignal dashboard checks

- User/subscription appears under audience/subscriptions
- Test push sent and received

---

## 8) Common Failure Modes

- **"skip POST /devices — not authenticated"**
  - Cause: push id created before login.
  - Fix: login, then trigger sync again (already done in auth flow).

- **"registerForPushNotificationsAsync returned null"**
  - Cause: permission denied, SDK unavailable, emulator/simulator path, or id not ready.
  - Fix: use real device, grant permissions, verify OneSignal app id.

- **Android emulator no push/device registration**
  - Common in dev flow; use real device for reliable push tests.

- **iOS simulator**
  - APNs push delivery not supported for normal app testing; use real iPhone.

- **Info.plist text keeps reverting**
  - Cause: `expo prebuild` regenerated native files.
  - Fix: set in `app.config.js` not only in `ios/.../Info.plist`.

---

## 9) Reuse Template for New Apps

For a new app, change only:

- Bundle ID / package name
- OneSignal App ID
- APNs `.p8` + Key ID + Team ID
- Firebase project and `google-services.json`
- Backend env keys (`ONESIGNAL_*`)

Keep architecture same:

- Frontend registers/subscribes and sends playerId to `/devices`
- Backend stores user-device mapping and sends by `include_player_ids`

---

## 10) Security Rules

- Never commit `.p8`, Firebase service account JSON, or REST API keys.
- Keep REST API keys backend-only.
- Rotate keys if leaked.
- Use separate OneSignal/Firebase projects for staging vs production when possible.

