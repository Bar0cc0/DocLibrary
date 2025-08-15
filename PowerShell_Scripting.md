# POWERSHELL SCRIPTING

### Help
```shell
Get-Alias				# lists all aliases
Get-Command				# lists all commands
Get-Help <cmdlet_name>	# params:  -detailed -showwindow -online
Get-Help about_*
```

### GREP
```shell
# Search for a pattern in the command history
Get-History | Select-Object * | Where-Object -Property <propertyName> -Like "*<pattern>*"
Get-History | Select-Object * | Where-Object -Property <propertyName> -Eq <somevalue>
			| Sort-Object -Property <propertyName> [-Descending]
			| Group-Object -Property <propertyName>

# Search for a pattern in the output of a command
Get-Process | Select-Object * | Where-Object -Property Name -Like "*<pattern>*"

# Search for a pattern in the output of a pip command
pip freeze | Select-String pandas
pip list | FindStr "pandas"

# Search for a pattern in a file
Get-Content <fileName> | Select-String -Pattern <pattern> -CaseSensitive

# Search for a pattern in a file and return the first 10 matches
Get-Content <fileName> | Select-String -Pattern <pattern> -CaseSensitive | Select-Object -First 10

```

### Checks if module is installed
```shell
Get-Module -ListAvailable -Name <moduleName> 
Get-Module -ListAvailable | Where-Object -Property Name -Eq <moduleName> | Select-Object *
```

### Tee-Object
```shell
# Capture all processes to a log file while only passing the high-CPU processes further down the pipeline
Get-Process | Tee-Object -FilePath "processes.log" | Where-Object {$_.CPU -gt 10} 

# Using the -Variable parameter to store output in a variable for later reference
Get-Process | Tee-Object -Variable HighCpuProcesses | Where-Object {$_.CPU -gt 10}
Get-Variable HighCpuProcesses | Select-Object -ExpandProperty Value
```

### Manage services
```shell
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Service <service_name> | Get-Member   # returns list of attributes and methods

Get-Service | Out-File -FilePath <path> | ConvertTo-Csv
Get-Service | Export-Clixml -Path <path> | Compare-Object -ReferenceObject (Import-Clixml <path>) -Diff -Process -Property [property]

Start-Service -Name <serviceName> [-ArgumentList "<args>"] # -ArgumentList "`"$SolutionPath`"" 
Stop-Service -Name <serviceName>
```

### Manage processes
```shell
Get-Process | Where-Object {$_.CPU -gt 10}
Get-Process <process_name> | Get-Member   # returns list of attributes and methods
Get-Process | Select-Object -Property Name, Id, CPU, Handles, StartTime | Sort-Object -Property CPU -Descending

Start-Process -Name <processName>
Stop-Process -Name <processName> [-Force]
```

### Variables
```shell
$x = "Hello, World!"
$myArray = @("Item1", "Item2", "Item3")
$myHashTable = @{
	Key1 = "Value1"
	Key2 = "Value2"
}
$_  									# represents the current object in the pipeline
$PSBoundParameters  					# contains all parameters passed to a function
$PSCmdlet.MyInvocation.MyCommand.Name   # returns the name of the current command
$PSVersionTable.PSVersion  				# returns the PowerShell version
$env:PATH  								# returns the PATH environment variable
$env:COMPUTERNAME  						# returns the computer name
```

### Remoting
Session = persistent connection to a remote computer
```shell
# Enable PowerShell Remoting
Enable-PSRemoting

# List all available sessions
Get-PSSession

# Enter/Exit a remote session
Enter-PSSession -ComputerName $env:COMPUTERNAME 
Enter-PSSession -HostName <hostname> -UserName <username> -SSHTransport # SSH session
Exit-PSSession

# Create/remove a new session
New-PSSession -ComputerName $env:COMPUTERNAME
Remove-PSSession -Session (Get-PSSession)

# Invoke a command on a remote session
Invoke-Command -Session (Get-PSSession) -ScriptBlock { Get-Process }

# Copy files to/from a remote session
Copy-Item -Path "C:\path\to\file.txt" -Destination "C:\path\to\remote\file.txt" -ToSession (Get-PSSession)
Copy-Item -Path "C:\path\to\remote\file.txt" -Destination "C:\path\to\local\file.txt" -FromSession (Get-PSSession)
```


### Network
```shell
# Show Wi-Fi profiles
netsh wlan show profiles key='clear'

# Show IP configuration
Get-NetIPConfiguration

# Show network adapters
Get-NetAdapter | Select-Object -Property Name, Status, MacAddress, LinkSpeed

# Show network connections
Get-NetTCPConnection | Select-Object -Property LocalAddress, LocalPort, RemoteAddress, RemotePort, State
```

### Curl
```shell
# Download a file
Invoke-WebRequest -Uri "https://www.url.com" -OutFile "C:\path\to\file.txt"

# POST request with body
$body = @{
	key1 = "value1"
	key2 = "value2"
}
$body = $body | ConvertTo-Json

# For JSON body, use -ContentType "application/json"
Invoke-WebRequest -Uri "https://www.url.com" -Method Post -Body $body -ContentType "application/json"
```

### File Operations
```shell
# File operations
Remove-Item -Path "C:\path\to\file.txt" 
Copy-Item -Path "C:\path\to\source.txt" -Destination "C:\path\to\destination.txt"  
Move-Item -Path "C:\path\to\source.txt" -Destination "C:\path\to\destination.txt" 
New-Item -Path "C:\path\to\newfile.txt" -ItemType [File | Directory]

# List files in a directory (alias: ls, dir)
Get-ChildItem -Path "C:\path\to\directory" -Recurse | Where-Object { $_.Extension -eq ".txt" }

# Search for a specific file
Get-ChildItem -Path "C:\path\to\directory" -Recurse | Where-Object { $_.Name -like "*searchTerm*" }

# Get file metadata
Get-Item -Path "C:\path\to\file.txt" | Select-Object Name, Length, LastWriteTime

# Read a file (alias: cat, type)
Get-Content -Path "C:\path\to\file.txt" | Select-String "search term"
(Get-Content -Path "C:\path\to\file.txt" -Raw) -split '\r?\n' | Where-Object { $_ -match '\S' }

# Write to a file
"Hello, World!" | Out-File -FilePath "C:\path\to\file.txt"		# Create new file
"Hello, PowerShell!" | Set-Content -Path "C:\path\to\file.txt"	# Overwrite the file
"Hello, Universe!" | Add-Content -Path "C:\path\to\file.txt"	# Append to the file

# Replace content using wildcards in a file
(Get-Content -Path "C:\path\to\file.txt") -replace "Hello.*", "Goodbye" | Set-Content -Path "C:\path\to\file.txt"
```


### Module that contains functions (scope = current session)
```shell
Import-Module <moduleName>.ps1
<functionName> [args]
	
Get-Command -CommandType Function
Get-ChildItem Function:<functionName> | select *
```

### Paths 
```shell
# Retrieve the parent folder of the current script
Split-Path -Parent $PSScriptRoot

# Retrieve the "grandparent" folder
Split-Path -Parent (Split-Path -Parent $PSScriptRoot)

# Join paths
Join-Path -Path $RootPath -ChildPath "Data\Test"
Join-Path -Path $RootPath -ChildPath "Unittests.sql"

# Get the current script directory
$CurrentPath = Split-Path -Parent $MyInvocation.MyCommand.Path
$CurrentPath = $CurrentPath -replace '\\', '/'  

# Test if a path exists
Test-Path -Path "C:\path\to\file.txt"


```

### Database Connection
```shell
Invoke-Sqlcmd -ServerInstance $env:COMPUTERNAME -Database "master" -InputFile "D:\Projects\PortefolioWebsite\Projects\CustomerChurn\Scripts\SQLQuery_ImportRawData.sql" -Variable @{Param="D:\Projects\PortefolioWebsite\Projects\CustomerChurn\Datasets\"}
```

### Scopes, certificates, and security
```shell
# Get the current scope
Get-PSCallStack

################
# Certificates #
################

## Get the current certificate
Get-ChildItem -Path Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*CN=*" } | Select-Object Subject, Thumbprint

## Create a self-signed certificate
New-SelfSignedCertificate -DnsName "localhost" -CertStoreLocation "Cert:\CurrentUser\My" -NotAfter (Get-Date).AddYears(1)

## Get the certificate thumbprint
$cert = Get-ChildItem -Path Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*CN=localhost*" }
$thumbprint = $cert.Thumbprint

## Export the certificate to a file
Export-Certificate -Cert $cert -FilePath "C:\path\to\certificate.cer" -Type CERT

## Import the certificate from a file
Import-Certificate -FilePath "C:\path\to\certificate.cer" -CertStoreLocation "Cert:\CurrentUser\My"


######################
# Execution policies #
######################

## Get the current execution policy
Get-ExecutionPolicy -List

## RemoteSigned = requires scripts downloaded from the internet to be signed by a trusted publisher
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope [ LocalMachine | CurrentUser | Process]

## AllSigned = requires all scripts to be signed by a trusted publisher
Set-ExecutionPolicy -ExecutionPolicy AllSigned -Scope [ LocalMachine | CurrentUser | Process]

## Unrestricted = Give a script permission to run
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope [ LocalMachine | CurrentUser | Process]

## Bypass =  Bypass the execution policy for the current session
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope [ LocalMachine | CurrentUser | Process]

```


# XML

```powershell
# Load XML from a file
# [xml] converts content to an XML document object
$xml = [xml](Get-Content -Path "file.xml") 

# Load XML from a web resource
$xml = [xml](Invoke-WebRequest -Uri "https://example.com/data.xml").Content

# Access elements
$xml.root.element                # Access the 'element' under 'root'
$xml.root.childElement.value     # Navigate deeper in the hierarchy

# Access attributes
$xml.root.element.GetAttribute("name")
$xml.root.element.name           # Alternative for attributes

# Find elements by name
$xml.SelectNodes("//element")    # Find all elements named 'element'
$xml.SelectSingleNode("//element[@name='specific']")  # Find by attribute

# Create new elements
$newElement = $xml.CreateElement("newElement")
$newElement.InnerText = "Element content"
$xml.root.AppendChild($newElement)

# Add attributes
$xml.root.element.SetAttribute("attribute", "value")

# Remove elements
$nodeToRemove = $xml.SelectSingleNode("//elementToRemove")
$nodeToRemove.ParentNode.RemoveChild($nodeToRemove)

# Save to a file
$xml.Save("output.xml")

# Get XML as a string
$xmlString = $xml.OuterXml

# Pretty-formatted XML string
$stringWriter = New-Object System.IO.StringWriter
$xmlWriter = New-Object System.Xml.XmlTextWriter($stringWriter)
$xmlWriter.Formatting = [System.Xml.Formatting]::Indented # Indent the XML for readability
$xml.WriteTo($xmlWriter)
$xmlWriter.Flush()
$formattedXml = $stringWriter.ToString()

# Convert XML to JSON
$xmlJson = $xml | ConvertTo-Json -Depth 10

# Convert JSON back to XML
$jsonString = '{"root":{"element":"value"}}'
$jsonObject = $jsonString | ConvertFrom-Json
$xmlFromJson = [xml]"<root><element>$($jsonObject.root.element)</element></root>"

# Practical Example
# Load an application config file
$configPath = "C:\App\config.xml"
$config = [xml](Get-Content $configPath)

# Modify a connection string
$config.configuration.connectionStrings.add | 
    Where-Object { $_.name -eq "MainDB" } |
    ForEach-Object { $_.connectionString = "Server=newserver;Database=MainDB;Trusted_Connection=True;" }

# Add a new app setting
$newSetting = $config.CreateElement("add")
$newSetting.SetAttribute("key", "FeatureFlag")
$newSetting.SetAttribute("value", "true")
$config.configuration.appSettings.AppendChild($newSetting)

# Save the modified config
$config.Save($configPath)
```


### Functions
```powershell
function MyFunction {
	param (
		[Parameter(Mandatory = $true)]
		[ValidateNotNullOrEmpty()]
		[string]$Name,
		
		[Parameter(Mandatory=$false)]
		[switch]$IsFlag = $false,
		
		[Parameter(Mandatory=$false)]
		[bool]$IsBoolean = $false
	)
	try {
		Write-Host "Hello, $Name!"
		
		if ($IsFlag) {
			$str = @"
This is a multi-line raw string.
It can contain special characters like $ and `n.
Formatting is preserved.
"@
		}
		
		if (-not $IsBoolean) {
			Get-Command -Name "Get-Process" -ErrorAction Stop |
				ForEach-Object {
					Write-Host "Found command: $($_.Name)"
				}
		}
	} catch {
		Write-Error "An error occurred: $_"
	} finally {
		Write-Host "Function execution completed."
	}
}


function Restart-VisualStudio {
    param (
        [switch]$Force = $false,
        [int]$WaitSeconds = 3
    )

    # Find running Visual Studio processes
    $vsProcesses = Get-Process -Name "devenv" -ErrorAction SilentlyContinue
    
    if ($vsProcesses) {
        # Get the solution path from command line arguments
        $solutionPath = $null
        foreach ($proc in $vsProcesses) {
            try {
                $cmdLine = (Get-WmiObject -Class Win32_Process -Filter "ProcessId = $($proc.Id)").CommandLine
                if ($cmdLine -match '(?<=")(.*\.sln)(?=")') {
                    $solutionPath = $matches[0]
                    break
                }
            }
            catch {
                Write-Warning "Could not retrieve command line for process $($proc.Id)"
            }
        }
        
        Write-Host "Found Visual Studio with solution: $solutionPath"
        
        # Close Visual Studio
        if ($Force) {
            $vsProcesses | ForEach-Object { 
                Write-Host "Force closing Visual Studio (ID: $($_.Id))"
                $_ | Stop-Process -Force 
            }
        } else {
            $vsProcesses | ForEach-Object { 
                Write-Host "Gracefully closing Visual Studio (ID: $($_.Id))"
                $_.CloseMainWindow() | Out-Null
            }
        }
        
        # Wait for processes to close
        Write-Host "Waiting $WaitSeconds seconds for Visual Studio to close..."
        Start-Sleep -Seconds $WaitSeconds
        
        # Reopen Visual Studio with the same solution
        if ($solutionPath) {
            Write-Host "Restarting Visual Studio with solution: $solutionPath"
            Start-Process "devenv.exe" -ArgumentList "`"$solutionPath`""
        } else {
            Write-Host "Restarting Visual Studio (no solution found)"
            Start-Process "devenv.exe"
        }
    } else {
        Write-Host "No running Visual Studio instances found."
        Start-Process "devenv.exe"
    }
}

```


### Database connection
#### PostgreSQL example
```powershell
# Define the connection string 
$connectionString = "Server=localhost;Port=5432;Database=mydb;User Id=postgres;Password=password;"

# Create a SQL connection
# Make sure Npgsql is installed: Install-Package Npgsql
$sqlConnection = New-Object Npgsql.NpgsqlConnection
$sqlConnection.ConnectionString = $connectionString

try {
    # Open the connection
    $sqlConnection.Open()
    Write-Host "Database connection successful."

    # Execute a simple query
    $sqlCommand = $sqlConnection.CreateCommand()
    $sqlCommand.CommandText = "SELECT * FROM myTable LIMIT 10"
    $sqlDataReader = $sqlCommand.ExecuteReader()

    # Process the results
    while ($sqlDataReader.Read()) {
        Write-Host "Row: $($sqlDataReader[0]), $($sqlDataReader[1])"
    }
} catch {
    Write-Error "Database connection failed: $_"
} finally {
    # Clean up
    $sqlDataReader.Close()
    $sqlConnection.Close()
}
```

#### SQL Server example
```PowerShell
# Create a connection
$connectionString = "Server=myServerAddress;Database=myDataBase;User Id=myUsername;Password=myPassword;"
$sqlConnection = New-Object System.Data.SqlClient.SqlConnection
$sqlConnection.ConnectionString = $connectionString
$sqlConnection.Open()

# Create a SQL command
$sqlCommand = $sqlConnection.CreateCommand()
$sqlCommand.CommandText = "SELECT * FROM myTable LIMIT 10"
$sqlDataReader = $sqlCommand.ExecuteReader()

# Process the results
while ($sqlDataReader.Read()) {
    Write-Host "Row: $($sqlDataReader[0]), $($sqlDataReader[1])"
}

# Alternative way to execute: Invoke-Sqlcmd
Invoke-Sqlcmd -Query "SELECT * FROM myTable LIMIT 10" -ConnectionString $connectionString
Invoke-Sqlcmd -Query "EXEC myStoredProcedure" -ConnectionString $connectionString

# Clean up
$sqlDataReader.Close()
$sqlConnection.Close()
```