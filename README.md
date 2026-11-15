# Learning Practice 4: Black-Box Testing, Part 2

**Course:** AD361 — Application Testing  
**Application:** North Vacation Booking  
**Testing methods:** Decision table testing and state transition testing

## 1. Conditions and actions

I used the reservation requirements to decide which inputs affect the result. I kept the pricing rule separate from the validation rules because a reservation can be valid but still cost more when there are extra guests.

### Five conditions

| ID | Condition | True (T) means |
|---|---|---|
| C1 | Check-in and checkout dates are valid | Check-in is before checkout. |
| C2 | Length of stay is valid | The stay is between 1 and 31 nights, inclusive. Checkout day is not counted. |
| C3 | Guest count is valid | There are between 1 and 12 guests, inclusive. |
| C4 | All requested nights are available | Every night between check-in (inclusive) and checkout (exclusive) is available. |
| C5 | More than five guests | There are 6–12 guests, so an extra guest charge applies to a valid booking. |

### Five actions

| ID | Action | What I expect |
|---|---|---|
| A1 | Create reservation | A valid reservation is created in the `Pending` state. |
| A2 | Calculate standard price | Charge $200 per night plus a one-time $50 cleaning fee. |
| A3 | Add extra guest fee | Add $25 per night for each guest above five. |
| A4 | Raise `ValueError` | Reject invalid dates, an invalid stay length, or an invalid guest count. |
| A5 | Raise `UnavailableError` | Reject an otherwise valid request if at least one requested night is unavailable. |

For a valid reservation, the total is:

`total = ($200 × nights) + $50 + ($25 × max(guests - 5, 0) × nights)`

## 2. Decision table

Each column is one rule. I used `-` when that condition is not needed to decide the outcome. For example, if the date order is wrong, the reservation should already fail validation, so the guest count does not change that rule.

| Conditions / Actions | R1 | R2 | R3 | R4 | R5 | R6 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Conditions** | | | | | | |
| C1: Dates valid? | T | T | F | T | T | T |
| C2: 1–31 nights? | T | T | - | F | T | T |
| C3: 1–12 guests? | T | T | - | - | F | T |
| C4: All nights available? | T | T | - | - | - | F |
| C5: More than five guests? | F | T | - | - | - | - |
| **Actions** | | | | | | |
| A1: Create reservation | X | X | | | | |
| A2: Calculate standard price | X | X | | | | |
| A3: Add extra guest fee | | X | | | | |
| A4: Raise `ValueError` | | | X | X | X | |
| A5: Raise `UnavailableError` | | | | | | X |

**Key:** T = true; F = false; - = not relevant to that rule; X = expected action.

- **R1:** Valid dates, nights, and guests; all nights available; five or fewer guests.
- **R2:** Valid reservation with six or more guests, so the extra nightly fee applies.
- **R3:** Check-in is not before checkout, so the dates are invalid.
- **R4:** The stay is outside the allowed 1–31-night range (for this test, 32 nights).
- **R5:** Guest count is outside the allowed 1–12 range (for this test, 13 guests).
- **R6:** The request is otherwise valid, but one of the requested nights is unavailable.

## 3. Decision table test cases

I picked specific dates to make the night counts easy to check. For a stay from December 1 to December 4, the booked nights are December 1, 2, and 3. December 4 is the checkout day, not another night.

| Test ID | Rule | Conditions (C1–C5) | Check-in → Checkout | Nights | Guests | Availability | Expected action | Expected price / exception |
|---|---|---|---|---:|---:|---|---|---|
| DT01 | R1 | T, T, T, T, F | 2026-12-01 → 2026-12-04 | 3 | 3 | All 3 nights available | A1 + A2; create `Pending` reservation | **$650** = (3 × $200) + $50 |
| DT02 | R2 | T, T, T, T, T | 2026-12-10 → 2026-12-14 | 4 | 8 | All 4 nights available | A1 + A2 + A3; create `Pending` reservation | **$1,150** = (4 × $200) + $50 + (3 extra guests × $25 × 4) |
| DT03 | R3 | F, -, -, -, - | 2026-12-06 → 2026-12-06 | 0 | 2 | Available | A4; reject reservation | `ValueError`; no price |
| DT04 | R4 | T, F, -, -, - | 2026-12-01 → 2027-01-02 | 32 | 2 | All 32 nights available | A4; reject reservation | `ValueError`; no price |
| DT05 | R5 | T, T, F, -, - | 2026-12-10 → 2026-12-13 | 3 | 13 | All 3 nights available | A4; reject reservation | `ValueError`; no price |
| DT06 | R6 | T, T, T, F, - | 2026-12-10 → 2026-12-13 | 3 | 2 | Dec. 11 is unavailable | A5; reject reservation | `UnavailableError`; no price |

**What I checked:** DT01 and DT02 cover the two pricing results. DT03–DT05 each cover a different invalid input, and DT06 checks the availability exception. The requirements do not specify which error comes first when several inputs are wrong, so I kept each invalid test focused on one problem.

## 4. Reservation states and events

### Five states

| State | What it means |
|---|---|
| `Pending` | The booking is valid and has been created, but payment has not succeeded yet. |
| `Confirmed` | Payment was successful, and the reservation is confirmed. |
| `Checked In` | The guest has checked in on the scheduled check-in date. |
| `Completed` | The guest has checked out and the stay is finished. |
| `Cancelled` | The reservation has been cancelled. |

### Five events

| Event | What triggers it |
|---|---|
| E1: Payment succeeds | A pending booking's payment is accepted. |
| E2: Payment fails | Payment for a pending booking is rejected. |
| E3: Guest cancels | Cancellation is requested while the reservation is pending or confirmed. |
| E4: Guest checks in | The guest requests check-in; this is allowed for a confirmed booking on its check-in date. |
| E5: Guest checks out | A checked-in guest checks out at the end of the stay. |

## 5. State transition table

These are **six valid** and **three invalid** transitions from the requirements. I included the date restriction for check-in instead of assuming a confirmed guest can check in at any time.

| ID | Starting state | Event / condition | Expected state | Expected result | Type |
|---|---|---|---|---|---|
| T01 | `Pending` | E1: Payment succeeds | `Confirmed` | Payment succeeds; reservation becomes confirmed. | Valid |
| T02 | `Pending` | E2: Payment fails | `Cancelled` | Payment fails; reservation is cancelled. | Valid |
| T03 | `Pending` | E3: Guest cancels | `Cancelled` | Pending reservation is cancelled. | Valid |
| T04 | `Confirmed` | E3: Guest cancels | `Cancelled` | Confirmed reservation is cancelled. | Valid |
| T05 | `Confirmed` | E4: Guest checks in **on check-in date** | `Checked In` | Check-in is accepted. | Valid |
| T06 | `Checked In` | E5: Guest checks out after stay | `Completed` | Checkout is recorded and stay is completed. | Valid |
| T07 | `Cancelled` | E4: Guest attempts to check in | `Cancelled` | `InvalidStateError`; state stays the same. | Invalid |
| T08 | `Completed` | E3: Guest attempts to cancel | `Completed` | `InvalidStateError`; state stays the same. | Invalid |
| T09 | `Confirmed` | E4: Guest attempts to check in **before check-in date** | `Confirmed` | `InvalidStateError`; state stays the same. | Invalid |

## 6. State transition test cases

**Shared test setup:** Each test starts with a valid, three-night booking for two guests, check-in **2026-12-10**, checkout **2026-12-13**, with every requested night available. The booking starts in `Pending`; the setup actions needed to reach other starting states are listed below. I am testing the specified state behavior, not any hidden implementation.

### Individual transitions (six valid, three invalid)

| Test ID | Transition | Starting setup / state | Event and input | Expected state | Expected result |
|---|---|---|---|---|---|
| ST01 | T01 | New reservation: `Pending` | Payment succeeds | `Confirmed` | Reservation is confirmed. |
| ST02 | T02 | New reservation: `Pending` | Payment fails | `Cancelled` | Reservation is cancelled. |
| ST03 | T03 | New reservation: `Pending` | Guest cancels | `Cancelled` | Reservation is cancelled. |
| ST04 | T04 | Reservation paid successfully: `Confirmed` | Guest cancels | `Cancelled` | Reservation is cancelled. |
| ST05 | T05 | Reservation paid successfully: `Confirmed` | Guest checks in on **2026-12-10** | `Checked In` | Check-in is accepted. |
| ST06 | T06 | Reservation paid and guest checked in: `Checked In` | Guest checks out on **2026-12-13** | `Completed` | Stay is completed. |
| ST07 | T07 | Reservation was cancelled: `Cancelled` | Guest attempts to check in | `Cancelled` | `InvalidStateError`; no state change. |
| ST08 | T08 | Reservation has been completed: `Completed` | Guest attempts to cancel | `Completed` | `InvalidStateError`; no state change. |
| ST09 | T09 | Reservation paid successfully: `Confirmed` | Guest attempts to check in on **2026-12-09** (a day early) | `Confirmed` | `InvalidStateError`; no state change. |

### Complete sequences (two more test cases)

**ST10 — Normal reservation sequence**

- **Starting conditions:** A new valid reservation for December 10–13, two guests, three available nights; initial state `Pending`.
- **Event 1:** Payment succeeds → expected `Confirmed`; payment accepted.
- **Event 2:** Guest checks in on December 10 → expected `Checked In`; check-in accepted.
- **Event 3:** Guest checks out on December 13 → expected `Completed`; stay completed.
- **Final expected result:** `Completed`, without errors.

**ST11 — Cancellation sequence**

- **Starting conditions:** A new valid reservation for December 10–13, two guests, three available nights; initial state `Pending`.
- **Event 1:** Payment succeeds → expected `Confirmed`; payment accepted.
- **Event 2:** Guest cancels → expected `Cancelled`; cancellation accepted.
- **Final expected result:** `Cancelled`, without errors.

## 7. Notes and AI use

This is a **test design** based on the supplied North Vacation Booking requirements. No application code was provided, so these are expected results, not test execution results. I did not claim that the tests passed in a running application.

I used ChatGPT to help organize the condition/action tables and improve grammar and wording.


