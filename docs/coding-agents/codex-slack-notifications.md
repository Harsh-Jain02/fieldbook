# Get Slack notifications when Codex needs approval or finishes

*Last updated: 19 October 2026*

When you work with ChatGPT Codex in VS Code, it does not tell you when it is stuck waiting for your approval or when it has finished. This page sets up a Codex hook that sends you a Slack message and a desktop notification in both cases.

Choose the instructions for Windows, WSL, or Ubuntu. Replace `YOUR_USERNAME` with your Windows username wherever it appears.

## How it works

Codex can run a command, called a hook, when certain events happen. This page registers the PowerShell script `slack_notifier.ps1` for two events:

- **`PermissionRequest`**: Codex is waiting for your approval. Slack receives **CODEX NEEDS ATTENTION** with the project folder, the tool Codex wants to use and the reason.
- **`Stop`**: Codex has finished. Slack receives **CODEX FINISHED** with the project folder and Codex's last message.

Both events also show a desktop notification: a Windows notification on Windows or WSL, and an Ubuntu notification through `notify-send` on Ubuntu. Long reasons and messages are cut to 1,000 characters.

The notifier reads the Slack webhook URL from `CODEX_SLACK_WEBHOOK`. Windows stores it as a user environment variable; Ubuntu keeps it in a private configuration file that the notifier reads. If the URL is missing or Slack cannot be reached, the notifier skips the message without an error, so a failed notification never interrupts Codex.

## Step 1: Create a Slack incoming webhook

1. Create a Slack account.
2. Create the channel where you want to receive the messages.
3. Go to [api.slack.com/apps](https://api.slack.com/apps).
4. Create an app.
5. Go to **Incoming Webhooks** and create a webhook for that channel. Copy its URL for the next step.

!!! warning "Keep the webhook URL private"
    Anyone who has the URL can post messages to your channel. Keep it out of pages, repositories and screenshots.

## Step 2: Store the webhook URL securely

### Windows

Open the Start menu, search for **environment variables** and open **Edit environment variables for your account**. Under **User variables**, click **New...** and enter:

- **Variable name:** `CODEX_SLACK_WEBHOOK`
- **Variable value:** your webhook URL, for example `https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxx`

Click **OK** in both windows.

### Ubuntu

The Ubuntu notifier needs Python 3 and `notify-send`. Check whether they are available:

```bash
python3 --version
notify-send --version
```

If either command is missing, install the required packages:

```bash
sudo apt update
sudo apt install python3 libnotify-bin
```

Create a private configuration directory and file:

```bash
mkdir -p ~/.config/codex
chmod 700 ~/.config/codex
nano ~/.config/codex/slack-notifier.env
```

Add the webhook URL, replacing the placeholder. Do not include quotes around the URL:

```text
CODEX_SLACK_WEBHOOK=https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxx
```

Save the file, then restrict it to your user account:

```bash
chmod 600 ~/.config/codex/slack-notifier.env
```

## Step 3: Save the notifier script

### Windows

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

### Ubuntu

Create the hooks directory and save this script as `~/.codex/hooks/slack_notifier.py`:

```bash
mkdir -p ~/.codex/hooks
nano ~/.codex/hooks/slack_notifier.py
```

??? example "slack_notifier.py"

    ```python
    #!/usr/bin/env python3
    import json
    import os
    from pathlib import Path
    import subprocess
    import sys
    from urllib.request import Request, urlopen


    def truncate(value, limit=1000):
        value = value if isinstance(value, str) else ""
        return value if len(value) <= limit else f"{value[:limit]}..."


    def webhook_url():
        url = os.environ.get("CODEX_SLACK_WEBHOOK", "").strip()
        if url:
            return url

        config_home = Path(os.environ.get("XDG_CONFIG_HOME", Path.home() / ".config"))
        config_file = config_home / "codex" / "slack-notifier.env"

        try:
            for line in config_file.read_text(encoding="utf-8").splitlines():
                key, separator, value = line.partition("=")
                if separator and key.strip() == "CODEX_SLACK_WEBHOOK":
                    return value.strip()
        except OSError:
            pass

        return ""


    def show_desktop_notification(title, message):
        try:
            subprocess.run(
                ["notify-send", "--expire-time=5000", title, message],
                check=False,
                stdin=subprocess.DEVNULL,
                stdout=subprocess.DEVNULL,
                stderr=subprocess.DEVNULL,
                timeout=2,
            )
        except (OSError, subprocess.TimeoutExpired):
            pass


    def send_slack_notification(message):
        url = webhook_url()
        if not url:
            return

        try:
            request = Request(
                url,
                data=json.dumps({"text": message}).encode("utf-8"),
                headers={"Content-Type": "application/json; charset=utf-8"},
                method="POST",
            )
            with urlopen(request, timeout=5):
                pass
        except (OSError, ValueError):
            pass


    try:
        event = json.load(sys.stdin)
    except (json.JSONDecodeError, OSError):
        event = {}

    if not isinstance(event, dict):
        event = {}

    event_name = event.get("hook_event_name")
    cwd = event.get("cwd") or "Unknown"

    if event_name == "PermissionRequest":
        tool = event.get("tool_name") or "Unknown"
        tool_input = event.get("tool_input")
        description = tool_input.get("description") if isinstance(tool_input, dict) else None
        description = truncate(description) or "Codex is waiting for your approval."

        message = f"""CODEX NEEDS ATTENTION

    Project:
    {cwd}

    Tool:
    {tool}

    Reason:
    {description}"""

        show_desktop_notification(
            "Codex Needs Attention",
            f"Codex is waiting for your approval in {cwd}",
        )
        send_slack_notification(message)

    elif event_name == "Stop":
        last_message = truncate(event.get("last_assistant_message")) or "Codex completed the task."

        message = f"""CODEX FINISHED

    Project:
    {cwd}

    Result:
    {last_message}"""

        show_desktop_notification(
            "Codex Finished",
            f"Codex has finished working on {cwd}",
        )
        send_slack_notification(message)


    # Stop hooks must return JSON on stdout. PermissionRequest ignores stdout.
    print("{}")
    ```

Make the script executable:

```bash
chmod 700 ~/.codex/hooks/slack_notifier.py
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

=== "Linux"

    Update `~/.codex/hooks.json`:

    ```json
    {
      "description": "Slack notifications for Codex",
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "python3 ~/.codex/hooks/slack_notifier.py",
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
                "command": "python3 ~/.codex/hooks/slack_notifier.py",
                "timeout": 15
              }
            ]
          }
        ]
      }
    }
    ```

## Step 5: Review and trust the hooks

Codex does not run new hooks until you review and trust them. In Codex CLI, run `/hooks`, then review and trust the two hook definitions. In VS Code, you can also open the Codex settings, go to **Hooks**, and allow them.

## References

- [Codex: Hooks](https://learn.chatgpt.com/docs/hooks)
- [Slack: Sending messages using incoming webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks)
