# gym_buddy — Product Spec (v1)

## What this is

An iPhone + Apple Watch app that reads your Apple Health data, tracks what you eat, and talks to you about your progress in the tone you pick. That includes roasting you when you skip the gym. Later on, it will match people with gym partners.

The core loop: **the watch records → the app notices something happened → the user gets one well-timed line about it.**

## Who it's for

People with an Apple Watch who have a fitness goal and keep falling off. They don't need more charts. They need something that notices when they skip and says so, in a voice they chose.

## v1 scope

In:
1. Apple Health / Apple Watch data reading
2. Food logging from text, photo, and barcode
3. Coach messages in three tones: Motivation, Normal, Roast
4. Friend accountability: invite link, shared streaks, send-a-roast
5. Minimal watch app: complication, quick log, "Roast me"

Out of v1 (planned for later):
- Stranger gym-buddy matching. It needs enough users in one area to work. Keep the data model ready for it (gym location, training schedule), but don't build matching.
- Android. The product depends on Apple Watch.
- Workout plan generation.

---

## Feature 1: Apple Health data

The app reads from HealthKit on the iPhone. The watch writes to Health, and the app reads from there.

**Read:** steps, active energy, basal energy, exercise minutes, workouts (type, duration, calories, heart rate), resting heart rate, HRV, sleep analysis (stages), VO2 max, body mass.

**Write:** dietary energy, protein, carbs, fat (from food logs).

**Background:** `HKObserverQuery` with background delivery. When a workout finishes, the app builds that day's summary and uploads it. This is also when coach messages should arrive.

**What goes to the server: daily summaries only.**

| Field | Example |
|---|---|
| date (user's local) | 2026-09-25 |
| workout_done | true |
| workout_types | ["traditionalStrengthTraining"] |
| workout_minutes | 52 |
| steps | 8,412 |
| active_kcal | 610 |
| basal_kcal | 1,720 |
| sleep_hours | 6.4 |
| resting_hr | 58 |
| hrv_ms | 47 |

Raw HealthKit samples never leave the phone.

Missing data is normal. Handle all of these without errors: no watch, denied permissions, a night with no sleep data, a day with the watch left on the charger.

---

## Feature 2: Food logging

### Inputs
- **Text:** "two eggs, toast with butter, black coffee"
- **Photo:** camera or photo library
- **Barcode:** VisionKit scanner → Open Food Facts

### Pipeline
1. The LLM parses the input into items with estimated grams and returns JSON validated by Pydantic.
2. Each item is looked up in USDA FoodData Central. **The LLM never produces calorie or macro numbers.**
3. The app shows an **editable card**: one row per item, each with a portion slider. The user adjusts it, then confirms.
4. Saved meals go to the server and get written to HealthKit.
5. Photos are deleted from storage as soon as they're parsed.

USDA lookups are cached in a table, since the same foods come up again and again.

### Metrics tracked
- Calories in (logged) vs calories out (active + basal)
- Protein, carbs, fat, fiber
- Water
- Weight trend (if the user logs weight or has a smart scale)
- Later: estimated real maintenance calories from weight trend vs intake

---

## Feature 3: Coach messages (Motivation / Normal / Roast)

### Two layers

**Layer 1: event engine (deterministic Python).** Turns daily summaries and user targets into events. It's testable and always right about the facts.

**Layer 2: tone generator (LLM).** Gets one event, 2–3 numbers, and the tone. Returns one line. It never sees raw data and never decides what happened.

### Events (v1)

| Event | Trigger | Tones allowed |
|---|---|---|
| workout_completed | a workout ≥ 20 min logged today | all |
| skipped_planned_workout | a planned gym day ended with no workout | all |
| streak_hit | streak reaches 3, 7, 14, 30, 60, 100 days | all |
| streak_broken | streak of ≥ 3 ends | all |
| comeback | first workout after ≥ 3 days off | all (roast stays light) |
| inactive_3_days | 3 days, no workout, low steps | all |
| weekly_target_hit | weekly gym target reached | all |
| low_sleep | sleep under 6h | motivation, normal only |
| step_goal_missed | steps under target by end of day | all |
| protein_target_hit | protein target reached | praise only, in every tone |

### Tone examples

| Event | Motivation | Normal | Roast |
|---|---|---|---|
| skipped_planned_workout | "Tomorrow's a clean slate. Get your bag packed tonight." | "You planned a gym day and missed it. Tomorrow?" | "Your gym membership is starting to look like a donation." |
| streak_hit (7) | "Seven days straight. That's a habit now." | "7-day streak." | "A full week. Your couch has filed a missing persons report." |
| comeback | "Welcome back. The hardest rep was walking in." | "First workout in 4 days. Good." | "Look who remembered where the gym is." |

### Hard rules
- **Roasts target behavior only.** Never body size, weight, appearance, or food eaten. This holds in every tone.
- **Food intake is never roast material.** No event is ever created from what someone ate, other than protein_target_hit, which is praise only.
- **A very low calorie day never earns praise or streak credit.**
- **Calorie targets have a minimum.** The backend rejects targets below it. Default floor: 1,200 kcal/day. Revisit before launch.
- The iOS app double-checks: if a roast mentions body, weight or food, it doesn't show it.
- Every rule above has a test.

### Delivery
- A nightly batch pre-generates each user's lines (keeps LLM cost per user low).
- Event-triggered messages (workout just finished) are generated live.
- Max 2 push notifications per day. The user can pick quiet hours.

---

## Feature 4: Friends (v1) → Gym buddy matching (later)

**v1:**
- Invite a friend by link (universal link)
- See each other's current streak and weekly progress (nothing more; no calories or health data)
- "Send a roast" to a friend who skipped. The friend picks tone limits for incoming roasts.

**Later (don't build yet, keep fields in the model):**
- Home gym location (PostGIS point)
- Training schedule (days + time windows)
- Training style and goal
- Swipe/match flow, chat, report + block (required by App Review for apps where strangers interact)

---

## Watch app

Minimal. Every action takes one or two taps.
- Complication / Smart Stack widget: today's status line (e.g. "Gym ✓ · 1,840 / 2,200 kcal · 🔥 6")
- Quick actions: log water, log a meal by voice (dictation → same text parser)
- "Roast me" button
- App Intent: "Hey Siri, roast me"

---

## Onboarding

1. Sign in with Apple
2. Goal: lose fat / build muscle / stay consistent
3. Weekly gym target + which days
4. Calorie and protein targets (suggested defaults, editable, floor enforced)
5. Tone picker with a sample line for each
6. HealthKit permissions, with plain-language reasons for each data type

---

## Data model (draft)

- **users**: id, apple_sub, created_at, timezone
- **profiles**: user_id, goal, weekly_gym_target, planned_days, tone, calorie_target, protein_target, quiet_hours, *(later: gym_location, schedule, training_style)*
- **daily_summaries**: user_id, date, the fields from Feature 1
- **food_logs**: id, user_id, logged_at, source (text/photo/barcode), raw_input
- **food_items**: log_id, name, grams, kcal, protein, carbs, fat, fiber, usda_id
- **usda_cache**: query, usda_id, per_100g nutrients, fetched_at
- **events**: user_id, date, type, payload (numbers used)
- **coach_messages**: user_id, event_id, tone, text, delivered_at
- **invites**: code, inviter_id, created_at, accepted_by
- **friendships**: user_a, user_b, created_at, roast_tone_limit
- **friend_roasts**: from_id, to_id, event, text, sent_at

---

## API (draft; openapi.json is the source of truth once it exists)

- `GET /health`
- `GET /me` · `PUT /me/profile`
- `POST /summaries` (one or more days)
- `POST /food/parse-text` → items
- `POST /food/parse-photo` → items
- `POST /food/barcode` → item
- `POST /food/logs` (confirmed items) · `GET /food/day?date=`
- `GET /coach/today`
- `POST /invites` · `POST /invites/{code}/accept`
- `GET /friends` · `POST /friends/{id}/roast`

---

## Tech stack

- **iOS + watchOS:** Swift 6, SwiftUI, HealthKit, WidgetKit, App Intents, SwiftData, VisionKit, XcodeGen, swift-openapi-generator, Supabase Swift SDK
- **Backend:** Python 3.12, uv, FastAPI, Pydantic v2, SQLAlchemy 2 async, Alembic
- **Data:** Supabase (Postgres, Auth with Sign in with Apple, Storage)
- **Nutrition:** USDA FoodData Central, Open Food Facts (barcodes)
- **AI:** vision-capable LLM behind one swappable interface
- **Payments (later):** RevenueCat
- **Deploy:** Railway or Fly.io

## Repo layout

```
gym_buddy/
├── docs/
│   └── spec.md          ← this file
├── backend/
│   └── openapi.json     ← exported after each backend milestone
└── frontend/
    ├── project.yml      ← XcodeGen source of truth
    └── gym_buddy.xcodeproj  ← generated, don't hand-edit
```

## Privacy and App Store notes

- HealthKit data is never used for ads and never shared with third parties.
- Friends see streaks and weekly progress only, never health or food data.
- Privacy policy lists every HealthKit type read and written.
- Age rating: 17+ (roast mode).
- Any stranger-to-stranger feature ships with report, block and content filtering on day one.

## Open questions

- App name
- Free vs paid split (likely: logging and Normal tone free; photo logging and Roast mode paid?)
- Which LLM for parsing vs tone lines (cost vs quality)
- Exact calorie floor and whether it varies by user
- Should Roast mode auto-soften after a long break (so it doesn't kick someone who's already down)?