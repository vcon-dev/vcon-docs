---
icon: face-glasses
description: Describes the 43 synthetic TADHack 2025 vCons, where to get them and what they contain.
---

# TADHack vCon

## Conversation Set

For the TADHack 2025 vCon Hackathon, we've generated a set of synthetic vCons for your use:

* You can download the set at [https://github.com/vcon-dev/tadhack-2025](https://github.com/vcon-dev/tadhack-2025)
* Audio lives in the repo alongside the vCons, in IETF vCon syntax 0.4.0
* The conversations are synthetic, generated with the vCon Faker pipeline. Each party is marked `"validation": "synthetic"`, and each vCon carries a `lawful_basis` attachment recording that it is demo data with no real data subject. The phone numbers and emails were not checked against real subscribers, so do not dial or email them.
* The dataset is also listed in [Public vCon Datasets](../tools/vcon-datasets.md), with the `vcon-data` command to load it.

## Overview

This dataset contains customer service conversation data from Aquidneck Yacht Brokers in vCon (virtual conversation) format. The conversations span from May 18-24, 2025, and represent typical customer interactions for a yacht brokerage company.  The dataset includes 43 customer service calls between Aquidneck Yacht Brokers agents and customers, covering various marine industry-specific support scenarios.

### Conversation Types

#### 1. Returns & Refunds

* Customers requesting returns for yacht equipment
* Processing refund requests
* Emotional customers (often expressing sadness about returns)

#### 2. Shipping & Logistics

* Yacht transportation inquiries (e.g., Fort Lauderdale to Newport)
* Delivery status updates
* Shipping cost questions

#### 3. Order Issues

* Wrong items received (e.g., yacht anchor instead of navigation system)
* Missing order investigations
* Order verification and corrections

#### 4. Equipment Support

* GPS malfunction troubleshooting
* Navigation system issues
* Equipment compatibility questions

#### 5. Business Services

* Yacht listing inquiries
* Brokerage service questions
* Pricing and commission discussions

#### 6. Account Management

* Membership cancellations
* Billing inquiries
* Privacy and data concerns
* Contact information updates

#### 7. Appointments & Scheduling

* Yacht viewing appointments
* Service scheduling
* Consultation bookings

### Call Characteristics

* **Duration**: 43 to 77 seconds, about 58 seconds on average
* **Call Disposition**: All marked as "ANSWERED" with "VM Left" status
* **Language**: English
* **Transcription Confidence**: 99%
* **Professional Tone**: Agents maintain consistent, helpful demeanor
* **Resolution Rate**: Most issues resolved or appropriately escalated

### Data Format

Each conversation includes:

* Audio recording (MP3 format)
* Full transcript with speaker diarization
* AI-generated summary
* Participant metadata (names, roles, contact info)
* Call metadata (duration, timestamp, disposition)

### Typical Interaction Flow

1. Agent greeting with company name and agent introduction
2. Customer name verification
3. Issue description by customer
4. Information gathering (order numbers, email verification)
5. Resolution or escalation
6. Professional closing

### Notable Patterns

* Customers frequently express emotions related to their issues
* Agents consistently follow verification protocols
* Marine industry-specific terminology used throughout
* Focus on high-value transactions typical of yacht brokerage

This dataset provides realistic examples of customer service interactions in the luxury marine industry, useful for training, analysis, or demonstration purposes.
