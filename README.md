Proposed Structure
1. Introduction & Overview

Goals, stakeholders, handoff context
High-level architecture diagram
15-minute tour: "Here's what everything does"

2. System Architecture

Functional diagram (what you have)
Network topology diagram
Resource naming conventions
Cost breakdown overview

3. Azure Infrastructure ← This becomes a major section

3.1 Resource Groups & Organization
3.2 SQL Database

Resource creation
Networking & access
Schema & tables
Common queries
Maintenance operations


3.3 Blob Storage

Resource creation
Folder structure
Access & monitoring


3.4 Function Apps (overview of all three)

Deployment process
Environment variables
Monitoring & logs


3.5 Static Web App
3.6 Networking (VNet, private endpoints)
3.7 Monitoring & Cost Management

4. Data Processing Components ← Code logic

4.1 Webhook Receiver Function

What it does
Code structure
Error handling


4.2 Blob Processor Function

Deduplication logic
Database writing


4.3 Daily Reporter Function
4.4 API Function

5. Public Dashboard

Frontend architecture
API endpoints
User features

6. Particle.io Platform

Webhook configuration
Monitoring & troubleshooting
Device management

7. Operations & Maintenance

Daily/weekly checks
Common issues
Making changes
Cost optimization tips

8. Appendices

A: Azure Portal Step-by-Step Guides (screenshots)

A1: Creating SQL Database
A2: Configuring Private Endpoints
A3: Deploying Function Apps
etc.


B: SQL Quick Reference (common queries)
C: Particle.io Screenshots
D: Code Repository Links
E: Contact Information

Key principle:

Main sections = concepts & "why"
Appendices = step-by-step "how" with screenshots