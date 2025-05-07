# TCP Connection Monitoring Script
# This script monitors specified TCP ports and logs detailed connection information
# including process details, user information, and connection statistics

param(
    # Optional parameter for log file directory
    [string]$LogDir = (Split-Path -Parent $MyInvocation.MyCommand.Path)
)

# List of target ports to monitor (temporarily adding common ports for testing)
$ports = @(7077, 9099, 8088)  # common ports for testing output - 80, 443, 3389, 445

function Write-LogAndConsole {
    param(
        $Message,
        $ForegroundColor = "White",
        [switch]$NoNewLine
    )
    
    if ($NoNewLine) {
        Write-Host $Message -ForegroundColor $ForegroundColor -NoNewline
    } else {
        Write-Host $Message -ForegroundColor $ForegroundColor
    }
    Add-Content -Path $logFile -Value "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss'): $Message"
}

# Create log file with timestamp
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$logFile = Join-Path $LogDir "tcp_connections_$timestamp.log"

Write-LogAndConsole "Starting TCP connection monitoring. Log file: $logFile" -ForegroundColor Cyan
Write-LogAndConsole "----------------------------------------" -ForegroundColor Cyan

while ($true) {
    Clear-Host
    $currentTime = Get-Date

    Write-LogAndConsole "Checking TCP connections to ports: $($ports -join ', ') on ALL local IP addresses and interfaces at $currentTime" -ForegroundColor Cyan
    Write-LogAndConsole "(This includes connections from remote clients and local processes)" -ForegroundColor Yellow

    # Get all TCP connections to the specified ports
    $connections = Get-NetTCPConnection | Where-Object {
        $ports -contains $_.LocalPort
    }

    if ($connections) {
        # Group by LocalPort and count connections
        $grouped = $connections | Group-Object LocalPort | Sort-Object Name
        Write-LogAndConsole "`nConnection counts per port:" -ForegroundColor Green
        foreach ($g in $grouped) {
            Write-LogAndConsole ("Port {0}: {1} connections" -f $g.Name, $g.Count) -ForegroundColor White
        }

        Write-LogAndConsole "`nConnection details:" -ForegroundColor Green

        # Create an array of connection objects with all details
        $connectionDetails = $connections | ForEach-Object {
            # Get process details
            $process = try { Get-Process -Id $_.OwningProcess -ErrorAction Stop } catch { $null }
            $procName = if ($process) { $process.ProcessName } else { "N/A" }
            
            # Get process owner and session info
            $procDetails = try {
                $proc = Get-WmiObject Win32_Process -Filter "ProcessId = $($_.OwningProcess)"
                $owner = $null
                $domain = $null
                $proc.GetOwner([ref]$owner, [ref]$domain)
                @{
                    Owner = $owner
                    Domain = $domain
                    SessionId = $proc.SessionId
                    CreationDate = $proc.CreationDate
                }
            } catch { @{ Owner = "N/A"; Domain = "N/A"; SessionId = "N/A"; CreationDate = "N/A" } }

            # Calculate connection duration
            $connectionAge = if ($_.CreationTime) {
                $duration = $currentTime - $_.CreationTime
                "{0:dd}d {0:hh}h {0:mm}m" -f $duration
            } else { "Unknown" }

            # Get idle time
            $idleTime = if ($_.State -eq 'Established') { 
                try { 
                    $idle = (Get-NetTCPConnection -OwningProcess $_.OwningProcess -State Established -ErrorAction Stop).IdleTime
                    if ($idle) { "{0:hh}h {0:mm}m" -f $idle }
                    else { "N/A" }
                } catch { "N/A" }
            } else { "N/A" }

            [PSCustomObject]@{
                'Local'          = "$($_.LocalAddress):$($_.LocalPort)"
                'Remote'         = "$($_.RemoteAddress):$($_.RemotePort)"
                'State'          = $_.State
                'Age'            = $connectionAge
                'Idle'           = $idleTime
                'Process'        = "$procName ($($_.OwningProcess))"
                'User'           = "$($procDetails.Domain)\$($procDetails.Owner)"
                'Session'        = $procDetails.SessionId
            }
        }

        # Output the connections in a table format
        $tableOutput = $connectionDetails | Format-Table -AutoSize | Out-String
        Write-LogAndConsole $tableOutput
    } else {
        Write-LogAndConsole "No connections found to the specified ports at this time." -ForegroundColor Yellow
    }

    Write-LogAndConsole "----------------------------------------`n"
    Start-Sleep -Seconds 10
}