# Azure-Website-Uptime-Monitor

## Video walkthrough: https://www.loom.com/share/0621ebede73446e6b1980083539cea82
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

### Folder Setup

```terraform
New-Item -ItemType Directory -Path "$HOME\uptime-monitor-001"
cd "$HOME\uptime-monitor-001"
New-Item -ItemType Directory -Path "function_app"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars
New-Item -ItemType File function_app\check_website.py, function_app\requirements.txt, function_app\function.json
```
### Write variables.tf
### Write terraform.tfvars

###
- Replace the values with your actual target website URL, email, and phone number.

### Write the Function Code

###
- Before writing the infrastructure, write the monitoring logic. Azure Functions are small, event-driven pieces of code — in this case, a Python function that runs on a timer.

### Write function_app/requirements.txt

### Write function_app/function.json

###
- This is the Function binding configuration. It tells Azure Functions what triggers the function and what its inputs and outputs are.

- schedule = "0 */5 * * * *" is a CRON expression meaning "run every 5 minutes." The six fields are: seconds, minutes, hours, day-of-month, month, day-of-week. */5 in the minutes field means "every 5 minutes."

### Write function_app/check_website.py

###
- This is the actual monitoring logic. Read through each section — it is explained inline.

### Write main.tf

### Resource group

```terraform
resource "azurerm_resource_group" "main" {
  name     = "rg-uptime-monitor-${var.yourname}"
  location = var.location
  tags     = var.tags
}
```

### Storage account and Table

###
- Azure Functions need a storage account to store their runtime state, logs, and deployment packages. The same account is used for the uptime check results table — this keeps the architecture simple.

- account_replication_type = "LRS" is sufficient here because this data is operational (check results), not business-critical backup data. If the storage account in a single region failed, the monitoring function itself would also be down, so geo-replication adds cost without adding meaningful protection.

```terraform
resource "azurerm_storage_account" "main" {
  name                     = "stuptime${var.yourname}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"
  tags                     = var.tags
}
 
resource "azurerm_storage_table" "uptime_checks" {
  name                 = "uptimechecks"
  storage_account_name = azurerm_storage_account.main.name
}
```

### Log Analytics Workspace and Application Insights

###
Log Analytics is the central log store. Application Insights is a monitoring layer that wraps your Function App — it tracks invocation counts, execution duration, failures, and exceptions automatically. Application Insights stores its data in a Log Analytics workspace, which is why both resources are needed.

workspace_id links Application Insights to Log Analytics. Without this link, Application Insights stores data independently and it cannot be queried alongside other Azure logs.

```terraform
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-uptime-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}
 
resource "azurerm_application_insights" "main" {
  name                = "appi-uptime-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
  tags                = var.tags
}
```

### App Service Plan and Function App

###
The App Service Plan is the compute that runs your Function App. kind = "FunctionApp" and sku { tier = "Dynamic" name = "Y1" } together configure a Consumption plan — Azure only charges you when the function is actually executing, not for idle time. For a function that runs 12 times an hour, this costs fractions of a cent per day.

The Function App is the container for your monitoring function. os_type = "linux" and runtime = "python" tell Azure to use a Linux-based Python runtime. version = "~3.10" specifies Python 3.10.

app_settings are environment variables injected into the Function at runtime. The function code reads TARGET_URL using os.environ["TARGET_URL"] — this is the correct pattern because it means you can change the monitored URL without modifying the function code.

APPINSIGHTS_INSTRUMENTATIONKEY and APPLICATIONINSIGHTS_CONNECTION_STRING connect the Function App to Application Insights automatically. Azure uses these values to know where to send telemetry data.

AzureWebJobsStorage gives the function runtime access to the storage account for its internal state management and for writing check results to Table Storage.

```terraform
resource "azurerm_service_plan" "main" {
  name                = "asp-uptime-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  os_type             = "Linux"
  sku_name            = "Y1"
  tags                = var.tags
}
 
resource "azurerm_linux_function_app" "monitor" {
  name                       = "func-uptime-${var.yourname}"
  resource_group_name        = azurerm_resource_group.main.name
  location                   = var.location
  storage_account_name       = azurerm_storage_account.main.name
  storage_account_access_key = azurerm_storage_account.main.primary_access_key
  service_plan_id            = azurerm_service_plan.main.id
 
  site_config {
    application_stack {
      python_version = "3.10"
    }
  }
 
  app_settings = {
    "TARGET_URL"                              = var.target_url
    "APPINSIGHTS_INSTRUMENTATIONKEY"          = azurerm_application_insights.main.instrumentation_key
    "APPLICATIONINSIGHTS_CONNECTION_STRING"   = azurerm_application_insights.main.connection_string
    "AzureWebJobsStorage"                     = azurerm_storage_account.main.primary_connection_string
    "FUNCTIONS_WORKER_RUNTIME"                = "python"
    "WEBSITE_RUN_FROM_PACKAGE"                = "1"
  }
 
  tags = var.tags
}
```

### Action Group with email and SMS

###
This Action Group sends alerts to both email and SMS. The sms_receiver block requires the phone number in E.164 format (country code + number, no spaces or dashes, e.g. +14045550100).

```terraform
resource "azurerm_monitor_action_group" "downtime_alerts" {
  name                = "ag-uptime-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "uptime"
 
  email_receiver {
    name                    = "owner-email"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }
 
  sms_receiver {
    name         = "owner-sms"
    country_code = "1"
    phone_number = replace(replace(var.alert_phone, "+1", ""), "-", "")
  }
 
  tags = var.tags
}
```

### Monitor alert on Function failures

###
This alert fires when the Function App logs errors — specifically, when the monitoring function writes logging.error() calls, which it does whenever the target site fails a check. Application Insights routes these error logs into Log Analytics, and this alert watches the log stream for them.

query uses KQL (the Kusto Query Language used by Log Analytics) to count error-level traces from the Function in the past 5 minutes. If any exist, the threshold of 0 is exceeded and the alert fires.

evaluation_frequency = "PT5M" means Azure evaluates this query every 5 minutes — matching the check interval of the Function itself.

```terraform
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "site_down" {
  name                = "alert-site-down-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  description         = "Fires when the uptime monitor detects site failure."
  severity            = 1
  enabled             = true
 
  scopes                  = [azurerm_log_analytics_workspace.main.id]
  evaluation_frequency    = "PT5M"
  window_duration         = "PT5M"
  auto_mitigation_enabled = true
 
  criteria {
    query = <<-QUERY
      AppTraces
      | where SeverityLevel == 3
      | where Message contains "SITE DOWN"
      | summarize count() by bin(TimeGenerated, 5m)
      | where count_ > 0
    QUERY
 
    time_aggregation_method = "Count"
    threshold               = 0
    operator                = "GreaterThan"
  }
 
  action {
    action_groups = [azurerm_monitor_action_group.downtime_alerts.id]
  }
 
  tags = var.tags
}
```

### Write outputs.tf

 ## Deploy

###
Windows (PowerShell):
```terraform
terraform init
terraform plan
terraform apply
```

## Deploy the Function Code

###
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
## Thank you for following along watching me create real-world cloud solutions. This is only part of my full Azure cloud portfolio.
