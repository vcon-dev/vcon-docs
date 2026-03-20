# VCONIC TADHack 2026: Hackathon Review

> **Event:** [VCONIC TADHack](https://blog.tadhack.com/2025/12/19/vconic-tadhack/) | March 7-8, 2026 | Virtual | [The Hacks blog post](https://blog.tadhack.com/2026/03/08/vconic-tadhack-the-hacks/)
>
> **Core technology:** [vCon](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-container/) (IETF standard) | [vCon MCP Server](https://github.com/vcon-dev/vcon-mcp) ([docs](https://www.conserver.io/mcp-server/what-is-the-vcon-mcp-server) | [live](https://mcp.conserver.io/))

### Overview

The VCONIC TADHack 2026 (March 7-8) drew 15 submissions from developers across four continents — Nigeria, Kenya, the United States, and Brazil — tackling problems in emergency response, financial services, education, compliance, supply chains, animal rescue, and personal memory. Every team built on the [vCon standard](https://datatracker.ietf.org/group/vcon/about/). Most connected to the [vCon MCP Server](https://github.com/vcon-dev/vcon-mcp). Several pushed the boundaries of what vCon can represent, demonstrating that a conversation container originally designed for telephony is becoming a general-purpose intelligence format.

This review examines each submission individually, then steps back to assess the hackathon as a whole: what it revealed about vCon's capabilities, where teams innovated, and what patterns emerged.

***

### Individual Reviews

#### 1. 911 First Response — Shouvik Sharma & Ankita Bhanushali

> [Video](https://www.youtube.com/watch?v=HRKcU5U-gzI) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/911-first-response.vcon.json) | Transcript

**The idea:** Turn a live 911 call into a structured dispatch action in real time. The moment a call comes in, it becomes a vCon. AI extracts location, emergency type, severity, and incident classification (medical, fire, burglary, accident), then triggers the appropriate dispatch workflow — choosing the right service, finding the nearest unit, and dispatching immediately.

**What they built:** Two working demos. First, a programmatic MCP consumption script that ingests vCon files from a `911_calls` folder, performs health checks, lists stored vCons, fetches by UUID, and prints the structured dispatch action (e.g., "Dispatch the nearest police unit to 5520 Cedar Lane. Reason: burglary in progress"). Second, a Streamlit-based Conversation Viewer with a table of all vCons, per-conversation detail views, audio playback, transcript display, and raw JSON inspection.

**vCon capabilities demonstrated:**

* vCon as the canonical record for an emergency call, from raw audio through AI-derived dispatch instructions
* The analysis array carrying transcript, summary, speaker diarization, and a structured action — all attached to the same vCon
* MCP server as the query layer for both scripts and AI assistants

**What makes it unique:** This is the most operationally immediate submission. It's not a dashboard or analytics tool — it's a real-time pipeline where seconds matter. The structured action output (service type, nearest unit, dispatch reason) shows vCon moving beyond record-keeping into decision-triggering.

***

#### 2. Apparitions — David Sikes & Jared Ashcraft

> [Video](https://www.youtube.com/watch?v=kYvfCVWyz1M) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/apparitions.vcon.json) | Transcript

**The idea:** A location-based augmented reality framework where each point of interest is a vCon. Users walk through physical spaces — museums, historical sites, mystery stories — and audio/media triggers as they approach GPS coordinates stored in vCon attachments.

**What they built:** A working mobile prototype demonstrating a mystery story scenario. As the user physically approaches real-world objects (a garage, a street), audio plays with volume that increases with proximity. The entire scenario is a list of vCons — swapping vCons changes events. Text scrolls from the dialog body; latitude, longitude, and audio file paths are stored as typed attachments.

**vCon capabilities demonstrated:**

* vCon as a unit of location-based experience, not a conversation record
* Attachments with type identifiers for coordinates, media paths, and metadata
* The list-of-vCons pattern for sequencing events
* Shareability — because each scenario is just a list of vCon JSON files, scenarios transfer trivially between devices

**What makes it unique:** This is the most creative reinterpretation of what a vCon can be. The team treated vCon not as a conversation format but as a general-purpose container for location-tagged, sequenceable media events. Their future plans (browsing scenarios by location and category, MCP-powered voice synthesis for user-generated content) point toward a vCon-native content distribution platform. No other submission reimagined the standard this boldly.

***

#### 3. Budget Yangu — Elvis Ogunga

> [Video](https://www.youtube.com/watch?v=fxBPIaZPSTk) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/budget-yangu.vcon.json) | Transcript

**The idea:** "My Budget" in Swahili — an AI-powered personal finance assistant where every conversation between the user and the AI becomes a vCon record that can be analyzed for financial insights.

**What they built:** A Laravel-backed application where users manually input financial records and then use an AI agent to analyze their data. The agent retrieves user data via authenticated sessions, analyzes it, and every interaction is captured as a vCon with parties, dialog, and analysis. The app sends vCons to the MCP server running on a local node. A dashboard shows total vCon count, dialog types, conversation subjects, and data health metrics.

**vCon capabilities demonstrated:**

* vCon as the persistence layer for human-AI financial conversations
* The full vCon creation pipeline in Laravel: initializing the vCon service, installing dialog, handling party names and roles
* MCP server integration for storage and retrieval
* Analysis array for AI-derived financial insights

**What makes it unique:** This is a solo developer building a complete financial product with vCon at the center. The insight that every financial AI conversation is worth preserving as a structured record — not just for the user, but for improving the AI and the business — is commercially astute.

***

#### 4. ConsentMate — Abdurrahman Umar & Berlu (Team Skyline Coders)

> [Video](https://www.youtube.com/watch?v=WUdpfmbMAAQ) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/main/vcons/consentmate.vcon.json) | Transcript

**The idea:** An AI-powered GDPR compliance dashboard that tracks customer consent across all recorded conversations, provides compliance scoring, and delivers daily briefings.

**What they built:** A Next.js dashboard with four core views: (1) a Daily Briefing that delivers a personalized compliance update each morning with a consent compliance score, (2) a Transcript tab previewing all vCon calls, (3) an Analysis tab breaking down each conversation for sentiment, compliance score, and key highlights, and (4) a Consent tab tracking active, expired, and expiring consent in one place. Backend AI generates real-time compliance messages and analyzes transcripts.

**vCon capabilities demonstrated:**

* vCon as the source of truth for consent state — every call is a vCon, and consent status is derived from the vCon record
* AI analysis of vCon transcripts for compliance scoring
* The analysis array carrying sentiment, compliance scores, and highlights

**What makes it unique:** This is the submission most directly aligned with the [vCon Lawful Basis extension](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/). The daily compliance briefing with a numerical score is a compelling UX pattern — it turns a legal obligation into something a business owner can glance at over coffee.

***

#### 5. ConvoLens — Josphat Mwangi

> [Video](https://www.youtube.com/watch?v=lX8pXsz-47c) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/convolens.vcon.json) | Transcript

**The idea:** A customer conversation intelligence platform for banking and financial services that ingests conversations from any channel, converts them to vCon, analyzes them with AI, and provides a live dashboard.

**What they built:** The most feature-rich analytics platform in the hackathon. ConvoLens does three things: (1) ingests conversations from WhatsApp exports, call transcripts, tweet threads, or any text source and converts them to IETF-standard vCon records; (2) analyzes every conversation using Claude for sentiment, issue classification, complaint flagging, and action recommendations — all stored in the vCon analysis array; (3) provides a live dashboard with compliance risk alerts, customer journey timelines, and an "Ask Claude" chat where teams can type natural language questions ("What are the top complaints this week?") and get instant answers. Also runs the official vCon MCP server alongside the platform, letting Claude Desktop query vCon records through MCP natively.

**vCon capabilities demonstrated:**

* Multi-channel ingestion normalized to vCon: WhatsApp, Twitter/X, Facebook, Instagram, call center, email, CRM
* The vCon analysis array as a rich structured output: sentiment, issue classification, compliance flags, action recommendations
* MCP server running alongside the app for native AI assistant access
* Pure vCon JSON API — any external tool that understands vCon can consume ConvoLens directly
* Open source (GitHub), built on Next.js, Supabase, and Anthropic Claude

**What makes it unique:** The banking/financial services focus is commercially sharp — this is a heavily regulated industry where conversation compliance is mandatory, not optional. The "Ask Claude" feature querying vCon data through MCP is a clean demonstration of the AI-native future of conversation intelligence.

***

#### 6. ConvoSense — Collins Omondi

> [Video](https://www.youtube.com/watch?v=XNn8HUwuzec) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/convosense.vcon.json) | Transcript

**The idea:** A platform that solves the three core problems of SMB customer support: data scattered across platforms, manual effort to derive insights, and vendor lock-in.

**What they built:** A dashboard that connects to VAPI (voice AI) and Intercom via internal plugins using API keys. Once connected, it pulls all conversations, then connects to the vCon MCP server. On connection, conversations are automatically synchronized into vCon format. An AI assistant with built-in visualization tools and all vCon MCP server tools can answer questions a support team would ask, providing analysis in seconds rather than hours.

**vCon capabilities demonstrated:**

* Plugin-based ingestion from VAPI and Intercom, converting proprietary formats to vCon
* The MCP server's `create_vcon` tool used programmatically for format conversion
* vCon as the anti-vendor-lock-in layer — businesses connect their existing tools without migrating
* AI assistant querying both platforms simultaneously through vCon tools

**What makes it unique:** The vendor lock-in angle is the most strategically important idea in the hackathon. By positioning vCon as the interoperability layer between VAPI, Intercom, and future platforms, ConvoSense demonstrates that vCon can be the "PDF of conversations" — a format that liberates data from proprietary silos.

***

#### 7. Life Canvas — Sabrina (Valencia College)

> [Video](https://www.youtube.com/watch?v=C0UhGJR72pk) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/life-canvas.vcon.json) | Transcript

**The idea:** A personal life intelligence system that turns everyday moments — journal entries, photos, voice notes, moods — into structured vCon records that become searchable, analyzable life data.

**What they built:** A full web application with: a Dashboard (journal entries, streaks, highlights, recent memories), a Journal (capture reflections via writing, photos, voice notes, or mood selection), Buckets (life categories: family, career, health, personal growth), a Calendar (revisit memories by date), Analytics (emotional trends, common topics, most-mentioned people), Memory Search, a Timeline (visual stories of personal growth), and Year in Review (annual milestones and insights). Behind the scenes, every entry becomes a structured vCon record, and AI detects meaningful patterns.

**vCon capabilities demonstrated:**

* vCon applied to personal life data, not business conversations
* Multi-modal capture (text, photo, voice, mood) all structured into the vCon format
* The analysis array used for emotional pattern detection and life insights
* Privacy-first design with user-controlled access through the MCP server

**What makes it unique:** This is the most emotionally resonant submission. The opening pitch — "How much of your life do you actually remember?" — reframes vCon from a business tool to a personal one. The idea that your life is a series of conversations worth structuring is philosophically aligned with vCon's vision, and the execution (buckets, analytics, year-in-review) shows a thoughtful product designer at work.

***

#### 8. Ollie — Anna Correa

> [Video](https://www.youtube.com/watch?v=guBpk1E9yZA) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/ollie.vcon.json) | Transcript

**The idea:** An AI platform that crawls Reddit for lost pet reports, converts them to vCon records, and provides actionable tools for animal rescue coordination.

**What they built:** A Next.js application deployed on Vercel that: (1) curls Reddit daily, finding posts about lost or stray animals; (2) converts each conversation into a vCon and stores it in MongoDB; (3) runs AI enrichment (summary, sentiment, location extraction); (4) provides a Live Feed, Map view (with animal locations plotted on a map), and Help Now page (filter by support/foster requests); (5) includes a chat assistant on every page ("Are there any dogs near Austin?") that queries the vCon data; (6) includes a custom MCP server built from scratch, designed to layer on top of the existing vCon MCP server.

**vCon capabilities demonstrated:**

* Automated web scraping → vCon conversion pipeline
* vCon as the normalization layer for unstructured Reddit data
* Custom MCP server tools: `search_animal_conversations`, `find_lost_pets`, `find_ways_to_help`
* MongoDB as the vCon store with AI enrichment
* Chat assistant querying vCon data across all pages

**What makes it unique:** This is the most heartwarming submission and demonstrates the most creative data source. Nobody else scraped external web content into vCons. The custom MCP server built from scratch, designed to compose with the existing vCon MCP server, shows strong architectural thinking. The constraint-driven development (Hugging Face free tier for AI, Vercel for deployment) makes this a genuine scrappy hack.

***

#### 9. OnePrice Sales Memory — Joan Ovalles Rosario (Valencia College)

> [Video](https://www.youtube.com/watch?v=XCIGV91PZn4) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/oneprice-sales-memory.vcon.json) | Transcript

**The idea:** End information decay in auto sales. Consolidate multi-channel customer interactions into a structured vCon-based memory system so salespeople never lose context on a deal.

**What they built:** A working sales workflow demo: create a lead (Juan Soto, interested in Ford F-150, budget $1,000/month), run AI tagging analysis (warm lead, payment-sensitive, shopping competitors, price shock, needs manager follow-up), search across all vCons for hot leads matching criteria ("search my vCons for any hot leads who are highly sensitive to monthly payments"), and generate personalized follow-up scripts built directly from the vCon profile. The system uses Docker, the MCP server, OpenAI for analysis, and Supabase for storage.

**vCon capabilities demonstrated:**

* vCon as CRM memory — each customer interaction is a persistent, searchable record
* AI-powered tagging stored in the vCon analysis array
* MCP-based search across the vCon corpus ("find payment-sensitive leads")
* Follow-up script generation grounded in vCon data, not generic templates
* Multi-location dealership support

**What makes it unique:** The "information decay" framing is brilliant. Every salesperson knows the pain of losing context on a deal. The demo flow — create lead, tag, search, generate personalized follow-up — is a complete sales workflow, not just a proof of concept. Joan's comment that she "can actually use it in my everyday job" is the highest compliment a hackathon project can receive.

***

#### 10. Patanisha — Charles Wachira

> [Video](https://www.youtube.com/watch?v=bl_YNeu3MCM) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/patanisha.vcon.json) | Transcript

**The idea:** "Patanisha" is Swahili for "unifying." The first support platform built on the vCon standard — every customer interaction (phone, SMS, email, chat) automatically becomes a vCon, unified into a single timeline.

**What they built:** A live demo using Africa's Talking APIs for voice and SMS in the Kenyan market. The dashboard shows cases on the left, loaded from TADHack 2025 data as proof of scale. A simulated customer interaction demonstrates a voice call, SMS, and email all unified into one case timeline. Agents can claim cases, send SMS responses via Africa's Talking, and mark cases as resolved. All data flows to a Supabase backend. The team used Conserver.io for vCon storage and AI processing.

**vCon capabilities demonstrated:**

* "Phone call, that's a vCon. SMS, that's a vCon. Email, that's a vCon." — the clearest articulation of vCon's unification promise
* [Africa's Talking](https://africastalking.com/) API integration for real-world telephony in the Kenyan market
* [Conserver.io](https://www.conserver.io/) as the vCon backend
* Supabase integration for the dashboard
* Ready for the [vCon App Store](mailto:apply@mindmaking.com) (MindMaking)

**What makes it unique:** Patanisha is the submission closest to production. It uses real telephony APIs (Africa's Talking) in a real market (Kenya), handles real communication channels (voice + SMS + email), and is deployed at a real URL (patanisha.ruita.co.ke). The "3 minutes not 30" support resolution claim, while aspirational, is grounded in a working demo. The Swahili naming and Kenyan market focus represent the growing East African developer community in vCon.

***

#### 11. TraceConnect — Jevans Otieno

> [Video](https://www.youtube.com/watch?v=POUeuloABtU) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/traceconnect.vcon.json) | Transcript

**The idea:** Enterprise-grade global visibility for distributed teams. HQ gets real-time intelligence on how branches in different countries handle issues and complaints, powered by vCon and Claude Desktop via MCP.

**What they built:** A Radar Dashboard showing activity across branches (Nairobi, Lagos, New York), with metrics like open issues, resolved issues, and average resolution time per branch. An "Ask TraceConnect" sidebar lets HQ query across all branches via natural language ("Walk me through the enterprise outage escalation in Lagos — show me the full conversation trail"). The system connects vCon to Claude Desktop via MCP for direct querying.

**vCon capabilities demonstrated:**

* vCon as the standardized conversation format across global branches
* Cross-region intelligence — querying vCons from Nairobi, Lagos, and New York through a single interface
* Claude Desktop connected to vCon via MCP for natural language queries
* Conversation trail reconstruction from vCon records
* Branch-level performance metrics derived from vCon data

**What makes it unique:** This is the only submission targeting enterprise HQ-to-branch visibility. The use case — "something might be happening in real-time but HQ only finds out at the weekly meeting" — is immediately recognizable to anyone who's managed distributed operations. The cross-border compliance angle (conversations in different regulatory jurisdictions) adds a sophisticated dimension.

***

#### 12. vChat — Ahmadu Suleiman

> [Video](https://www.youtube.com/watch?v=7_PImyiISn8) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/vchat.vcon.json) | Transcript

**The idea:** Bring vCon to everyone's phone. A general-purpose messaging app where you chat normally and vCon structures everything in the background — who said what, in what role, when, and what was agreed.

**What they built:** A mobile-optimized prototype with two modes: (1) start a new conversation in-app with role assignment, structured in real time; (2) import existing WhatsApp, SMS, or email threads, assign roles, and convert them into the same structured vCon format. Three demo vCons show use cases in land disputes, healthcare, and education. MCP server attachment enables AI capabilities directly on conversations — insights, semantic search, and more. Supports vCon export/download for full data portability and optional redaction for privacy.

**vCon capabilities demonstrated:**

* Real-time vCon creation during live chat
* Import and conversion of WhatsApp/SMS/email threads to vCon
* Role-based party attribution (critical for mediation, legal, and dispute contexts)
* MCP integration for AI-powered conversation analysis
* vCon export for institutional sharing (banks, lawyers, government offices)
* Selective redaction before sharing

**What makes it unique:** vChat's positioning as "vCon for everyone" is the most ambitious consumer play. The specific use cases — community mediation, tenant-landlord disputes, delivery confirmations — demonstrate that structured conversation records aren't just for enterprises. The import-from-WhatsApp feature is strategically important: it means existing conversations, not just future ones, can enter the vCon ecosystem. The emphasis on verifiability ("signed and independently verifiable") positions vCon as a trust layer for informal agreements.

***

#### 13. vCohort — Ziyad Shuaibu, Abdulalim Ladan & Mubarak Ibrahim

> [Video](https://www.youtube.com/watch?v=j6f88p8DIZU) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/vcohort.vcon.json) | Transcript

**The idea:** A conversation intelligence layer for Nigeria's booming cohort-based learning market ($1.5B in 2025). Learning sessions on Zoom and Google Meet become vCon records, generating engagement analytics, confusion signals, and intervention recommendations.

**What they built:** A full platform with: organization management, cohort creation, session upload/live recording ingestion. Per-session analytics include duration, speaking time vs. silence, engagement scores, questions asked, key moments, action items, and per-participant speaking time breakdowns. A transcript preview is available for each session, and vCon files can be downloaded in JSON format. The team built integrations for Zoom, Google Meet, and planned Trello/Slack/Google Calendar integration.

**vCon capabilities demonstrated:**

* vCon as the intelligence layer for educational conversations
* Bot-joins-meeting pattern for live vCon capture
* Session upload for asynchronous recording ingestion
* Rich analysis array: engagement scores, confusion signals, speaking time distribution, action items
* vCon download/export for portability
* Organization → Cohort → Session hierarchy modeled with vCon

**What makes it unique:** The market insight is sharp — Nigeria's education sector is growing fast, and the gap between "we had a Zoom call" and "we understand what happened in that call" is enormous. The "at-risk participants" detection (learners who aren't engaging) demonstrates vCon enabling proactive intervention, not just passive recording. This team clearly listened to Thomas's comment about enterprise tools and found the intersection with their market.

***

#### 14. vCon Example App — Muntaser Syed (Submission 1)

> [Video](https://www.youtube.com/watch?v=msvvAcZFEng) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/vcon-example-app.vcon.json) | Transcript | [Pull request on vcon-mcp](https://github.com/vcon-dev/vcon-mcp)

**The idea:** A reference implementation demonstrating vCon enhancements: JSON-LD extensions for richer metadata, cryptographic signing for integrity verification, and MongoDB integration for analytics and vector search.

**What they built:** An interactive web app where users can: load sample vCons, add JSON-LD context and extensions (external types, confidence labels), enrich analysis with extended metadata, cryptographically sign a vCon, verify integrity (tampering detection demonstrated by modifying a field and showing the signature check fail), and perform MongoDB-backed vector search across vCon embeddings.

**vCon capabilities demonstrated:**

* JSON-LD and JSON-LD-EX extensions for semantic enrichment
* Cryptographic signing and integrity verification ([JWS](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-container/))
* Tamper detection — changing a single field breaks the signature
* MongoDB vector search over vCon embeddings
* Confidence labels for agentic AI trust decisions

**What makes it unique:** This is the most technically deep submission. The cryptographic signing demo — sign, tamper, verify failure, restore — is the clearest demonstration of vCon's integrity guarantees in the entire hackathon. The JSON-LD extensions and confidence labels point toward a future where AI agents can assess trust in vCon data programmatically. Submitted as an actual pull request to the vCon MCP repository.

***

#### 15. vCon Intelligence Platform — Muntaser Syed (Submission 2)

> [Video](https://www.youtube.com/watch?v=h4ehOvFSqLw) | [vCon](https://github.com/vcon-dev/vcon-the-hacks/blob/84b3dcf5d4589653eaaba75d32c0d5e3857cf343/vcons/vcon-intelligence-platform.vcon.json) | Transcript

**The idea:** A comprehensive multi-backend intelligence platform that ingests conversations from every source, enriches them, stores them in multiple database backends, and provides graph-based visualization and AI-powered querying.

**What they built:** A full platform supporting: direct input, SIP REC recordings, Microsoft Teams transcripts, WhatsApp chat exports, and direct audio upload with automatic Whisper transcription. Multiple storage backends: MongoDB, Neo4j (graph database), ChromaDB (vector store), and Supabase. MQTT-based event system for real-time ingestion notifications. A Neo4j-powered graph visualization showing agents, customers, conversations, and topic nodes with their relationships. A JSON-LD-EX inspector for enhanced vCon viewing. AI chat interface for querying across all ingested vCons ("What was the most frequent issue?" → "Billing and account issues, including duplicate charges"). Sentiment analysis timeline across all conversations.

**vCon capabilities demonstrated:**

* The widest ingestion surface: direct input, SIP REC, Teams, WhatsApp, audio upload
* MQTT integration for real-time event-driven architecture (aligns with UNS/manufacturing use cases from training sessions)
* Neo4j graph visualization of conversation networks
* Multi-backend storage: MongoDB, Neo4j, ChromaDB, Supabase
* JSON-LD-EX enhanced vCon format with extended analytics fields
* Whisper-based automatic transcription on audio upload
* AI-powered cross-corpus querying

**What makes it unique:** This is the most architecturally ambitious submission. The multi-backend approach (relational + graph + vector + document) acknowledges that different query patterns need different storage engines. The MQTT integration directly addresses the UNS (Unified Namespace) pattern discussed in [Matthew Smith's training session](https://www.youtube.com/watch?v=QNLUXTr3JI4) ([starter kit](https://github.com/fieldcloud/vcons-uns-starter-kit)), showing that vCon can participate in industrial event-driven architectures. The SIP REC ingestion connects vCon to traditional telephony infrastructure.

***

### Themes and Patterns

#### Geographic Reach

The hackathon drew developers from at least four countries across three continents:

* **Kenya:** Patanisha (Charles Wachira), ConvoLens (Josphat Mwangi), ConvoSense (Collins Omondi), TraceConnect (Jevans Otieno), Budget Yangu (Elvis Ogunga)
* **Nigeria:** vCohort (Ziyad Shuaibu et al.), vChat (Ahmadu Suleiman), ConsentMate (Abdurrahman Umar & Berlu)
* **United States:** 911 First Response (Shouvik Sharma & Ankita Bhanushali), Apparitions (David Sikes & Jared Ashcraft), Life Canvas (Sabrina), OnePrice Sales Memory (Joan Ovalles Rosario), vCon Example App & Intelligence Platform (Muntaser Syed)
* **Brazil:** Ollie (Anna Correa)

The strong East African and West African representation is notable. Five of the six Kenyan/Nigerian submissions address local market needs (financial services in Kenya, education in Nigeria, Swahili-named products). This suggests vCon is finding natural traction in markets where conversation data is plentiful but infrastructure for structuring it is scarce.

#### vCon as More Than Telephony

The original vCon use case was capturing phone calls. This hackathon proved the format's generality:

| What vCon represented                      | Submission                 |
| ------------------------------------------ | -------------------------- |
| Emergency 911 calls                        | 911 First Response         |
| Location-based AR experiences              | Apparitions                |
| Human-AI financial conversations           | Budget Yangu               |
| GDPR consent records                       | ConsentMate                |
| Multi-channel banking complaints           | ConvoLens                  |
| Voice AI + Intercom chats                  | ConvoSense                 |
| Personal journal entries                   | Life Canvas                |
| Reddit posts about lost pets               | Ollie                      |
| Auto dealership sales interactions         | OnePrice Sales Memory      |
| Phone + SMS + email support cases          | Patanisha                  |
| Global enterprise branch conversations     | TraceConnect               |
| Community mediation agreements             | vChat                      |
| Educational bootcamp sessions              | vCohort                    |
| Semantically enriched conversation records | vCon Example App           |
| SIP REC + Teams + WhatsApp + audio         | vCon Intelligence Platform |

The most surprising entries — Apparitions (vCon as AR event container), Life Canvas (vCon as life journal record), and Ollie (vCon as Reddit post container) — show that developers intuitively extend the standard beyond its original scope when the format is flexible enough.

#### The MCP Server as Enabler

Nearly every submission connected to the vCon MCP Server, but they used it differently:

* **Query layer for AI assistants:** 911 First Response, ConvoLens, TraceConnect, OnePrice Sales Memory
* **Format conversion gateway:** ConvoSense (VAPI/Intercom → vCon), vCon Intelligence Platform (Teams/WhatsApp/SIP REC → vCon)
* **Storage and retrieval backend:** Budget Yangu, Patanisha, vCohort
* **Custom MCP server built on top:** Ollie (animal-specific tools layered on vCon MCP)

The pattern of "connect the MCP server, then query with natural language" appeared repeatedly, validating the MCP approach as the AI-native interface for conversation data.

#### Unique Ideas Worth Highlighting

1. **Apparitions' vCon-as-content-unit pattern** — Each location event is a vCon; a scenario is a list of vCons. This is the most novel structural use of the format and could inspire a "vCon playlist" pattern for sequential experiences.
2. **ConvoSense's anti-vendor-lock-in positioning** — vCon as the interoperability layer between proprietary platforms. This is strategically important for vCon adoption.
3. **Ollie's web-scraping-to-vCon pipeline** — Proving that vCon can normalize unstructured web content, not just structured API data.
4. **vChat's import-existing-conversations feature** — The insight that vCon's value isn't limited to future conversations; existing WhatsApp/SMS threads can be retroactively structured.
5. **vCon Example App's tamper-detection demo** — The most visceral demonstration of vCon integrity: change one character, signature breaks.
6. **vCohort's "at-risk learner" detection** — Using conversation analysis not just for reporting but for proactive intervention.
7. **OnePrice's "information decay" framing** — Naming the problem that vCon solves in sales contexts. Good framing accelerates adoption.
8. **vCon Intelligence Platform's MQTT integration** — Connecting vCon to industrial event-driven architectures, bridging the gap between conversation data and IoT/manufacturing systems.

#### What the Hackathon Proved About vCon

1. **The format is genuinely general-purpose.** Developers applied it to AR experiences, personal journals, Reddit posts, and emergency dispatch — none of which are traditional "conversations" — and the format held up.
2. **The MCP Server is the right abstraction.** Teams spent their time building applications, not fighting with data ingestion. The MCP server handled the plumbing.
3. **The analysis array is vCon's killer feature for AI.** Nearly every submission used it to store AI-derived insights (sentiment, compliance scores, dispatch actions, engagement metrics) alongside the raw conversation. This "raw data + derived intelligence in one container" pattern is what makes vCon more than just a recording format.
4. **vCon attracts developers who think in systems.** These aren't toy demos. Multiple submissions (ConvoLens, vCon Intelligence Platform, Patanisha, vCohort) are architecturally complete enough to evolve into products.
5. **The East African developer community is a natural fit.** Five Kenyan and three Nigerian submissions, many addressing local market needs with local telephony APIs ([Africa's Talking](https://africastalking.com/)), suggest that vCon has found an enthusiastic early-adopter community in a region where conversation-heavy business processes (customer support, financial services, education) are growing rapidly.

***

### Resources

#### Specifications and Standards

* [vCon Container Format](https://datatracker.ietf.org/doc/draft-ietf-vcon-vcon-container/) — The core IETF standard that every submission built on
* [vCon Working Group](https://datatracker.ietf.org/group/vcon/about/) — IETF working group developing the standard
* [vCon Lawful Basis Extension](https://datatracker.ietf.org/doc/draft-howe-vcon-lawful-basis/) — Consent and legal basis tracking (relevant to ConsentMate, vChat)
* [vCon Lifecycle Management](https://datatracker.ietf.org/doc/draft-howe-vcon-lifecycle/) — SCITT-based transparency services
* [SIP Extension for MCP](https://datatracker.ietf.org/doc/draft-howe-sipcore-mcp-extension/) — SIP protocol extension for MCP discovery (stretch objective for the hackathon)

#### Tools and Platforms

* [vCon MCP Server](https://github.com/vcon-dev/vcon-mcp) — The core MCP server used by most submissions ([docs](https://www.conserver.io/mcp-server/what-is-the-vcon-mcp-server) | [live server](https://mcp.conserver.io/))
* [vCon GitHub Organization](https://github.com/vcon-dev) — All vCon open source projects
* [Conserver.io](https://www.conserver.io/) — VCONIC's vCon platform (used by Patanisha, others)
* [vCon UNS Starter Kit](https://github.com/fieldcloud/vcons-uns-starter-kit) — vCon + Unified Namespace for manufacturing (relevant to vCon Intelligence Platform)
* [Africa's Talking](https://africastalking.com/) — Voice and SMS APIs (used by Patanisha)

#### Training Sessions

These sessions were held in the weeks before the hackathon and informed many submissions:

| Session                                 | Presenter                                                                          | Video                                                |
| --------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------- |
| VCONIC TADHack Background and Resources | Thomas McCarthy-Howe                                                               | [Watch](https://www.youtube.com/watch?v=Tm6EewfOa4M) |
| AI Coding for Beginners                 | Rob Pickering                                                                      | [Watch](https://www.youtube.com/watch?v=byP02fQe7sI) |
| Consent and Lifecycle                   | Thomas McCarthy-Howe                                                               | [Watch](https://www.youtube.com/watch?v=jzOw94GKcbs) |
| Spec-Driven Development                 | Jason Goecke ([sdd.goecke.io](https://sdd.goecke.io/))                             | [Watch](https://www.youtube.com/watch?v=FnuzyBi2ntw) |
| The VCON App Store                      | Audrey Hayn, MindMaking                                                            | [Watch](https://www.youtube.com/watch?v=cwgY3D7DVdY) |
| VCON + UNS for Manufacturing            | Matthew Smith ([starter kit](https://github.com/fieldcloud/vcons-uns-starter-kit)) | [Watch](https://www.youtube.com/watch?v=QNLUXTr3JI4) |

#### Event Links

* [VCONIC TADHack Announcement](https://blog.tadhack.com/2025/12/19/vconic-tadhack/) — Original event page with challenge details
* [VCONIC TADHack — The Hacks](https://blog.tadhack.com/2026/03/08/vconic-tadhack-the-hacks/) — Blog post covering all submissions
* [How to Submit](https://blog.tadhack.com/2026/02/20/how-do-i-submit-my-vconic-hack/) — Submission process
* [TADHack](https://tadhack.com) — Telecom Application Developer Hackathon
