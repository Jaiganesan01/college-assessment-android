---
description: Build Cloud-based College Assessment Management Android Application
---

# Workflow Overview
This workflow guides you through creating a **free, open‑source, cloud‑hosted Android application** for managing college assessments. It uses **Kotlin**, **Android Jetpack**, and **Firebase** (Authentication, Firestore, Cloud Functions) as the backend. All steps are designed for a team of >2500 students and faculty and can be run on free tiers.

## Prerequisites
- Windows machine with **Android Studio** installed (latest stable version).
- **Git** installed and a GitHub repository created for version control.
- A **Firebase** project (free tier) with Billing disabled (optional for Cloud Functions, but free tier suffices for basic usage).
- Java JDK 11+ (bundled with Android Studio).

## Steps
1. **Create Android Studio project**
   ```
   // turbo
   // In Android Studio: File → New → New Project → "Empty Activity"
   // Package name: com.example.collegeassessment
   // Language: Kotlin
   // Minimum SDK: API 21 (Android 5.0)
   ```
2. **Initialize Git repository and push to GitHub**
   ```
   // turbo
   git init
   git add .
   git commit -m "Initial commit – Android project scaffold"
   git remote add origin <YOUR_GITHUB_REPO_URL>
   git branch -M main
   git push -u origin main
   ```
3. **Add Firebase SDKs**
   - Open `build.gradle (Project)` and add Google Services classpath:
   ```gradle
   classpath 'com.google.gms:google-services:4.4.2'
   ```
   - Open `build.gradle (Module: app)` and add dependencies:
   ```gradle
   implementation platform('com.google.firebase:firebase-bom:33.2.0')
   implementation 'com.google.firebase:firebase-auth-ktx'
   implementation 'com.google.firebase:firebase-firestore-ktx'
   implementation 'com.google.firebase:firebase-functions-ktx'
   implementation 'com.google.android.gms:play-services-auth:21.0.0'
   implementation 'com.google.android.gms:play-services-auth-api-phone:18.0.1'
   ```
   - Apply Google Services plugin at the bottom of the file:
   ```gradle
   apply plugin: 'com.google.gms.google-services'
   ```
   - Sync Gradle.
4. **Configure Firebase project**
   - In the Firebase console, enable **Authentication** (Email/Password and Phone).
   - Add Android app with the package name `com.example.collegeassessment` and download `google-services.json`.
   - Place `google-services.json` in the app module (`app/`).
5. **Set up Firestore data model**
   - Collections:
     - `users` (fields: uid, role[student|faculty], email, phone, name, department)
     - `assessments` (fields: id, title, subject, department, startTime, endTime, durationMinutes, createdBy, questions[])
     - `responses` (fields: assessmentId, studentUid, answers[], submittedAt, score)
   - Create **Firestore security rules** to enforce role‑based access (see step 10).
6. **Implement OTP‑based authentication**
   - Use `FirebaseAuth` for email/password sign‑in.
   - For phone OTP, call `PhoneAuthProvider.verifyPhoneNumber` and handle callbacks.
   - After successful sign‑in, fetch the user document to determine role and store it in a `UserSession` singleton.
7. **Create UI components**
   - **LoginActivity** – email/password fields, "Sign in with Phone" button.
   - **StudentDashboardFragment** – list of assigned assessments (RecyclerView).
   - **FacultyDashboardFragment** – buttons to *Create Assessment*, *View Reports*.
   - **AssessmentActivity** – displays questions, a countdown timer, and a *Submit* button.
   - **CreateAssessmentActivity** – form for faculty to input title, subject, department, duration, and add questions.
   - Use **Material Design 3**, Google Fonts (e.g., `Inter`), and a dark‑mode friendly palette (primary #0D47A1, secondary #FF6F00, background #F5F5F5).
8. **Enforce one‑attempt‑only rule**
   - When a student opens an assessment, check `responses` collection for an existing document with the same `assessmentId` and `studentUid`. If found, block entry and show a message.
9. **Automatic submission on interruption**
   - In `AssessmentActivity`, implement `onPause()`, `onStop()`, and `onDestroy()` callbacks:
     ```kotlin
     override fun onPause() {
         super.onPause()
         if (!isSubmitted) autoSubmit()
     }
     ```
   - `autoSubmit()` gathers current answers, writes a response document with `submittedAt = ServerTimestamp`, and marks `isSubmitted = true`.
   - Also set a **CountDownTimer**; when it finishes, call `autoSubmit()`.
10. **Firestore security rules (example)**
    ```
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        // Users can read/write their own profile
        match /users/{uid} {
          allow read, write: if request.auth.uid == uid;
        }
        // Faculty can create and manage assessments
        match /assessments/{aid} {
          allow create, update, delete: if get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'faculty';
          allow read: if request.auth != null;
        }
        // Students can read assessments assigned to them and write a single response
        match /responses/{rid} {
          allow create: if request.auth != null &&
                         get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'student' &&
                         !exists(/databases/$(database)/documents/responses/$(rid));
          allow read: if request.auth != null &&
                       (resource.data.studentUid == request.auth.uid ||
                        get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == 'faculty');
        }
      }
    }
    ```
11. **Deploy Cloud Functions (optional for OTP verification & analytics)**
    ```
    // turbo
    firebase init functions
    // Choose TypeScript, install dependencies, then write functions in functions/src/index.ts
    // Example: export const onAssessmentSubmit = functions.firestore.document('responses/{rid}').onCreate(...)
    firebase deploy --only functions
    ```
12. **Testing**
    - Run the app on an Android emulator (API 30) and on a physical device.
    - Verify:
      * Email & phone OTP login works.
      * Role‑based navigation.
      * Faculty can create assessments.
      * Student can start, see timer, and auto‑submit on background/minimize.
      * Firestore rules prevent unauthorized reads/writes.
13. **Continuous Integration (GitHub Actions)**
    - Add a workflow `.github/workflows/android.yml` to run lint and unit tests on each push.
    - Example step:
      ```yaml
      - name: Run unit tests
        run: ./gradlew test
      ```
14. **Release**
    - Generate a signed APK/AAB via Android Studio.
    - Publish to **Google Play** (free tier) or distribute the APK directly to students/faculty.
    - Backend remains on Firebase free tier; monitor usage in the console.

---
**End of Workflow**

*Feel free to adjust any step (e.g., replace Firebase with another open‑source backend such as Supabase) while keeping the overall architecture.*
