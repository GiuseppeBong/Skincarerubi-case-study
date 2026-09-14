Skincare Rubí — A Custom Booking System for a Beauty Clinic
A PHP/MySQL site for a beauty clinic in Maracay, Venezuela, built around a scheduling engine that follows the clinic's own treatment rules. Live at skincarerubi.com.

Context
Skincare Rubí is a single-practitioner beauty clinic offering five services: laser hair removal, electrolysis, brows and lashes, facial treatments, and micropigmentation.

The owner wanted something that fit how she actually runs her business, and specifically how she organizes treatment schedules. That turned out to be the deciding factor against a template or a site builder.

Laser hair removal isn't a single appointment. It's a course of sessions spaced 21 days apart, and the clinic only performs laser on Mondays, Wednesdays, Thursdays, and Saturdays. So when a client books, the system has to project a whole calendar forward, keep the 21-day cadence, and never land on a day the clinic doesn't do laser. If a client misses one, every session after it moves.

That's the part no builder gives you. A template can render a services page and a contact form, but the scheduling rules are specific to this business, and they're the reason the site exists. Everything else — the service pages, the price list, the galleries — is supporting material around that core.

This was my first professional project. The first version took 25 days.

What the site does
Public side. A landing page with the five services plus a price list. Each service has its own page with a photo and video carousel, treatment information, and pre-appointment instructions. Laser has additional pages for pricing and active promotions. Location, hours, and contact links sit in a shared footer.

Client accounts. Clients register and get a personal area where they see their projected schedule: every session, its date, whether it's been completed, and whether it's been paid.

Admin panel. The owner manages everything herself without touching code:

Appointment list, filterable by current month, next month, all pending, overdue, completed, and a trash bin with restore
Mark sessions completed or paid, reschedule with automatic cascade to later sessions
WhatsApp link per client, prefilled and ready to send
Full price editor across all service categories — edit, add, and delete rows
Promotional flyer uploads with activate/pause
Per-service media galleries with drag-and-drop reordering that works on desktop and touch
Added later, at the client's request. The original build assumed prepaid packages of eight sessions with fixed zone combinations. A year in, the owner wanted to change how she sells:

Free-form zone selection. Instead of picking from preset combinations, clients now choose any mix of the fifteen individual treatment areas.
Per-session payment. No prepaid packages — each session is paid on the day, and both the owner and the client can mark it.
Variable course length. Clients pick 1, 4, or 8 sessions instead of always committing to the full course.
Discount bundles the owner defines herself, priced below the sum of their parts, which appear as one-tap shortcuts when booking.
There's also a WhatsApp reminder engine built to run from cron. It's written and idempotent — each appointment carries flags so a reminder is never sent twice — but it isn't enabled yet, since it needs approved message templates from Meta.

Stack and infrastructure constraints
PHP 8.1 and MariaDB on shared cPanel hosting. PDO with prepared statements throughout. Tailwind and Flatpickr from CDN. Apache mod_rewrite for extensionless URLs.

The hosting shaped more decisions than the language did.

No Node, no build step. Which is why Tailwind comes from a CDN rather than a compiled stylesheet. It's the wrong answer for production — it costs a round trip and ships far more CSS than the site uses — but adding a build step to an environment with no deploy pipeline means someone has to remember to run it, and that someone eventually forgets. I took the consistency over the bytes.

No background workers. Anything scheduled has to be a cron entry point that runs, does its work, and exits. That's why the reminder engine is a standalone script with database flags instead of a queue: it has to be safe to run repeatedly, possibly overlapping, with no coordinator.

No deploy pipeline. Deploys are files uploaded through a file manager. That single constraint drove the schema strategy. There's no migrate command to run, so the project has one idempotent setup script that can be opened in a browser: it creates tables if missing, adds columns only when they don't exist, and reports what it changed. Every schema change since has been appended to it as a new guarded block. Running it twice is a no-op, which matters when the only way to run it is for a human to visit a URL.

Shared resource limits. The account runs under per-account caps on processes, CPU, and I/O. That makes anything expensive per-request a real risk rather than a theoretical one, which is where the pagination work below came from.

Technical decisions
Projecting a schedule onto restricted days
Sessions are 21 days apart, but the clinic only does laser four days a week. Adding 21 days repeatedly lands on a closed day roughly half the time.

Three options. Generate the rigid 21-day sequence and let the owner fix collisions by hand — rejected, because it pushes the work back onto the person the software is supposed to help. Always push forward to the next open day — simple, but it drifts: push forward enough times and a course stretches weeks past where it should end. Or snap to the nearest open day in either direction.

I went with nearest-in-either-direction, searching up to three days out and taking the smaller offset. It keeps the course close to its intended length because errors cancel instead of accumulating — a session pushed two days later is often followed by one pulled a day earlier.

The harder half was rescheduling. When a session moves, every later session has to move with it, keeping the cadence and the day restriction. That's the same generator run again from the new date, applied to the remaining sessions. Writing it once and calling it from both paths meant booking and rescheduling can't drift apart in behavior — which matters more than it sounds, because they're triggered from completely different parts of the app.

Storing arbitrary zone combinations
The rewrite replaced fixed packages with free selection from fifteen treatment areas. That changed what a booking is: previously one value from a known list, now any subset of fifteen.

I considered a junction table — properly normalized, one row per zone per booking. I also considered a JSON column. I went with a delimited string in the existing column, and I want to be honest that this was a trade-off rather than the textbook answer.

The reasoning: nothing in the application queries by individual zone. The zones are displayed as a unit on the client's schedule, in the admin table, and in the packages themselves. A junction table would have meant a new table, a migration on a live database, and joins on every read, to support queries the product doesn't make. The delimited string kept the change to one widened column.

Two things make it survivable. Everything is validated server-side against a canonical list before it's stored, so nothing outside the fifteen known zones can be written. And the separator is deliberately not +, because zone names already contain it — "PECHO + ABDOMEN" is one zone, and joining with + produces text nobody, including the parser, can unambiguously split. I used a middle dot instead, a character that can't appear inside a zone name.

If the product ever needs "how many clients book underarms," this becomes the wrong choice and I'd normalize it then.

Paginating the appointment table
The admin appointment list rendered every matching row in one page. That's fine for a few dozen appointments. It stops being fine as the table grows, and on shared hosting with per-account resource caps, a page that grows without bound is a liability rather than an annoyance.

The trigger was that I made it worse. Adding a payment toggle meant another form per row, which grew each row by about a quarter. The "all pending" filter was already producing several megabytes of HTML per request; the addition pushed it further.

I reproduced production volume locally with generated data so I could measure rather than guess. Options were client-side filtering of a fully-rendered table — which doesn't help, since the cost is in rendering and transferring it — infinite scroll, or server-side pagination.

Server-side pagination, 40 rows per page, with a separate count query for the pager. The measured result was roughly a 95% reduction in response size for the heaviest filter. Two details mattered beyond the query: the page number survives form actions, so marking an appointment doesn't bounce the owner back to page one, and the filter is validated against an allowlist before the WHERE clause is assembled rather than after, so the query and the highlighted button can't disagree.

Data model
Seven tables, three of which carry the scheduling logic.

Users holds both clients and the administrator, separated by a role column. Phone and email are each unique but individually optional, since a client may register with either.

Laser packages is a course of treatment belonging to one user. It records the selected zones, how many sessions the course runs, how many are done, and whether the course is active, completed, or cancelled — cancelled being the trash bin, which is why deletion from the panel is reversible.

Laser sessions is an individual appointment belonging to a package: its number in the course, its scheduled date, its state, whether it's been paid, and two flags recording which reminders have already gone out. The state distinguishes upcoming appointments from ones whose date has passed without being marked complete, which is what drives the "overdue" filter in the panel.

Both child tables cascade on delete, so removing a client removes her courses and their appointments in one operation rather than leaving orphans.

The remaining three are content the owner edits rather than transactional data: service prices, grouped by category with a display order; promotions, which are flyer images with an active flag; and carousel media, which stores per-service images and videos with their ordering. Keeping these in the database rather than in code is what lets the owner change prices and swap photos without a developer.

There's also a legacy bookings table left over from an earlier approach to scheduling. It's empty and nothing reads it — I've left it in place rather than dropping a table from a live database for no operational gain.

Maintenance
The site is in production and I still maintain it. The per-session billing model, free-form zone selection, variable course lengths, and owner-defined discount bundles were all built after launch, in response to how the business actually changed.

That ongoing relationship has shaped how I work on it. Schema changes go into the same idempotent setup script rather than ad-hoc SQL, because the person running them is not a developer and the instruction has to be "open this URL" rather than a sequence of statements. Anything that changes the panel gets explained in plain language when it ships, because a feature the owner doesn't know exists may as well not.

What I'd do differently
Separate logic from presentation. Every page is its own controller and view in one file: request handling at the top, HTML below. It's readable at this size and it has no dependencies, but it means the admin sidebar is copy-pasted across four files and drifts when one changes. A shared layout and a thin routing layer would have cost little up front and saved real friction.

Commit smaller and deploy properly. My first commit was the entire finished platform, and there were month-long gaps between commits after that. That makes the history useless for understanding why anything is the way it is. Deployment is also manual file uploads, which means no atomic deploy and no rollback — if half the files land, the site is half-broken. Even a git pull on the server would be a significant improvement. Related: the database connection settings live in a version-controlled file that has to be hand-edited on the server, so every deploy has to carefully avoid overwriting it. That belongs in a separate, untracked config file.

Write tests, at least for the date logic. There are none. The scheduling generator is pure logic with clear inputs and outputs — the easiest thing in the project to test and the most expensive to get wrong, since a bad date reaches a real person's calendar. I also once changed the values the code writes to a column without changing the column to accept them; the database quietly stored something else, and nothing caught it for months because nothing was checking. A few assertions would have.

Model the data properly where it counts. Prices are stored as text rather than a numeric type, which means any arithmetic on them requires parsing first. Zone selections are a delimited string for the reasons above. Both are defensible at this scale and both would be wrong in a larger system — and I'd rather write down that I know the difference than pretend the shortcuts were the plan.
