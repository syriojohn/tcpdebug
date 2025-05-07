# TCP Connection Monitor

This PowerShell script monitors TCP connections on specified ports and provides detailed information about each connection, including process details, user information, and connection statistics.

## Features

- Monitors specified TCP ports (default: 7077, 9099, 8088)
- Logs all connection information to a timestamped file
- Provides detailed information for each connection:
  - Connection age and idle time
  - Process details (name, PID, start time)
  - User and session information
  - Connection state and endpoints
- Real-time console output with automatic refresh
- Configurable log file location

## Usage

### Basic Usage

```powershell
.\tcp_debug.ps1
```

This will run the script with default settings and create log files in the same directory as the script.

### Specify Custom Log Directory

```powershell
.\tcp_debug.ps1 -LogDir "C:\CustomLogPath"
```

### Running on Remote Server

To run this on a remote server, you can:
1. Copy the script to the server
2. Run it using PowerShell with appropriate permissions

```powershell
# Example running from a remote session
Enter-PSSession -ComputerName SERVERNAME
.\tcp_debug.ps1 -LogDir "D:\Logs"
```

## Output

The script generates two types of output:
1. Console output with color-coded information
2. Log file with timestamp (format: tcp_connections_YYYYMMDD_HHMMSS.log)

## Requirements

- Windows PowerShell 5.1 or later
- Administrative privileges (for accessing process and connection information)
- Appropriate execution policy (can be bypassed using):
  ```powershell
  powershell -ExecutionPolicy Bypass -File tcp_debug.ps1
  ```

## Stopping the Script

There are several ways to stop the script:

1. **If you have access to the PowerShell window running the script:**
   - Press Ctrl+C in the PowerShell window

2. **Using PowerShell commands (requires admin privileges):**
   ```powershell
   # Find and stop the script by looking for PowerShell processes running tcp_debug.ps1
   Get-Process | Where-Object {$_.ProcessName -eq 'powershell' -and $_.CommandLine -like '*tcp_debug.ps1*'} | Stop-Process -Force

   # Alternative: If you know the Process ID (PID), you can stop it directly
   Stop-Process -Id <PID> -Force
   ```

3. **Using Task Manager (requires admin privileges):**
   1. Open Task Manager (Ctrl+Shift+Esc)
   2. Go to the "Details" tab
   3. Look for "powershell.exe" processes
   4. Right-click and select "End Task" on the process running tcp_debug.ps1
   
4. **Finding and stopping from remote machine (requires admin privileges):**
   ```powershell
   # Connect to remote machine and stop the script
   $computerName = "SERVERNAME"
   Invoke-Command -ComputerName $computerName -ScriptBlock {
       Get-Process | Where-Object {$_.ProcessName -eq 'powershell' -and $_.CommandLine -like '*tcp_debug.ps1*'} | Stop-Process -Force
   }
   ```

Note: If multiple instances of the script are running, these commands will stop all instances. To stop a specific instance, you'll need to identify its specific PID first using:
```powershell
Get-Process | Where-Object {$_.ProcessName -eq 'powershell' -and $_.CommandLine -like '*tcp_debug.ps1*'} | Select-Object Id, StartTime, CPU
