# OJT Tasking System — Handover

## 1. What this is
Two mobile apps (built from web code with Capacitor) that share one **real-time Firebase** backend:

| App | Who | What they can do |
|---|---|---|
| **admin-app** | Admin (email + password) | Dashboard, OJT fields, roster of interns, create/edit/delete tasks, "View" any intern's task list and mark on their behalf |
| **intern-app** | OJT interns (no password) | Pick their own name from the roster, see only their field's tasks, mark Pending / Done / Unfinished |

Changes appear instantly on every device (Firestore live listeners). Pending tasks whose day has passed show as Unfinished automatically.

## 2. Package contents
```
ojt-tasking-system/
  HANDOVER.md            this file
  firestore.rules        security rules (REQUIRED, see step 4)
  firebase.json          rules deploy config
  admin-app/   www/index.html, www/firebase-config.js, capacitor.config.json, package.json
  intern-app/  same structure
```

## 3. Firebase setup (once, ~15 min)
1. https://console.firebase.google.com → **Add project**.
2. **Build → Firestore Database** → Create database (production mode; pick the region nearest your users).
3. **Build → Authentication → Sign-in method**: enable **Email/Password** and **Anonymous**.
4. **Project settings → Your apps → Web (</>)** → register; copy the config values into **both** `admin-app/www/firebase-config.js` and `intern-app/www/firebase-config.js`.
5. **Firestore → Rules** → paste the content of `firestore.rules` → **Publish**. (Or `npm i -g firebase-tools`, `firebase login`, `firebase deploy --only firestore:rules`.)
6. **Create the admin**: Authentication → Users → **Add user** (email + password). Copy that user's **UID**. In Firestore create collection `admins` → document ID = the UID → add any field (e.g. `role: "admin"`). Repeat for extra admins.

## 4. Try it in a browser first
```
npx serve admin-app/www      # sign in with the admin account
npx serve intern-app/www     # open in a second browser/device
```
Test checklist:
- [ ] Admin signs in; a non-admin account is refused.
- [ ] Add an OJT field, add two interns, create a task.
- [ ] Intern app lists only unpicked names; picking one locks it (a second device can't pick it).
- [ ] Intern marks a task Done; admin dashboard updates without refresh.
- [ ] Admin → Interns → **View** shows that intern's tasks.
- [ ] Admin **release** button frees a wrongly picked name.

## 5. Build the mobile apps (Capacitor)
Needs Node 18+, Android Studio (Android), Xcode on a Mac (iOS).
1. Edit `appId` in each `capacitor.config.json` to your own reverse-domain id (e.g. `com.school.ojt.admin`) **before** the next step.
2. For each app folder:
```
npm install
npx cap add android        # and/or: npx cap add ios
npx cap sync
npx cap open android       # Build → Generate Signed Bundle/APK
```
3. Add icons/splash if wanted (`npm i -D @capacitor/assets`, put `assets/icon.png`, `npx capacitor-assets generate`).
4. Publish: Google Play (one-time fee) / Apple App Store (yearly fee). The **admin app can stay private** (share the APK or use TestFlight / internal testing).
After any change to `www/`, run `npx cap sync` again.

## 6. Security model (enforced by `firestore.rules`, not just the UI)
- Only users listed in `admins` can write fields, tasks, or the roster.
- An intern can only claim a **free** profile, and can only hand back **their own**.
- An intern can only write progress for the profile they own; they cannot change another intern's status.
- Interns are anonymous accounts tied to the device. If an intern reinstalls the app or clears its data, the admin presses **release** on their name and they pick it again.

## 7. Data model (Firestore)
`admins/{uid}` · `fields/{id}: name` · `interns/{id}: name, field, claimedBy` · `tasks/{id}: title, desc, field, assigneeId, due, createdAt` · `progress/{taskId_internId}: taskId, internId, status, updatedAt`

## 8. Known limits / suggested next steps
- **Not tested against a live Firebase project** — the code was checked for syntax only. Run the checklist in step 4 before release.
- Firebase libraries load from Google's CDN, so the apps need internet. For offline use, install the `firebase` npm package and bundle it.
- "Unfinished after the day passes" is calculated on the device when the app is open; there are no push notifications or server-side jobs yet.
- Consider Firebase App Check and a daily Firestore export for backups.
- Firebase's free tier is usually enough for a class-sized group; check current pricing.
- Existing data from the online demo page is **not** migrated; start with a fresh roster.
