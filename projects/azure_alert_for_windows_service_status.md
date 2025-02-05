## Monitor a Service on an Azure Windows VM and Create an Alert Notification if the Windows Service is not Running
_Date: 04 Feb 2025_  

**Scenario**: A critical Windows service runinng on an Azure VM crashes and needs to be manually restarted due to some manual authentication steps required by the service.  

**Requirement**: The IT team needs to be alerted when the Windows service is not running so that remediation efforts can be made as soon as it occurs.  

**Summary**: 

It is possible to create Alerts in Azure based on performance metrics and Windows Event Logs. The Alert can be used to trigger a notification using an Action Group. 

We can collect Event Logs from VMs using Data Collection Rules that employ Azure Monitoring Agent (AMA) installed on the VMs to send the logs to a Log Analytics WorkSpace. Then, we create an Alert based on a Custom Log Search query which can trigger an alert notification to be sent.

In Windows Server OS, we can monitor events with the Event ID 7038 for services changing running state or Event ID 7000 if the service fails to start.However, Windows Client OSes (such as Windows 8, 10 or 11) do not trigger the 7038 events. To address this limitation, we can use a PowerShell script to monitor the Service state and log an event.
&nbsp; 

---

### Steps:

- From Azure Portal, search for **Log Analytics WorkSpace**. Create a Log Analytics WorkSpace that will be the destination of the Windows event logs that will be collected from the Windows VM(s).   
&nbsp; 
- From Azure Portal, search for **Data Collection Rule**. Create a Data Collection Rule (DCR) using the settings mentioned below.  
    
    - For the _Platform Type_, choose the _Windows_ option.

    - For the _Resources_ page, add the VM(s) you wish to monitor the service and collect the event logs from.

    - For the _Data Sources_ page, click the _+ Add data source_ button and select _Windows Event Logs_ from the drop-down. Then, select the Event Log Types and Log Level. You can choose from here as it suits you or use an _XRPath filter_ under the _Custom_ option.

        I am choosing to collect only the _Warning_ level events from the _Application_ log as this is where our PowerShell script will log its findings.     

        ![](/assets/img/projects/vm_service_alert/dcr_log_type.jpg)

    - For the _Destination_ prompt, select _Azure Monitor Logs_ for the _Destination Type_. Then, select the Log Analytics WorkSpace you created earlier and its correspong Subscription. 

    - Creating the DCR will automatically prompt the **Azure Monioring Agent (AMA)** to be installed on the VMs specified under _Resources_ using VM Extensions.

    - To verify that the Monitoring Agent is operating correctly, go to your "Log Analytics WorkSpace" from the Azure Portal and select _Logs_ and run the KQL query below.

        ```
        Heartbeat
        | where TimeGenerated > ago(24h)
        | where Computer has "<<Your-Computer-Name>>"
        | project TimeGenerated, Computer, Category, Version
        | order by TimeGenerated desc
        ```    
        You should be able to see Heartbeat events being collected from the AMA.

    - To verify that the Event Logs are being collected, replace the above query with the query below and run it.

        ```
        Event
        | where Computer has "<<Your-Computer-Name>>"
        | where TimeGenerated > ago(48h)
        | order by TimeGenerated desc
        ```

        You should be able to see events for the Event Log type and level you selected for the DCR.

&nbsp;

- On the VM(s), I have used **Task Scheduler** and created a task that runs every 5 mins to trigger a PowerShell script (code provided below). 
    - For the _Trigger_, choose _On a schedule_ and _One time_ under settings provided the start time. 
    
        Under _Advanced settings_ check the option to _Repeat task every_ 5 minutes for a duration of _Indefinitely_      

        ![](/assets/img/projects/vm_service_alert/task_trigger.jpg)

    - For the _Action_, choose _Start a program_. For _Program/Script_, type _powershell.exe_ and for Arguments type _-File "C:\path\to\PS-script.ps1"_

&nbsp;

- The **PowerShell script** checks if the specified service is running, and if not it logs a "Warning" level event in the Windows Application event log. 
    
    The PS v5.1 script is shown below. As an example, this is monitoring the Print Spooler service status.

    ```powershell
    # Define the service name you want to monitor
    $serviceName = "Spooler"  # Display Name = Print Spooler

    # Define the event log source and log name
    $logSource = "ServiceMonitor"
    $logName = "Application"



    # Check if the event log source exists, if not, create it
    if (-Not [System.Diagnostics.EventLog]::SourceExists($logSource)) {

        New-EventLog -LogName $logName -Source $logSource

    }


    Try {
        
        # Get the service object
        $service = Get-Service -Name $serviceName

        # Check the status of the service
        if ($service.Status -eq 'Running') {
            
            # Only used for testing, no need to log event when service is running normally 
            # Write-EventLog -LogName $logName -Source $logSource -EntryType Information -EventId 1000 -Message "The service '$serviceName' is currently running."

        } else {

            # When service is not running, log a Warning event
            Write-EventLog -LogName $logName -Source $logSource -EntryType Warning -EventId 1001 -Message "The service '$serviceName' is not running. Current status: $($service.Status)."

        }
    } 
    Catch {

        # When service is not running or unknown state, log a Warning event
        Write-EventLog -LogName $logName -Source $logSource -EntryType Error -EventId 1002 -Message "Failed to retrieve the status of the service '$serviceName'. Error: $_"

    }
    ```

    - To verify if the script is working, open _Services.msc_ on the VM and stop the service you are monitoring such as the _Print Spooler_ service. 
    
        Next, run the script manually or trigger it from the Task Scheduler and then open the _Event Viewer_ > _Application_ log and verify if the service is logging a _Warning_ event.

&nbsp;

- Finally, from Azure Portal, search for **Alerts** and create an _Alert Rule_.

    - For the _Scope_ page, select the VM(s) you are monitoring.

    - For the _Condition_ page, select _Custom log search_ option for the _Signal Name_ and add the query below:

    ```
    Event
    | where Computer has "<<your-computer-name>>"
    | where Source has "ServiceMonitor"
    | where EventLevel == 3
    | where EventID == 1001 or EventID == 1002
    | where TimeGenerated > ago(15m)
    | order by TimeGenerated desc
    ```
    
    Under _Measurement_, enter:    
    _Mesure_ = _Table Rows_,    
    _Aggregation Type_:_Count_    
    _Aggregation granularity_:_15 minutes_    
    _Operator_ = _Greater than or equal to_    
    _Threshold value_ = _2_    
    _Frequency of evaluation_ = _15 minutes_  

    This will trigger the alert if 2 or more Warning events are registered in the previous 15-minute period. 

    - For the _Actions_ page, select an existing Action Group or create a new one to send the notification. For example, I chose to send an email to myself.

    - For the _Details_ page, enter the Alert _Severity_, _Name_ and other details to create the Alert Rule.

&nbsp;

You should now start to receive an alert notification every 15 minute if the monitored service is not in a running state over 15 minutes.     

![](/assets/img/projects/vm_service_alert/vm_alerts.jpg)


Hope this was helpful.  

&nbsp;

---

[Back to Project List](../projects) &emsp; &emsp; &emsp; [Back to Top](#top)