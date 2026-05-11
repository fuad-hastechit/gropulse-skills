---
description: Add free appointment booking (Growth Strategy Call) to a Gropulse Shopify app
---

Invoke the gropulse-skills:appointment-booking skill.

Add the "Book a Free Growth Strategy Call" feature to this Gropulse Shopify app:
- Install @gropulse/booking-widget
- Add shop context fields to the root app loader (bookingApiUrl defaults to https://booking.gropulse.com)
- Create the /app/growth-call page with exact verbatim content and inline BookingWidget (Button 2)
- Add floating CTA button to the app shell (Button 1) — visible on every page, hidden on /app/growth-call

Note: there are TWO distinct "Book Free Strategy Call" buttons:
- Button 1 (floating, dark navy, fixed bottom-left) — navigates TO the growth-call page, visible everywhere except that page
- Button 2 (widget-styled) — lives ON the growth-call page, opens the AI booking chat

Follow the skill steps exactly. State assumptions before starting.
