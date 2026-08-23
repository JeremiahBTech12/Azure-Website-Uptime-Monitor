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

The system operates using an automated, timer-driven architecture. An Azure Function runs every five minutes and sends an HTTP request to the target website to evaluate its availability, response time, and content. Each check is classified as a pass, slow response, or failure, with the results written to Azure Table Storage for historical tracking. When the function detects a failure, it records a SITE DOWN error in Application Insights, which sends the telemetry to the connected Log Analytics workspace. An Azure Monitor alert rule continuously evaluates these logs for failure events and, when a failure is detected, triggers an Action Group that immediately sends an email and SMS notification to the business owner.

Supporting this workflow are several monitoring and data components. Azure Table Storage maintains a record of every website check, including timestamps, response times, and status information, while Application Insights and Log Analytics provide visibility into the Function App’s executions, errors, and overall health. Together, these services create a fully automated monitoring pipeline that continuously checks the website, records its performance, and alerts the appropriate recipient when an issue occurs.

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
- Terraform installed
- Azure CLI installed and authenticated (az login)
- Python 3.10 installed

## Terraform Configuration

###

 
 ## Deploy

###
Windows (PowerShell):
```terraform
terraform init
terraform plan
terraform apply
```

## Deploy the Function Code

Terraform provisions the Function App container. You deploy the function code separately using the Azure CLI.

First, package the function:

###
Windows (PowerShell):
```terraform
cd function_app
pip install -r requirements.txt --target .python_packages\lib\site-packages
Compress-Archive -Path * -DestinationPath ..\function_deploy.zip -Force
cd ..
```

## Deploy to the Function App:

###
Windows (PowerShell):
```terraform
az functionapp deployment source config-zip `
  --resource-group rg-uptime-monitor-jeremiah `
  --name func-uptime-jeremiah `
  --src function_deploy.zip
```

## Verify Running Monitor
Wait 5 minutes for the first execution, then check the results:
###
Windows (PowerShell):
```
az storage entity query `
  --account-name stuptimejeremiah `
  --table-name uptimechecks `
  --auth-mode login `
  --output table
```

You should see rows appearing with Status values of PASS, SLOW, or FAIL. A healthy site will show PASS for every row.

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

The errors is this project ranged from simple requests, to real headaches diagnosing/resolving.

- `terraform apply` failed provisioning the App Service Plan with a quota error on the Y1 (Consumption) SKU**
The subscription had no quota allocated for the Y1 (Dynamic/Consumption) tier needed to host
`asp-uptime-jeremiah`, so Terraform couldn't create it. The fix here was to
request a quota increase for the Y1/Consumption tier directly through Azure (Portal → Support →
Quotas) and wait for approval before re-running `terraform apply`.

- `check_website` still showed 0 functions found 
PowerShell's `Compress-Archive` wrote the zip's internal file paths using backslashes
(`check_website\function.json`) instead of the forward slashes the ZIP format — and the
Linux-based Function host — require. The host couldn't see `check_website` as a folder at all,
so it reported zero functions no matter how many times the app was restarted. Fixed by
rebuilding the zip with Python's `zipfile` module, which writes correct forward-slash paths.

- Running `terraform apply` after deploying function code would silently break the deployment again
`main.tf` hardcodes `WEBSITE_RUN_FROM_PACKAGE = "1"`, but
`az functionapp deployment source config-zip` overwrites that same setting with a real package
URL after every deploy. Terraform has no visibility into that change, so a plain `apply` treats
the real URL as drift and resets it back to `"1"` — at which point the Function App no longer
knows where its code lives. Fixed by adding
`lifecycle { ignore_changes = [app_settings["WEBSITE_RUN_FROM_PACKAGE"]] }` to the
`azurerm_linux_function_app` resource.

## Teardown
###
``` terraform
Terraform Destroy
```

