# Cisco Backup Bot

## Problem Solved

Network device configurations need to be backed up reliably, but manual backup routines are easy to miss and hard to audit. This project automates Cisco configuration backup activity so technical teams can reduce operational risk and recover faster when configuration changes cause problems.

## Business Context

For technical operations teams, network configuration backups are a basic but important control. A small automation tool can provide meaningful resilience by making backup collection consistent, logged, and easier to verify.

## Tech Stack

- Python
- Network automation libraries and command execution patterns
- Scheduled automation
- Configuration file handling
- Logging and status reporting

## Key Features

- Connects to supported Cisco network devices
- Retrieves running or startup configuration data
- Stores backups using predictable naming and timestamp conventions
- Logs success and failure results for operational review
- Supports repeatable scheduled execution
- Keeps credentials and device-specific details outside published code

## Data / Automation Work Involved

- Automated device interaction that would otherwise require manual login
- Standardized backup file naming and retention-friendly structure
- Captured operational logs for traceability
- Designed failure visibility for unreachable devices or authentication issues
- Supported repeatable backup checks across a device list

## Testing and Quality Notes

- Tested connection handling, timeout behavior, and failed authentication scenarios
- Reviewed backup output structure for consistency
- Kept device IPs, hostnames, credentials, and network diagrams out of public materials
- Used logging to make scheduled runs easier to troubleshoot

## What This Demonstrates To Employers

- Python automation applied to real technical operations work
- Understanding of network administration and operational risk
- Ability to build practical tools that improve reliability without unnecessary complexity
- Awareness of credential safety and sensitive infrastructure information

## Repository Status

Internal. This case study excludes network identifiers, credentials, configuration files, and environment-specific details.

