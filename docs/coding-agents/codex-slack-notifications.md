# Get Slack notifications when Codex needs approval or finishes

*Last updated: 9 October 2026*

When you work with ChatGPT Codex in VS Code, it does not tell you when it is stuck waiting for your approval or when it has finished. This page sets up a Codex hook that sends you a Slack message, and shows a Windows desktop notification, in both cases.

The setup is for Windows, including Codex running in WSL. Replace `YOUR_USERNAME` with your Windows username wherever it appears.

## How it works

Codex can run a command, called a hook, when certain events happen. This page registers the PowerShell script `slack_notifier.ps1` for two events:

- **`PermissionRequest`**: Codex is waiting for your approval. Slack receives **CODEX NEEDS ATTENTION** with the project folder, the tool Codex wants to use and the reason.
- **`Stop`**: Codex has finished. Slack receives **CODEX FINISHED** with the project folder and Codex's last message.

Both events also show a Windows desktop notification. Long reasons and messages are cut to 1,000 characters.

The script reads the Slack webhook URL from the `CODEX_SLACK_WEBHOOK` environment variable. If the variable is missing or Slack cannot be reached, the script skips the message without an error, so a failed notification never interrupts Codex.

## Step 1: Create a Slack incoming webhook

1. Create a Slack account.
2. Create the channel where you want to receive the messages.
3. Go to [api.slack.com/apps](https://api.slack.com/apps).
4. Create an app.
5. Go to **Incoming Webhooks** and create a webhook for that channel. Copy its URL for the next step.

!!! warning "Keep the webhook URL private"
    Anyone who has the URL can post messages to your channel. Keep it out of pages, repositories and screenshots.

## Step 2: Save the webhook URL in an environment variable

Open the Start menu, search for **environment variables** and open **Edit environment variables for your account**. Under **User variables**, click **New...** and enter:

- **Variable name:** `CODEX_SLACK_WEBHOOK`
- **Variable value:** your webhook URL, for example `https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxx`

Click **OK** in both windows.

## Step 3: Save the notifier script

Create the folder `C:\Users\YOUR_USERNAME\.codex\hooks\` and save this script in it as `slack_notifier.ps1`:

??? example "slack_notifier.ps1"

    ```powershell
    $ErrorActionPreference = "Stop"

    function Show-DesktopNotification {
        param(
            [string]$Title,
            [string]$Message
        )

        try {
            Add-Type -AssemblyName System.Windows.Forms
            Add-Type -AssemblyName System.Drawing

            $notification = New-Object System.Windows.Forms.NotifyIcon
            $notification.Icon = [System.Drawing.SystemIcons]::Information
            $notification.BalloonTipTitle = $Title
            $notification.BalloonTipText = $Message
            $notification.Visible = $true

            $notification.ShowBalloonTip(5000)

            # Keep process alive briefly so Windows can display it
            Start-Sleep -Seconds 3

            $notification.Dispose()
        }
        catch {
            # Notification failure should not break Codex
        }
    }

    function Send-SlackNotification {
        param(
            [string]$Message
        )

        try {
            # First check process environment
            $webhookUrl = [Environment]::GetEnvironmentVariable(
                "CODEX_SLACK_WEBHOOK",
                "Process"
            )

            # Fall back to Windows user environment
            if ([string]::IsNullOrWhiteSpace($webhookUrl)) {
                $webhookUrl = [Environment]::GetEnvironmentVariable(
                    "CODEX_SLACK_WEBHOOK",
                    "User"
                )
            }

            if ([string]::IsNullOrWhiteSpace($webhookUrl)) {
                return
            }

            $body = @{
                text = $Message
            } | ConvertTo-Json -Compress

            $bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($body)

            Invoke-RestMethod `
                -Uri $webhookUrl `
                -Method Post `
                -ContentType "application/json; charset=utf-8" `
                -Body $bodyBytes | Out-Null
        }
        catch {
            # Slack failure should not break Codex
        }
    }


    # -----------------------------
    # Read Codex hook input
    # -----------------------------

    try {
        $rawInput = [Console]::In.ReadToEnd()

        if ([string]::IsNullOrWhiteSpace($rawInput)) {
            Write-Output "{}"
            exit 0
        }

        $event = $rawInput | ConvertFrom-Json
    }
    catch {
        Write-Output "{}"
        exit 0
    }


    $eventName = $event.hook_event_name
    $cwd = $event.cwd

    if ([string]::IsNullOrWhiteSpace($cwd)) {
        $cwd = "Unknown"
    }


    # -----------------------------
    # Handle events
    # -----------------------------

    switch ($eventName) {

        "PermissionRequest" {

            $tool = $event.tool_name

            if ([string]::IsNullOrWhiteSpace($tool)) {
                $tool = "Unknown"
            }

            $description = $event.tool_input.description

            if ([string]::IsNullOrWhiteSpace($description)) {
                $description = "Codex is waiting for your approval."
            }

            # Prevent unexpectedly huge Slack messages
            if ($description.Length -gt 1000) {
                $description = $description.Substring(0, 1000) + "..."
            }

            $message = @"
    CODEX NEEDS ATTENTION

    Project:
    $cwd

    Tool:
    $tool

    Reason:
    $description
    "@

            Show-DesktopNotification `
                -Title "Codex Needs Attention" `
                -Message "Codex is waiting for your approval in $cwd"
        }


        "Stop" {

            $lastMessage = $event.last_assistant_message

            if ([string]::IsNullOrWhiteSpace($lastMessage)) {
                $lastMessage = "Codex completed the task."
            }

            if ($lastMessage.Length -gt 1000) {
                $lastMessage = $lastMessage.Substring(0, 1000) + "..."
            }

            $message = @"
    CODEX FINISHED

    Project:
    $cwd

    Result:
    $lastMessage
    "@

            Show-DesktopNotification `
                -Title "Codex Finished" `
                -Message "Codex has finished working on $cwd"
        }


        default {
            Write-Output "{}"
            exit 0
        }
    }


    # -----------------------------
    # Send Slack
    # -----------------------------

    Send-SlackNotification -Message $message


    # Codex Stop hooks expect valid JSON on stdout
    Write-Output "{}"

    exit 0
    ```

## Step 4: Register the hooks in `hooks.json`

Which `hooks.json` to update depends on where Codex runs. Create the file if it does not exist.

=== "Windows"

    Update `C:\Users\YOUR_USERNAME\.codex\hooks.json`:

    ```json
    {
      "description": "Slack notifications for Codex",
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1",
                "commandWindows": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1",
                "timeout": 15
              }
            ]
          }
        ],
        "Stop": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1",
                "commandWindows": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1",
                "timeout": 15
              }
            ]
          }
        ]
      }
    }
    ```

=== "WSL"

    If Codex runs inside WSL, update `~/.codex/hooks.json` in your WSL home folder instead. The script stays on the Windows side, where you saved it in step 3:

    ```json
    {
      "description": "Slack notifications for Codex",
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File 'C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1'",
                "timeout": 15
              }
            ]
          }
        ],
        "Stop": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File 'C:\\Users\\YOUR_USERNAME\\.codex\\hooks\\slack_notifier.ps1'",
                "timeout": 15
              }
            ]
          }
        ]
      }
    }
    ```

## Step 5: Allow the hooks in VS Code

Codex does not run new hooks until you allow them. In VS Code, open the Codex settings, go to **Hooks** and allow the hooks.

## References

- [Codex: Hooks](https://learn.chatgpt.com/docs/hooks)
- [Slack: Sending messages using incoming webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks)
