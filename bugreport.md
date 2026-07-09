Bug Report — CoWork: Multi-Tenant Coworking Space Booking API

Each entry lists the file and function, what was wrong, why it produced incorrect behavior, and how it was fixed.


Auth

1. Access tokens lived 900 minutes instead of 900 seconds

File: app/auth.py, create_access_token()
Bug: lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60) — the config value (15) is already minutes, so multiplying by 60 produced 900 minutes (54,000 s) instead of the required exact 900 s expiry.
Fix: lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES).

2. Logout did not actually revoke tokens

File: app/auth.py, get_token_payload()
Bug: revoke_access_token() stores the token's jti in _revoked_tokens, but the check on incoming requests was payload.get("sub") in _revoked_tokens — comparing a user id against a set of jtis. It never matched, so a logged-out token kept working (rule 8 requires 401 after logout).
Fix: check payload.get("jti") in _revoked_tokens.

3. Refresh tokens were reusable (not single-use)

File: app/routers/auth.py, refresh() (+ helpers in app/auth.py)
Bug: /auth/refresh decoded the presented refresh token and issued new tokens without invalidating it. The same refresh token could be replayed forever; rule 8 requires reuse → 401.
Fix: added revoke_refresh_token() / is_refresh_token_revoked() (jti blacklist). refresh() now returns 401 if the jti was already used and marks it used before issuing new tokens.

4. Duplicate registration returned the existing user instead of 409

File: app/routers/auth.py, register()
Bug: when the username already existed in the org, the endpoint returned the existing user's data with a success status instead of failing. Rule 15 requires 409 USERNAME_TAKEN. (Returning the existing account is also an account-takeover-style information leak.)
Fix: raise AppError(409, "USERNAME_TAKEN", ...); the commit is also guarded with IntegrityError → 409 so the unique constraint holds under concurrent registrations.


Time handling

5. Datetimes with a non-UTC offset were stored wrong

File: app/timeutils.py, parse_input_datetime()
Bug: dt = dt.replace(tzinfo=None) simply dropped the offset instead of converting. 10:00+06:00 was stored as 10:00 UTC instead of 04:00 UTC — corrupting conflict checks, quota windows, refund-notice tiers, availability dates and report ranges for any offset input (rule 1).
Fix: dt = dt.astimezone(timezone.utc).replace(tzinfo=None).


Booking creation (app/routers/bookings.py)

6. 5-minute grace window for past start times

Bug: if start <= now - timedelta(seconds=300) accepted bookings starting up to 5 minutes in the past. Rule 2: start must be strictly in the future, no grace window.
Fix: if start <= now: reject.

7. Minimum duration never enforced

Bug: only duration_hours > MAX_DURATION_HOURS was checked; MIN_DURATION_HOURS was unused. A 0-hour booking (end == start) or negative-duration whole-hour booking passed validation, violating "min 1 hour" and "end strictly after start".
Fix: if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS: 400 INVALID_BOOKING_WINDOW.

8. Back-to-back bookings rejected as conflicts

Function: _has_conflict()
Bug: overlap test used inclusive comparisons b.start_time <= end and start <= b.end_time. A booking ending 10:00 conflicted with one starting 10:00. Rule 3 defines overlap with strict inequalities and explicitly allows back-to-back.
Fix: b.start_time < end and start < b.end_time.

9. Double-booking and quota not enforced under concurrency

Function: create_booking() (with _pricing_warmup/_quota_audit sleeps widening the race)
Bug: conflict check → quota check → insert was a non-atomic check-then-act sequence. Two simultaneous requests for the same slot both saw "no conflict" (the deliberate time.sleep between read and write makes this near-certain) and both got confirmed; same for the 3-booking quota. Violates rules 3 and 4 "must hold under concurrent requests".
Fix: module-level threading.Lock (_booking_lock) held across conflict check, quota check, and insert+commit, serializing booking writes.

10. Pagination broken three ways

Function: list_bookings()
Bug: (a) sorted start_time.desc() instead of ascending (rule 11); (b) offset was page * limit, skipping the first page entirely (page 1 started at item 11); (c) .limit(10) hardcoded, ignoring the limit parameter.
Fix: order_by(start_time.asc(), id.asc()).offset((page - 1) * limit).limit(limit).

11. Booking detail returned created_at as start_time

Function: get_booking()
Bug: after serializing, the code overwrote response["start_time"] = iso_utc(booking.created_at), so the detail endpoint reported the wrong start time.
Fix: removed the overwrite.

12. Members could read other members' bookings

Function: get_booking()
Bug: the query filtered by org only. Any member could fetch any booking in their org by id; rule 10 says another member's booking id must behave as 404 BOOKING_NOT_FOUND. (cancel_booking had the ownership check; get_booking was missing it.)
Fix: added if user.role != "admin" and booking.user_id != user.id: 404 BOOKING_NOT_FOUND.


Cancellation & refunds

13. Refund tiers wrong at both boundaries

File: app/routers/bookings.py, cancel_booking()
Bug: (a) notice_hours = int(notice.total_seconds() // 3600) then if notice_hours > 48 — exactly 48h notice (and anything up to 48h59m due to truncation) failed the 100% tier, violating "notice ≥ 48 hours → 100%"; (b) the final else branch returned 50 instead of 0, so <24h cancellations were refunded 50% instead of 0%.
Fix: compare the timedelta directly: >= timedelta(hours=48) → 100, >= timedelta(hours=24) → 50, else 0.

14. Refund amount: wrong rounding, and response ≠ ledger

Files: app/services/refunds.py log_refund(), and cancel_booking()
Bug: the ledger computed the amount via floats and int() (truncation toward zero), while the response computed it separately with Python's round() (banker's rounding, half-to-even). Rule 6 requires half-cents to round up and the response amount to equal the RefundLog amount — both violated (e.g. price 101¢ at 50%: ledger 50¢, response 50¢, correct 51¢; price 103¢ at 50%: ledger 51¢, response 52¢ — mismatch).
Fix: single integer computation in log_refund(): amount_cents = (price_cents * percent + 50) // 100 (exact half-up), and the cancel response now returns the ledger entry's amount_cents.

15. Concurrent cancels produced duplicate refunds

File: app/routers/bookings.py, cancel_booking()
Bug: status check → refund log → status update was non-atomic, with _settlement_pause() (0.12 s sleep) between the refund write and the status flip. Two simultaneous cancels both passed the status == "cancelled" check and both wrote a RefundLog — violating "exactly one RefundLog entry" / one 409 for the loser.
Fix: the whole fetch → check → refund → status-update sequence now runs under _booking_lock; the second request sees cancelled and gets 409 ALREADY_CANCELLED.

16. Stale caches: reports and availability not invalidated

File: app/routers/bookings.py
Bug: creating a booking invalidated only the availability cache (usage reports kept serving stale cached data, violating rule 12 "must reflect the current state immediately"); cancelling invalidated only the report cache (availability kept showing the cancelled booking as busy, violating rule 13).
Fix: create now also calls cache.invalidate_report(user.org_id); cancel now also calls cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat()).


Services

17. Rate limiter lost requests under concurrency

File: app/services/ratelimit.py, record_and_check()
Bug: unsynchronized read → trim → (0.1 s sleep) → append → store on a shared dict. Concurrent requests each read the same bucket, slept, then each wrote back a bucket containing only their own timestamp — losing counts, so a burst of 30+ requests never hit the 20/60s limit (rule 5 "must hold under concurrent requests").
Fix: the trim/append/check sequence is guarded by a threading.Lock; the bookkeeping pause runs before the lock so requests aren't serialized on the sleep. Rejected requests still record their timestamp ("all requests count").

18. Duplicate reference codes under concurrency

File: app/services/reference.py, next_reference_code()
Bug: current = _counter["value"] → 0.12 s sleep → _counter["value"] = current + 1. Concurrent creators read the same counter value and got identical codes, violating rule 7 uniqueness.
Fix: the read+increment is atomic under a threading.Lock; the formatting pause happens after the lock is released.

19. Room stats lost updates under concurrency

File: app/services/stats.py, record_create() / record_cancel()
Bug: same read-modify-write race as above (read counts → 0.1 s sleep → write back), so bursts of concurrent bookings produced counts/revenue lower than the real number of confirmed bookings, violating rule 14.
Fix: the read-modify-write on _stats is guarded by a threading.Lock (pause moved before the critical section).

20. Deadlock between create and cancel notifications

File: app/services/notifications.py
Bug: notify_created acquired _email_lock then _audit_lock; notify_cancelled acquired _audit_lock then _email_lock. A concurrent create + cancel could each grab their first lock and wait forever on the other — a classic lock-ordering deadlock that hangs the service (violates rule 16 liveness; sleeps inside the critical sections make it easy to trigger).
Fix: both functions now acquire locks in the same order (email → audit).

21. CSV export leaked other organizations' bookings

File: app/services/export.py, generate_export()
Bug: with include_all=true&room_id=<id>, rows were loaded via fetch_bookings_raw(), which filters by room id only — no org check. An admin could pass another org's room id and export their bookings, violating rule 9 multi-tenancy.
Fix: that branch now uses the org-scoped query (_fetch_scoped(db, org_id, None, room_id)), so cross-org room ids yield no rows.
