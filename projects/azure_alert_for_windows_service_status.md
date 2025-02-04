## Monitor a Service on an Azure Windows VM and Create an Alert Notification if the Windows Service is not Running
_Date: 04 Feb 2025_  

**Scenario**: A critical Windows service runinng on an Azure VM crashes and needs to be manually restarted due to some manual authentication steps required by the service.  

**Requirement**: The IT team needs to be alerted when the Windows service is not running so that remediation efforts can be made as soon as it occurs.  

**Research**: 

It is possible to create Alerts in Azure based off performance metrics and Windows Event Logs. The Alert can be used to trigger a notification using an Action Group. 

We can collect Event Logs from VMs using Data Collection Rules that employ Azure Monitoring Agent (AMA) installed on the VMs into a Log Analytics WorkSpace. Then, we create an alert based on a Custom Log Search query which can trigger a n alert notification to be sent.

In Windows Server OS, we can monitor events with the Event ID 7038 for services changing running state or Event ID 7000 if the service fails to start.  
However, Windows Client OS (such as Windows 8, 10 or 11) does not trigger the 7038 events. To address this limitation, we can use a PowerShell script to monitor the Service state and log an event.
&nbsp; 

---

### Steps:

- From Azure Portal, search for "Log Analytics WorkSpace". Create a Log Analytics WorkSpace that will be the destination of the Windows event logs being collected.

    ![](/assets/img/projects/vm_service_alert/_)


- From Azure Portal, search for "Data Collection Rule". Create a Data Collection Rule (DCR).  
    
    - Choose the _Windows_ option for the _Platform Type_
    ![](/assets/img/projects/vm_service_alert/_)

    - For the _Resources_ page, add the VM(s) you wish to monitor the service and collect the event logs from.

    - For the _Data Sources_ page, click the _+ Add data source_ button and select _Windows Event Logs_ from the drop-down. Then, select the Event Log Types and Log Level. You can choose from here as it suits you or use an _XRPath filter_ under the _Custom_ option.

        I am choosing to collect only the _Warning_ level events from the _Application_ log as this is where our PowerShell script will log its findings.
        ![](/assets/img/projects/vm_service_alert/_)

    - For the _Destination_ prompt, select _Azure Monitor Logs_ for the _Destination Type_. Then, select the Log Analytics WorkSpace you created earlier and its correspong Subscription. 
    ![](/assets/img/projects/vm_service_alert/_)

    - Creating the DCR will automatically prompt the Azure Monioring Agent (AMA) to be installed on the VMs specified under _Resources_.

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

- On the VM(s), I have used Task Scheduler and created a task that runs every 5 mins to trigger a PowerShell script. 

    - The script checks if the specified service is running and if not it logs a "Warning" level event in the Windows Application event log. The PS v5.1 script is shown below:

    ```
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

- To add MFA prompt for the RDP access, create a **Conditional Policy** that Grants access to Users for All Resources when the MFA Requirement is met. This will cover the RDP access to the VM and require MFA.

- To RDP to the VM:
    - Connect to the Azure VPN from your local device.

    - Open Remote Desktop Connection app from your local device and from the Advanced tab, select _"Use a web account to sign in to the remote computer"_ 
    ![](/assets/img/projects/rdp_entra_id/rdc_web_account.png)
    
    - Add the remote computer's FQDN (OS hostname plus the DNS Suffix)
    
    - Enter the username, password and MFA code when you the MS Entra ID modern authentication credential prompt. 
    
    - Select _Yes_ when prompted to _Allow remote desktop connection?_ 
    ![](/assets/img/projects/rdp_entra_id/rdp_prompt.png) 


You should now be able to access the Azure VM via RDP using the Entra ID credentials and MFA over a VPN connection. Hope this was helpful.  

&nbsp;

---

[Back to Project List](../projects) &emsp; &emsp; &emsp; [Back to Top](#top)