# GlassChat — The Complete Guide to Building a Scalable Encrypted Chat Application

GlassChat is a modern, real-time, offline-first multi-user chat application built for Android. It features **military-grade end-to-end encryption (AES-256-GCM, X25519 ECDH)**, encrypted local storage (SQLCipher), a stunning glassmorphic UI, real-time messaging, encrypted media sharing (images, videos, voice notes, documents), encrypted voice/video calling (WebRTC), and content-stripped push notifications.

**Version 4.0.0** — Major security update with comprehensive end-to-end encryption across all data layers.

This guide explains the complete architecture and step-by-step process of how to build this application **from scratch**, focusing heavily on the configuration of Cloud Services (Firebase, Supabase, OneSignal) and the end-to-end encryption layer without showing the underlying Android source code.

## 🛠️ Technology Stack

| Component | Technology Used | Purpose |
| :--- | :--- | :--- |
| **Language** | Kotlin | Primary programming language |
| **UI Framework** | Jetpack Compose + Material 3 | Declarative UI, animations, glassmorphic themes |
| **Authentication**| Firebase Auth | User identity and secure login |
| **Database (Remote)**| Firebase Firestore | Real-time messaging, presence, call signaling |
| **Database (Local)** | Room + **SQLCipher** | **Encrypted offline-first caching (AES-256)** |
| **File Storage** | Supabase Storage | **Encrypted media files (images, videos, voice, docs)** |
| **Encryption** | **AES-256-GCM** + **X25519 ECDH** + **HKDF-SHA256** | **End-to-end encryption** |
| **Credential Storage** | **EncryptedSharedPreferences** | **AES-256-GCM encrypted local credentials** |
| **Notifications** | OneSignal → FCM | **Content-stripped push notifications** |
| **Image Loading** | Coil | Efficient asynchronous image rendering |
| **Media Playback** | Media3 (ExoPlayer) | Video and audio message playback |
| **Video Calling** | Stream WebRTC | Peer-to-peer voice & video calls |
| **Video Compression** | Media3 Transformer | **Hardware-accelerated video compression** |

---

## 🔐 End-to-End Encryption

GlassChat encrypts data at every layer:

### What's Encrypted

| Data | Where Stored | Encryption |
| :--- | :--- | :--- |
| Message text, sender name/email, replies | Firestore | AES-256-GCM with conversation key |
| Media files (photos, videos, voice, docs) | Supabase Storage | AES-256-GCM with per-file key |
| Call signaling (SDP, ICE candidates, names) | Firestore | AES-256-GCM with conversation key |
| Profile bio & phone number | Firestore | AES-256-GCM with user-local key |
| Push notification preview | OneSignal data payload | AES-256-GCM with conversation key |
| Conversation last message | Firestore | AES-256-GCM with conversation key |
| All credentials & cached keys | Device SharedPreferences | EncryptedSharedPreferences (AES-256) |
| Local chat database | Device SQLite | SQLCipher (AES-256-CBC) |

### Key Exchange

- **DM conversations**: X25519 ECDH key agreement → HKDF-SHA256 → AES-256 conversation key. Both parties derive the same key independently.
- **Group conversations**: Random AES-256 group key, envelope-encrypted per member using ECDH-derived wrapping keys. Stored in Firestore `conversations/{id}/memberKeys`.
- **Identity keys**: X25519 keypair generated on first login, stored in EncryptedSharedPreferences (backed by Android Keystore). Public key published to Firestore `users/{uid}/publicKey`.

### Backward Compatibility

Messages without the `ENC:` prefix are treated as plaintext (pre-encryption messages). The `smartDecrypt()` function automatically detects and handles both formats.

> **See also**: `changes.md` for complete encryption architecture documentation.

---

## 🏗️ Step-by-Step Setup Guide

### Phase 1: Environment & Project Initialization

**1. Create the Android Project**
* Open Android Studio and create a new project using the **Empty Compose Activity** template.
* Set the minimum SDK to **26 (Android 8.0)** and Target SDK to **35**.
* Enable Kotlin Symbol Processing (KSP) for Room database support.

**2. Configure Dependencies**
* Add dependencies in `build.gradle.kts` for Compose, Navigation, Room, Coil, ExoPlayer, Firebase BoM, Supabase BoM, OneSignal, SQLCipher, and `security-crypto`.
* Implement a `local.properties` setup to securely load API keys without exposing them in version control.

---

### Phase 2: Firebase Configuration (Database & Auth)

Firebase acts as our primary Authentication provider and our real-time messaging backbone.

**1. Project Setup**
* Go to the [Firebase Console](https://console.firebase.google.com/) and create a new project.
* Click the **Android** icon to add an app. Enter your package name (e.g., `com.nityam.glasschat`).
* Download the `google-services.json` file and place it inside the `app/` directory of your Android project.

**2. Authentication Setup**
* In the left menu, go to **Build > Authentication**.
* Click **Get Started**, navigate to the **Sign-in method** tab.
* Enable **Email/Password**.

**3. Firestore Database Setup**
* Go to **Build > Firestore Database** and click **Create database**.
* Choose **Production mode** (we will configure rules next) and pick a location close to your users.

**4. Firestore Security Rules**

To secure the chat and support encryption, you need rules that:
- Guarantee users can only read/write their own data
- Allow encryption fields (`publicKey`, `publicKeyAlgorithm`, `encryptedFileKey`, `groupKeyOwnerUid`, `memberKeys`)
- Increase size limits for encrypted values (bio: 500, phone: 200, text: 10000)

The complete rules are in `firestore.rules` in the project root. Deploy with:

```bash
firebase deploy --only firestore:rules
```

Or paste the content of `firestore.rules` into the Firebase Console → Firestore → Rules tab.

**5. Firestore Indexes**

No manual composite indexes are required. GlassChat handles complex sorting locally using Room. All Firestore queries rely on single-field lookups that Firestore indexes automatically.

---

### Phase 3: Supabase Configuration (Encrypted Media Storage)

Supabase serves as the encrypted media storage backend. Media files are encrypted client-side with AES-256-GCM before upload — Supabase never sees plaintext content.

**1. Project & Auth Setup**
* Go to [Supabase](https://supabase.com/) and create a new project.
* Navigate to **Authentication > Providers** and ensure **Email** is enabled.
* Supabase Auth is used only for storage access tokens.

**2. Create Storage Buckets**
Navigate to **Storage** in Supabase and create the following **Private** buckets:
* `avatars` — profile pictures (not encrypted)
* `group_avatars` — group profile pictures (not encrypted)
* `chat_media` — images, videos, voice notes, documents (**encrypted client-side**)
* `chat_wallpapers` — custom wallpapers (not encrypted)

**3. Supabase SQL & RLS Policies**

Go to the **SQL Editor** in Supabase and run the complete SQL from `backend_deployment_guide.md` to set up Row Level Security (RLS) for all buckets.

---

### Phase 4: OneSignal Configuration (Content-Stripped Push Notifications)

OneSignal handles push notification delivery. After the encryption update, OneSignal servers **never see message content** — the push body is always "New message" with an encrypted preview in the data payload.

**1. Obtain FCM Credentials for OneSignal**
* In the Firebase Console, go to **Project Settings > Cloud Messaging**.
* Under **Firebase Cloud Messaging API (V1)**, ensure it is enabled.
* Go to the **Service accounts** tab, click **Generate new private key**.

**2. Setup OneSignal App**
* Go to the [OneSignal Dashboard](https://onesignal.com/) and create a new App/Website.
* Select **Google Android (FCM)**.
* Upload the Firebase Service Account JSON file.

**3. App Keys & Routing**
* Navigate to **Settings > Keys & IDs** in OneSignal.
* Copy the **OneSignal App ID** and **REST API Key**.
* Place both values in Android `local.properties` as `ONESIGNAL_APP_ID` and `ONESIGNAL_REST_API_KEY`.
* This keeps the project on Firebase Spark, but the REST key can be exposed if the APK is decompiled.

**4. Notification Privacy**
* When User A sends a message, the push contains `contents: "New message"` (generic).
* Actual text is in `data.encryptedPreview`, encrypted with the conversation key.
* The receiver's device decrypts the preview locally for display.
* `collapse_id: conversationId` groups notifications per chat thread.

---

### Phase 5: Encryption Implementation Highlights

**1. Key Generation (Signup/Login)**
* On signup, a X25519 keypair is generated. The private key goes to EncryptedSharedPreferences (backed by Android Keystore). The public key is published to Firestore.
* On login, existing keypair is loaded or a new one is generated for the device.

**2. Message Encryption (Sending)**
* Before writing to Firestore, all sensitive fields (text, senderName, senderEmail, replyTo, etc.) are encrypted with the conversation key using AES-256-GCM.
* Encrypted values are prefixed with `ENC:` for detection.

**3. Media Encryption (Uploading)**
* A random AES-256 key is generated per file.
* File bytes are encrypted with this key and uploaded to Supabase.
* The file key is encrypted with the conversation key and stored as `encryptedFileKey` in the Firestore message.

**4. Message Decryption (Receiving)**
* On receiving messages from Firestore, `smartDecrypt()` detects the `ENC:` prefix and decrypts transparently.
* Decrypted plaintext is stored in the local Room database for fast UI rendering.

**5. Local Storage Encryption**
* Room database is encrypted with SQLCipher (AES-256-CBC, 256-bit random passphrase).
* SharedPreferences are encrypted with EncryptedSharedPreferences (AES-256-GCM keys, AES-256-SIV key names).

### Phase 6: Architecture Implementation Highlights

1. **Room Database (Offline-First):** All UI reads from the encrypted local Room database (`LocalMessage` and `LocalConversation` DAOs).
2. **Optimistic Updates:** When a message is sent, it is instantly written to Room (plaintext). Concurrently, it is encrypted and uploaded to Firestore.
3. **Real-time Sync:** The app listens to the `/conversations/{id}/messages` Firestore collection. Changes are decrypted and saved to Room, which automatically updates the Compose UI via Flow.
4. **Media Handling:** Media is compressed locally, encrypted with a per-file AES-256 key, uploaded to Supabase Storage, and the encrypted file key is attached to the Firestore message.
5. **Call Signaling:** WebRTC SDP offers/answers and ICE candidates are encrypted with the conversation key before being written to Firestore call documents.

---

## 📚 Documentation

| Document | Content |
| :--- | :--- |
fields |
| `firestore.rules` | Production Firestore security rules (650+ lines) |
| `SCHEMA.md` | Firestore and Room database schema documentation |