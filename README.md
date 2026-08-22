# Azure-Website-Uptime-Monitor

## Video walkthrough: 
Jeremiah Brown | Azure Cloud Engineer | Linkedin: https://www.linkedin.com/in/jeremiah-brown12/

## Project Overview
###
Website availability is critical to any business that relies on its online presence. When a website goes down, becomes slow, or returns incorrect content, customers may encounter errors, abandon the site, and move to a competitor. The bigger problem is that businesses may not know an issue has occurred until a customer reports it or performance and sales begin to decline.

This creates a significant visibility gap: a business can be losing customers without knowing that its website is the reason.

This project eliminates that blind spot by building an automated website monitoring and alerting system in Azure. The system continuously monitors a target website every five minutes, 24 hours a day, and provides automated visibility into its availability and performance.

System Capabilities
- Availability Monitoring: Checks whether the target website successfully loads.
- Performance Monitoring: Measures how long the website takes to respond.
- Content Validation: Verifies that the expected content is present on the page.
- Automated Alerting: Immediately triggers an alert when a monitoring check fails.
- Historical Tracking: Records every check in Azure Table Storage, including the timestamp, response time, and pass/fail status.
- Performance Visibility: Provides historical data that can be used to identify downtime, performance issues, and recurring incidents.

The result is an automated monitoring solution that transforms website availability from something a business discovers after the fact into something it can continuously observe, measure, and respond to.

## Architecture Flow
 
### <img width="983" height="535" alt="image" src="https://github.com/user-attachments/assets/f369d3cf-25ec-4910-a97a-228c398306cf" />


##  What gets built

###
```
rg-uptime-monitor-jeremiah
├── Storage Account (stuptimejeremiah)
│   └── Table: uptimechecks        → every check result written here
├── App Service Plan (Y1 Consumption)
├── Function App (func-uptime-jeremiah)
│   └── Function: check_website    → timer trigger, runs every 5 minutes
├── Log Analytics Workspace
├── Application Insights           → monitors the Function App itself
├── Monitor Action Group           → email + SMS alert targets
└── Monitor Alert Rule             → fires on SITE DOWN log entries
```

## Prerequisites

###
 Before deploying, install and configure:
Terraform installed
Azure CLI installed and authenticated (az login)
Python 3.10 installed

## Terraform Configuration

###

## Verification Checklist
###
- Function App func-uptime-[yourname] exists and shows Running status
- Function App → Functions shows check_website function listed
- Table Storage uptimechecks contains rows with check results
- Application Insights → Live Metrics shows function invocations
- Alert rule alert-site-down-[yourname] exists in Monitor → Alerts
- Action Group has both email and SMS receivers configured

<img width="827" height="491" alt="image" src="https://github.com/user-attachments/assets/9ac8c3fc-76cc-44a4-b217-61b2e66ff5c1" />


<img width="1229" height="578" alt="image" src="https://github.com/user-attachments/assets/af9ddceb-a60c-4f06-8e18-39b8db10c0c6" />

<img width="968" height="267" alt="image" src="https://github.com/user-attachments/assets/a34ed602-3ebd-458b-a4f3-d4a8b73809c4" />


## Troubleshooting & Lessons Learned

###

## Teardown
###
``` terraform
Terraform Destroy
```

