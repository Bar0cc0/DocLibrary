# POWERSHELL SCRIPTING

### Help
```shell
get-alias				# lists all aliases
get-command				# lists all commands
get-help <cmdlet_name>	# params:  -detailed -showwindow -online
get-help about_*
```

### GREP
```shell
# Search for a pattern in the command history
get-history | select * | where -property <propertyName> -like "*<pattern>*"
get-history | select * | where -property <propertyName> -eq <somevalue>
			| sort  -property <propertyName> [-descending]
			| group -property <propertyName>

# Search for a pattern in the output of a command
get-process | select * | where -property Name -like "*<pattern>*"

# Search for a pattern in the output of a pip command
pip freeze | select-string pandas
pip list | findstr "pandas"

# Search for a pattern in a file
get-content <fileName> | select-string -pattern <pattern> -caseSensitive

# Search for a pattern in a file and return the first 10 matches
get-content <fileName> | select-string -pattern <pattern> -caseSensitive | select -first 10

```

### Checks if module is installed
```shell
get-module -listavailable -name <moduleName> 
get-module -listavailable | where -property Name -eq <moduleName> [| select *]
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
get-service | where-object {$_.status -eq 'Running'}
get-service <service_name> | get-member   # returns list of attributes and methods

get-service | out-file -filepath <path> | convertto-csv
get-service | export-clixml -path <path> | compare-object -referenceobject (import-clixml <path>) -diff -process -property [property]

start-service -name <serviceName> [-WhatIf]
stop-service  -name <serviceName>
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
Remove-Item -Path "C:\path\to\file.txt" 
Copy-Item -Path "C:\path\to\source.txt" -Destination "C:\path\to\destination.txt"  
Move-Item -Path "C:\path\to\source.txt" -Destination "C:\path\to\destination.txt" 
New-Item -Path "C:\path\to\newfile.txt" -ItemType [File | Directory]

# List files in a directory (alias: ls, dir)
Get-ChildItem -Path "C:\path\to\directory" -Recurse | Where-Object { $_.Extension -eq ".txt" }
# Read a file (alias: cat, type)
Get-Content -Path "C:\path\to\file.txt" | Select-String "search term"
(Get-Content -Path "C:\path\to\file.txt" -Raw) -split '\r?\n' | Where-Object { $_ -match '\S' }
```


### Module that contains functions (scope = current session):
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
```