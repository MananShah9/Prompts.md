# RestartTasks.ps1
$tasks = @("temp1", "temp2", "temp3", "temp4", "temp5")

foreach ($task in $tasks) {
    try {
        # Stop the task if it is running
        if ((Get-ScheduledTask -TaskName $task).State -eq "Running") {
            Stop-ScheduledTask -TaskName $task
            Start-Sleep -Seconds 2 # Wait a bit before restarting
        }

        # Start the task
        Start-ScheduledTask -TaskName $task
        Write-Host "$task restarted successfully"
    }
    catch {
        Write-Host "Error restarting $task: $_"
    }
}
