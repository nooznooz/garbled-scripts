# garbled-scripts
temp folder for random snippets


<#
.SYNOPSIS
    Provides a scaffolding for launching a non-blocking Windows Form
    that can execute commands back in the host PowerShell terminal.

.DESCRIPTION
    This script defines two functions:
    1. Start-Assignment: The main function you call. It takes parameters,
       compiles a C# Windows Form in-memory, and launches it in a separate
       PowerShell runspace so it doesn't block your terminal. It also starts
       a background timer to listen for commands from the form.

    2. Stop-Assignment: A helper function to clean up the runspace and
       stop the timer. This is triggered automatically when the form is closed.

.EXAMPLE
    .\WinForm_Scaffolding.ps1
    Start-Assignment -Rack "R007" -Ip "192.168.1.101"

    # A Windows Form will appear. Your terminal prompt will return.
    # Click the buttons on the form.
    # The commands (e.g., plink, Invoke-RestMethod) will execute
    # directly in your terminal, and you will see their output.
#>

function Start-Assignment {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]$Rack,

        [Parameter(Mandatory = $true)]
        [string]$Ip
    )

    Write-Host "Starting assignment for Rack: $Rack, IP: $Ip" -ForegroundColor Cyan

    # 1. Define the inline C# code for the Windows Form.
    #    This form will be compiled against the .NET Framework version
    #    that PowerShell 5.1 is running on.
    $csharpCode = @"
    using System;
    using System.Drawing;
    using System.Windows.Forms;
    using System.Collections.Concurrent;

    // This is the namespace and class we will create in PowerShell
    namespace PsFormHost {
        public class MainForm : Form {

            // This queue is the communication bridge.
            // The form ADDS command strings to it.
            private ConcurrentQueue<string> commandQueue;
            private string rack;
            private string ip;

            // Controls
            private Label rackLabel;
            private Label ipLabel;
            private Button sshButton;
            private Button apiButton;
            private Button exitButton;

            // Constructor that accepts our PS parameters and the queue
            public MainForm(string rack, string ip, ConcurrentQueue<string> queue) {
                this.rack = rack;
                this.ip = ip;
                this.commandQueue = queue;
                
                InitializeComponent();
            }

            private void InitializeComponent() {
                this.Text = "PowerShell Assignment Runner";
                this.Size = new Size(400, 250);
                this.FormBorderStyle = FormBorderStyle.FixedDialog;
                this.StartPosition = FormStartPosition.CenterScreen;

                // Rack Label
                this.rackLabel = new Label();
                this.rackLabel.Text = "Rack: " + this.rack;
                this.rackLabel.Location = new Point(20, 20);
                this.rackLabel.AutoSize = true;
                this.rackLabel.Font = new Font(this.rackLabel.Font.FontFamily, 12);
                this.Controls.Add(this.rackLabel);

                // IP Label
                this.ipLabel = new Label();
                this.ipLabel.Text = "IP Address: " + this.ip;
                this.ipLabel.Location = new Point(20, 50);
                this.ipLabel.AutoSize = true;
                this.ipLabel.Font = new Font(this.ipLabel.Font.FontFamily, 12);
                this.Controls.Add(this.ipLabel);

                // SSH Button
                this.sshButton = new Button();
                this.sshButton.Text = "Run SSH (plink)";
                this.sshButton.Location = new Point(20, 100);
                this.sshButton.Size = new Size(340, 40);
                this.sshButton.Click += new EventHandler(SshButton_Click);
                this.Controls.Add(this.sshButton);

                // API Button
                this.apiButton = new Button();
                this.apiButton.Text = "Run REST API (Example)";
                this.apiButton.Location = new Point(20, 150);
                this.apiButton.Size = new Size(340, 40);
                this.apiButton.Click += new EventHandler(ApiButton_Click);
                this.Controls.Add(this.apiButton);

                // Add a FormClosing event handler to clean up
                this.FormClosing += new FormClosingEventHandler(MainForm_Closing);
            }

            private void SshButton_Click(object sender, EventArgs e) {
                // --- THIS IS THE CORE LOGIC ---
                // The user clicked the button. We will build a STRING
                // containing the PowerShell command we want to run
                // back in the main terminal.

                // IMPORTANT: Update this path to your plink.exe
                string plinkPath = @"C:\Program Files\PuTTY\plink.exe";
                
                // We use single quotes for PS strings to avoid expansion issues
                string command = string.Format(
                    "Write-Host '--- [FORM] Starting SSH via plink... ---' -ForegroundColor Yellow; & '{0}' -ssh 'user@{1}'", 
                    plinkPath, 
                    this.ip
                );
                
                // Add the command string to the queue for the host to pick up.
                this.commandQueue.Enqueue(command);
            }
            
            private void ApiButton_Click(object sender, EventArgs e) {
                // Example of a different command
                string command = string.Format(
                    "Write-Host '--- [FORM] Running REST API call for rack {0}... ---' -ForegroundColor Yellow; " +
                    "try {{ Invoke-RestMethod -Uri 'https://jsonplaceholder.typicode.com/posts/1' }} catch {{ Write-Warning $_.Exception.Message }}",
                    this.rack
                );
                this.commandQueue.Enqueue(command);
            }

            private void MainForm_Closing(object sender, FormClosingEventArgs e) {
                // When the form closes, send a special "STOP" command
                // to tell the host terminal to stop the timer and clean up.
                this.commandQueue.Enqueue("STOP_FORM_PROCESSOR");
            }
        }
    }
"@

    # 2. Create the synchronized queue.
    #    We use [string] because it's easy to pass from C# to PowerShell.
    $commandQueue = [System.Collections.Concurrent.ConcurrentQueue[string]]::new()

    # 3. Define the script block that will run in the NEW runspace
    $scriptBlock = {
        param($rack, $ip, $commandQueue, $csharpCode)

        # Add required assemblies for WinForms
        Add-Type -AssemblyName System.Windows.Forms
        Add-Type -AssemblyName System.Drawing

        # Compile our C# code in-memory
        Add-Type -TypeDefinition $csharpCode

        # Create an instance of our form, passing in the parameters
        $form = New-Object PsFormHost.MainForm($rack, $ip, $commandQueue)

        # Show the form and start the application message loop
        # [Application]::Run() is crucial.
        # Do NOT use .ShowDialog() as that would block this thread.
        [System.Windows.Forms.Application]::Run($form)
    }

    # 4. Create and configure the new runspace
    $rs = [runspacefactory]::CreateRunspace()
    $rs.Open()

    # Set variables in the new runspace's session state
    $rs.SessionStateProxy.SetVariable("rack", $Rack)
    $rs.SessionStateProxy.SetVariable("ip", $Ip)
    $rs.SessionStateProxy.SetVariable("commandQueue", $commandQueue)
    $rs.SessionStateProxy.SetVariable("csharpCode", $csharpCode)

    # 5. Create a [powershell] object to run the script block
    $ps = [powershell]::Create()
    $ps.Runspace = $rs
    $ps.AddScript($scriptBlock) | Out-Null

    # 6. Invoke the script asynchronously.
    #    This starts the form in the other runspace and
    #    immediately returns control to this script.
    $ps.BeginInvoke() | Out-Null

    Write-Host "Form launched in separate runspace. Terminal is active." -ForegroundColor Green

    # 7. Create the Timer in the HOST runspace.
    #    This timer will periodically check the queue for new commands.
    $timer = [System.Timers.Timer]::new(250) # Check 4 times a second

    # 8. Define the action for the timer's "Elapsed" event
    #    This script block runs in the HOST runspace.
    $timerAction = {
        param($commandQueue)

        $timer = $event.Sender
        $commandString = $null

        # Check if the queue has an item.
        # [ref] is used to get the "out" value from TryDequeue.
        if ($commandQueue.TryDequeue([ref]$commandString)) {
            
            # Check for our special "stop" command
            if ($commandString -eq "STOP_FORM_PROCESSOR") {
                Write-Host "[Host] Form closed. Stopping command processor timer." -ForegroundColor Green
                $timer.Stop()
                $timer.Dispose()
                
                # Clean up the runspace
                if ($global:formRunspace) {
                    $global:formRunspace.Close()
                    $global:formRunspace.Dispose()
                    Remove-Variable -Name formRunspace -Scope Global
                }
                if ($global:formCommandTimer) {
                    Remove-Variable -Name formCommandTimer -Scope Global
                }
            }
            else {
                # This is a normal command. Execute it!
                try {
                    # Create a script block from the string
                    $scriptBlock = [scriptblock]::Create($commandString)
                    
                    # Execute it in the host runspace
                    Invoke-Command -ScriptBlock $scriptBlock
                }
                catch {
                    Write-Error "Error executing command from form: $_"
                }
            }
        }
    }

    # 9. Register the event and start the timer
    #    We use -MessageData to pass the $commandQueue to the $timerAction
    Register-ObjectEvent -InputObject $timer -EventName "Elapsed" -Action $timerAction -MessageData $commandQueue | Out-Null
    $timer.Start()

    # 10. Store timer and runspace in global variables
    #     so they don't get garbage-collected.
    $global:formCommandTimer = $timer
    $global:formRunspace = $rs
}

# Helper function to manually clean up if needed
function Stop-Assignment {
    Write-Host "Stopping assignment processor..."
    if ($global:formCommandTimer) {
        $global:formCommandTimer.Stop()
        $global:formCommandTimer.Dispose()
        Remove-Variable -Name formCommandTimer -Scope Global
    }
    if ($global:formRunspace) {
        $global:formRunspace.Close()
        $global:formRunspace.Dispose()
        Remove-Variable -Name formRunspace -Scope Global
    }
    
    # Unregister the event listener
    Get-EventSubscriber -SourceIdentifier "Event.*" | Unregister-Event
}

Write-Host "Functions 'Start-Assignment' and 'Stop-Assignment' are loaded."
Write-Host "Example Usage:"
Write-Host "Start-Assignment -Rack 'R001' -Ip '192.168.1.50'"
