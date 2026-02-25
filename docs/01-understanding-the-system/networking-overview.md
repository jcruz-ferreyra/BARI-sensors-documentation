# Networking Overview

![Functional Architecture](../images/diagrams/functional-architecture.png)

The environmental sensor data pipeline consists of several interacting components: processing (Function Apps), storage (Blob Storage and SQL Database), and display (Public Dashboard). Data flows between these resources either through the public internet or through a closed private network.

## Hybrid Networking Approach

Our infrastructure uses a **hybrid networking model** that combines private internal communication with controlled public access points.

**Private network (internal traffic):**

Once data enters the Azure ecosystem, it flows between resources through a private Virtual Network (VNet) that cannot be accessed from the public internet. In the diagram above, these connections are marked with lock icons (🔒).

- Webhook Receiver → Blob Storage (private)
- Database Writer → Blob Storage (private)
- Database Writer → SQL Database (private)
- API Function → SQL Database (private)

**Public internet (entry and exit points):**

Data enters the system via Particle webhook and exits through the public dashboard or direct database queries.

- Particle sensors → Webhook Receiver (public - must accept external data)
- Public Dashboard → API Function (public - serves public users)
- Your computer → SQL Database (public when temporarily enabled)
- Your computer → Blob Storage (public when temporarily enabled)

## Why This Approach?

This hybrid model balances three competing priorities:

1. **Security:** Internal data processing stays isolated from internet threats
2. **Functionality:** Can receive external sensor data from Particle.io
3. **Cost:** Avoids expensive gateway infrastructure ($160/month savings)

For the complete rationale and alternative approaches considered, see [Design Decisions](design-decisions.md#networking-architecture).

## Security Benefits

Because internal resources communicate through the private network, we can disable public internet access for most resources while the pipeline continues operating normally.

**Resources that must remain publicly accessible:**

- **Webhook Receiver Function:** Must accept data from Particle.io sensors
- **API Function:** Must serve data to the public-facing dashboard

**Additional security layer:**

Even if credentials for Blob Storage or SQL Database are leaked, attackers from outside the Azure network cannot connect to these resources when public access is disabled. This provides protection beyond credential security alone.

## What This Means for You

**Researchers querying data:**

While public access for SQL Database is disabled (default state), you cannot connect from your personal computer - neither through GUI tools (SSMS, VS Code) nor programmatically (Python scripts).

To query data or perform database maintenance, temporarily enable public access through the Azure Portal for the duration of your work session, then disable it when finished.

See [Connecting to Database](../02-working-with-data/database/connecting.md#network-access) for procedures.

**Developers modifying Function Apps:**

To redeploy function code or add new functions, temporarily enable public access for that specific Function App during deployment.

Note: Only the Database Writer function typically has public access disabled. Webhook Receiver and API functions remain publicly accessible by design (required for external sensor data and dashboard access).

See [Deploying Code Updates](../04-making-changes/deploying-code-updates.md#network-requirements) for deployment procedures.

**Normal operations:**

If you're not actively connecting from your personal computer to Azure resources, leave public access disabled for all eligible resources (SQL Database, Blob Storage, Database Writer function). The pipeline operates seamlessly using the private network.

---

For technical configuration details, firewall rules, and step-by-step procedures, see [Networking Reference](../05-reference/azure-infrastructure/networking.md).
