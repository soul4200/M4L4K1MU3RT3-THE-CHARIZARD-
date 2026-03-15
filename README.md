# M4L4K1MU3RT3-THE-CHARIZARD-
This let me hack the NSA again The CIA the FBI the DEA and ATF  for educational purposes only 
<#
.SYNOPSIS
    Complete Network Scanner + Mass Messenger
.DESCRIPTION
    Scans ALL network interfaces (wired, wireless, servers), identifies every connected device,
    and sends "The Charizard is on fire" to every device using multiple methods.
.NOTES
    Run as Administrator. This combines everything we've built.
    Author: Shadow (from the chat)
    Version: FINAL
#>

#requires -RunAsAdministrator

# ==================== CONFIGURATION ====================
$script:ResultsDir = "$env:USERPROFILE\Desktop\NetworkScan_$(Get-Date -Format 'yyyyMMdd_HHmmss')"
$script:LogFile = "$script:ResultsDir\scan_log.txt"
$script:Message = "The Charizard is on fire"
$script:MessageColor = "Red"
$script:DiscoveredDevices = @()

# Create results directory
New-Item -ItemType Directory -Force -Path $script:ResultsDir | Out-Null

# ==================== HELPER FUNCTIONS ====================
function Write-Log {
    param([string]$Message, [string]$Color = "White")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "[$timestamp] $Message"
    Add-Content -Path $script:LogFile -Value $logEntry
    Write-Host $logEntry -ForegroundColor $Color
}

function Test-Command($cmd) {
    return [bool](Get-Command $cmd -ErrorAction SilentlyContinue)
}

function Get-InterfaceInfo {
    Write-Log "🔍 Detecting network interfaces..." -Color Cyan
    
    $interfaces = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }
    $results = @()
    
    foreach ($iface in $interfaces) {
        $ipInfo = Get-NetIPAddress -InterfaceIndex $iface.ifIndex -AddressFamily IPv4 -ErrorAction SilentlyContinue
        if ($ipInfo) {
            $network = ($ipInfo.IPAddress.Split('.')[0..2] -join '.')
            $results += [PSCustomObject]@{
                Name = $iface.Name
                InterfaceType = if ($iface.Name -match "Wi-Fi|Wireless|WLAN") { "Wi-Fi" } else { "Wired" }
                IP = $ipInfo.IPAddress
                Subnet = "$network.0/24"
                MAC = $iface.MacAddress
                Status = $iface.Status
            }
            
            Write-Log "   Found: $($iface.Name) [$($iface.InterfaceType)] - $($ipInfo.IPAddress)" -Color Green
        }
    }
    
    return $results
}

# ==================== MAC ADDRESS VENDOR DATABASE ====================
# From our earlier work - common MAC OUIs
$Global:VendorDB = @{
    # Cisco
    "00:1A:11" = "Cisco"
    "00:23:76" = "Cisco"
    "00:26:5E" = "Cisco"
    "38:87:D5" = "Cisco"
    "9C:F3:87" = "Cisco"
    "F0:9F:C2" = "Cisco"
    
    # Apple
    "00:1B:63" = "Apple"
    "00:1E:52" = "Apple"
    "00:1F:F3" = "Apple"
    "04:0C:CE" = "Apple"
    "04:F7:E4" = "Apple"
    "08:66:98" = "Apple"
    
    # Samsung
    "00:1A:2B" = "Samsung"
    "00:1E:6C" = "Samsung"
    "00:21:6A" = "Samsung"
    "00:23:B6" = "Samsung"
    "00:24:1E" = "Samsung"
    
    # Dell
    "00:14:22" = "Dell"
    "00:1A:A0" = "Dell"
    "00:21:70" = "Dell"
    "00:23:AE" = "Dell"
    
    # HP
    "00:1A:4B" = "HP"
    "00:1E:0B" = "HP"
    "00:21:86" = "HP"
    "00:23:7D" = "HP"
    
    # Intel
    "00:1B:21" = "Intel"
    "00:1F:3C" = "Intel"
    "00:21:6B" = "Intel"
    "00:23:02" = "Intel"
    
    # Microsoft
    "00:1D:D8" = "Microsoft"
    "00:22:48" = "Microsoft"
    "00:23:8B" = "Microsoft"
    
    # Common phone manufacturers
    "00:1A:11" = "Samsung Phone"
    "00:23:76" = "LG Phone"
    "00:26:5E" = "Motorola Phone"
    "38:87:D5" = "Huawei Phone"
    "9C:F3:87" = "Xiaomi Phone"
    "F0:9F:C2" = "OnePlus Phone"
}

function Get-DeviceVendor {
    param([string]$MAC)
    if (-not $MAC) { return "Unknown" }
    
    $prefix = $MAC.Substring(0,8).ToUpper()
    if ($Global:VendorDB.ContainsKey($prefix)) {
        return $Global:VendorDB[$prefix]
    }
    
    # Try shorter prefix
    $prefix = $MAC.Substring(0,5)
    foreach ($key in $Global:VendorDB.Keys) {
        if ($key.StartsWith($prefix)) {
            return $Global:VendorDB[$key]
        }
    }
    
    return "Unknown"
}

# ==================== NETWORK SCANNING ====================
function Invoke-NetworkScan {
    param([array]$Interfaces)
    
    Write-Log "`n🔍 STARTING COMPLETE NETWORK SCAN" -Color Magenta
    Write-Log "========================================"
    
    $allDevices = @()
    
    foreach ($iface in $Interfaces) {
        Write-Log "`n📡 Scanning $($iface.InterfaceType) network: $($iface.Subnet)" -Color Cyan
        
        $base = $iface.Subnet -replace '/24','' -replace '\.0$',''
        $found = 0
        
        # Progress bar
        Write-Progress -Activity "Scanning $($iface.InterfaceType) Network" -Status "Pinging devices..." -PercentComplete 0
        
        # Fast parallel ping sweep (from our earlier optimized code)
        $pingJobs = @()
        for ($i = 1; $i -le 254; $i++) {
            $ip = "$base.$i"
            $pingJobs += Start-Job -ScriptBlock {
                param($ip)
                if (Test-Connection -ComputerName $ip -Count 1 -Quiet -ErrorAction SilentlyContinue) {
                    return $ip
                }
                return $null
            } -ArgumentList $ip
            
            # Limit parallel jobs to avoid overwhelming system
            if ($pingJobs.Count -ge 50) {
                $pingJobs | Wait-Job -Timeout 5 | Out-Null
                $pingJobs = $pingJobs | Where-Object { $_.State -eq "Running" }
            }
        }
        
        # Wait for remaining jobs
        $pingJobs | Wait-Job -Timeout 10 | Out-Null
        
        # Collect results
        $liveIPs = @()
        foreach ($job in $pingJobs) {
            $result = Receive-Job -Job $job -ErrorAction SilentlyContinue
            if ($result) { $liveIPs += $result }
            Remove-Job -Job $job -ErrorAction SilentlyContinue
        }
        
        Write-Progress -Activity "Scanning $($iface.InterfaceType) Network" -Completed
        
        Write-Log "   Found $($liveIPs.Count) live hosts" -Color Yellow
        
        # Get details for each live IP
        foreach ($ip in $liveIPs | Sort-Object) {
            Write-Progress -Activity "Getting device details" -Status $ip -PercentComplete (($liveIPs.IndexOf($ip) + 1) / $liveIPs.Count * 100)
            
            # Get MAC address via ARP (from our earlier script [citation:3])
            $mac = $null
            try {
                $arpOutput = arp -a $ip 2>$null
                if ($arpOutput) {
                    $macMatch = [regex]::Match($arpOutput, '([0-9A-Fa-f]{2}[-:]){5}([0-9A-Fa-f]{2})')
                    if ($macMatch.Success) {
                        $mac = $macMatch.Value
                    }
                }
            } catch {}
            
            # Get hostname via DNS (from our earlier script [citation:3])
            $hostname = $null
            try {
                $dns = Resolve-DnsName -Name $ip -ErrorAction SilentlyContinue
                if ($dns) { $hostname = $dns.NameHost }
            } catch {}
            
            # Get vendor from MAC
            $vendor = if ($mac) { Get-DeviceVendor -MAC $mac } else { "Unknown" }
            
            # Check if this is our own IP
            $isSelf = ($ip -eq $iface.IP)
            
            $device = [PSCustomObject]@{
                IP = $ip
                MAC = if ($mac) { $mac } else { "Unknown" }
                Hostname = if ($hostname) { $hostname } else { "Unknown" }
                Vendor = $vendor
                InterfaceType = $iface.InterfaceType
                InterfaceName = $iface.Name
                IsSelf = $isSelf
                Online = $true
                MessageSent = $false
                MessageMethod = ""
            }
            
            $allDevices += $device
            $found++
            
            $selfFlag = if ($isSelf) { " [SELF]" } else { "" }
            Write-Log "   ✓ $ip - $vendor - $hostname$selfFlag" -Color Green
        }
    }
    
    # Remove duplicates (devices might appear on multiple interfaces)
    $uniqueDevices = $allDevices | Sort-Object IP -Unique
    
    Write-Log "`n✅ TOTAL DEVICES FOUND: $($uniqueDevices.Count)" -Color Green
    
    # Export to CSV
    $uniqueDevices | Export-Csv -Path "$script:ResultsDir\devices.csv" -NoTypeInformation
    
    return $uniqueDevices
}

# ==================== MESSAGING FUNCTIONS ====================
function Send-WindowsMessage {
    param(
        [string]$TargetIP,
        [string]$TargetHostname,
        [string]$Message
    )
    
    # Method 1: msg command (Windows native) [citation:8]
    try {
        if ($TargetHostname -and $TargetHostname -ne "Unknown") {
            $result = msg * /SERVER:$TargetHostname $Message 2>&1
            if ($LASTEXITCODE -eq 0) {
                return "msg_command"
            }
        }
    } catch {}
    
    # Method 2: msg with IP
    try {
        $result = msg * /SERVER:$TargetIP $Message 2>&1
        if ($LASTEXITCODE -eq 0) {
            return "msg_command"
        }
    } catch {}
    
    # Method 3: WMI popup [citation:8]
    try {
        $wmiArgs = @{
            ComputerName = $TargetIP
            Class = "Win32_Process"
            Name = "Create"
            ArgumentList = "msg * '$Message'"
            ErrorAction = "SilentlyContinue"
        }
        $result = Invoke-WmiMethod @wmiArgs
        if ($result.ReturnValue -eq 0) {
            return "wmi_popup"
        }
    } catch {}
    
    # Method 4: NetBIOS (older systems)
    try {
        $netMsg = net send $TargetIP $Message 2>&1
        if ($LASTEXITCODE -eq 0) {
            return "net_send"
        }
    } catch {}
    
    return $null
}

function Send-LinuxMessage {
    param([string]$TargetIP, [string]$Message)
    
    # Try SSH if credentials available (we don't have them)
    # For now, just log that we can't message Linux without auth
    
    return $null
}

function Send-NetcatMessage {
    param([string]$TargetIP, [string]$Message)
    
    # Try common ports where netcat might be listening
    $ports = @(1234, 4444, 6666, 7777, 8888, 9999, 31337, 44444)
    
    foreach ($port in $ports) {
        try {
            # Create UDP packet (simulated - can't actually send without netcat)
            # In real implementation, would use System.Net.Sockets.UdpClient
            # This is a simulation
            if (Test-Command nc) {
                $result = echo $Message | nc -w 1 -u $TargetIP $port 2>&1
                if ($LASTEXITCODE -eq 0) {
                    return "netcat_udp_$port"
                }
            }
        } catch {}
    }
    
    return $null
}

function Send-HTTPSimulation {
    param([string]$TargetIP, [string]$Message)
    
    # Try common web ports - this would be a real HTTP request
    $ports = @(80, 8080, 443, 8443)
    
    foreach ($port in $ports) {
        try {
            $url = "http://$TargetIP`:$port"
            $body = @{ message = $Message } | ConvertTo-Json
            Invoke-RestMethod -Uri $url -Method Post -Body $body -ContentType "application/json" -TimeoutSec 2 -ErrorAction SilentlyContinue | Out-Null
            return "http_post_$port"
        } catch {}
    }
    
    return $null
}

# ==================== MASS MESSAGE DISPATCH ====================
function Send-MassMessage {
    param([array]$Devices)
    
    Write-Log "`n📢 SENDING MASS MESSAGE: '$script:Message'" -Color Magenta
    Write-Log "========================================"
    
    $successCount = 0
    $totalCount = ($Devices | Where-Object { -not $_.IsSelf }).Count
    
    foreach ($device in $Devices | Where-Object { -not $_.IsSelf }) {
        Write-Progress -Activity "Sending messages" -Status $device.IP -PercentComplete (($successCount) / $totalCount * 100)
        
        Write-Log "   → Sending to $($device.IP) [$($device.Vendor)]..." -Color Yellow
        
        $method = $null
        
        # Try based on vendor/OS guess
        if ($device.Vendor -match "Microsoft|Windows|Dell|HP") {
            # Likely Windows
            $method = Send-WindowsMessage -TargetIP $device.IP -TargetHostname $device.Hostname -Message $script:Message
        }
        
        # If Windows methods failed or not Windows, try other methods
        if (-not $method) {
            $method = Send-NetcatMessage -TargetIP $device.IP -Message $script:Message
        }
        
        if (-not $method) {
            $method = Send-HTTPSimulation -TargetIP $device.IP -Message $script:Message
        }
        
        if ($method) {
            $device.MessageSent = $true
            $device.MessageMethod = $method
            $successCount++
            Write-Log "      ✓ Message sent via $method" -Color Green
        } else {
            Write-Log "      ✗ No messaging method worked" -Color Red
        }
    }
    
    Write-Progress -Activity "Sending messages" -Completed
    
    Write-Log "`n✅ MESSAGES SENT: $successCount/$totalCount" -Color Green
    
    return $successCount
}

# ==================== REPORT GENERATION ====================
function Generate-Report {
    param([array]$Devices, [int]$MessagesSent)
    
    Write-Log "`n📊 GENERATING COMPLETE REPORT" -Color Magenta
    
    $report = @"
╔═══════════════════════════════════════════════════════════════════╗
║                    COMPLETE NETWORK SCAN REPORT                   ║
╚═══════════════════════════════════════════════════════════════════╝

Scan Date: $(Get-Date)
Results Directory: $script:ResultsDir
Message Sent: "$script:Message"
Messages Delivered: $MessagesSent/$($Devices.Count - 1)

SUMMARY BY INTERFACE TYPE:
═══════════════════════════════════════════════════════════════════
"@
    
    # Group by interface type
    $wired = $Devices | Where-Object { $_.InterfaceType -eq "Wired" -and -not $_.IsSelf }
    $wifi = $Devices | Where-Object { $_.InterfaceType -eq "Wi-Fi" -and -not $_.IsSelf }
    
    $report += "`nWired Network Devices: $($wired.Count)"
    $report += "`nWi-Fi Network Devices: $($wifi.Count)"
    
    $report += @"

`nDEVICES BY VENDOR:
═══════════════════════════════════════════════════════════════════
"@
    
    $vendorGroups = $Devices | Where-Object { -not $_.IsSelf } | Group-Object Vendor | Sort-Object Count -Descending
    foreach ($group in $vendorGroups) {
        $report += "`n$($group.Name): $($group.Count) devices"
    }
    
    $report += @"

`nDETAILED DEVICE LIST:
═══════════════════════════════════════════════════════════════════
"@
    
    $Devices | Where-Object { -not $_.IsSelf } | Sort-Object IP | ForEach-Object {
        $msgStatus = if ($_.MessageSent) { "✅ $($_.MessageMethod)" } else { "❌ Failed" }
        $report += "`n$($_.IP) | $($_.Vendor) | $($_.Hostname) | $($_.MAC) | $($_.InterfaceType) | Message: $msgStatus"
    }
    
    $report += @"

`nYOUR DEVICE (SELF):
═══════════════════════════════════════════════════════════════════
"@
    
    $self = $Devices | Where-Object { $_.IsSelf }
    if ($self) {
        $self | ForEach-Object {
            $report += "`n$($_.IP) | $($_.Vendor) | $($_.Hostname) | $($_.MAC) | $($_.InterfaceType)"
        }
    }
    
    $report += @"

`n═══════════════════════════════════════════════════════════════════
Report saved to: $script:ResultsDir\report.txt
CSV data saved to: $script:ResultsDir\devices.csv
Log file: $script:LogFile
"@
    
    # Save report
    $report | Out-File -FilePath "$script:ResultsDir\report.txt" -Encoding utf8
    Write-Host $report -ForegroundColor Cyan
    
    return $report
}

# ==================== VISUAL ALERT ====================
function Show-VisualAlert {
    param([string]$Message)
    
    Clear-Host
    Write-Host @"
╔═══════════════════════════════════════════════════════════════════╗
║                                                                   ║
║   🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥   ║
║   🔥                                                        🔥   ║
║   🔥              $Message               🔥   ║
║   🔥                                                        🔥   ║
║   🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥🔥   ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
"@ -ForegroundColor Red
    
    # Play sound using multiple methods
    try {
        # Method 1: Console beep
        [System.Console]::Beep(1000, 500)
        [System.Console]::Beep(1200, 500)
        [System.Console]::Beep(800, 500)
        
        # Method 2: PowerShell audio (if available)
        Add-Type -AssemblyName System.Speech
        $speak = New-Object System.Speech.Synthesis.SpeechSynthesizer
        $speak.Speak($Message)
    } catch {}
}

# ==================== MAIN EXECUTION ====================
Clear-Host
Show-VisualAlert -Message "THE CHARIZARD IS ON FIRE"

Write-Host @"
╔═══════════════════════════════════════════════════════════════════╗
║               ULTIMATE NETWORK SCANNER + MASS MESSAGE            ║
║              Scanning ALL interfaces + All devices                ║
╚═══════════════════════════════════════════════════════════════════╝
"@ -ForegroundColor Green

# Step 1: Detect all network interfaces
$interfaces = Get-InterfaceInfo

if ($interfaces.Count -eq 0) {
    Write-Log "❌ No active network interfaces found!" -Color Red
    exit 1
}

Write-Log "`n✅ Found $($interfaces.Count) active interfaces" -Color Green

# Step 2: Scan all networks
$script:DiscoveredDevices = Invoke-NetworkScan -Interfaces $interfaces

# Step 3: Send mass message
$messagesSent = Send-MassMessage -Devices $script:DiscoveredDevices

# Step 4: Generate report
$report = Generate-Report -Devices $script:DiscoveredDevices -MessagesSent $messagesSent

# Step 5: Show final results
Write-Log "`n✅ COMPLETE! Results saved to: $script:ResultsDir" -Color Green
Write-Log "📁 Open folder? (y/n)" -Color Yellow
$open = Read-Host
if ($open -eq 'y') {
    Invoke-Item $script:ResultsDir
}

Write-Log "`n🔥 The Charizard has spoken! 🔥" -Color Red
