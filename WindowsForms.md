# Guidelines for Fully Responsive Windows Forms Applications

## Basic Concepts for Responsive Design

### 1. Use Anchoring and Docking

```csharp
// Example: Make a button anchor to the bottom right
button1.Anchor = AnchorStyles.Bottom | AnchorStyles.Right;

// Example: Dock a panel to fill its parent container
panel1.Dock = DockStyle.Fill;
```

### 2. Layout Panels for Responsive Design

```csharp
// Create a TableLayoutPanel for grid-like layouts
TableLayoutPanel tablePanel = new TableLayoutPanel
{
	Dock = DockStyle.Fill,
	RowCount = 3,
	ColumnCount = 2
};

// Create a FlowLayoutPanel for flowing content
FlowLayoutPanel flowPanel = new FlowLayoutPanel
{
	Dock = DockStyle.Fill,
	FlowDirection = FlowDirection.TopDown,
	AutoScroll = true
};
```

### 3. Handle DPI Scaling

```csharp
public partial class Form1 : Form
{
	public Form1()
	{
		InitializeComponent();
		
		// Enable DPI awareness
		this.AutoScaleDimensions = new SizeF(96F, 96F);
		this.AutoScaleMode = AutoScaleMode.Dpi;
		
		// Subscribe to DPI changed events
		this.DpiChanged += Form1_DpiChanged;
	}
	
	private void Form1_DpiChanged(object sender, DpiChangedEventArgs e)
	{
		// Handle DPI changes here
		// Adjust custom elements that don't scale automatically
	}
	
	private void button1_Click(object sender, EventArgs e)
	{
		label1.Text = "Button clicked!";
	}
}
```

## Advanced Techniques

### 4. Multiple Monitor Support

```csharp
// Get information about all screens
Screen[] screens = Screen.AllScreens;

// Get the screen where the form is currently displayed
Screen currentScreen = Screen.FromControl(this);

// Adjust form position based on current screen
this.Location = new Point(
	currentScreen.WorkingArea.X + (currentScreen.WorkingArea.Width - this.Width) / 2,
	currentScreen.WorkingArea.Y + (currentScreen.WorkingArea.Height - this.Height) / 2
);
```

### 5. Responsive Sizing Approach

```csharp
// Create a method to calculate relative sizes
private int GetResponsiveSize(int baseSize)
{
	float scaleFactor = this.CreateGraphics().DpiX / 96f;
	return (int)(baseSize * scaleFactor);
}

// Usage example
button1.Font = new Font("Segoe UI", GetResponsiveSize(9));
button1.Padding = new Padding(GetResponsiveSize(10));
```

### 6. Set Minimum Size Constraints

```csharp
// Set minimum size to prevent unusable UI
this.MinimumSize = new Size(800, 600);
```

### 7. Form Load Positioning and Scaling

```csharp
private void Form1_Load(object sender, EventArgs e)
{
	// Position relative to screen size
	Screen screen = Screen.FromControl(this);
	int screenWidth = screen.WorkingArea.Width;
	int screenHeight = screen.WorkingArea.Height;
	
	// Set form size as percentage of screen
	this.Width = (int)(screenWidth * 0.8);
	this.Height = (int)(screenHeight * 0.8);
	
	// Center on screen
	this.Left = (screenWidth - this.Width) / 2;
	this.Top = (screenHeight - this.Height) / 2;
}
```


## Handling DPI

```csharp

// In Program.cs
// Enable DPI awareness for the application
Application.SetHighDpiMode(HighDpiMode.SystemAware);
// or for per-monitor DPI awareness in .NET 5 and later
Application.SetHighDpiMode(HighDpiMode.PerMonitorV2);

// In Form1 
using System;
using System.Runtime.InteropServices;
using System.Windows.Forms;
using System.Drawing;
using System.Text;
using System.Windows.Forms.VisualStyles;

public partial class Form1 : Form
{
	// DPI awareness context constant for per-monitor DPI awareness
	// The value -4 corresponds to DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 in Windows 10 and later
	private static readonly IntPtr DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2 = new IntPtr(-4);

	// Sets the DPI awareness context for the process per-monitor
	[DllImport("user32.dll")]
	private static extern bool SetProcessDpiAwarenessContext(IntPtr dpiContext);

	public Form1()
	{
		InitializeComponent();

		// Enable DPI awareness for the form
		this.AutoScaleMode = AutoScaleMode.Dpi;

		// Subscribe to DPI changed events
		this.DpiChanged += Form1_DpiChanged;
	}

	// This method sets the DPI awareness context for the process
	private void SetProcessDpiAwareness()
	{
		// Check if the OS version supports per-monitor DPI awareness
		if (Environment.OSVersion.Version.Major >= 6)
		{
			try
			{
				SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
			}
			catch (Exception ex)
			{
				MessageBox.Show($"Failed to set DPI awareness: {ex.Message}");
			}
		}
	}

	private void Form1_DpiChanged(object? sender, DpiChangedEventArgs e)
	{
		// Handle DPI change event
		try
		{
			// Log that event fired
			Console.WriteLine($"DPI changed event fired: {e.DeviceDpiOld} -> {e.DeviceDpiNew}");

			MessageBox.Show($"DPI changed from {e.DeviceDpiOld} to {e.DeviceDpiNew}.",
				"DPI Change Detected", MessageBoxButtons.OK, MessageBoxIcon.Information);
		}
		catch (Exception ex)
		{
			MessageBox.Show($"Error in DPI handler: {ex.Message}");
		}

		// Update the form's controls or layout to adapt to the new DPI
		try
		{
			// Calculate scaling factor between old and new DPI
			float scaleFactor = (float)e.DeviceDpiNew / e.DeviceDpiOld;

			// Scale the form's font
			this.Font = new Font(this.Font.FontFamily, this.Font.Size * scaleFactor, this.Font.Style);

			// Scale all controls
			ScaleControlsOnDpiChange(this.Controls, scaleFactor);
		}
		catch (Exception ex)
		{
			MessageBox.Show($"Error adapting controls to new Dpi: {ex.Message}");
		}

	}

	protected override void OnHandleCreated(EventArgs e)
	{
		base.OnHandleCreated(e);
		// Force Windows to recognize this window for DPI change notifications
		SetProcessDpiAwareness();
		Console.WriteLine($"Window created with DPI: {this.DeviceDpi}");
	}


	// Recursive method to scale all controls in the hierarchy
	private void ScaleControlsOnDpiChange(Control.ControlCollection controls, float scaleFactor)
	{
		foreach (Control control in controls)
		{
			// Check if the control has a font that's different from its parent
			bool hasCustomFont = control.Parent != null &&
								control.Font != null &&
								!control.Font.Equals(control.Parent.Font);

			if (hasCustomFont || control.Parent == null)
			{
				control.Font = new Font(
					control.Font.FontFamily,
					control.Font.Size * scaleFactor,
					control.Font.Style);
			}

			// Handle specific control types that need special scaling
			if (control is Label || control is Button || control is TextBox)
			{
				// Scale control size if it's not anchored/docked
				if (control.Anchor == (AnchorStyles.Top | AnchorStyles.Left))
				{
					control.Width = (int)(control.Width * scaleFactor);
					control.Height = (int)(control.Height * scaleFactor);
				}
				
				// Scale control location
				control.Left = (int)(control.Left * scaleFactor);
				control.Top = (int)(control.Top * scaleFactor);
			}

			// Recursively process child controls
			if (control.Controls.Count > 0)
			{
				ScaleControlsOnDpiChange(control.Controls, scaleFactor);
			}
		}
	}

	// Controls affected by DPI changes
	private void button1_Click(object sender, EventArgs e)
	{
		label1.Text = "Button clicked!";
	}

	private void tableLayoutPanel1_Paint(object sender, PaintEventArgs e)
	{

	}
}

