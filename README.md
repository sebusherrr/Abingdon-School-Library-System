# A Modern School Library Platform — Full System Design

*"Netflix + Google Workspace + a professional library system + an AI librarian assistant."*

---

## 1. Executive Summary

This document designs a school-wide library management platform that replaces a spreadsheet-era catalogue with a modern, Google Workspace–integrated service. Students discover books the way they discover shows on Netflix; librarians run the collection the way a university library team would; Gemini adds natural-language search and recommendation explanations without ever being load-bearing — the library works perfectly with AI switched off.

Three design principles run through everything below:

1. **Database and rules do the heavy lifting; Gemini explains and converses.** Availability checks, due dates, and ranking are ordinary code. AI is reserved for things only language understanding can do well.
2. **Privacy by default.** Borrowing history is private per-student, analytics are aggregated, and Gemini never sees more than it needs.
3. **Librarians and admins stay in control.** No AI auto-purchases books, auto-changes permissions, or overrides a librarian's judgement.

---

## 2. Product Vision

A single site, reachable with the student's existing school Google account, that:

- Feels personal (a homepage of curated shelves, not a table of rows)
- Makes the physical library easier to run (barcode scanning, shelf locations, live availability)
- Helps librarians make better purchasing and promotion decisions
- Scales from one school library to a multi-school deployment without a redesign

---

## 3. What Makes It Unique

| Compared to a typical school library system | This platform |
|---|---|
| Static, alphabetical catalogue | Personalised, shelf-based homepage |
| "Search box only" discovery | Fuzzy search + natural-language discovery |
| Spreadsheet-style book records | Proper Book vs Copy data model |
| Manual purchasing decisions | Data-informed "collection gap" signals for librarians |
| Generic due-date emails | Contextual, low-noise notifications |
| No insight into recommendation quality | AI dashboard tracking click-through and borrow-through rates |
| AI treated as a gimmick or not used at all | AI scoped precisely to where it adds value, with full fallback |

---

## 4. User Roles

| Role | Core capabilities | Cannot do |
|---|---|---|
| **Student** | Search/browse, view availability, borrow/return/renew/reserve, rate & review, wishlist, request books, view *own* history, tune recommendation preferences | See other students' data, manage catalogue |
| **Teacher** | Everything a student can browse, plus reading lists, class recommendations, subject collections, book requests | See individual student borrowing histories (only aggregated class-level stats, and only for their own classes) |
| **Librarian** | Full catalogue CRUD, copy management, checkout/return, reservations, overdue management, student/teacher account admin, category management, statistics | Change system-wide configuration or AI settings |
| **Administrator** | System configuration, permissions, integrations, AI configuration, audit logs, borrowing-rule policy | Day-to-day catalogue work (by convention, not a hard restriction) |

Role assignment is derived from the school's **Google Workspace Groups** (e.g. `students@`, `staff-library@`, `librarians@`), not maintained by hand in the app.

---

## 5. Complete Feature List

| Area | Features |
|---|---|
| Discovery | Personalised homepage shelves, genre browsing, fuzzy/typo-tolerant search, filters (availability, reading level, subject, series) |
| Book page | Cover, metadata, live copy availability, shelf location, ratings/reviews, "similar books," teacher recommendations, AI explanation |
| Borrowing | Checkout, return, renewal, reservation/holds queue, overdue tracking, lost/damaged handling, reference-only flagging |
| Personal | Wishlist, reading history, ratings, recommendation preference controls |
| Requests | Student/teacher book requests with status tracking through to catalogue addition |
| Teacher tools | Reading lists, class recommendations, subject collections |
| Librarian tools | Catalogue CRUD, copy/condition/location tracking, reservation & overdue management, user administration, statistics dashboard |
| Admin tools | Role/permission config, integration management, AI configuration, audit log viewer, borrowing-rule policy editor |
| AI | Recommendation explanations, natural-language discovery, librarian analytics assistant, reading-list generation |
| Notifications | Due-soon, overdue, reservation-ready, request-approved, digest of new/recommended books |
| Analytics | Circulation trends, genre/author popularity, year-group patterns, collection gaps, recommendation performance |
| Physical integration | Barcode/QR checkout, shelf labelling, self-service kiosk mode |

---

## 6. Gemini Recommendation System

### 6.1 What Gemini is and isn't responsible for

Gemini never scores or ranks books directly against the whole catalogue — that would be slow, expensive, and hard to audit. Instead, a conventional recommendation layer narrows the catalogue to a shortlist of *candidates*, and Gemini's job is limited to:

- Turning a natural-language request into structured search parameters
- Writing a short, honest explanation of *why* a candidate was shortlisted
- Assembling a personalised reading list from candidates a librarian or the ranking algorithm has already approved

### 6.2 Inputs Gemini may use

| Allowed | Never used |
|---|---|
| Borrowed/returned/rated books, genres, subjects, series, authors | Any protected characteristic (ethnicity, disability, religion, etc.) |
| Year group / reading level (for age-appropriateness only) | Free-text inference about a student's personal life |
| Wishlist, search terms | Data belonging to other students |
| Teacher recommendations tied to the student's subjects | Anything outside the approved candidate list (no hallucinated books) |

### 6.3 Explanation style

> "You enjoyed *Percy Jackson and the Lightning Thief*, so this adventure series with a similar pace and humour was shortlisted for you."

Explanations always reference something real (a borrowed book, a genre, a teacher tag) — never invented signals.

### 6.4 Choosing an architecture: A vs B vs C

| Approach | Pros | Cons |
|---|---|---|
| **A. Gemini-only** | Simple to prototype | Expensive at scale, slow, hard to make deterministic, risk of hallucinated titles, difficult to audit for fairness |
| **B. Traditional algorithm + Gemini for explanation only** | Fast, cheap, predictable, auditable ranking; Gemini adds only natural language | Recommendations feel slightly less "conversational" without explicit AI interaction |
| **C. Hybrid (algorithm ranks + Gemini explains + Gemini handles free-text discovery)** | Combines B's reliability with a genuinely useful conversational layer for ad-hoc requests | More moving parts to build and test |

**Recommendation: C**, the hybrid model — algorithmic ranking for the homepage shelves (fast, cheap, reliable), with Gemini layered on for (a) explanations and (b) the separate natural-language discovery feature. This keeps the core library fully functional with AI disabled while still delivering the "ask for a book in your own words" experience.

---

## 7. Recommendation Algorithm

```
USER PROFILE
  borrowing history · ratings · wishlist · favourite genres · teacher tags
        │
        ▼
CANDIDATE GENERATION
  content-based similarity (genre/subject/author/series overlap)
  + collaborative signal (what similar year-group readers borrowed)
  + "new" and "popular" pools
        │
        ▼
SCHOOL RULES / AVAILABILITY / AGE-APPROPRIATENESS FILTER
  reading level ceiling, availability, reference-only exclusion
        │
        ▼
RANKING ALGORITHM (traditional, e.g. weighted scoring — no AI)
        │
        ▼
GEMINI EXPLANATION LAYER (only for the final shortlist, batched)
        │
        ▼
FINAL RECOMMENDATIONS (cached)
```

**Cold start:** a first-login preference picker ("choose 5 genres/authors you like") seeds an initial profile; until then, shelves default to *Popular in your year group* and *New to the library*, which need no personal data at all.

---

## 8. Google Workspace Integration

| Integration | Purpose | Necessary? | Permissions | Privacy note | Add later? |
|---|---|---|---|---|---|
| Google OAuth / Workspace login | Single sign-on with existing school account | **Yes — core** | OpenID/email/profile scope only | No password storage needed | — |
| Google Groups | Derive role (student/teacher/librarian) automatically | **Yes — core** | Read group membership | No new personal data collected | — |
| Google Admin controls | Central org policy, account lifecycle (joiners/leavers) | **Yes — core** | Admin-managed; app just trusts group membership | Deprovisioning is automatic when a Workspace account is suspended | — |
| Gemini for Education | Powers explanations & natural-language search | **Yes, scoped** | Server-side API calls only, never client Gemini access to student data beyond what's designed | Must confirm licensing/age-appropriateness settings with the Workspace admin | — |
| Google Sheets | Bulk catalogue import/export for librarians | Useful, not core | Drive file read/write scoped to librarian-selected sheet | No student data in these sheets | Yes, Phase 5+ |
| Google Forms | Simple book-request intake for students without needing new UI | Nice-to-have | Form responses read into request queue | Low risk if scoped to just the form itself | Yes |
| Google Classroom | Attach reading lists to a class | Useful for teachers | Classroom API, read-only course roster | Should not expose grades or other Classroom data | Yes, later phase |
| Google Drive | Store cover images / import assets | Optional | File read scoped to a library folder | Not for student personal files | Yes |

*Anything marked "must confirm" is flagged for the school's Workspace administrator to verify against their edition and licensing — this is not something the app can guarantee on its own.*

---

## 9. Privacy & Safety

- **Least privilege everywhere.** A student's JWT/session only carries their own ID and role; no endpoint returns another student's loan data.
- **Borrowing history is private.** Never surfaced to other students, never searchable, never in public "leaderboards."
- **Aggregated analytics only.** Dashboards show counts and trends, never "Student X borrowed Book Y" outside the librarian's loan-management screen (which is operationally necessary and audit-logged).
- **No sensitive-characteristic inference.** The system does not attempt to infer race, disability, religion, family situation, etc., and such fields do not exist in the schema.
- **What Gemini receives, explicitly:** book metadata, the student's own borrowing/rating/wishlist signals, year group/reading level, and the pre-filtered candidate list. **What Gemini never receives:** other students' data, protected characteristics, free-text profile inference, raw account credentials.
- **Data retention:** loan history retained for the pedagogic/operational period the school defines (e.g. current + 1 prior year), then anonymised into aggregate stats; accounts are purged on leaver deprovisioning per the school's data policy.
- **Audit logging** on all librarian/admin actions (catalogue edits, account changes, permission changes, AI configuration changes).

---

## 10. Database Architecture

Core principle: a **Book** is catalogue metadata; a **BookCopy** is a physical, trackable item.

| Table | Purpose | PK | Key FKs |
|---|---|---|---|
| `users` | All accounts (linked to Workspace identity) | `user_id` | `role_id` |
| `roles` | Student/Teacher/Librarian/Admin | `role_id` | — |
| `books` | Catalogue-level metadata (ISBN, title, description, series, reading level…) | `book_id` | — |
| `authors` | Author records | `author_id` | — |
| `book_authors` | Many-to-many book↔author | `book_id, author_id` | `book_id`, `author_id` |
| `book_copies` | Physical item: barcode, condition, location, status | `copy_id` | `book_id` |
| `genres` / `book_genres` | Genre taxonomy and linkage | `genre_id` | `book_id` |
| `subjects` | Curriculum subject tagging | `subject_id` | `book_id` |
| `loans` | Checkout/return records | `loan_id` | `copy_id`, `user_id` |
| `reservations` | Hold queue | `reservation_id` | `book_id`, `user_id` |
| `ratings` | Star ratings | `rating_id` | `book_id`, `user_id` |
| `reviews` | Optional text reviews | `review_id` | `book_id`, `user_id` |
| `wishlists` / `wishlist_items` | Personal wishlists | `wishlist_id` | `user_id`, `book_id` |
| `recommendations` | Cached, generated recommendation sets + explanation text | `rec_id` | `user_id`, `book_id` |
| `book_requests` | Acquisition request workflow with status | `request_id` | `user_id`, `book_id`(nullable) |
| `teacher_reading_lists` / `reading_list_items` | Curated class lists | `list_id` | `teacher_id`, `book_id` |
| `notifications` | Queued/sent alerts | `notification_id` | `user_id` |
| `audit_logs` | Who did what, when | `log_id` | `user_id` |

**Relationships in brief:** a `book` has many `book_copies`, many `authors` (via `book_authors`), many `genres`/`subjects`; a `user` has many `loans`, `reservations`, `ratings`, `wishlist_items`; a `loan` always references exactly one `book_copy` and one `user`; `recommendations` link a `user` to a `book` with the stored explanation and the algorithm's score, refreshed on a schedule rather than per page load.

---

## 11. System Architecture

```
┌──────────────┐     ┌───────────────────┐     ┌─────────────────┐
│  Frontend      │──▶│  API / App Server   │──▶│  PostgreSQL       │
│  (React/Next)  │   │  (business rules,    │   │  (catalogue,      │
│                │◀──│   auth, ranking)     │◀──│   loans, users)   │
└──────────────┘     └────────┬──────────┘     └─────────────────┘
                                │
                ┌───────────────┼────────────────┐
                ▼                                ▼
        ┌───────────────┐                ┌───────────────────┐
        │ Google OAuth /  │                │ Recommendation      │
        │ Workspace       │                │ worker (scheduled)  │
        └───────────────┘                └─────────┬─────────┘
                                                        ▼
                                               ┌───────────────────┐
                                               │ Gemini API          │
                                               │ (explanations,      │
                                               │ NL search only)     │
                                               └───────────────────┘
```

Gemini sits behind a scheduled worker and a narrow "explain/parse" API — it is never in the request path of an ordinary page load.

---

## 12. UI/UX Design

- **Palette:** a calm primary (deep teal or indigo) for navigation/CTAs, warm neutral backgrounds, a single accent colour reserved for "available now" status — avoids feeling like a corporate SaaS dashboard while staying school-appropriate.
- **Typography:** a humanist sans-serif for UI text, generous size for readability (WCAG AA contrast minimum), a slightly larger base font for younger year groups if the school wants an accessibility mode.
- **Cards:** book cover-led cards, consistent aspect ratio, availability badge overlay (green "available," amber "reserved," grey "checked out").
- **Navigation:** persistent top nav (Home / Catalogue / My Books / Recommended / Wishlist / Requests) for students; a left-rail dashboard nav for librarians.
- **States:** clear empty states ("No results — try fewer keywords"), honest AI states ("Gemini couldn't find an exact match — here's the closest available"), skeleton loaders instead of spinners for shelf rows.
- **Accessibility:** full keyboard navigation, alt text on covers, AA contrast, scalable text, screen-reader-friendly status badges (not colour-only).

---

## 13–16. Role Experiences

| Role | Landing view | Signature screens |
|---|---|---|
| **Librarian** | Today's activity dashboard | Loans, Copies, Reservations, Requests, Analytics, AI Tools |
| **Student** | Personalised Netflix-style homepage | Catalogue, My Books, Recommended, Wishlist, Requests |
| **Teacher** | Class + subject overview | Reading list builder, class recommendation tool, subject collections |
| **Administrator** | System health & configuration | Permissions, Integrations, AI Configuration, Audit Logs, Borrowing Rules |

---

## 14. (continued) Teacher Experience Detail

Teachers see aggregated engagement for lists they've created (e.g. "12 of 28 students have borrowed a book from this list") but never a named breakdown of who has or hasn't — preserving student privacy while still giving teachers useful signal.

---

## 17. Search System

- Full-text search across title/author/ISBN/genre/subject/series/keyword, with structured filters for availability, location, reading level, and publication date.
- **Fuzzy matching** (trigram or Postgres `pg_trgm`) handles "Harry potter" → *Harry Potter and the Philosopher's Stone*.
- **Typo tolerance** (edit-distance matching) handles "philosphers stone" → correct title, without needing AI for the common case.
- Natural-language requests ("something like Percy Jackson but less mythology") are the one place Gemini is used, converting free text into the same structured filters the ordinary search uses — so results are still real, ranked catalogue items.

---

## 18. Borrowing System

**Workflow:** Scan copy barcode → system identifies the copy and its book → checks eligibility (loan limit, no existing overdue, not reference-only) → creates `loan` with due date per year-group policy → confirmation shown/printed if desired.

- **Renewal** allowed up to a school-configured limit, blocked if a reservation is waiting.
- **Reservations** form a queue per book; the next available copy auto-notifies the top of the queue.
- **Overdue/lost/damaged** are copy-status states with librarian-managed workflows (fines are optional and school-policy-driven, not assumed).
- **Checkout hardware:** barcode/QR scanner is the primary method; supports existing student ID cards if they already carry a barcode/QR, avoiding new hardware spend. A self-service kiosk (a tablet + USB scanner) is realistic for most schools; no specialist RFID hardware is assumed.

---

## 19. Notification System

| Trigger | Channel | Notes |
|---|---|---|
| Due in 2 days | Email (Workspace) + in-app | One reminder, not repeated daily |
| Overdue | Email + in-app | Escalating tone after repeated overdue, still respectful |
| Reservation ready | Email + in-app | Time-boxed hold (e.g. 48 hrs) before it passes to the next student |
| Request approved/rejected | In-app | Also visible on the request status page |
| New/recommended books | Weekly digest, opt-out available | Batched to avoid spam |

Workspace makes email delivery straightforward (school-managed addresses, no separate email infra needed) — this is a "yes, useful, low-effort" integration.

---

## 20. Analytics

Aggregated-only dashboards: borrows/week and /month, top genres/authors/books, year-group borrowing patterns, new-book performance, reservation demand, overdue trend lines, "books never borrowed" (collection-gap signal), and recommendation performance (click-through vs. actual borrow rate). No individual-student breakdowns appear outside the librarian's operational loan screen.

---

## 21. Smart Collection Management

The system surfaces patterns like "Science fiction searches are 3× higher than the section's copy count would suggest" or "15 unique students have requested robotics-related books" as **signals on the librarian dashboard** — plain aggregated statistics, not AI decisions. The librarian reviews and decides; nothing is auto-ordered.

---

## 22. AI Librarian Assistant

A separate, librarian-only Gemini-backed chat that can answer questions like "which genres grew this term?" or "create a Year 9 sci-fi reading list," by first querying the librarian's own authorised aggregate data and then having Gemini summarise/format it. It never accesses data the logged-in librarian couldn't otherwise see, and every generated reading list is a *draft* the librarian must approve before publishing.

---

## 23. Natural-Language Book Search

Student free-text ("I need a book for a WW2 history project") is parsed by Gemini into structured filters (subject=History, keyword=World War Two, availability=true), which run through the *normal* search — so results are always real catalogue items. If nothing matches well, the system says so plainly rather than inventing a title:

> "I couldn't find that exact book in our library, but these available books are similar."

---

## 24. Physical Library Integration

Barcode on spine/label → USB or Bluetooth scanner at the desk (or a tablet for self-service) → system resolves the copy → checkout/return recorded instantly. Shelf locations are stored per copy and shown on the book page ("Fiction → Fantasy → F12"). QR codes are a low-cost alternative to barcodes if the school prefers printing them in-house. No RFID or specialist inventory hardware is assumed necessary.

---

## 25. Security

- **Authentication:** Google Workspace OAuth only — no passwords stored by the app at all.
- **Authorisation:** role-based access control enforced server-side on every endpoint, not just hidden in the UI.
- **Session management:** short-lived tokens, refreshed via Google's standard OAuth flow.
- **API security:** input validation and parameterised queries (SQL-injection safe by construction with an ORM), output encoding (XSS-safe), CSRF tokens on state-changing requests, rate limiting on search and auth endpoints.
- **Audit logs** on catalogue changes, permission changes, and AI configuration changes.
- **Backups:** scheduled database backups with tested restore procedure; point-in-time recovery for the loan/catalogue data.

---

## 26. Performance

Recommendations are **generated on a schedule** (e.g. nightly, or on a meaningful event like a new loan) and cached in the `recommendations` table. The homepage always reads from cache — Gemini is never called on page load. A background worker refreshes stale recommendation sets; Gemini calls are batched across many candidate books at once rather than one call per book.

---

## 27. Cost Control

| Task | AI needed? |
|---|---|
| Check availability, calculate due dates, ISBN lookup | No — plain queries |
| Rank candidate books | No — weighted algorithm |
| Explain *why* a book was recommended | Yes — genuine natural-language value |
| Parse a free-text discovery request | Yes |
| Summarise a month of activity for a librarian | Yes |

Gemini calls are batched, cached, and scheduled — never fired per page view — which keeps API cost roughly proportional to catalogue/user growth, not to page traffic.

---

## 28. MVP

| Tier | Features |
|---|---|
| **Must have** | Google login/roles, catalogue browse & search, book page, checkout/return/renew, librarian book & copy management, basic due-date notifications |
| **Should have** | Reservations, wishlist, ratings, book requests, librarian dashboard stats |
| **Nice to have** | Algorithmic (non-AI) recommendation shelves, teacher reading lists |
| **Future / advanced** | Gemini explanations, natural-language discovery, AI librarian assistant, smart collection-gap signals, Classroom/Sheets/Forms integration |

---

## 29. Development Roadmap

| Phase | Focus |
|---|---|
| 1 | Database schema + Google OAuth/role sync |
| 2 | Catalogue (books, copies, search) |
| 3 | Borrowing system (checkout/return/renew/reserve) |
| 4 | Librarian dashboard + user management |
| 5 | Google Workspace extras (Sheets import, Forms requests) |
| 6 | Algorithmic recommendation engine (no AI yet) |
| 7 | Gemini integration (explanations + NL search) |
| 8 | Analytics & smart collection signals |
| 9 | Security hardening + full test suite |
| 10 | Deployment & school onboarding |

---

## 30. Testing Plan

| Type | Example cases |
|---|---|
| Permission tests | Student cannot reach librarian endpoints; teacher cannot view individual student loan history; librarian cannot change system-wide AI config |
| Auth tests | Only Workspace-authenticated accounts can log in; deprovisioned accounts lose access immediately |
| Database tests | Loan creation respects copy availability; deleting a book cascades correctly to copies/genres |
| AI tests | Gemini output validated against the real candidate list before display; a fabricated title is rejected and falls back to search; timeout/error triggers graceful fallback |
| Recommendation tests | Cold-start user gets sensible defaults; recommendation cache refresh doesn't call Gemini per page load |
| Accessibility tests | Full keyboard navigation; screen-reader labels on status badges; AA contrast check |
| Performance tests | Homepage loads from cache under load without live AI calls |
| Security tests | SQLi/XSS/CSRF attempts blocked; rate limits enforced |

---

## 31. Future Features

Multi-school/district deployment, ebook/audiobook integration, reading-streak gamification (privacy-safe, opt-in), Classroom-linked assignment reading lists, richer author/series pages, accessibility text-to-speech for book descriptions.

---

## 32. Final Recommendation

Build the MVP first with **no AI at all** — a rock-solid catalogue, borrowing system, and librarian dashboard. Layer in the algorithmic recommendation engine next (still no AI dependency). Only then add Gemini for explanations and natural-language discovery, scoped and cached exactly as described above, so the library never depends on an external AI service to function.

---

# THE 10 FEATURES I WOULD BUILD FIRST

1. **Google Workspace login + role sync** — everything else depends on trustworthy identity and roles.
2. **Book/Copy data model + catalogue CRUD** — the foundation of every other feature.
3. **Search with fuzzy/typo tolerance** — the single most-used feature, day one.
4. **Checkout/return via barcode scan** — the core physical-library workflow librarians need immediately.
5. **Student "My Books" view (loans, due dates, renew)** — the core student-facing utility.
6. **Librarian dashboard (today's activity, overdue, inventory)** — makes the system immediately useful operationally.
7. **Reservations/holds queue** — high value, purely algorithmic, no AI needed.
8. **Book requests workflow** — gives students/teachers a voice and librarians a pipeline.
9. **Algorithmic (non-AI) recommendation shelves** — delivers the "feels personal" experience cheaply and reliably.
10. **Gemini explanation layer + natural-language discovery** — the standout feature, but deliberately last, once everything beneath it is solid.
