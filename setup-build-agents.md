# Setup Build-Agents

## Prerequisites

1) Get your [VSTS-Token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops)
1) Install [AzuredevOpsAPIUtils](https://www.powershellgallery.com/packages/AzureDevOpsAPIUtils) with PowerShell:

```PowerShell
Install-Module AzuredevOpsAPIUtils -Force
Update-Module  AzuredevOpsAPIUtils
Import-Module  AzuredevOpsAPIUtils -Force
```

## Create Azure DevOps Agent Pool

Create an Azure DevOps [Agent Pool](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops) for your organization with PowerShell:

```PowerShell
$poolName       = "<POOL-NAME>"
$devOpsURL      = "https://dev.azure.com/<YOUR-ORGANIZATION>"
$vstsToken      = "<YOUR-VSTS-TOKEN>"

$pools          = (Get-AzureDevOpsAgentPools -organizationUri $devOpsURL -vstsToken $vstsToken)
$pool           = ($pools | Where-Object { $_.name -eq $poolName } | Select-Object -First 1)

if (! $pool) {
    $pool = (Add-AzureDevOpsAgentPool -name $poolName -organizationUri $devOpsURL -vstsToken $vstsToken)
}
```

## Install Local Build Agent

Download, Install, and Run a [Self Hosted Agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops) with PowerShell:

```PowerShell
# 1) replace placeholders in these variables:
$poolName       = "<POOL-NAME>"
$devOpsURL      = "https://dev.azure.com/<YOUR-ORGANIZATION>"
$vstsToken      = "<YOUR-VSTS-TOKEN>"
$credential     = $null  # or ... ([PSCredential]::new("username", (ConvertTo-SecureString -String "password" -AsPlainText -Force)))

# 2) Ensure, you download the newest Agent
$agentURL       = "https://vstsagentpackage.azureedge.net/agent/2.155.1/vsts-agent-win-x64-2.155.1.zip"

# 3) Prepare variables for execution
$agentFile      = "$($env:TEMP)/agent.zip"
$agentName      = $env:COMPUTERNAME

Write-Host "Download VSTS-Agent"
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri "$agentURL" -OutFile "$agentFile"

Write-Host "Extract VSTS-Agent"
Expand-Archive $agentFile "C:/agent"

$installCmd     = Get-AzureDevOpsAgentInstallParameters `
                        -poolName        $poolName `
                        -organizationUri $devOpsURL `
                        -vstsToken       $vstsToken `
                        -agentName       $agentName `
                        -credential      $credential `
                        -runAsService:($credential -ne $null)

# Install the Agent
& cmd.exe /c """C:\agent\config.cmd $installCmd""" 2>%1
# Start the Deployment Agent
& cmd.exe /c """C:\agent\run.cmd"""
```

## Install Build Agent in NAV-/BC-Docker Container

To create the docker container use [NavContainerHelper](https://www.powershellgallery.com/packages/navcontainerhelper/) (see also [Freddy's Blog](https://freddysblog.com/category/navcontainerhelper/))

Download, Install, and Run a [Self Hosted Agent](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/v2-windows?view=azure-devops) as Service inside of a NAV-/BC-Docker Container with PowerShell:

```PowerShell
# 1) replace placeholders in these variables:
$containerName  = "<container-name>" # your BC container
$poolName       = "<POOL-NAME>"
$devOpsURL      = "https://dev.azure.com/<YOUR-ORGANIZATION>"
$vstsToken      = "<YOUR-VSTS-TOKEN>"
$credential     = ([PSCredential]::new("<docker-username>", (ConvertTo-SecureString -String "<docker-password>" -AsPlainText -Force)))

# 2) Ensure, you download the newest Agent
$agentURL       = "https://vstsagentpackage.azureedge.net/agent/2.155.1/vsts-agent-win-x64-2.155.1.zip"

# 3) Prepare variables for execution
$agentName      = $containerName
$installCmd     = Get-AzureDevOpsAgentInstallParameters `
                        -poolName        $poolName `
                        -organizationUri $devOpsURL `
                        -vstsToken       $vstsToken `
                        -agentName       $agentName `
                        -credential      $credential `
                        -runAsService:($credential -ne $null)
$uninstallCmd   = Get-AzureDevOpsAgentUnInstallParameters -vstsToken $vstsToken

# Run this script inside of the Container
Invoke-ScriptInNavContainer -containerName $containerName -scriptblock {
        Param($agentURL, $installCmd, $uninstallCmd, $poolName, $agentName)

    try {
            Add-LocalGroupMember -Group Administrators -Member "Network Service" -ErrorAction SilentlyContinue

            Write-Host "Setup VSTS-Agent $agentName for pool $poolName" -f Cyan
            $agentFile = "C:/run/my/agent.zip"
            Set-Location "C:/run/my"

            Write-Host "Download VSTS-Agent"
            [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
            Invoke-WebRequest -Uri "$agentURL" -OutFile "$agentFile"

            Write-Host "Extract VSTS-Agent"
            Expand-Archive $agentFile "C:/agent"

            Write-Host "Register VSTS-Agent $agentName @ $poolName"

            # remove agent, if already registered
            & cmd.exe /c """C:\agent\config.cmd $uninstallCmd""" 2>%1

            # register agent
            & cmd.exe /c """C:\agent\config.cmd $installCmd""" 2>%1

            Write-Host "Setup VSTS-Agent done." -f Green
    } catch {
        Write-Warning "Install VSTS-Agent ERROR: $($_.Exception.Message)"
    }

} -argumentList $agentURL, $installCmd, $uninstallCmd, $poolName, $agentName

```
