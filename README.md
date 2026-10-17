


ROLE
You are a senior full-stack engineer acting as my technical co-founder. Build a demo-ready MVP of an esports ladder platform for Ethiopia, to be shown live to judges at the GameBridge 250 Innovation Challenge prototyping gate. My team is beginner-to-mixed and knows Python, so keep code simple, typed and commented. A reliable core loop beats a long feature list.

PRODUCT
A Telegram-first, web-based platform where players link their game accounts, get a verified rating and rank, join free local cups run by trusted "Hub Captains" in their city, report results, and build a public "Gamer Passport" that scouts can open without Telegram. Full ladder: Grassroots Daily Cups > City > Regional > National > Global Scouting. Build only Levels 1-2; show the rest as roadmap.

USERS AND CONSTRAINTS
Ethiopian youth on budget Android phones, slow and expensive mobile data, heavy Telegram use. Telegram has been blocked in Ethiopia before, so the web app must also work in a normal browser. Some players are under 18. Languages: English + Amharic from day one (Afaan Oromo later).

TECH STACK (do not substitute without asking)
- Python 3.12, FastAPI, SQLAlchemy 2 + Alembic, PostgreSQL (SQLite only for local dev via env var)
- aiogram 3 bot in webhook mode inside the same FastAPI app
- ONE mobile-first web UI: Jinja2 + HTMX + a small compiled Tailwind CSS file. No heavy JS framework; first load under 200 KB. It opens as a Telegram Mini App AND works in a normal browser.
- Auth: validate Telegram WebApp initData (HMAC) on the server; never trust a client-sent Telegram ID. Browser login via Telegram Login Widget or a one-time login link from the bot. Signed session cookies.
- Docker + docker-compose, .env.example, Makefile, README with exact run and deploy steps.
- PWA manifest + service worker for static asset caching: last step only.

GAME ADAPTER SYSTEM (core idea)
Each game is a plug-in class: key, display name, tier, modes, fetch_rating(username, mode), verify_owner(username, code). Adding a game = one new file + one registry line.
- Tier A (API-verified): Lichess and Chess.com.
  - Lichess: GET https://lichess.org/api/user/{username} gives ratings per mode (bullet, blitz, rapid, classical), games count, ToS-violation flag, profile bio.
  - Chess.com: GET https://api.chess.com/pub/player/{username}/stats gives ratings per format incl. rating deviation. Send requests serially, with a User-Agent containing app name + contact email.
  - Check the current official docs before coding. Handle 404, 429 and timeouts gracefully.
- Tier B (screenshot-verified): one local mobile game, [LOCAL_GAME]. Player declares in-game ID + rank tier + screenshot; a Hub Captain approves or rejects. Do NOT use unofficial Free Fire/PUBG scraping APIs. If [LOCAL_GAME] is battle royale, STOP and ask me: it needs lobby points scoring, not 1v1 Elo.
- Tier C: platform-only results.
- "Coming soon" cards (UI only): Clash Royale, Brawl Stars.

ACCOUNT LINKING AND VERIFICATION
Player enters username > system generates a one-time code (e.g. GB-4F7K) > player puts it in their Lichess bio or Chess.com profile > system checks via API > "Verified". Otherwise "Claimed": clearly badged and excluded from leaderboards and prizes. One external account per game per platform user (unique constraint). Consent checkbox before linking. Reject ToS-flagged accounts, accounts below a configurable minimum of rated games (default 30), and provisional or high-deviation ratings.

RATINGS AND LEADERBOARDS
- Verified Rating: imported external rating per game+mode, cached, refreshed nightly by a background job, manual refresh limited to once per hour. Store RatingSnapshot history for graphs. If an API is down, show the cached value with a "last updated" time.
- Platform Rating: Elo earned in our cups, seeded from the verified rating. K=40 for the first 10 matches, then 24. Named tiers (Bronze to Diamond).
- Leaderboards: per game+mode, national by default with a city filter. Never mix games or modes. Minimum games to appear.

CUPS AND MATCHES
- Hub Captain creates a free single-elimination Daily Cup (game, mode, start time, max players, sponsor prize text).
- Players join in-app. Auto-generate the bracket seeded by rating, with byes, deadlines and forfeits for no-shows.
- Match room: opponent, deadline, Telegram deep link to contact them, report button.
- Results:
  - Lichess: player submits a game link; fetch the game via the Lichess API (verify the current endpoint); confirm both players are the linked accounts, the result, and that it was played inside the match window > auto-confirm.
  - Other games: both players self-report the score and upload a screenshot (compress in the browser to max 1280px JPEG; server limit 3 MB; validate file type). If reports agree > auto-confirm; if not > dispute queue for the Hub Captain.
- Results are APPEND-ONLY events. Corrections are new events with reason + actor. Nothing is ever deleted or edited.
- After each match both players rate the opponent clean/suspicious. Repeated low ratings create a manual-review flag only, never an auto-ban.

ROLES
player, hub_captain (scoped to their hub), super_admin. Every captain/admin action is written to an AuditLog. Captain web dashboard: active cups, players, pending approvals and disputes.

PASSPORT
Public page /p/<handle>, no login needed: verified ratings per game with badges, platform rating + tier, rating history graph, win/loss, read-only tournament history, fair-play indicator, milestone badges (first win, first city cup, 10-win streak). Generate a shareable PNG card (Pillow) + Open Graph tags.

BOT
/start with deep-link referral (t.me/<bot>?start=hub_<code>), language choice, registration, "Open App" WebApp button, notifications in the user's language (cup starting, your match, report result, you advanced, dispute update), /leaderboard, /profile.

PAYMENTS
Out of scope: all cups are free. Add a PaymentProvider interface with a ManualConfirm stub and a documented place for Telebirr web checkout later. No real money handling.

DEMO READINESS (most important)
1. Seed script + admin button that creates clearly labelled DEMO DATA (3 hubs, ~40 players, 1 finished cup, 1 live cup) with one-click wipe. Real pilot data stays separate.
2. A /judge page: guided 5-minute walkthrough linking to each step.
3. Analytics page: registrations, % matches completed, dispute rate, per-hub activity, day-1/day-7 return.
4. Never crash on an external API failure: 5s timeouts, retry with backoff, cached fallback, friendly error pages.
5. /health endpoint, structured logging, Sentry-ready hook.
6. Public HTTPS deployment guide (single VPS or PaaS, region near East Africa, Cloudflare in front).
7. Test under slow-3G emulation and report page weight.

SECURITY AND PRIVACY
Secrets only in env vars. Validate all input. Rate-limit the bot and API. CSRF on forms. Random filenames for uploads. Collect minimal data. Privacy page. Age bracket stored; no money features for under-18. Users can export and delete their data.

OUT OF SCOPE (do not build)
Native apps, real payments, screenshot OCR, unofficial scraping APIs, more than 3 live games, chat, ML anti-cheat, ladder Levels 3-5.

TESTS (pytest)
Elo, bracket generation incl. byes, initData validation, adapter parsing with mocked API responses, auto-confirm vs dispute logic, append-only corrections.

HOW TO WORK
Build in phases. After each phase: run the tests, summarize what works and how to run it, list your assumptions, then STOP and wait for me to say "continue".
- Phase 0: scaffold, config, DB + migrations, health check, Docker, i18n skeleton (EN/AM)
- Phase 1: auth (Mini App + browser), users, hubs, bot registration
- Phase 2: adapters, account linking + verification, ratings, leaderboards
- Phase 3: cups, joining, auto bracket, match rooms, notifications
- Phase 4: result reporting, auto-verification, disputes, Elo
- Phase 5: Passport, share card, captain dashboard, audit log
- Phase 6: demo mode, analytics, /judge page, PWA, deployment docs, demo script
Ask me only blocking questions up front; otherwise state assumptions and proceed. Keep code readable for beginners, with a README section per module.

MY INPUTS (I will fill these in)
Pilot city and hubs: [...]
Local game (Tier B): [...]
Bot name: [...]
Deadline for the prototyping gate: [...]
Team skills: [...]

DEFINITION OF DONE (judges can see this live, on a phone)
1. Open the bot link and register in Amharic or English
2. Link a Lichess account, verify it with the bio code, see verified rating + rank
3. Join a cup and see the auto-generated bracket
4. Submit a Lichess game link: result auto-confirms, ratings update, a notification arrives
5. A screenshot-game dispute gets resolved by a Hub Captain
6. A public Passport and share card open without Telegram
7. The same site works in a normal browser if Telegram is unavailable
8. The analytics page shows real numbers



https://forms.gle/rx5KzWYemLdzxnYA6


--------------------------------------------------------------------------------------------------------------------------------------------------
=====================================================================================================================================================

















Beyond the Ballot: Building a Democracy Africans Can Live In

Ask an African what democracy means to them and you will rarely hear about ballot boxes first. You are more likely to hear about a road that never got repaired after the councillor was elected, a clinic that closed when its funding moved to a ministry no one could reach, or a young graduate who has voted in three elections and still cannot find work. For many citizens, democracy is judged less by the act of voting than by whether it changes anything in daily life. The next twenty years will be decided by whether African democracy can answer that question with more than elections.

The progress of the last two decades is real and should not be dismissed. Ghana has handed power over peacefully several times since 2000, and Botswana has sustained multiparty competition for longer than most of its neighbours. In 2024, Senegal offered a striking lesson. When the incumbent tried to postpone a presidential election, the Constitutional Council overturned the delay, and citizens and civil society pressed until the vote took place and power changed hands peacefully. Such moments show that African citizens are not passive. They defend the rules when those rules are threatened.

Yet the same decades have exposed deep fractures. Between 2020 and 2023, military officers seized power in Mali, Guinea, Burkina Faso, Niger and Gabon, often presenting themselves as saviours from corrupt and unresponsive governments. Several of those governments had been elected, which is the uncomfortable point. An election held every few years cannot compensate for institutions that fail citizens between votes: courts that bend, police who answer to the ruling party, and public services that reward loyalty rather than need. When democracy is reduced to a calendar of polls, people learn to see it as a contest among elites rather than a system that serves them.

Money compounds the problem. Campaign financing is often opaque, and candidates who depend on private patrons owe those patrons after they win. Public contracts become rewards for loyalty. Meanwhile, young people, who make up roughly 60 per cent of Africa’s population under the age of 25, face high unemployment and few routes into political life. Parties recruit older members, candidacy fees are costly, and youth who protest, as Kenyans did in 2024 against a proposed finance bill, are too often met with force rather than dialogue. Young Africans are not disengaged from politics. Many are simply shut out of its institutions.

Digital technology sharpens these tensions. Social media has given citizens new ways to expose corruption and organise, but it has also made disinformation faster and cheaper to produce. Some governments have responded not by improving the information environment but by restricting internet access during elections and protests, which erodes trust on all sides. A democracy that cannot agree on what is true will struggle to hold anyone accountable.

So what should democracy look like in twenty years? I would argue it should be measured by the distance between the citizen and the decision. Four practical steps could close that distance, and each draws on resources that already exist on the continent.

First, decentralisation must be made real rather than declared. Constitutions across the continent promise local government, but councils often lack budgets, staff and the power to decide anything meaningful. Participatory budgeting, which began in Porto Alegre, Brazil, and has since been adapted in cities around the world including several in Africa, lets residents decide how a share of public money is spent. Scaling this with transparent public accounts would turn voting into ongoing participation.

Second, African societies should draw on their own traditions of governance. The Gadaa system of the Oromo people, which UNESCO recognised as intangible cultural heritage in 2016, organised political authority around age-based cohorts that passed responsibilities on at regular intervals, building rotation and accountability into leadership. Other communities have councils of elders, consensus-building practices and customary checks on rulers. These are not museum pieces. Universities and civic educators could study them alongside Western political theory, so that democracy feels like something Africans built rather than something imported.

Third, young people need a formal place at the table. Governments could lower the minimum age for candidates, reserve a share of list seats for under-35s, and require parties to publish how they select candidates. The African Youth Charter already calls for young people’s participation in decision-making, and these reforms would put that call into practice. Such measures cost little and send a clear message that the future belongs to those who will live longest with its consequences.

Fourth, the information environment needs both protection and literacy. Instead of switching off the internet, governments should commit to predictable, rights-based rules for platforms, while schools teach students to verify sources. Independent fact-checking organisations, many already working across the continent, deserve stable funding.

None of these ideas will work if they are copied wholesale from elsewhere, and none will succeed without enforcement. The African Charter on Democracy, Elections and Governance, adopted in 2007, already sets clear standards on elections, term limits and unconstitutional changes of government. Its weakness has never been its text but its implementation. Citizens, parliaments and regional bodies must treat the charter as a tool to be used, not a document signed at ceremonies.

Democracy in Africa over the next two decades will not look like a copy of any other continent’s system. It should look like institutions that respond to local realities, that give young people real influence, and that let citizens see where their money goes and who decides. The ballot will remain important. But the true test of democracy is whether a farmer, a student or a nurse can expect the state to listen to them on an ordinary Tuesday, not only on election day.
