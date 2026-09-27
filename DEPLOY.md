# Setting up push reminders for Bunny's Planner

This adds real push notifications (work even with the app closed) at 9am, 12pm,
3pm, 6pm and 9pm local time. It needs three things deployed alongside your
existing Firebase project: a Cloud Function, a service worker file, and a
couple of static files. None of this touches your existing Firestore data or
the pairing/sync features - it's additive.

## 1. Get a VAPID key (2 min)
1. Go to the [Firebase Console](https://console.firebase.google.com) → your
   `bunnybuddy-6622c` project → **Project settings** → **Cloud Messaging** tab.
2. Under **Web Push certificates**, click **Generate key pair**.
3. Copy the key string it gives you.
4. Open `index.html`, find the line near the top:
   ```js
   var VAPID_KEY = "PASTE_YOUR_VAPID_PUBLIC_KEY_HERE";
   ```
   and paste your key in there.

## 2. Put the static files in place
`manifest.json`, `icon-192.png`, `icon-512.png`, and `firebase-messaging-sw.js`
all need to sit in the **same folder as `index.html`**, at whatever URL serves
your app (e.g. if `index.html` is at `https://yourapp.com/index.html`, these
need to be at `https://yourapp.com/manifest.json`, etc. - not in a subfolder).

If you don't already have a host for this, Firebase Hosting is the easiest
match since you're already on Firebase:
```
npm install -g firebase-tools
firebase login
firebase init hosting     # pick your existing bunnybuddy-6622c project,
                           # public directory = the folder with these files
firebase deploy --only hosting
```

## 3. Deploy the Cloud Function
The `functions/` folder contains the scheduled sender. This step requires
the **Blaze (pay-as-you-go) plan** on your Firebase project - scheduled
functions need Cloud Scheduler, which isn't available on the free Spark plan.
For a 2-person app checking in a few times a day, the actual cost will be
essentially $0 (well within the free monthly quota that Blaze still includes).

```
firebase init functions    # pick the same project, choose "JavaScript",
                           # when it asks to overwrite index.html/package.json,
                           # say NO and instead copy in the files from
                           # this functions/ folder
cd functions
npm install
firebase deploy --only functions
```

By default the reminders fire in **Central time** (`America/Chicago`). If
that's wrong, open `functions/index.js` and change the `TIMEZONE` constant
near the top, then redeploy with `firebase deploy --only functions`.

## 4. Turn it on in the app
Open the app → Settings → Reminders → tap to turn on. Your phone will ask
for notification permission - allow it. On iPhone, push notifications for
web apps only work if you've added the app to your Home Screen first
(Share button → Add to Home Screen) - a regular Safari tab can't receive
them at all, that's an Apple platform restriction, not something the code
can work around.

## Changing the schedule
The cron string `"0 9,12,15,18,21 * * *"` in `functions/index.js` sets the
hours (24hr, in `TIMEZONE`). E.g. for every 2 hours from 8am-10pm instead:
`"0 8,10,12,14,16,18,20,22 * * *"`.

## Notes / limits
- Firestore stores one token per device at `pushTokens/{uid}`, written when
  reminders are turned on and deleted when turned off. The Cloud Function
  reads this collection each run and prunes tokens Firebase reports as dead
  (app uninstalled, permission revoked, etc.) automatically.
- This piece can't be tested from within this chat - I don't have network
  access in this sandbox to actually deploy or fire a real push. Everything
  above has been checked for syntax correctness, but the first real test
  will be on your end after deploying.
