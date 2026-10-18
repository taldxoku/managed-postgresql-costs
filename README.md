# managed postgresql: how it works, what it really costs, and how to pick a provider without overpaying

Managed PostgreSQL means a provider runs the unforgiving parts of Postgres for you — backups, failover, patching, monitoring — while you keep the part that's actually yours: the schema and the queries. That's the whole deal in one sentence.

Everything else in this article is about how differently each provider packages that deal. The monthly number on the front of a pricing page is rarely the full bill: storage gets billed separately in several services, replica nodes multiply compute costs, and backup retention defaults are shorter than most people assume. We'll walk through all of it with real, published numbers, then match provider types to situations.

> **In a hurry?** For most self-serve teams, Vultr's Managed Databases for PostgreSQL is the pragmatic default: pick a size and a region, deploy from the console in minutes, pay as you go — no sales call, no migration project. 👉 [Check Vultr's managed PostgreSQL plans and current prices](https://www.vultr.com/products/managed-databases/?ref=7268283). If you need French sovereign hosting, or you'd rather have an ops team run the database for you, the comparison further down covers those cases too.

## What "managed PostgreSQL" actually has to include

Six things separate a real managed service from a Postgres container someone else restarts for you: automated backups with point-in-time recovery, high availability with automatic failover, connection pooling, monitoring and metrics, encryption in transit and at rest, and current PostgreSQL versions.

Backups are table stakes. Point-in-time recovery (PITR) is the feature that actually saves you at 2 a.m., because it restores the database to a specific moment using continuously archived WAL logs rather than just the last nightly snapshot. Cloud Temple, for example, archives WALs to object storage for exactly this purpose. One caveat worth burning into memory: PITR is only as deep as the retention window behind it, and that window is shorter than most buyers expect (more below).

Versions matter more than marketing pages suggest. PostgreSQL 13 hit end of life in November 2025, according to Clouddigit's version table. In 2026, any provider still selling 13 as a "supported" version is telling you how seriously they take patching. Current sensible targets are 16 and 17 — Cloud Temple runs both, with EOL dates of 2028 and 2029 — and Temps already ships PostgreSQL 18 as its default image.

Connection pooling is the item people skip and then regret. PostgreSQL spawns a process per connection, so a few hundred app connections will eat memory fast. That's why Cloud Temple calls PgBouncer essential and runs it active/active, and why Fly.io routes every attached application through a pooled endpoint automatically. If a provider doesn't offer pooling at all, the "unlimited connections" on your app framework is a trap.

Failover and encryption round out the list. Two of the providers researched here publish their failover times — both under 30 seconds — but "automatic failover" still isn't zero downtime, which we'll unpack shortly. On encryption, AES-256 at rest and TLS 1.3 in transit (Cloud Temple's published baseline) should be treated as a minimum question to ask, not a premium feature.

## Vultr Managed Databases for PostgreSQL: the decisions you're making

Vultr runs managed PostgreSQL as part of its managed database family — a lineup that also covers other engines such as MySQL and Redis. The workflow is deliberately boring: pick a plan size and a region from the console (or automate it through the API, CLI, or Terraform), and the database bills pay-as-you-go alongside the rest of your Vultr infrastructure. Entry pricing has sat around $15/month since the service launched, though Vultr revises sizes and prices regularly — the pricing page is the source of truth here, not this article or any other blog.

What that setup buys you in practice:

- **One bill and one private network** for your app and its database, instead of gluing two vendors together.
- **Pay-as-you-go billing**, so a staging database you delete after three days costs three days, not a month.
- **Regions spread across North America, Europe, Asia-Pacific and beyond**, which means you can usually put the database in the same data center as the app and stop paying the latency tax.

Rather than paste a plan grid that goes stale the moment Vultr adds a size or a region, here's the choice you're actually making — every current number is one click away:

| What you choose | Typical options | Who should think about it | Check it live |
| --- | --- | --- | --- |
| Node size | Small single-node instances up through larger multi-vCPU/memory plans | Start small; most databases grow slower than traffic dashboards suggest | [See current managed PostgreSQL sizes and prices](https://www.vultr.com/pricing/?ref=7268283) |
| Availability | Single node vs. a high-availability deployment with automatic failover | Production with paying users → HA; dev and staging → single node is fine | [Compare plans on the managed PostgreSQL page](https://www.vultr.com/products/managed-databases/?ref=7268283) |
| Read replicas | Add read-only nodes for read-heavy traffic | Reporting, analytics, dashboards hammering the same cluster | [Check plan and replica options](https://www.vultr.com/products/managed-databases/?ref=7268283) |
| Region | Vultr's globally distributed data centers | Latency between app and database shows up in every request | [Browse regions and plans](https://www.vultr.com/products/managed-databases/?ref=7268283) |
| Billing | Pay-as-you-go, billed like the rest of Vultr's resources | Short-lived test databases, event-driven workloads | [See pricing details](https://www.vultr.com/pricing/?ref=7268283) |

One honest note: I'm not quoting per-plan prices I can't verify against Vultr's live page. Managed database grids change too often, and year-old price screenshots are how most reviews end up wrong. The two links above will always beat any static table.

## The costs that hide behind the sticker price

The number on the front page is usually compute only. Three providers illustrate what gets added on top.

Fly.io bills storage separately: $0.28 per provisioned GB per month, on top of whatever plan you pick, and clusters start at 10 GB — so roughly $2.80/month of storage before you even look at plan pricing. The upside is that storage auto-grows with your data up to 1 TB and is replicated across all cluster nodes, so there's nothing to pre-provision. The downside is that your real monthly cost is plan + storage, always.

Clouddigit splits the bill four ways: hourly compute for every node (writer, standbys, replicas), storage per GiB-month, a separately priced provisioned-IOPS knob on block storage, and backups billed at object-storage rates once you exceed the default retention. That default is 7 days, configurable up to 35 — and the PITR window matches snapshot retention, so extending history means extending the bill.

Canonical is the most candid about it: their management fee explicitly does not include infrastructure. Their pitch — total cost up to five times lower than public cloud "when fully costed" — only holds up if you actually cost everything, which is precisely the exercise most teams skip when comparing a $15 database to a $15 database.

The counter-example is Temps, which runs PostgreSQL 18 with WAL-G on your own server and charges no per-GB fees at all. The tradeoffs are real, though: you own and babysit the server, SSL is off by default because traffic stays on the internal network, and if you bring a custom image instead of the managed one you lose PITR and get snapshot backups only.

The practical takeaway: before comparing two providers, compute (compute price × node count) + storage + IOPS if metered + backups beyond retention. The ranking often flips once the full bill is on the table.

## "High availability," decoded: RPO, RTO, and the 30-second truth

High availability means a standby takes over automatically when the primary dies. It does not mean zero downtime. Clouddigit documents failover at roughly 15–30 seconds of downtime; Cloud Temple guarantees failover in under 30 seconds. Either way, your application needs retry logic on database connections regardless of which provider you choose — treating the database connection as permanent is how apps turn a 20-second blip into an outage.

The distinction that actually matters is synchronous versus asynchronous replication:

- **Async standby**: the primary acknowledges a write before the standby has it. Fast, but a primary crash can lose the last few transactions.
- **Sync standby**: the primary waits for the standby to confirm before acknowledging. That's RPO = 0 — no committed transaction lost. Cloud Temple ships it via synchronous replication in production, Clouddigit makes "primary + sync standby" its default production topology, and both providers place the sync replica in a different availability zone.

Sync replication costs real money. Clouddigit's own topology table prices primary + sync replica at about 2.1× a single instance, with each async read replica adding another 1×. Cloud Temple goes further with one primary plus two standbys, one per AZ. High availability without sync protects uptime, not the last committed writes — worth knowing before you assume "HA" on the pricing page means "nothing can be lost."

A workable rule: payment ledgers, billing data, and anything an auditor would ask about justify synchronous replication. Content sites, internal tools, and most SaaS workloads run fine on async HA, and the 2.1× premium buys them little.

## Managed PostgreSQL providers compared

The services above represent the main delivery models on the market right now. Here they are side by side — information only, since the right pick depends entirely on your constraints:

| Provider | Delivery model | Standout | Watch out for |
| --- | --- | --- | --- |
| Vultr | Self-serve DBaaS on Vultr's cloud; console, API, CLI, Terraform; pay-as-you-go | One bill and private network for app + database; hourly billing; broad region coverage | Sizes and prices get revised; verify on the live page |
| Fly.io | Self-serve DBaaS tied to Fly apps; `fly mpg create` and attach sets a pooled `DATABASE_URL` | PgBouncer on every plan; PostGIS and pgvector in the extension catalog; 13 regions | Storage billed at $0.28/provisioned GB/month on top of the plan; clusters start at 10 GB |
| Cloud Temple | Sovereign managed PostgreSQL on SecNumCloud 3.2; hosting 100% in France; dedicated clusters per customer | RPO = 0 via synchronous replication; failover under 30 s via CloudNativePG; HDS and ISO 27001 certified; AES-256 at rest, TLS 1.3; no public IP on the database | France-only hosting; access only via private network (VPN or interconnect) |
| Clouddigit | Classic cloud DBaaS with four topologies, from single instance to cross-region replicas | 99.95% monthly uptime SLA on HA clusters; compute up to 96 vCPU / 768 GiB; separate writer/reader endpoints | Storage and provisioned IOPS billed separately; snapshots kept 7 days by default |
| Temps | Self-hosted container on your own server; dashboard, CLI, or REST API | PostgreSQL 18 with WAL-G point-in-time recovery and no per-GB fees; AI coding agents can drive the CLI | SSL off by default; custom images get snapshot backups only, no PITR |
| Canonical | Human-operated fleet service on your cloud tenancy or data center | 24/7 ops staff; ISO 27001 / SOC-2 Type II audited; optional handover back to your team | Management fee excludes infrastructure costs; engagement starts with a workshop, not a signup form |

Notice how different the "same product" is across these rows. Three are self-serve platforms you can provision tonight; one is sovereign infrastructure for compliance-bound workloads; one runs on hardware you already own; one is people, not panels. The feature checklists from earlier — PITR, failover, pooling — apply to all of them, but the buying process and the bill look nothing alike.

## Which one fits your situation

Pick by constraint, not by feature-count comparison:

- **Already running your app on Fly.io?** Two commands get you a cluster, and attaching sets a pooled `DATABASE_URL` on your app automatically. Don't migrate platforms just for the database.
- **EU data residency, health data, or public-sector compliance?** Cloud Temple's SecNumCloud setup — France-only hosting, HDS and ISO 27001, RPO = 0, databases with no public IP — exists precisely for this. It's overkill for everyone else.
- **Own a server and hate per-GB fees?** Temps gives you PITR on your own hardware. Accept the SSL-off-by-default and self-hosted tradeoffs with eyes open.
- **Enterprise fleet, existing DBA team, want offloading rather than outsourcing?** Canonical manages day-2 operations on your own tenancy and can hand operations back to you later — a genuinely unusual option worth knowing about.
- **Everyone else** — solo developers, startups, most production SaaS — should default to self-serve DBaaS: fast to provision, no procurement process, easy to resize. That's the category Vultr plays in, and it's where I'd start for a typical app. 👉 [Browse Vultr's managed PostgreSQL plans and regions](https://www.vultr.com/products/managed-databases/?ref=7268283) and see whether a size-region combo matches your app's footprint.

## FAQ: the questions people actually ask

**Does automatic failover mean zero downtime?**
No. The published numbers are roughly 15–30 seconds (Clouddigit) and under 30 seconds (Cloud Temple). Plan for connection retries in your application; anything promising more is overselling.

**What is RPO = 0, and do I need it?**
It means no committed transaction is lost during failover, and it requires synchronous replication — which Clouddigit prices at about 2.1× a single instance. You need it for payments and anything with legal or audit consequences. Most workloads don't.

**How long are my backups actually kept?**
Shorter than you'd hope by default. Clouddigit keeps snapshots 7 days (configurable to 35), and the PITR window matches retention. Whatever provider you pick, find the retention number before assuming "we have backups" means "we have history."

**Why does every provider push connection pooling?**
PostgreSQL runs a process per connection, so connection counts translate directly into memory pressure. PgBouncer multiplexes many app connections onto fewer database connections — Fly.io puts it on every plan and routes attached apps through it by default.

**Is managed PostgreSQL worth it for a small side project?**
Honestly, sometimes not. A hobby site on a cheap VPS with scheduled dumps is a perfectly rational setup. Managed starts earning its price the moment a failed disk at 2 a.m. would cost you real data — or when you'd rather never learn what a replication slot is.

## Before you hit deploy

Whatever provider you land on, confirm five things from their current docs — not from a blog, this one included:

1. Which PostgreSQL major versions they support today (nothing at or past EOL; ideally 16 or 17).
2. Backup retention and the PITR window, in days.
3. Whether storage and IOPS are billed separately from compute.
4. Failover behavior: RTO, and whether replication is synchronous or asynchronous.
5. Which regions are actually available, and whether the database sits on a private network.

If those check out and the self-serve route sounds right for your situation, 👉 [open Vultr's managed PostgreSQL page and compare current plans](https://bit.ly/VulTR). Ten minutes of checking beats a migration later.
