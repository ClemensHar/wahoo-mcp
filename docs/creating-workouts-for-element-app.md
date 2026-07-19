# Wahoo Cloud API: Creating Workouts That Sync to the ELEMNT App

## Key Findings

1. **Plans and Workouts are separate objects.**
   - `create_plan` creates a reusable **template** (intervals, power/HR targets, structure). It has no date and is not tied to a specific ride.
   - `create_workout` creates a **scheduled instance** — a calendar entry with a name, start time, and duration. Attaching a `plan_id` links a template to that scheduled slot.
   - A plan alone (without a linked workout) does **not** appear in the ELEMNT app.

2. **Only `create_workout` triggers app sync — and only within a date window.**
   - A workout must be scheduled from **today through 6 days in the future** to sync to the ELEMNT app.
   - Confirmed by testing: scheduling a workout 3 days in the **past** succeeded via the API (no error, workout confirmed via `list_workouts`), but it **never appeared in the ELEMNT app**. Only future-dated workouts (within the window) synced.
   - This means the date restriction is enforced **client-side** (by the app's sync logic), not by the API at creation time.

3. **Plans are reusable.**
   - The same `plan_id` can be attached to multiple `create_workout` calls, each with a new date and a unique `workout_token`.
   - No need to recreate the plan each time you want to schedule the same session — just create a new workout instance referencing the existing plan.

4. **Third-party integrations (e.g. TrainingPeaks) work differently.**
   - Workouts synced from TrainingPeaks appear directly as scheduled workouts (with their own `plan_id` under the hood), via a separate integration path — not through the same manual two-step process described here.

---

## How-To: Create a Workout That Shows Up in the ELEMNT App

### Step 1 — Create the plan (template)
Define the structure once: intervals, durations, and power/HR targets.

```
create_plan(
  external_id: "<unique-id>",
  filename: "<name>.json",
  plan: {
    name: "<Workout Name>",
    workout_type: "bike",
    description: "<description>",
    intervals: [
      { name, interval_type, duration (seconds), targets: [{ target_type: "power", target_min, target_max, unit: "watts" }] },
      ...
    ]
  },
  provider_updated_at: "<ISO 8601 timestamp>"
)
```

This returns a `plan_id` — save it for reuse.

### Step 2 — Schedule a workout instance, linked to the plan
This is the step that actually makes it appear in the app.

```
create_workout(
  name: "<Workout Name>",
  plan_id: <plan_id from Step 1>,
  starts: "<ISO 8601 datetime, TODAY through +6 days>",
  minutes: <duration>,
  workout_token: "<unique token per instance>",
  workout_type_id: <e.g. 0 = outdoor biking, 12 = indoor biking, 1 = running outdoor>
)
```

**Critical constraint:** `starts` must fall between now and 6 days from now, or the workout will not sync to the app — even though the API call itself will succeed silently.

### Step 3 — Verify
- `list_plans` — confirms the plan exists in your library (cloud-side only, not app visibility).
- `list_workouts` — confirms the scheduled workout exists, including its `plan_id` link.
- Final check: open the ELEMNT app and confirm the workout appears — this is the only way to confirm client-side sync actually happened.

---

## Reusing a Template for Future Sessions

Once a plan exists, repeat only Step 2 with a new date and a new `workout_token` each time:

```
create_workout(
  name: "<Workout Name> - <date>",
  plan_id: <existing plan_id>,
  starts: "<new date, within 6-day window>",
  minutes: <duration>,
  workout_token: "<new unique token>",
  workout_type_id: <same as before>
)
```

No need to touch `create_plan` again unless the interval structure or targets themselves need to change.

---

## Open Questions / Not Yet Verified

- Whether workouts scheduled exactly on day +7 or beyond behave the same as past-dated ones (assumed yes, not tested).
- Whether there's a way to trigger an on-demand sync from the API side (currently appears to be entirely handled by the app itself, outside API control).
