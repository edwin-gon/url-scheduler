# Url Scheduler 

## Problem Statement

A user is provided CLI Application to schedule URL invocations. User should be able to CRUD a scheduled entry.   

Assumptions:
- Only GET method allowed — URL invocations do not allow specified headers or body requests. 
- No Auth implementation 
- No restriction on number of requests or specific actions per user
- Fixed Timestamp format/time zone
- Users can schedule requests however far in the future

---
# System Design
! [Architecture Diagram](url_scheduler.png)

## API Design

***Bonus Points:***

Protected Endpoints
- Number invocations by status, date, and time range
- Validation: 
    - See if endpoint matches required criteria. 
    - Output list of criteria that needs to be satisfied
- Test: allow users to test payload against expected responses/payloads


