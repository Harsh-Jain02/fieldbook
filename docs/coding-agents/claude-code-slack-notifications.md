# Get Slack notifications when Claude Code needs you or finishes

*Last updated: 20 October 2026*

When you use the Claude Code extension in VS Code, a hook can send a Slack message and a desktop notification when Claude needs an approval or when its turn has finished. The setup is personal and global: it applies to every project opened through the extension or the Claude Code CLI on that computer.

Choose the instructions for the system where Claude Code runs. If VS Code is connected to WSL, Claude Code runs in WSL, but this guide uses a Windows PowerShell notifier so the desktop alert appears in Windows. Replace `YOUR_USERNAME` with your Windows username wherever it appears.

!!! warning "Slack receives parts of your work"
    The notification can include a project path, a permission request, and Claude's final message. Use a private Slack channel and do not enable it for work that must not leave your computer. Keep the incoming-webhook URL out of repositories, pages, screenshots, and messages.

## What you need

- The Claude Code extension for VS Code, signed in and up to date.
- A Slack workspace and a private channel where you can create an incoming webhook.
- On Ubuntu, Python 3 and `notify-send` (`libnotify-bin`) for desktop notifications. Slack still works if `notify-send` is unavailable.

## How it works

Claude Code runs the same hooks in the VS Code extension and in its CLI. This setup registers one script for three events:

- **`PermissionRequest`**: Claude is about to show an approval or input control. The alert is immediate and names the requested tool; a question from Claude appears as the `AskUserQuestion` tool.
- **`Notification`**: an MCP server needs your input through an elicitation form or URL, or an agent session needs your input.
- **`Stop`**: the main Claude agent has completed its response. The script does not report a finish while Claude still has background work or a scheduled wake-up.

Both attention events use a warning desktop icon and `:warning:` in Slack. A completed turn uses an information icon and `:white_check_mark:`. The notifier limits the included text to 1,000 characters. If Slack or the desktop notifier fails, the hook exits successfully, so Claude is never interrupted.

## Step 1: Create a Slack incoming webhook

1. Create the private Slack channel that should receive the alerts.
2. Go to [Slack API: Your Apps](https://api.slack.com/apps) and create an app.
3. Open **Incoming Webhooks**, activate it, then add a webhook for that channel.
4. Copy the webhook URL for the next step.

## Step 2: Store the webhook URL securely

=== "Windows and WSL"

    Open the Start menu, search for **environment variables**, and open **Edit environment variables for your account**. Under **User variables**, click **New...** and enter:

    - **Variable name:** `CLAUDE_SLACK_WEBHOOK`
    - **Variable value:** the webhook URL, for example `https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxx`

    Click **OK** in both windows. The PowerShell script below reads this user-level value, so it works whether Claude Code is launched from VS Code or from WSL.

=== "Ubuntu"

    First check that Python and `notify-send` are available:

    ```bash
    python3 --version
    notify-send --version
    ```

    If either is missing, install the required packages:

    ```bash
    sudo apt update
    sudo apt install python3 libnotify-bin
    ```

    Create a private configuration directory and file:

    ```bash
    mkdir -p ~/.config/claude
    chmod 700 ~/.config/claude
    nano ~/.config/claude/slack-notifier.env
    ```

    Add the webhook URL, replacing the placeholder. Do not include quotes:

    ```text
    CLAUDE_SLACK_WEBHOOK=https://hooks.slack.com/services/xxxxxxxxxxx/xxxxxxxxxxxx/xxxxxxxxxxxxxxxxxxxxxxx
    ```

    Save the file, then restrict it to your account:

    ```bash
    chmod 600 ~/.config/claude/slack-notifier.env
    ```

## Step 3: Save the notifier script

=== "Windows and WSL"

    Create `C:\Users\YOUR_USERNAME\.claude\hooks\` and save the following as `slack_notifier.ps1` in that folder.

    ??? example "slack_notifier.ps1"

        ```powershell
        $ErrorActionPreference = "Stop"

        function Truncate-Text {
            param(
                [AllowNull()][object]$Value,
                [int]$Limit = 1000
            )

            $text = if ($null -eq $Value) { "" } else { [string]$Value }
            if ($text.Length -le $Limit) {
                return $text
            }

            return $text.Substring(0, $Limit) + "..."
        }

        function Show-DesktopNotification {
            param(
                [string]$Title,
                [string]$Message,
                [string]$Icon = "Info"  # "Info" or "Warning"
            )

            try {
                Add-Type -AssemblyName System.Windows.Forms
                Add-Type -AssemblyName System.Drawing

                $notification = New-Object System.Windows.Forms.NotifyIcon
                if ($Icon -eq "Warning") {
                    $notification.Icon = [System.Drawing.SystemIcons]::Warning
                    $notification.BalloonTipIcon = [System.Windows.Forms.ToolTipIcon]::Warning
                }
                else {
                    $notification.Icon = [System.Drawing.SystemIcons]::Information
                    $notification.BalloonTipIcon = [System.Windows.Forms.ToolTipIcon]::Info
                }

                $notification.BalloonTipTitle = $Title
                $notification.BalloonTipText = $Message
                $notification.Visible = $true
                $notification.ShowBalloonTip(5000)

                # Keep the process alive long enough for Windows to display the alert.
                Start-Sleep -Seconds 3
                $notification.Dispose()
            }
            catch {
                # A desktop-notification failure must not affect Claude Code.
            }
        }

        function Send-SlackNotification {
            param([string]$Message)

            try {
                $webhookUrl = [Environment]::GetEnvironmentVariable(
                    "CLAUDE_SLACK_WEBHOOK",
                    "Process"
                )

                if ([string]::IsNullOrWhiteSpace($webhookUrl)) {
                    $webhookUrl = [Environment]::GetEnvironmentVariable(
                        "CLAUDE_SLACK_WEBHOOK",
                        "User"
                    )
                }

                if ([string]::IsNullOrWhiteSpace($webhookUrl)) {
                    return
                }

                $body = @{ text = $Message } | ConvertTo-Json -Compress
                $bodyBytes = [System.Text.Encoding]::UTF8.GetBytes($body)

                Invoke-RestMethod `
                    -Uri $webhookUrl `
                    -Method Post `
                    -ContentType "application/json; charset=utf-8" `
                    -Body $bodyBytes | Out-Null
            }
            catch {
                # A Slack failure must not affect Claude Code.
            }
        }

        function Get-ToolDetails {
            param($ToolName, $ToolInput)

            if ($null -eq $ToolInput) {
                return "Claude is waiting for your input."
            }

            if ($ToolName -eq "AskUserQuestion" -and $ToolInput.questions) {
                $questions = @($ToolInput.questions | ForEach-Object { $_.question } | Where-Object { $_ })
                if ($questions.Count -gt 0) {
                    return "Question: " + ($questions -join " | ")
                }
            }

            foreach ($property in "description", "command", "file_path", "prompt") {
                $value = $ToolInput.$property
                if (-not [string]::IsNullOrWhiteSpace([string]$value)) {
                    return "${property}: $value"
                }
            }

            try {
                return $ToolInput | ConvertTo-Json -Depth 5 -Compress
            }
            catch {
                return "Claude is waiting for your input."
            }
        }

        try {
            $rawInput = [Console]::In.ReadToEnd()
            $event = if ([string]::IsNullOrWhiteSpace($rawInput)) {
                @{}
            }
            else {
                $rawInput | ConvertFrom-Json
            }
        }
        catch {
            Write-Output "{}"
            exit 0
        }

        $cwd = if ([string]::IsNullOrWhiteSpace($event.cwd)) { "Unknown" } else { $event.cwd }
        $message = $null

        switch ($event.hook_event_name) {
            "PermissionRequest" {
                $tool = if ([string]::IsNullOrWhiteSpace($event.tool_name)) { "Unknown" } else { $event.tool_name }
                $details = Truncate-Text (Get-ToolDetails -ToolName $tool -ToolInput $event.tool_input)

                $message = @"
        :warning: CLAUDE CODE NEEDS ATTENTION

        Project:
        $cwd

        Tool:
        $tool

        Details:
        $details
        "@

                Show-DesktopNotification `
                    -Title "Claude Code Needs Attention" `
                    -Message "Claude is waiting for your input in $cwd" `
                    -Icon "Warning"
            }

            "Notification" {
                $type = $event.notification_type
                if ($type -notin "elicitation_dialog", "elicitation_url_dialog", "agent_needs_input") {
                    break
                }

                $title = Truncate-Text $event.title 200
                if ([string]::IsNullOrWhiteSpace($title)) {
                    $title = "Claude Code Needs Attention"
                }
                $details = Truncate-Text $event.message

                $message = @"
        :warning: CLAUDE CODE NEEDS ATTENTION

        Project:
        $cwd

        Request:
        $title

        Details:
        $details
        "@

                Show-DesktopNotification `
                    -Title "Claude Code Needs Attention" `
                    -Message "$title in $cwd" `
                    -Icon "Warning"
            }

            "Stop" {
                # A Stop event can occur while Claude has work that will resume later.
                $backgroundTasks = @($event.background_tasks | Where-Object { $null -ne $_ })
                $sessionCrons = @($event.session_crons | Where-Object { $null -ne $_ })
                if ($backgroundTasks.Count -gt 0 -or $sessionCrons.Count -gt 0) {
                    break
                }

                $result = Truncate-Text $event.last_assistant_message
                if ([string]::IsNullOrWhiteSpace($result)) {
                    $result = "Claude completed the task."
                }

                $message = @"
        :white_check_mark: CLAUDE CODE FINISHED

        Project:
        $cwd

        Result:
        $result
        "@

                Show-DesktopNotification `
                    -Title "Claude Code Finished" `
                    -Message "Claude has finished working on $cwd" `
                    -Icon "Info"
            }
        }

        if (-not [string]::IsNullOrWhiteSpace($message)) {
            Send-SlackNotification -Message $message
        }

        # These hooks only notify; no hook decision is returned.
        Write-Output "{}"
        exit 0
        ```

=== "Ubuntu"

    Create the hooks directory and save this as `~/.claude/hooks/slack_notifier.py`:

    ```bash
    mkdir -p ~/.claude/hooks
    nano ~/.claude/hooks/slack_notifier.py
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
            text = value if isinstance(value, str) else ""
            return text if len(text) <= limit else f"{text[:limit]}..."


        def webhook_url():
            url = os.environ.get("CLAUDE_SLACK_WEBHOOK", "").strip()
            if url:
                return url

            config_home = Path(os.environ.get("XDG_CONFIG_HOME", Path.home() / ".config"))
            config_file = config_home / "claude" / "slack-notifier.env"

            try:
                for line in config_file.read_text(encoding="utf-8").splitlines():
                    key, separator, value = line.partition("=")
                    if separator and key.strip() == "CLAUDE_SLACK_WEBHOOK":
                        return value.strip()
            except OSError:
                pass

            return ""


        def show_desktop_notification(title, message, icon, urgency="normal"):
            try:
                subprocess.run(
                    [
                        "notify-send",
                        "--app-name=Claude Code",
                        f"--icon={icon}",
                        f"--urgency={urgency}",
                        "--expire-time=5000",
                        title,
                        message,
                    ],
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


        def tool_details(tool_name, tool_input):
            if not isinstance(tool_input, dict):
                return "Claude is waiting for your input."

            if tool_name == "AskUserQuestion":
                questions = tool_input.get("questions")
                if isinstance(questions, list):
                    text = [item.get("question") for item in questions if isinstance(item, dict)]
                    text = [item for item in text if isinstance(item, str) and item]
                    if text:
                        return "Question: " + " | ".join(text)

            for key in ("description", "command", "file_path", "prompt"):
                value = tool_input.get(key)
                if isinstance(value, str) and value:
                    return f"{key}: {value}"

            try:
                return json.dumps(tool_input, ensure_ascii=False, separators=(",", ":"))
            except (TypeError, ValueError):
                return "Claude is waiting for your input."


        try:
            event = json.load(sys.stdin)
        except (json.JSONDecodeError, OSError):
            event = {}

        if not isinstance(event, dict):
            event = {}

        cwd = event.get("cwd") or "Unknown"
        event_name = event.get("hook_event_name")
        message = ""

        if event_name == "PermissionRequest":
            tool = event.get("tool_name") or "Unknown"
            details = truncate(tool_details(tool, event.get("tool_input")))
            message = f""":warning: CLAUDE CODE NEEDS ATTENTION

        Project:
        {cwd}

        Tool:
        {tool}

        Details:
        {details}"""
            show_desktop_notification(
                "Claude Code Needs Attention",
                f"Claude is waiting for your input in {cwd}",
                icon="dialog-warning",
                urgency="critical",
            )

        elif event_name == "Notification":
            alert_types = {"elicitation_dialog", "elicitation_url_dialog", "agent_needs_input"}
            if event.get("notification_type") in alert_types:
                title = truncate(event.get("title"), 200) or "Claude Code Needs Attention"
                details = truncate(event.get("message"))
                message = f""":warning: CLAUDE CODE NEEDS ATTENTION

        Project:
        {cwd}

        Request:
        {title}

        Details:
        {details}"""
                show_desktop_notification(
                    "Claude Code Needs Attention",
                    f"{title} in {cwd}",
                    icon="dialog-warning",
                    urgency="critical",
                )

        elif event_name == "Stop":
            if not event.get("background_tasks") and not event.get("session_crons"):
                result = truncate(event.get("last_assistant_message")) or "Claude completed the task."
                message = f""":white_check_mark: CLAUDE CODE FINISHED

        Project:
        {cwd}

        Result:
        {result}"""
                show_desktop_notification(
                    "Claude Code Finished",
                    f"Claude has finished working on {cwd}",
                    icon="emblem-default",
                )

        if message:
            send_slack_notification(message)

        # These hooks only notify; no hook decision is returned.
        print("{}")
        ```

    Make the script executable:

    ```bash
    chmod 700 ~/.claude/hooks/slack_notifier.py
    ```

## Step 4: Register the hooks in your user settings

Use the user settings file so this is your personal notification setup, not a project setting that could be committed. If `settings.json` already exists, merge the `hooks` entries below with its existing contents rather than replacing the file.

=== "Windows"

    Add these entries under the top-level `"hooks"` key in `C:\Users\YOUR_USERNAME\.claude\settings.json`:

    ```json
    {
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ],
        "Notification": [
          {
            "matcher": "elicitation_dialog|elicitation_url_dialog|agent_needs_input",
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ],
        "Stop": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ]
      }
    }
    ```

=== "WSL"

    Add these entries under the top-level `"hooks"` key in `~/.claude/settings.json` in the WSL distribution. The script remains in the Windows location from step 3 so the desktop alert appears in Windows:

    ```json
    {
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ],
        "Notification": [
          {
            "matcher": "elicitation_dialog|elicitation_url_dialog|agent_needs_input",
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ],
        "Stop": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "powershell.exe",
                "args": [
                  "-NoProfile",
                  "-ExecutionPolicy",
                  "Bypass",
                  "-File",
                  "C:\\Users\\YOUR_USERNAME\\.claude\\hooks\\slack_notifier.ps1"
                ],
                "timeout": 10
              }
            ]
          }
        ]
      }
    }
    ```

=== "Ubuntu"

    Add these entries under the top-level `"hooks"` key in `~/.claude/settings.json`:

    ```json
    {
      "hooks": {
        "PermissionRequest": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "python3 ~/.claude/hooks/slack_notifier.py",
                "timeout": 10
              }
            ]
          }
        ],
        "Notification": [
          {
            "matcher": "elicitation_dialog|elicitation_url_dialog|agent_needs_input",
            "hooks": [
              {
                "type": "command",
                "command": "python3 ~/.claude/hooks/slack_notifier.py",
                "timeout": 10
              }
            ]
          }
        ],
        "Stop": [
          {
            "hooks": [
              {
                "type": "command",
                "command": "python3 ~/.claude/hooks/slack_notifier.py",
                "timeout": 10
              }
            ]
          }
        ]
      }
    }
    ```

    This command uses Claude Code's shell form, so `~` expands to the home directory of the user who runs Claude Code.

## Step 5: Verify it in VS Code

1. Open the Claude Code panel, type `/`, then choose **Hooks** to confirm that the `PermissionRequest`, `Notification`, and `Stop` hooks are loaded. This interface requires Claude Code 2.1.269 or later.
2. Use **Manual** permission mode and give Claude a task that needs an approval. You should immediately receive a warning alert with the tool name.
3. Give Claude a small, safe task that it can complete. When its turn ends, you should receive the finished alert.

Claude Code watches its settings files, so it normally applies hook changes to the open session. If the hooks do not appear, use **Developer: Reload Window** from the VS Code Command Palette, then check the runtime-specific `settings.json` again.

## Troubleshooting

- **Desktop alert works but Slack does not:** confirm that the webhook is stored under `CLAUDE_SLACK_WEBHOOK` and that the webhook remains enabled for the intended Slack channel.
- **No notification at all:** make sure the settings file is on the computer or WSL distribution where Claude Code runs. The VS Code extension shares the user file `~/.claude/settings.json` with the CLI.
- **No desktop alert from WSL:** check that `powershell.exe` is available in WSL and that the PowerShell script uses the actual Windows username and path.
- **Claude pauses for a sandbox network approval:** this is a special approval that does not emit `PermissionRequest`. To also receive a delayed (about six seconds) reminder for it, add `permission_prompt` to the `Notification` matcher. Standard approval prompts will then produce a second reminder if they remain unanswered for about six seconds.

## References

- [Claude Code: Hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code: Use Claude Code in VS Code](https://code.claude.com/docs/en/vs-code)
- [Slack: Sending messages using incoming webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks)
