---
layout: post
title: "Building a Geospatial Monitoring System for Payment Terminals"
date: 2026-06-13
categories: [iot, geospatial, aws]
tags: [postgis, geoserver, aws-iot, fraud-detection, postgresql]
source_code: "https://github.com/walterdeane/blog-drafts/tree/main/blogs/004_building_a_geospatial_monitoring_system_for_payment_terminals"
---

In this blog I want to demonstrate a geospatial system that I had planned to build for monitoring payment terminals used by merchants. While the original use case was focused on payments, I think the same concepts apply to a wide range of connected devices, including payment terminals, kiosks, sensors, and other IoT deployments.

I was disappointed that I never had the opportunity to finish this project before leaving Tyro. Rather than let the idea disappear, I decided to document it here and provide a proof-of-concept repository so others can experiment with the approach and determine whether it could be useful in their own environments.

## Potential Use Cases

### Automatic Detection of Cell Tower Outages

One of the first questions during an incident is whether the issue is being caused by one of the cellular providers.

The challenge is that payment terminals often do not generate explicit errors when connectivity is lost. They simply stop transacting.

There are many reasons why a terminal may not be processing transactions:

- The merchant may be closed.
- Business activity may be unusually slow due to weather or local events.
- There may be a software issue.
- There may be a network outage affecting a specific area.

Without location intelligence, determining the root cause can take significant time. My team's goal was to reduce **MTTI (Mean Time to Identify)** during incidents. By monitoring the geographic distribution of terminal activity, it becomes much easier to identify patterns that suggest a carrier or infrastructure outage.

### Enhanced Fraud Detection

A common fraud scenario involves the theft of a payment terminal followed by refund activity from a different location.

If terminal location is being monitored, a fraud engine can be notified when a device moves significantly from its expected location. If suspicious transactions occur shortly after that movement, the fraud detection system has an additional signal that can be used to identify potentially fraudulent behaviour.

Location alone is not enough to prove fraud, but it can be a valuable feature when combined with existing fraud models.

### Enhanced Business Analytics

A geospatial platform can also provide sophisticated business intelligence for enterprise customers.

Examples include:

- "Show me suburbs with the highest coffee shop sales where my business does not currently have a store."
- "Show me stores that are underperforming compared to the industry average in the surrounding area."
- "Show me sales trends by region over the last six months."
- "Identify growth corridors where transaction volume is increasing rapidly."

These types of insights become much easier when transaction and device data can be analysed spatially.

## The Main Challenge: Cost

The biggest challenge was obtaining location data at scale.

After evaluating several providers, we decided that Google's Geolocation API offered the best accuracy for our use case.

Although newer payment terminals often include GPS hardware, GPS is frequently unreliable indoors where most terminals operate. Many devices do not have a direct line of sight to satellites. Some devices use cellular connectivity, while others may only have Wi-Fi available.

Google's Geolocation API can combine information from:

- IP addresses
- Wi-Fi access points
- Cell tower data

to estimate the most accurate location possible.

### The Cost Problem

Google's pricing initially appeared reasonable. However, the numbers become much larger when scaled across an entire fleet.

For example:

- 130,000 terminals
- One location lookup per hour
- 24 hours per day
- 30.5 days per month

Results in approximately:

**95,160,000 API calls per month**

Using Google's published pricing at the time, Gemini estimated this would cost somewhere between **$34,000 and $64,000 per month**.

That estimate is likely higher than reality because:

- Not all terminals are powered on 24 hours a day.
- Many merchants only operate during business hours.
- Volume discounts would likely apply.
- Location changes occur far less frequently than telemetry updates.

Even so, it is a significant cost for a single metric, regardless of how valuable that metric may be.

## Reducing Costs Through Caching

To make the solution economically viable, I designed a caching strategy that dramatically reduced the number of external geolocation requests.

### Step 1: Cell Tower Dataset

I sourced a publicly available global cell tower dataset and filtered it down to Australian towers only.

The dataset was then cleaned so it contained only the fields required for geolocation matching.

### Step 2: Location Cache

The system would maintain a cache of previously resolved locations.

When new telemetry arrived, the processing service would follow this sequence:

1. Check whether the device already has a known latitude and longitude.
2. Query the geospatial cache to determine whether the current cell tower or network signature has already been resolved.
3. Reuse the cached location if available.
4. If no cached location exists, call Google's Geolocation API.
5. Store the result in both the cache table and the device location table.

The cache would be indexed using a Top-K style lookup structure to allow efficient matching of frequently observed tower combinations.

### Step 3: Minimise External Calls

Once the cache had been populated, the vast majority of location requests would never leave our environment.

In the ideal case:

- Devices remain stationary.
- Cell tower associations remain stable.
- Cached locations are continuously reused.

Under those conditions, monthly costs could potentially drop from tens of thousands of dollars to approximately **$700 per month**.

Before rolling this out across the production fleet, my plan was to enable telemetry collection while feature-toggling the Google API calls. This would allow us to gather real-world metrics for a month and validate the assumptions before incurring any significant cost.

Based on a rough Fermi estimate, I expected annual external API costs to remain below **$3,000 per year** if the cache behaved as anticipated.

There was also a mechanism planned to bypass the cache entirely when required. This would allow fraud analysts or support engineers to force a live geolocation lookup for diagnostics or investigations. Routine monitoring would continue to rely on cached tower locations.

## System Architecture

The proposed solution consisted of four primary components.

```mermaid
flowchart LR

    subgraph Devices
        T1["Payment Terminals"]
    end

    subgraph Ingestion
        IOT["AWS IoT Core"]
        SQS["SQS"]
    end

    subgraph Processing
        PROC["Geospatial Processing Service"]
        CACHE["Location Cache"]
        GOOGLE["Google Geolocation API"]
    end

    subgraph Storage
        POSTGIS["Aurora PostgreSQL + PostGIS"]
    end

    subgraph Mapping
        GEOSERVER["GeoServer"]
        MAPS["OpenLayers Maps"]
    end

    subgraph Analytics
        FRAUD["Fraud Detection"]
        INCIDENT["Incident Detection"]
        BI["Business Intelligence"]
    end

    subgraph Users
        OPS["Operations Teams"]
        ANALYSTS["Data Analysts"]
        CUSTOMERS["Enterprise Customers"]
    end

    T1 --> IOT
    IOT --> SQS
    SQS --> PROC

    PROC --> CACHE
    CACHE --> GOOGLE

    PROC --> POSTGIS
    GOOGLE --> POSTGIS

    POSTGIS --> GEOSERVER
    GEOSERVER --> MAPS

    PROC --> FRAUD
    PROC --> INCIDENT

    POSTGIS --> BI

    MAPS --> OPS
    MAPS --> ANALYSTS

    BI --> ANALYSTS
    BI --> CUSTOMERS
```

### AWS IoT

AWS IoT handled ingestion and routing of:

- Device telemetry
- Connectivity events
- Last Will and Testament (LWT) messages

This provided a scalable and reliable mechanism for collecting data from the fleet.

### GeoServer

A GeoServer instance running in EKS would provide:

- Map services
- Feature services
- Spatial layers
- Tile generation

GeoServer was configured to store its tile cache in S3 using a GeoServer plugin, allowing the cache to be shared across multiple instances.

### Aurora PostgreSQL with PostGIS

All geospatial data would be stored in Aurora PostgreSQL with the PostGIS extension enabled.

PostGIS provides a rich set of spatial capabilities including:

- Spatial indexing
- Distance calculations
- Polygon operations
- Geofencing
- Heat maps
- Spatial aggregations

This became the foundation for both operational monitoring and analytics.

### Telemetry Processing Service

The final component was a custom application running in EKS.

This service consumed telemetry from an SQS queue and acted as the central processing engine.

Its responsibilities included:

- Processing location telemetry
- Querying the location cache
- Calling external geolocation providers when required
- Updating device records
- Generating fraud signals
- Generating operational alerts
- Feeding incident detection workflows

This service was effectively the brain of the platform.

### Data Consumers

The processed geospatial information could then be consumed by a variety of internal systems.

Most visualisation would be provided through OpenLayers-based maps embedded within existing operational and business applications.

## Current State

Before leaving, I had already built several of the major components:

- GeoServer running in EKS
- S3-backed tile caching
- Aurora PostgreSQL with PostGIS
- Initial geospatial schemas and services

For the purposes of this blog and the accompanying proof-of-concept repository, I'll keep things much simpler.

Rather than recreating the full AWS deployment, I'll run GeoServer and PostGIS locally using Docker and focus on demonstrating the core concepts. The S3-backed tile cache and production infrastructure can be ignored for now so we can concentrate on the geospatial workflows themselves.