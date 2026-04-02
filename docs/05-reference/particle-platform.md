# Particle Platform Reference

## Overview

- Brief: What Particle.io platform does in this system
- Role: Cloud service between sensors and Azure

## Sensor Hardware

- Physical sensors collect data (Arduino-based)
- Cellular data transmission
- SD card backup (logistics-intensive, rarely used)
- Link to Amy's team for firmware questions

## Account Structure

- Multiple Particle accounts (6-7) due to free tier limits
- Credentials: Contact Amy's team
- Link to sensor-configuration.md for adding new sensors

## Webhook Integrations

- What webhooks are
- How they deliver data to Azure endpoints
- Integration vs Product vs Account level webhooks

## Platform Navigation

- Logging in (link to credentials source)
- Key sections: Devices, Integrations, Console, Events
- What to monitor

## Message Types & Data Format

### Environmental Data Messages

- Format, fields, examples
- See data-flow.md for processing details

### Error Messages

- Format, fields, examples

### Startup Messages

- Format, fields, examples

## Monitoring & Troubleshooting

- How to check if sensors are publishing
- Event logs
- Integration status
- Distinguishing Particle vs Azure issues
