# User Stories & User Journeys

This document maps the user stories and journeys for all roles in the Trimly telehealth platform.

---

## Roles Overview

| Role | Access Via | Primary Responsibility |
|---|---|---|
| Patient | Booking funnel (web) + Patient app (mobile) | Book consultations, attend video calls, view medications |
| Doctor / Practitioner | Medplum admin app (web) | Conduct consultations, prescribe medications, record notes |
| Clinic Admin / Staff | Medplum admin app (web) | Process payments, manage schedules, customer service, book follow-ups |
| Pharmacy Staff | Medplum admin app (web) | Fulfill prescriptions, track deliveries |

### Auth Model

- **Email OTP** as primary auth (no passwords) — patient enters email, receives one-time code, verifies
- **Google OAuth** as secondary option
- Consistent across both web booking funnel and mobile app
- New patients must register through the web booking funnel — the mobile app is login-only

---

## Patient

### P1. New Patient Booking (Booking Funnel)

**As a** new patient, **I want to** book a telehealth consultation online **so that** I can receive medical care from home.

**Journey:**

```
Landing Page
  |
  v
Quiz / Intake Questionnaire              <- unauthenticated, no login needed
  |  Answer medical screening questions
  |  Answers held in client-side state (React state / sessionStorage)
  |  NOT persisted to server until after registration
  |
  v
Quiz Validation
  |  If answers indicate ineligibility:
  |    -> Soft block: "Based on your answers, we recommend seeing a doctor in person"
  |    -> WhatsApp button to contact customer service (preset message)
  |    -> Cannot proceed to booking
  |  If eligible:
  |    -> Continue
  |
  v
Schedule Selection                        <- unauthenticated, browsing available slots
  |  View available dates & time slots
  |  Select a preferred slot
  |  (reads Schedule + Slot resources via client credentials)
  |
  v
Authentication                            <- email OTP or Google OAuth
  |  Enter email -> receive OTP -> verify
  |  Or sign in with Google
  |  Medplum creates account + FHIR Patient resource
  |  QuestionnaireResponse now submitted and linked to Patient
  |  Patient receives session token
  |
  v
Payment Setup                             <- authenticated
  |  Enter credit/debit card via Stripe Elements
  |  Card saved (Setup Intent, no charge yet)
  |  Stripe Customer ID + PaymentMethod stored on Patient resource
  |
  v
Confirmation                              <- authenticated
  |  Re-fetch slot availability — if taken, redirect back to slot selection
  |  If available: Appointment resource created (status=booked)
  |  Video room auto-created (Daily.co)
  |  Receives confirmation email + WhatsApp (both include video join link)
  |  Discord alert fires for clinic admin visibility
  |
  v
Done — waiting for appointment day
```

**Acceptance criteria:**
- Patient can complete quiz and browse slots without logging in
- Quiz answers held client-side, submitted only after registration
- Ineligible patients see soft block with WhatsApp CS button
- Slot availability re-checked at confirmation step — if taken, patient picks a new slot
- Card is tokenized but not charged during booking
- Confirmation sent via both email and WhatsApp, both include Daily.co video join link
- Discord alert fires for admin

---

### P2. Returning Patient Booking

**As a** returning patient, **I want to** book another consultation using my existing account **so that** I don't have to re-enter my details.

**Journey:**

```
Landing Page
  |
  v
Quiz / Intake Questionnaire               <- unauthenticated, answers held client-side
  |
  v
Quiz Validation (same soft block logic)
  |
  v
Schedule Selection                         <- unauthenticated
  |
  v
Authentication (email OTP or Google OAuth) <- existing account recognized
  |
  v
Payment Setup                              <- may already have a saved card
  |  Option to use existing card or add new one
  |
  v
Confirmation (with slot re-check)
```

**Acceptance criteria:**
- Returning patient authenticates via OTP or Google OAuth (no password)
- Same credentials work across booking funnel and patient mobile app
- Can reuse saved payment method or add a new one
- New QuestionnaireResponse created for each booking

---

### P3. Abandoned Cart Recovery

**As a** patient who started but didn't complete booking, **I want to** receive a reminder **so that** I can come back and finish booking.

**Journey:**

```
Patient registers (account created)
  |
  v
Task resource created (type=abandoned-cart, 2-hour deadline)
  |
  v
Patient leaves without booking
  |
  v
Cron bot checks every 30 min
  |  Patient has no booked Appointment?
  |  Past 2-hour deadline?
  |  Within allowed hours (8am-10pm SGT)?
  |
  v
WhatsApp reminder sent via Twilio
  |
  v
Patient returns and completes booking (or doesn't)
```

**Acceptance criteria:**
- Reminder only sent after 2-hour window has passed
- Quiet hours respected (10pm-8am SGT) — deferred to next run after 8am
- No reminder sent if patient already has a booked appointment
- Note: only triggers after registration — patients who leave before creating an account cannot be reminded

---

### P4. Appointment Reminders

**As a** patient with an upcoming appointment, **I want to** receive a reminder **so that** I don't forget my consultation.

**Journey:**

```
Daily 7am SGT — cron bot runs
  |
  v
Finds all Appointments starting within next 24 hours
  |
  v
Sends WhatsApp reminder + email to each patient
  |  Both include Daily.co video join link
```

**Acceptance criteria:**
- Reminder sent exactly once per appointment
- Both WhatsApp and email channels used
- Sent at 7am SGT regardless of appointment time (as long as it's within 24h)
- Includes video join link for easy access

---

### P5. Attending a Video Consultation

**As a** patient, **I want to** join a video call with my doctor **so that** I can have my telehealth consultation.

**Journey:**

```
Option A: Join from web
  |  Click Daily.co link from confirmation email or WhatsApp
  |  Opens in browser — no app required
  |
Option B: Join from mobile app
  |  Open patient app -> Appointments tab
  |  Tap "Join Call" on upcoming appointment
  |  Opens Daily.co video room (React Native SDK)
  |
  v
Video consultation with doctor
  |  (20-min typical session)
  |
  v
Call ends
  |  Room expires (appointment end time + 30-min buffer)
```

**Acceptance criteria:**
- Video call works from any browser (web) or from the mobile app
- No app download required to join a consultation
- Daily.co link included in confirmation email, WhatsApp, and appointment reminders
- Room auto-expires after appointment window

---

### P6. Viewing Medication Status

**As a** patient, **I want to** see my prescription and delivery status **so that** I know when my medication will arrive.

**Journey:**

```
Patient opens patient app -> Medications tab
  |
  v
Views list of MedicationRequests (prescriptions)
  |  Each shows: medication name, dosage, status (active/completed)
  |
  v
Taps a prescription to see details
  |  Sees MedicationDispense info:
  |    - Fulfillment status: preparation -> in-progress -> completed
  |    - Delivery status: pending -> shipped -> delivered
  |
  v
(Read-only — patient cannot modify)
```

**Acceptance criteria:**
- Patient can only see their own prescriptions (scoped by Patient Access Policy)
- Both MedicationRequest and MedicationDispense statuses visible
- Delivery tracking shown when available

---

### P7. Managing Profile

**As a** patient, **I want to** view and update my personal information **so that** my records are accurate.

**Journey:**

```
Patient opens patient app -> Profile tab
  |
  v
Views personal info (name, email, phone, DOB, sex, address)
  |
  v
Edits fields as needed
  |  Updates written to FHIR Patient resource
  |
  v
Views appointment history
  |
  v
Views payment methods (Stripe)
```

**Acceptance criteria:**
- Patient can read/write own Patient resource
- Cannot see other patients' data
- Payment method display (read-only in app, card management via Stripe)

---

### P8. Patient Login (Mobile App)

**As a** returning patient, **I want to** log in to the mobile app **so that** I can access my appointments and health information.

**Journey:**

```
Opens Expo patient app
  |
  v
Login screen
  |  Option 1: Email OTP (enter email -> receive code -> verify)
  |  Option 2: Google OAuth
  |  (Medplum OAuth2 PKCE flow)
  |
  |  No registration option — new patients directed to web booking funnel
  |  e.g., "Don't have an account? Book your first consultation at [website]"
  |
  v
Authenticated -> Home tab
  |  Sees upcoming appointments, recent medications, profile
```

**Acceptance criteria:**
- OAuth2 PKCE flow used (mobile-safe, no client secret exposed)
- Email OTP as primary, Google OAuth as secondary
- Same credentials as booking funnel
- No registration in-app — must go through web funnel first
- Session persisted locally

---

### P9. Viewing Appointment History

**As a** patient, **I want to** see my past consultations **so that** I can reference previous visits.

**Journey:**

```
Patient opens patient app -> Appointments tab
  |
  v
Views list of appointments (upcoming and past)
  |  Each shows: date, time, doctor name, status
  |
  v
Taps past appointment to see details
  |  Sees encounter notes (if documented)
  |  Sees linked prescriptions
```

**Acceptance criteria:**
- Appointments sorted by date (upcoming first, then past)
- Read-only access to Encounter and DocumentReference resources
- Patient cannot modify past appointments

---

### P10. Receive Cancellation Notification

**As a** patient whose appointment was cancelled, **I want to** be notified **so that** I know and can rebook if needed.

**Journey:**

```
Admin or doctor cancels appointment in admin app
  |
  v
on-appointment-cancelled trigger fires
  |
  v
Patient receives email + WhatsApp notification
  |  "Your appointment on [date] at [time] has been cancelled"
```

**Acceptance criteria:**
- Patient notified via both email and WhatsApp
- Cancelled slot freed up for other bookings

---

### P11. No-Show Notification + Fee

**As a** patient who missed an appointment, **I want to** be informed **so that** I'm aware of the no-show and any associated fee.

**Journey:**

```
Patient doesn't join video call
  |
  v
Doctor waits ~10 min
  |
  v
Doctor or admin marks appointment as no-show
  |
  v
Patient receives invoice via email ($20 no-show fee)
  |
  v
Admin charges $20 no-show fee (process-payment bot)
  |
  v
Patient receives receipt via email
  |
  v
Patient receives WhatsApp message
  |  "You missed your appointment on [date]. A $20 no-show fee has been charged."
```

**Acceptance criteria:**
- Fixed $20 no-show fee
- Invoice sent via email before charge
- Receipt sent via email after charge
- Charged to saved card via Stripe
- WhatsApp notification sent to patient about the no-show

---

### P12. Update Failed Payment Card

**As a** patient whose card was declined, **I want to** update my payment method **so that** my consultation/medication can be processed.

**Journey:**

```
Payment fails (expired/declined card)
  |
  v
Admin triggers card update link
  |
  v
Patient receives WhatsApp + email with link to card update page
  |
  v
Patient opens web page, enters new card via Stripe Elements
  |  New PaymentMethod saved to Patient resource
  |
  v
Admin retries payment
```

**Acceptance criteria:**
- Dedicated web page for card updates (not in-app)
- Link sent via both WhatsApp and email
- New card saved to Stripe Customer
- Admin can retry payment after card is updated

---

### P13. Medication Status WhatsApp Notifications

**As a** patient, **I want to** receive WhatsApp updates when my medication ships and delivers **so that** I don't have to keep checking the app.

**Journey:**

```
MedicationDispense deliveryStatus changes
  |
  v
Shipped -> WhatsApp: "Your medication has been shipped"
Delivered -> WhatsApp: "Your medication has been delivered"
```

**Acceptance criteria:**
- WhatsApp notifications on delivery status changes (shipped, delivered)
- Sent automatically on status update

---

### P14. Weight / Health Tracking (Future)

**As a** patient, **I want to** log and track my weight over time **so that** I can monitor my progress.

**Journey:**

```
Patient opens patient app -> Health/Progress tab
  |
  v
Logs current weight
  |  Stored as FHIR Observation resource
  |
  v
Views weight trend chart (BMI over time)
```

**Acceptance criteria:**
- Observation resources used for health metrics
- Historical data displayed as a trend
- Future enhancement, not in MVP

---

## Doctor / Practitioner

### D1. Viewing Scheduled Appointments

**As a** doctor, **I want to** see my upcoming appointments in a calendar view **so that** I can prepare for consultations.

**Journey:**

```
Doctor logs into Medplum admin app
  |
  v
Views appointment calendar
  |  Filtered to own patients (by Practitioner reference)
  |  Shows: patient name, time, status, visit type
  |
  v
Clicks an appointment to see details
  |  Patient demographics, intake quiz answers, past encounters, active medications
```

**Acceptance criteria:**
- Calendar view of appointments in provider portal
- No push notifications — doctor checks the calendar
- Can view patient details from the appointment

---

### D2. Conducting a Consultation

**As a** doctor, **I want to** conduct a video consultation and document it **so that** I can provide and record care.

**Journey:**

```
Doctor opens appointment in admin app
  |
  v
Pre-call: Review patient info
  |  - Demographics (name, age, sex)
  |  - Intake quiz answers (QuestionnaireResponse)
  |  - Past encounters + clinical notes
  |  - Active medications (MedicationRequests)
  |
  v
Start call: Click Daily.co video link
  |  Opens in a new browser tab
  |
  v
Clinical notes: Open in admin app tab (side-by-side with video tab)
  |  Template pre-populated based on visit type
  |  (e.g., sections: "Chief Complaint: [fill in]", "History: [fill in]",
  |   "Assessment: [fill in]", "Plan: [fill in]")
  |  Doctor fills in / replaces placeholder text
  |  Each save is timestamped
  |
  v
During/after call: Continue editing notes
  |  Notes remain editable — no hard deadline to finalize
  |
  v
Optionally: Create prescription (MedicationRequest)
  |
  v
Communicate follow-up recommendation to admin
  |  (e.g., "patient should return in 2 weeks")
```

**Acceptance criteria:**
- Patient info visible before and during consultation
- Video call opens in separate tab (Daily.co link)
- Clinical notes use templates pre-populated per visit type
- Notes are free-form text with template structure
- Notes editable during and after the call, each save timestamped
- No requirement to "close" or finalize the encounter — flexible workflow
- Doctor joins via browser, no special app needed

---

### D3. Creating a Prescription

**As a** doctor, **I want to** prescribe medication after a consultation **so that** the patient can receive treatment.

**Journey:**

```
After consultation
  |
  v
Doctor creates MedicationRequest in admin app
  |  Specifies: medication, dosage, frequency, quantity
  |  Links to Patient and Encounter
  |  Status: active
  |
  v
Prescription visible to:
  |  - Clinic admin (for payment processing)
  |  - Pharmacy staff (for fulfillment, after payment)
  |  - Patient (read-only in mobile app)
```

**Acceptance criteria:**
- MedicationRequest created with status=active
- References correct Patient, Practitioner, and Encounter
- Visible to all relevant roles with appropriate access
- Pharmacy does NOT begin fulfillment until admin processes payment

---

### D4. Reviewing Patient History

**As a** doctor, **I want to** review a patient's past encounters, prescriptions, and intake responses **so that** I have full context before a consultation.

**Journey:**

```
Doctor opens patient record in admin app
  |
  v
Views:
  |  - Patient demographics
  |  - Past Encounters + clinical notes (timestamped edits)
  |  - MedicationRequests (active and completed)
  |  - QuestionnaireResponses (intake answers)
  |  - Appointment history
```

**Acceptance criteria:**
- Full read access to all clinical resources for any patient
- History sorted chronologically
- Can view resources created by other practitioners

---

### D5. Managing Own Availability

**As a** doctor, **I want to** adjust my schedule and block out times **so that** my availability is accurate.

**Journey:**

```
Doctor opens schedule management in admin app
  |
  v
Views own recurring schedule (within clinic base hours)
  |
  v
Option A: Adjust recurring hours
  |  e.g., change Wednesday from 9am-6pm to 9am-1pm
  |  Must stay within clinic base hours
  |
Option B: Add block-out
  |  Mark specific dates/times as unavailable
  |  e.g., "Thursday 2pm-6pm — personal leave"
  |
  v
generate-slots bot respects changes on next run
  |  Blocked times will not have slots generated
```

**Acceptance criteria:**
- Doctor can adjust own recurring schedule within clinic base hours
- Doctor can add/remove block-outs for specific dates/times
- Changes reflected in next slot generation cycle
- Existing booked appointments in blocked times are NOT auto-cancelled (admin handles conflicts)

---

### D6. Handling No-Shows

**As a** doctor, **I want to** mark a patient as a no-show **so that** the admin can process the no-show fee.

**Journey:**

```
Appointment time arrives, patient doesn't join
  |
  v
Doctor waits ~10 min
  |
  v
Doctor marks appointment as no-show in admin app
  |
  v
Admin takes over: charges $20 no-show fee, patient notified via WhatsApp
```

**Acceptance criteria:**
- Doctor can mark appointment status as no-show
- Triggers admin workflow for fee processing and patient notification

---

## Clinic Admin / Staff

### A1. Processing Consultation Fee

**As a** clinic admin, **I want to** charge the consultation fee after an appointment **so that** the clinic receives payment for the visit.

**Journey:**

```
Doctor completes consultation
  |
  v
Admin views completed appointment in admin app
  |
  v
Step 1: Send invoice
  |  Admin clicks "Send Invoice"
  |  Patient receives invoice via email (informational, no action required)
  |
  v
Step 2: Charge card
  |  Admin clicks "Charge"
  |  Calls process-payment bot via $execute
  |  Bot charges patient's saved Stripe PaymentMethod
  |
  v
Payment result:
  |  Success -> payment recorded on Appointment
  |    -> Admin clicks "Send Receipt"
  |    -> Patient receives receipt via email
  |  Failure -> payment marked as failed
  |    -> Send card update link to patient (WhatsApp + email)
  |    -> Retry after patient updates card
```

**Acceptance criteria:**
- Consultation fee charged separately from medication fee
- Charged post-consultation (card was saved at booking, not charged)
- Admin manually sends invoice (email) before charging
- Admin manually sends receipt (email) after successful charge
- Payment status tracked
- On failure: card update link sent to patient

---

### A2. Processing Medication Fee

**As a** clinic admin, **I want to** charge for prescribed medication **so that** pharmacy can begin fulfillment.

**Journey:**

```
Doctor creates MedicationRequest (prescription)
  |
  v
Admin views prescription in admin app
  |  Sees paymentStatus: pending
  |
  v
Step 1: Send invoice
  |  Admin clicks "Send Invoice"
  |  Patient receives invoice via email (informational, no action required)
  |
  v
Step 2: Charge card
  |  Admin clicks "Charge"
  |  Calls process-payment bot via $execute
  |  Bot charges patient's saved Stripe PaymentMethod
  |
  v
Payment result:
  |  Success -> paymentStatus updated to "charged"
  |    -> Admin clicks "Send Receipt"
  |    -> Patient receives receipt via email
  |    -> Pharmacy can now begin fulfillment
  |  Failure -> paymentStatus updated to "failed"
  |    -> Send card update link to patient (WhatsApp + email)
  |    -> Pharmacy does NOT start fulfillment
  |    -> Retry after patient updates card
```

**Acceptance criteria:**
- Medication fee charged separately from consultation fee
- Admin manually sends invoice (email) before charging
- Admin manually sends receipt (email) after successful charge
- Payment must succeed before pharmacy begins fulfillment
- Multiple charges supported (multiple medications)
- Payment status tracked on MedicationDispense extension
- On failure: card update link sent, fulfillment blocked

---

### A3. Managing Prescription Fulfillment (Oversight)

**As a** clinic admin, **I want to** oversee prescription fulfillment status **so that** I can ensure patients receive their medications.

**Journey:**

```
Admin views prescriptions dashboard
  |
  v
Sees status of all MedicationRequests:
  |  - Pending payment
  |  - Payment charged, awaiting fulfillment
  |  - In fulfillment (pharmacy handling)
  |  - Shipped
  |  - Delivered
  |  - Payment failed (needs follow-up)
```

**Acceptance criteria:**
- Admin has visibility across all prescriptions
- Can filter by status
- Can identify stuck or failed items for follow-up

---

### A4. Booking Follow-Up Appointments

**As a** clinic admin, **I want to** schedule follow-up appointments for patients **so that** continuity of care is maintained.

**Journey:**

```
Doctor recommends follow-up (e.g., "return in 2 weeks")
  |
  v
Admin opens patient record in admin app
  |
  v
Navigates to scheduling
  |  Views available slots
  |  Selects a slot for the patient
  |
  v
Creates Appointment (status=booked)
  |  Same bot pipeline triggers:
  |    - Video room created
  |    - Confirmation email + WhatsApp sent to patient (with video link)
  |    - Discord alert
```

**Acceptance criteria:**
- Follow-up booking is admin's responsibility (not doctor)
- Admin can create appointments for any patient
- All automated notifications fire correctly
- Appointment visible to both patient and assigned doctor

---

### A5. Monitoring Appointment Activity

**As a** clinic admin, **I want to** see all appointments across the clinic **so that** I can manage daily operations.

**Journey:**

```
Admin opens admin app
  |
  v
Views all Appointments (not filtered by practitioner)
  |  Sees: patient name, doctor, time, status
  |
  v
Can filter by date, status, practitioner
  |
  v
Receives Discord alerts for new bookings (real-time awareness)
```

**Acceptance criteria:**
- Admin sees all appointments, not just one doctor's
- Discord webhook provides real-time alerts for new bookings
- Can view appointment details including video link

---

### A6. Cancelling an Appointment

**As a** clinic admin, **I want to** cancel an appointment **so that** the slot is freed and the patient is informed.

**Journey:**

```
Patient requests cancellation via WhatsApp (to customer service)
  |  OR doctor/admin decides to cancel
  |
  v
Admin cancels appointment in admin app
  |  Appointment status -> cancelled
  |  Slot freed for other bookings
  |
  v
on-appointment-cancelled trigger fires
  |  Patient notified via email + WhatsApp
```

**Acceptance criteria:**
- Only admin/doctor/staff can cancel (no patient self-service in MVP)
- Patients request cancellation through WhatsApp customer service
- Free cancellation (no cancellation fee)
- Cancelled slot becomes available for others
- Patient notified via both email and WhatsApp

---

### A7. Handling No-Shows

**As a** clinic admin, **I want to** process no-show fees **so that** missed appointments have consequences.

**Journey:**

```
Doctor marks appointment as no-show
  |
  v
Step 1: Send invoice
  |  Admin clicks "Send Invoice" for $20 no-show fee
  |  Patient receives invoice via email
  |
  v
Step 2: Charge card
  |  Admin clicks "Charge"
  |  Calls process-payment bot via $execute
  |  Charges patient's saved Stripe PaymentMethod
  |
  v
Payment result:
  |  Success -> Admin clicks "Send Receipt"
  |    -> Patient receives receipt via email
  |    -> Patient also receives WhatsApp notification about the no-show
  |  Failure -> Send card update link to patient (WhatsApp + email)
  |    -> Retry after patient updates card
```

**Acceptance criteria:**
- Fixed $20 no-show fee
- Admin manually sends invoice (email) before charging
- Admin manually sends receipt (email) after successful charge
- WhatsApp notification sent to patient about the no-show
- Same failed-payment recovery flow as other charges

---

### A8. Handling Failed Payments

**As a** clinic admin, **I want to** identify and recover failed payments **so that** revenue is not lost.

**Journey:**

```
Payment fails (consultation fee, medication fee, or no-show fee)
  |
  v
Admin sees failed payment in admin app
  |
  v
Sends card update link to patient (WhatsApp + email)
  |  Link goes to dedicated card update web page
  |
  v
Patient updates card
  |
  v
Admin retries payment via process-payment bot
```

**Acceptance criteria:**
- Failed payments clearly visible in admin app
- Card update link sent via both WhatsApp and email
- Retry mechanism available after card update
- Works for all payment types (consultation, medication, no-show)

---

### A9. Managing Doctor Schedules

**As a** clinic admin, **I want to** set and adjust doctor schedules **so that** appointment availability is correct.

**Journey:**

```
Admin opens schedule management in admin app
  |
  v
Views clinic base hours (e.g., 9am-6pm Mon-Sat)
  |  Base hours configured once, rarely changed
  |
  v
Sets doctor recurring schedule within base hours
  |  e.g., Dr. Lee: Mon/Wed/Fri 9am-6pm, Tue/Thu 9am-1pm
  |
  v
Manages block-out calendar
  |  Add/remove block-outs for specific dates/times
  |  e.g., public holiday, doctor leave
  |
  v
generate-slots bot runs weekly
  |  Creates slots respecting: base hours -> doctor schedule -> block-outs
```

**Acceptance criteria:**
- Three-layer schedule model: clinic base hours > doctor schedule > block-outs
- Admin can set/adjust doctor recurring schedules
- Admin can add/remove block-outs
- Slots auto-generated weekly respecting all three layers
- Existing booked appointments in newly blocked times are NOT auto-cancelled (admin handles conflicts manually)

---

### A10. Managing Clinical Note Templates

**As a** clinic admin, **I want to** create and manage clinical note templates **so that** doctors have consistent documentation for each visit type.

**Journey:**

```
Admin opens template management in admin app
  |
  v
Views existing templates (one per visit type)
  |  e.g., "Initial Consultation", "Follow-Up", "Weight Management Review"
  |
  v
Creates or edits a template
  |  Free-form text with placeholder sections
  |  e.g., "Chief Complaint: [fill in]"
  |       "History: [fill in]"
  |       "Assessment: [fill in]"
  |       "Plan: [fill in]"
  |
  v
Assigns template to a visit type
  |
  v
Doctors see the template pre-populated when documenting that visit type
```

**Acceptance criteria:**
- Admin can create, edit, and delete templates
- Each template assigned to a visit type
- Templates pre-populate the notes editor for doctors
- Doctors can modify the pre-populated text freely

---

### A11. Sending Invoices and Receipts

**As a** clinic admin, **I want to** send invoices before charging and receipts after charging **so that** patients have proper billing documentation.

**Journey:**

```
Any charge (consultation fee, medication fee, no-show fee)
  |
  v
Before charging:
  |  Admin clicks "Send Invoice"
  |  Invoice sent to patient via email
  |  (Informational only — no patient action required)
  |
  v
After successful charge:
  |  Admin clicks "Send Receipt"
  |  Receipt sent to patient via email
```

**Acceptance criteria:**
- Invoice and receipt sent via email (not WhatsApp)
- Manual trigger — admin clicks a button for each
- Available for all charge types (consultation, medication, no-show)
- Invoice sent before charging, receipt sent after
- Informational only — patient does not need to approve or confirm

---

### A12. Viewing Abandoned Cart Alerts

**As a** clinic admin, **I want to** know when patients abandon the booking process **so that** I can follow up if needed.

**Journey:**

```
Patient registers but doesn't complete booking
  |
  v
Task resource created (type=abandoned-cart)
  |
  v
After 2 hours, WhatsApp reminder auto-sent
  |
  v
Admin can view Task resources in admin app
  |  See which patients were reminded
  |  Manually follow up if WhatsApp didn't convert
```

**Acceptance criteria:**
- Abandoned cart Tasks visible in admin app
- Admin can see whether automated reminder was sent
- Can manually intervene for high-value leads

---

## Pharmacy Staff

### PH1. Fulfilling a Prescription

**As a** pharmacy team member, **I want to** track medication preparation and dispatch **so that** patients receive their prescriptions promptly.

**Journey:**

```
MedicationRequest created by doctor (status: active)
  |
  v
Admin processes medication payment (must succeed first)
  |
  v
Pharmacy staff views prescriptions dashboard
  |  Sees new paid prescriptions ready for fulfillment
  |  (No push notification — checks dashboard)
  |
  v
Creates MedicationDispense (status: preparation)
  |  Selects medication, quantity, packaging
  |
  v
Prepares medication
  |  Updates status: in-progress
  |
  v
Medication packaged and ready
  |  Updates status: completed
  |
  v
Hands off to delivery
  |  Updates deliveryStatus: shipped
  |  -> Patient receives WhatsApp: "Your medication has been shipped"
  |
  v
Delivery confirmed
  |  Updates deliveryStatus: delivered
  |  -> Patient receives WhatsApp: "Your medication has been delivered"
  |  MedicationRequest marked: completed
```

**Acceptance criteria:**
- Pharmacy only starts fulfillment after payment succeeds
- Pharmacy checks dashboard for new prescriptions (no push notifications)
- Status transitions are sequential and tracked
- Patient receives WhatsApp notifications on shipped and delivered
- MedicationRequest closed on delivery completion

---

## Cross-Role Journeys

### End-to-End Consultation Flow

This maps the complete lifecycle from booking to medication delivery, showing all role interactions.

```
PATIENT                    SYSTEM (BOTS)              DOCTOR                  ADMIN                PHARMACY
-------                    -------------              ------                  -----                --------
Completes quiz
(client-side hold)

Selects slot

Authenticates (OTP) -------> on-patient-registered
                              |-- Creates abandoned
                                  cart Task

Quiz answers submitted
(linked to Patient)

Enters card (Stripe)

Confirms booking -----------> on-appointment-created
(slot re-checked)             |-- Creates Daily.co room
                              |-- Sends email + WhatsApp
                              |   (with video join link)
                              |-- Discord admin alert ---------------------> Sees alert

(24h before)                  send-appointment-reminders
Receives reminder <-----------/  (with video join link)

Joins video call --------------------------------------------------> Joins video call
(web or mobile)                                                      (browser tab)
                                                                     |
                                                                     Reviews patient info
                                                                     Writes templated notes
                                                                     |
                                                                     Creates prescription ----> Sees Rx
                                                                     (MedicationRequest)
                                                                     |
                                                                     Recommends follow-up ----> Books follow-up
                                                                                                |
                                                                                                Sends consult invoice (email)
Receives invoice <---------------------------------------------------------------------         |
                                                                                                Charges consult fee
                                                                                                (process-payment bot)
                                                                                                Sends consult receipt (email)
Receives receipt <---------------------------------------------------------------------         |
                                                                                                Sends medication invoice (email)
Receives invoice <---------------------------------------------------------------------         |
                                                                                                Charges medication fee
                                                                                                (process-payment bot)
                                                                                                Sends medication receipt (email)
Receives receipt <---------------------------------------------------------------------
                                                                                                |
                                                                                                Payment success ---------> Sees paid Rx
                                                                                                                           |
                                                                                                                           Prepares medication
                                                                                                                           Ships medication
Receives WhatsApp: <---------------------------------------------------------------                                        |
"Medication shipped"                                                                                                       Updates deliveryStatus
                                                                                                                           |
Receives WhatsApp: <---------------------------------------------------------------                                        Marks delivered
"Medication delivered"

Receives follow-up <------ send-appointment-reminders
reminder
```

---

## Summary Table

| # | Role | Story | Priority |
|---|---|---|---|
| P1 | Patient | New patient booking (full funnel with quiz validation + slot re-check) | Must-have |
| P2 | Patient | Returning patient booking | Must-have |
| P3 | Patient | Abandoned cart recovery | Should-have |
| P4 | Patient | Appointment reminders (24h, with video link) | Must-have |
| P5 | Patient | Attend video consultation (web or mobile) | Must-have |
| P6 | Patient | View medication status | Must-have |
| P7 | Patient | Manage profile | Should-have |
| P8 | Patient | Login to mobile app (OTP, login-only, no registration) | Must-have |
| P9 | Patient | View appointment history | Should-have |
| P10 | Patient | Receive cancellation notification | Must-have |
| P11 | Patient | No-show notification + $20 fee | Must-have |
| P12 | Patient | Update failed card via link | Must-have |
| P13 | Patient | Medication status WhatsApp notifications | Should-have |
| P14 | Patient | Weight/health tracking | Future |
| D1 | Doctor | View scheduled appointments (calendar) | Must-have |
| D2 | Doctor | Conduct consultation (patient info + video + templated notes) | Must-have |
| D3 | Doctor | Create prescription | Must-have |
| D4 | Doctor | Review patient history | Should-have |
| D5 | Doctor | Manage own availability + block-outs | Should-have |
| D6 | Doctor | Mark no-show | Must-have |
| A1 | Admin | Process consultation fee | Must-have |
| A2 | Admin | Process medication fee (gates fulfillment) | Must-have |
| A3 | Admin | Manage prescription fulfillment (oversight) | Must-have |
| A4 | Admin | Book follow-up appointments | Must-have |
| A5 | Admin | Monitor appointment activity | Should-have |
| A6 | Admin | Cancel appointment (free, notifies patient) | Must-have |
| A7 | Admin | Handle no-show ($20 fee + WhatsApp notification) | Must-have |
| A8 | Admin | Handle failed payments (send card update link, retry) | Must-have |
| A9 | Admin | Manage doctor schedules (base hours + recurring + block-outs) | Must-have |
| A10 | Admin | Manage clinical note templates per visit type | Must-have |
| A11 | Admin | Send invoices (before charge) and receipts (after charge) via email | Must-have |
| A12 | Admin | View abandoned cart alerts | Nice-to-have |
| PH1 | Pharmacy | Fulfill prescription + delivery tracking (after payment) | Must-have |

### MVP: External Tools

- **Finance reporting**: Use Stripe Dashboard. Tag each charge with metadata (type: consultation/medication/no-show) for filtering. No custom dashboard.
- **Inventory management**: Google Sheets. Pharmacy staff tracks stock levels and cost-per-unit manually. Profit = Stripe revenue export minus inventory costs from sheet.

### Future Enhancements (Not in MVP)

- Custom finance reporting dashboard (consolidated view of revenue by type, outstanding payments, monthly profit — query Medplum + Stripe)
- Inventory management in-platform (FHIR SupplyDelivery resources or lightweight custom model)
- Patient self-service cancellation/rescheduling in mobile app
- Photo/attachment support on encounters
- Weight/health tracking (Observation resources)
- Push notifications (currently all notifications via WhatsApp + email)
- In-app messaging (FHIR Communication resources)
