# FLAME Shuttle Tracker — Capstone Plan

## Context

FLAME University students currently have no way to know where a campus shuttle actually is or when it will arrive at their stop — they just show up and wait. The capstone goal is an AI-assisted mobile app that shows live shuttle location and predicts arrival time, so students can time their walk to the stop instead of guessing.

**Hard constraint driving every decision below: solo build, 3 weeks, coursework-level coding experience.** That timeline cannot fit a native app, a custom backend, and a trained ML model built from scratch. The plan below is scoped to what is actually achievable in 3 weeks by reusing managed services (Firebase) and a beginner-friendly cross-platform framework (Expo/React Native), with a heuristic (not trained-model) ETA for the MVP and real ML explicitly pushed to "long-term" once trip data exists to train on. The user confirmed access to 1–2 real drivers for a pilot test, which the plan uses in Week 2.

---

## 1. Problem Being Solved
Students have no visibility into shuttle location or arrival time, so they either wait too long at a stop or miss a shuttle because they left too late. There's no live signal — only a static/word-of-mouth schedule.

## 2. Target Users
- **Primary:** FLAME students who ride the campus shuttle daily and want to time their arrival at a stop.
- **Secondary (enables the primary use case):** Shuttle drivers/conductors, who run a lightweight companion app that broadcasts the bus's location. They are not end-beneficiaries but are required participants.

## 3. Core Product
A two-role mobile app (single Expo codebase, two simple modes):
- **Driver mode:** driver opens the app at the start of a route, picks the route, and the app silently pings GPS location every ~10–15 seconds while running.
- **Student mode:** student picks a route, sees the shuttle's live position on a map, picks their stop, and sees an estimated arrival time (ETA) for that stop.

## 4. MVP Scope (achievable in 3 weeks)
- One shuttle, one fixed route, with a hardcoded list of stops (no admin panel to manage routes).
- Driver mode: start/stop broadcasting GPS for that one route.
- Student mode: live map showing the bus marker updating in near-real-time; a list of stops for the route; tap a stop to see ETA.
- ETA is a **heuristic**, not a trained model: remaining distance along the route ÷ recent average speed (computed from the last few GPS pings), recalculated on every update.
- No login, or the simplest possible login (anonymous Firebase auth) — just enough to tell "driver" and "student" app instances apart.
- Tested with at least one real driver on the actual FLAME shuttle route.

## 5. Final / Long-Term Goals (beyond the 3-week capstone)
- Multiple routes and multiple buses tracked simultaneously.
- A real ML-based ETA model (e.g. regression trained on logged historical trip data — time of day, day of week, traffic patterns) replacing the heuristic, once enough pings have been logged to train on.
- Push notifications ("your bus is 5 minutes away").
- Admin dashboard for the transport office to add/edit routes and stops without a code change.
- Ridership/demand analytics to help the transport office optimize schedules.
- Public app store release (Play Store / App Store) instead of Expo Go distribution.

## 6. Features That Should NOT Be in the MVP
- Multi-route / multi-bus support — one route only.
- A trained ML model — no historical data exists yet to train one; use the heuristic instead.
- Push notifications / alerts.
- An admin panel for managing routes and stops (hardcode them).
- Full user accounts, profiles, or social features.
- Crowding/capacity tracking on the bus.
- Payment, ticketing, or attendance features.
- Native iOS and Android app store submission — demo via Expo Go instead.
- Offline mode / background-location-when-app-closed on iOS (background GPS on iOS is a whole extra permissions project by itself).

## 7. AI-Involvement Level
**Level: light/heuristic AI, not deep learning — "AI-assisted estimation," with a clear upgrade path to real ML.**
For the MVP, "AI" = a speed/distance-based ETA estimator (a small, explainable calculation), not a trained model. As a stretch goal *if time allows in Week 3*, a very simple linear regression (e.g. scikit-learn `LinearRegression`) can be trained on the trip data logged during the Week 2 pilot to slightly refine the estimate — but this is explicitly optional, not required for MVP success.

## 8. Why That AI-Involvement Level Is Appropriate
- **No historical data exists on day 1.** A trained model needs training data; this project only starts generating GPS trip logs once the driver pilot begins in Week 2. There isn't time to collect enough data *and* train/validate/tune a model *and* ship a UI in 3 weeks solo.
- **A heuristic is honest and testable.** "Distance remaining ÷ recent average speed" is simple enough to build and debug in days, and its accuracy can still be measured against real pilot trips — giving a legitimate, defensible MVP success metric without pretending to have "AI" that doesn't exist yet.
- **It leaves a real growth story for the capstone writeup/defense**: "the MVP ships a heuristic; the logged trip data it collects is exactly what's needed to later train the regression model described in long-term goals" — this is a stronger, more credible narrative than an over-scoped model that doesn't actually work by the deadline.

## 9. Recommended Tech Stack
Chosen specifically to minimize new concepts for a beginner in a 3-week window by using managed/batteries-included services instead of building a custom backend.

| Layer | Choice | Why |
|---|---|---|
| Mobile app | **Expo (React Native)** | One JS codebase for both driver & student modes; live-reload; run instantly on a phone via Expo Go — no app-store build needed for the demo. Large beginner tutorial base. |
| Maps | **react-native-maps** + Google Maps | Drop-in map component with marker support; free tier is enough for a capstone demo. |
| Location | **expo-location** | Simple foreground GPS polling API, no native code needed. |
| Backend/DB | **Firebase Firestore** (or Realtime Database) | Real-time listeners built in — a Firestore doc update from the driver's phone pushes to the student's phone automatically. No custom server, no REST API to write. |
| Auth | **Firebase Anonymous Auth** | Just enough to distinguish a driver session from a student session; zero login-screen work. |
| ETA logic | Plain JS (haversine distance + rolling average speed) | Runs on-device, no ML infra needed for MVP. |
| Stretch ML | **Python + scikit-learn**, trained offline on exported Firestore logs | Only attempted in Week 3 if the heuristic is done and pilot data exists. |
| Hosting | Firebase free tier | No cost, no server to manage. |

## 10. Technical Architecture
```
Driver's phone (Expo app, "driver mode")
   └─ expo-location polls GPS every ~10-15s
        └─ writes {lat, lng, timestamp, speed} to Firestore doc: routes/route1/liveLocation
        └─ also appends the ping to routes/route1/tripLogs (for future ML training)

Firestore (Firebase)
   └─ liveLocation doc: onSnapshot real-time listener

Student's phone (Expo app, "student mode")
   └─ subscribes to routes/route1/liveLocation via onSnapshot
   └─ renders bus marker on react-native-maps
   └─ on stop selection: runs local ETA heuristic
        ETA = remainingDistanceToStop / recentAverageSpeed(last N pings)
```
No custom backend server is needed — Firestore's real-time sync replaces it. This is the single biggest scope-reduction decision in the stack and is what makes the 3-week timeline realistic for a beginner.

## 11. Development Phases (3 weeks, solo)
**Week 1 — Foundation**
- Set up Expo project (driver mode + student mode as two screens/tabs in one app) and Firebase project.
- Hardcode one route with its stops (lat/lng list) in a JSON file.
- Driver mode: get GPS pings writing to Firestore.
- Student mode: get the map rendering with a live-updating bus marker from Firestore data (can test solo by carrying the "driver" phone around campus).

**Week 2 — Core feature + real pilot**
- Add stop list + stop selection in student mode.
- Implement the ETA heuristic and show it in the UI.
- Start logging each GPS ping to a trip-log collection.
- Run a real pilot: get an actual driver to run driver mode on an actual shuttle run, ride along or track separately as a student, and record how close the ETA prediction was to actual arrival.
- Fix whatever breaks (GPS gaps, app backgrounding, stale data handling).

**Week 3 — Refinement, validation, demo prep**
- Use pilot data to sanity-check/tune the heuristic (e.g. adjust the "recent N pings" window).
- *Stretch, only if ahead of schedule:* train a simple regression on logged trip data and compare its accuracy to the heuristic.
- Polish UI (loading states, "no signal" states, basic styling).
- Run 2–3 more real pilot trips to gather the accuracy numbers used for the MVP success writeup.
- Prepare the live/recorded demo and capstone report.

## 12. Risks and Challenges
- **Driver compliance:** the driver must remember to open the app and keep it running; if they close it, tracking stops. Mitigate by keeping driver mode as close to a single "Start" button as possible.
- **iOS background GPS restrictions:** if the driver's phone is iOS and the app is backgrounded, location updates can stop. Mitigate by scoping the MVP to foreground-only tracking and testing on whatever phone the driver actually has as early as possible.
- **GPS accuracy/signal gaps on campus:** buildings or dead zones can cause jumpy or missing pings — the ETA heuristic must tolerate stale/missing data gracefully (e.g. show "no recent signal" rather than a wrong ETA).
- **Cold-start data problem:** there is no historical data to validate the heuristic against until the Week 2 pilot happens — this pushes real validation late into the timeline, so Week 2's pilot session is the critical path item, not optional.
- **Beginner + new stack in parallel:** Expo, Firebase, and maps are all new tools being learned while building — budget extra time in Week 1 for setup/debugging friction, and be willing to cut scope (e.g. skip the stretch ML model entirely) rather than slip the deadline.
- **Single point of failure:** only one driver/bus pilot means one bad data collection day (driver unavailable, phone dies) can cost a large fraction of the available validation time — try to line up 2 pilot sessions, not 1.

## 13. How to Define MVP Success
- **Functional:** a student, using only the app, can see the shuttle moving live on the map and see an ETA for their chosen stop, tested end-to-end with a real driver on the real route.
- **Accuracy (measurable):** across at least 3 real pilot trips, the predicted ETA is within ±3 minutes of actual arrival at least 70% of the time (numbers are a starting bar — tune after seeing Week 2 pilot data, but must be a measured number, not a guess).
- **Usability:** someone unfamiliar with the app can open student mode and correctly answer "where is the bus and when will it reach my stop" within 60 seconds, with no explanation from you.
- **Demo:** a live (or screen-recorded) end-to-end demo showing an actual driver's phone updating an actual student's phone in real time — this is the single most convincing capstone deliverable and should be treated as non-negotiable.