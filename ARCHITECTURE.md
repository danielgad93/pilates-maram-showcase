# Architecture notes

This document summarizes the architecture verified in the application. It does
not include source files, deployment credentials, or internal database records.

## Request and persistence flow

1. Customers and administrators use the same Hebrew/RTL web application.
2. Next.js App Router pages render the public, customer, and admin interfaces.
3. Route handlers process same-origin authentication, scheduling, booking,
   membership, profile, notification, and administration requests.
4. Signed-session identity is checked before protected operations; admin APIs
   also check the session role.
5. Booking and membership domain logic reads and writes through Prisma.
6. Prisma connects to PostgreSQL. The schema includes users, plans, memberships,
   classes, bookings, studio settings, testimonials, notifications, and blocked dates.
7. Selected OTP and studio-message flows call the external SMS4Free API.

## Design decisions represented in the code

### Class credits and booking

Booking eligibility checks for an active membership with available credits.
The booking is connected to the membership charged, and the balance update and
booking write occur inside a database transaction. The schema prevents a
duplicate booking for the same customer and class.

The current implementation checks capacity before beginning that transaction.
That means simultaneous high-contention booking behavior should be treated as
an unverified reliability boundary rather than a proven guarantee.

### Cancellation policy

The cancellation service reads the configured cutoff and refund setting.
Customer cancellation inside the cutoff is rejected; an administrator can
cancel on the customer's behalf. Booking status and an optional credit return
are persisted transactionally.

### Israel local time

Class calendar date and start time are separate fields. The shared class-time
helper converts the studio's `Asia/Jerusalem` local date/time into an instant
using timezone rules, including the summer/winter offset difference. The
cancellation-cutoff path uses this helper.

The separate past-class check in booking still constructs a fixed UTC+3 offset,
so winter behavior in that booking check needs further verification; it is not
covered by the three helper tests reported in the README.

## External dependencies

- **PostgreSQL:** application data.
- **SMS4Free:** verification codes and selected studio messages.
- **Bit:** the interface offers an off-app payment handoff. The app code does not
  implement a card processor or payment-settlement webhook.

Provider credentials are supplied through configuration and are not reproduced
in this case study.


## Verification boundaries and follow-up work

- Class capacity is checked before the booking transaction; behavior under simultaneous high-contention booking has not been load-tested.
- A separate past-class booking check currently uses a fixed UTC+3 offset; winter behavior for that check is not covered by the three timezone-helper tests.
- The reported checks do not establish complete coverage of authentication, messaging, payment handoff, integration, production build, or security.
