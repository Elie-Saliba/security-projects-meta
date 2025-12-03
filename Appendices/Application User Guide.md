# AI-Security Platform - User Guide

Complete guide for using the AI-Security frontend dashboard to monitor, analyze, and manage security log detections.


## Introduction

This appendix provides the operational User Guide for the AI-Security Dashboard, supporting analysts and administrators in navigating the interface and using its core functionality. It consolidates the interface components, workflows, and configuration procedures referenced in Chapters 3 and 4, including dashboard visualisations, log exploration, detection triage, rules management, feedback workflows, and AI configuration operations. The guide serves as the primary user-facing documentation for the proof-of-concept deployment of the platform.

## User Roles and Responsibilities

Although the proof-of-concept implementation does not enforce role-based access control, the system’s functionality aligns with three conceptual user roles that support operational clarity:

- **Security Analyst**  
  Reviews logs, investigates detections, assesses severity, applies feedback, and monitors real-time alerts. Primarily interacts with the Dashboard, Logs, and Detections pages.

- **Administrator**  
  Manages system-wide settings, configures rules, adjusts AI model behaviour, maintains feedback patterns, and oversees the operational mode of the platform through the Configuration area.

- **System Maintainer / DevOps Operator**  
  Oversees Docker deployment, service health checks, logging, and connectivity between Log-Transformer, the AI-Security Backend, and the PostgreSQL database, as described in the deployment appendices.

These roles provide a functional framework for understanding how the UI, APIs, and backend services support different responsibilities across an operational security workflow.

## Mapping UI Features to Backend Architecture

The user interface components presented in this guide integrate with backend subsystems defined in Chapters 3 and 4. The table below summarises the relationships between key UI features and the underlying services and APIs:

| UI Feature                   | Backend Component                            | API / Mechanism                          |
|-----------------------------|----------------------------------------------|------------------------------------------|
| Dashboard metrics & charts  | DetectionPipeline, aggregation queries       | `GET /api/logs`, `GET /api/detections`   |
| Log Explorer                | Log-Transformer normalisation engine         | `GET /api/logs`, filters, pagination     |
| Log Detail modal            | `normalized_logs` JSONB schema               | `GET /api/logs/{id}`                     |
| Detections Explorer         | RuleEngine + AI Agents + DetectionOrchestrator | `GET /api/detections`                  |
| Detection Detail modal      | DetectionOrchestrator, evidence linking      | `GET /api/detections/{id}` + WebSocket   |
| Real-time alerts panel      | WebSocket Broadcaster                        | `ws://host/ws/detections`                |
| Rule Management UI          | Rule loader and RuleEngine                   | YAML rule files + rules endpoints (if exposed) |
| AI Model Settings           | MCP servers and AI provider integration      | Backend configuration & environment vars |
| Feedback History & actions  | QualityAgent and `feedback_patterns` table   | `POST /api/feedback`, feedback queries   |
| System Settings             | Backend configuration layer                  | Database config + environment variables  |

This mapping connects the visual elements of the platform to the underlying architecture and data flows evaluated earlier in the thesis.


---

## Table of Contents

- [Overview](#overview)
- [Getting Started](#getting-started)
- [Navigation](#navigation)
- [Dashboard Page](#dashboard-page)
- [Logs Page](#logs-page)
- [Detections Page](#detections-page)
- [Configuration](#configuration)
  - [System Settings](#system-settings)
  - [Rule Management](#rule-management)
  - [Feedback History](#feedback-history)
  - [AI Agent Instructions](#ai-agent-instructions)
- [Common Components](#common-components)
- [Keyboard Shortcuts](#keyboard-shortcuts)
- [Tips & Best Practices](#tips--best-practices)

---

## Overview

The AI-Security Platform provides a comprehensive web-based interface for monitoring security logs, detecting threats using AI and rule-based systems, and managing security configurations. The platform is designed to be intuitive while providing powerful features for security analysts and administrators.

### Key Features

- **Real-time Dashboard** - Overview of security metrics and recent detections
- **Log Management** - Search, filter, and analyze security logs
- **Detection Analysis** - Review AI-detected threats with remediation guidance
- **Rule Configuration** - Create and manage custom security detection rules
- **AI Configuration** - Configure AI model providers and detection behavior
- **Feedback System** - Improve detection accuracy through feedback

---

## Getting Started

### First Login

When you first access the platform at `http://localhost:8080` (or your configured URL), you'll land on the Dashboard page.

![alt text](<Documentation Screenshots/Initial Dashboard View.png>)

### Required Setup (First Time Users)

Before using the platform, ensure:

1. **AI Model is Configured** - Navigate to Configuration → System Settings → AI Model Configuration
2. **Detection Mode is Set** - Choose between Polling, Realtime, or AI-only mode
3. **Rules are Loaded** (Optional) - Upload or create detection rules if using rule-based detection

![alt text](<Documentation Screenshots/System Settings - AI Model Configuration.png>)

---

## Navigation

### Top Navigation Bar

The navigation bar is always visible at the top of the screen and provides access to all main sections.

![alt text](<Documentation Screenshots/Navigation Bar.png>)

#### Navigation Items

1. **Dashboard** - Home page with overview statistics and recent detections
2. **Detections** - Dedicated view for all security detections with filtering
3. **Logs** - View and search all ingested security logs
4. **Configuration** - Access to system settings, rules, feedback, and AI agent configuration

#### Action Buttons

- **Search Icon (🔍)** - Opens the Command Palette for quick navigation
- **Notification Bell (🔔)** - Shows notification sidebar with system alerts
  - Red badge indicates unread notifications
  - Click to toggle notification panel

### Mobile Navigation

On mobile devices, tap the **menu icon (☰)** to open the side navigation drawer.

![alt text](<Documentation Screenshots/Mobile Navigation Bar.png>)

---

## Dashboard Page

The Dashboard provides a high-level overview of your security posture with real-time statistics and visualizations.

### Page Header

- **Date Range Picker** - Select time period for statistics (presets: Today, Last 7 days, Last 30 days, Custom)
- **Download Report Button** - Export detections or logs for the selected date range


### Statistics Cards

Four key metrics are displayed at the top:

#### 1. Total Logs
- **Purpose**: Shows total number of logs ingested in the selected time period
- **Trend Indicator**: Green/red arrow shows increase/decrease from previous period
- **Action**: Click to navigate to Logs page

#### 2. Total Detections
- **Purpose**: Shows all security detections created in the time period
- **Trend Indicator**: Percentage change from previous period
- **Action**: Click to navigate to Detections page

#### 3. Active Detections
- **Purpose**: Shows detections with "active" status requiring attention
- **Trend Indicator**: Change in active detection count
- **Action**: Click to view active detections with pre-applied filter

#### 4. Critical Alerts
- **Purpose**: Shows high-severity detections requiring immediate attention
- **Trend Indicator**: Change in critical alerts count
- **Action**: Click to view critical detections with severity filter

### Detections & Logs Chart (Radar Chart)


- **Purpose**: Visual comparison of logs vs detections across different categories
- **Grouping Options**: 
  - By Source (evtx, syslog, custom)
  - By Event Type (Security, Application, System)
- **How to Use**:
  - Click the **three-dot menu** in card header to change grouping
  - Hover over chart points to see exact values
  - Compare log volume with detection rate per category


### Detections Table

Displays the most recent detections with quick actions.

![alt text](<Documentation Screenshots/Dashboard Detection Table.png>)

#### Table Columns

| Column | Description |
|--------|-------------|
| **Severity** | Visual indicator (Critical, High, Medium, Low, Info) |
| **Title** | Detection name/summary |
| **Category** | Type of threat detected |
| **Status** | Current state (active, investigating, resolved, false_positive) |
| **Created At** | When detection was created |
| **Actions** | Quick action menu (⋮) |

#### Table Features

- **Click Row** - Opens detailed detection modal
- **Sort Columns** - Click column headers to sort
- **Pagination** - Navigate through pages at bottom
- **Bulk Actions** - Select multiple rows with checkboxes
  - Update status for multiple detections
  - Delete multiple detections


#### Quick Actions (Three-Dot Menu)

- **Update Status** - Change to: investigating, resolved, false_positive, active
- **Delete** - Remove detection permanently

---

## Logs Page

The Logs page provides comprehensive tools for searching, filtering, and analyzing security logs from various sources.

![alt text](<Documentation Screenshots/Full Logs Page.png>)

### Page Header

- **Title & Description** - Page identification
- **Export Logs Button** - Download logs in CSV or JSON format

### Log Distribution Chart (Sunburst Chart)

![alt text](<Documentation Screenshots/Log Distribution.png>)

- **Purpose**: Hierarchical visualization of log distribution
- **Grouping Options** (three-dot menu):
  - Source (where logs came from)
  - Severity (critical, high, medium, low, info)
  - Event Type (category of events)
- **How to Read**:
  - Inner ring: Primary category (e.g., sources)
  - Outer rings: Sub-categories and severity levels
  - Hover to see exact counts
  - Click segments to drill down
- **Chart Stats Panel**: Shows total count and breakdown with color legend


### Log Activity Timeline

![alt text](<Documentation Screenshots/Log Activity TimeLine.png>)

- **Purpose**: Shows log volume over time
- **Grouping Options**:
  - **Hour** - Hourly breakdown (best for single day)
  - **Day** - Daily breakdown (best for weeks/months)
- **How to Use**:
  - Toggle between Hour/Day view using buttons
  - Hover over line to see exact count at each point
  - Identify spikes in activity that may indicate issues

### Logs Table

![alt text](<Documentation Screenshots/Logs Table.png>)

#### Search & Filter Toolbar

- **Search Box** - Free-text search across all log fields
- **Filters Button** - Toggle advanced filtering panel
  - Active filter indicator shows when filters are applied

![alt text](<Documentation Screenshots/Logs Filters.png>)

#### Filter Panel Options

When you click **Filters**, a panel appears with options:

- **Source** - Filter by log source (evtx, syslog, etc.)
- **Severity** - Filter by severity level
- **Event Type** - Filter by event category
- **Date Range** - Custom start and end dates
- **Processed Status** - Show only processed/unprocessed logs
- **Has Detections** - Show only logs with associated detections

**Buttons**:
- **Apply** - Apply current filters and close panel
- **Clear** - Remove all filters
- **Close (✕)** - Close panel without applying changes

#### Table Columns

| Column | Description |
|--------|-------------|
| **Timestamp** | When the event occurred |
| **Source** | Origin of the log (evtx, syslog, etc.) |
| **Event Type** | Category/channel of the event |
| **Severity** | Importance level with color coding |
| **Processed** | Checkmark if log has been analyzed |
| **Detections** | Count of associated detections |
| **Actions** | Quick action menu |

#### Table Interactions

- **Click Row** - Opens log detail modal
- **Sort** - Click column headers to change sort order
- **Pagination** - Navigate pages at bottom
- **Bulk Selection** - Select multiple logs with checkboxes
  - Trigger detection for multiple logs at once

#### Row Actions (Three-Dot Menu)

- **View Detections** - Navigate to Detections page filtered by this log
- **Trigger Detection** - Manually run detection analysis on this log

### Log Detail Modal

When you click a log row, a modal shows complete log information:

![alt text](<Documentation Screenshots/Log Details.png>)
![alt text](<Documentation Screenshots/Log Details - detection.png>)

#### Modal Sections

1. **Header**
   - Title with timestamp
   - Close button (✕)

2. **Overview Tab**
   - Source, Event Type, Severity
   - Processed status
   - Associated detection count
   - Timestamps (created, updated)

3. **Raw Data Tab**
   - Complete original log data in JSON format
   - Syntax highlighted for readability
   - Copy button to copy entire JSON

4. **Normalized Data Tab**
   - Structured fields extracted from log:
     - Host, User, Source IP, Destination IP
     - Process, File Path, Action, Result
     - Event ID, Channel
     - Additional custom fields

5. **Associated Detections** (if any)
   - List of detections created from this log
   - Click detection to open its detail view

#### Modal Actions

- **View Detection** - Opens first associated detection
- **Close** - Close modal

---

## Detections Page

The Detections page is the core of threat analysis, providing detailed information about AI-detected security threats and remediation guidance.

![alt text](<Documentation Screenshots/Detection Name.png>)

### Page Header

- **Title & Description**
- **Download Report Button** - Export detections in CSV, JSON, Markdown, or HTML

### Detections Chart (Sunburst Chart)

![alt text](<Documentation Screenshots/Detection Sunburst Chart.png>)

- **Purpose**: Hierarchical view of detections by category and severity
- **Grouping Options** (three-dot menu):
  - **Severity** - Group by critical, high, medium, low, info
  - **Category** - Group by threat type (malware, intrusion, data_breach, etc.)
- **How to Use**:
  - Inner ring shows primary grouping
  - Outer rings show sub-categories
  - Hover for exact counts
  - Color-coded for quick severity assessment

### Top Detections Card

![alt text](<Documentation Screenshots/Top Detections Chart.png>)

- **Purpose**: Quick view of the 5 most recent or highest-priority detections
- **Information Shown**:
  - Detection title
  - Severity badge
  - Source IP (if available)
- **Action**: Click any detection to open its detail modal

### Detections Table

![alt text](<Documentation Screenshots/Detection Table.png>)

#### Search & Filter Toolbar

- **Search Box** - Search detection titles, descriptions, categories
- **Filters Button** - Toggle advanced filter panel

![alt text](<Documentation Screenshots/Detection Filter Table.png>)

#### Filter Panel Options

- **Status** - active, investigating, resolved, false_positive
- **Severity** - critical, high, medium, low, info
- **Category** - malware, intrusion, data_breach, unauthorized_access, etc.
- **Date Range** - Start and end dates
- **Source** - Log source that triggered detection
- **Log ID** - Specific log identifier

#### Table Columns

| Column | Description |
|--------|-------------|
| **Severity** | Visual badge (color-coded) |
| **Title** | Detection summary |
| **Category** | Threat classification |
| **Status** | Current investigation status |
| **Confidence** | AI confidence score (0-100%) |
| **Created At** | Detection timestamp |
| **Actions** | Quick actions menu |

#### Table Features

- **Click Row** - Opens detection detail modal (primary action)
- **Sort** - Click headers to sort
- **Bulk Selection** - Select multiple detections
  - Update status for multiple
  - Delete multiple detections
- **Pagination** - Navigate through pages

#### Quick Actions Menu

- **Update Status** - Change to investigating/resolved/false_positive/active
- **Delete** - Permanently remove detection

### Detection Detail Modal

The most important component - provides complete information about a detected threat.

![alt text](<Documentation Screenshots/Detection Details + MITRE.png>)

#### Modal Header

- **Title** - Detection name
- **Severity Badge** - Color-coded severity level
- **Status Badge** - Current status
- **Close Button (✕)**

#### Modal Tabs

##### 1. Overview Tab

**Key Information Section**:
- **Severity** - Threat severity level
- **Category** - Type of threat detected
- **Status** - Current investigation status
- **Confidence** - AI confidence percentage
- **Created At** - When detected
- **Updated At** - Last modification time

**Description Section**:
- Detailed explanation of what was detected
- Why it's classified as this threat type
- Context about the security event


##### 2. Detection Tab

**Analysis Section**:
- Complete AI-generated analysis
- Technical details about the threat
- Indicators of compromise (IoCs)
- Attack patterns identified

**Evidence Section**:
- Specific log entries that triggered detection
- Suspicious patterns or behaviors
- Timeline of events

![alt text](<Documentation Screenshots/Detction Details - Logs Related to detection.png>)

##### 3. Remediation Tab

**Action Plan**:
- Step-by-step remediation instructions
- Immediate actions to contain threat
- Long-term security improvements
- Best practices to prevent recurrence

**Actions Available**:
- **Regenerate Remediation** - Ask AI to create new remediation plan
  - Useful if you want different approach
  - Uses updated threat intelligence

![alt text](<Documentation Screenshots/Detection Details- Remediation Plan.png>)

##### 4. Logs Tab

**Associated Logs**:
- All logs that contributed to this detection
- Mini-table with key log information:
  - Timestamp
  - Source
  - Event Type
  - Severity

**Actions**:
- **Click Log Row** - Opens log detail modal
- **View in Logs Page** - Navigate to Logs page with filter applied


#### Modal Footer Actions

![alt text](<Documentation Screenshots/Detection Details- Status- Notes - Feedback.png>)

Available actions depend on detection status:

- **Update Status Dropdown** - Change detection status:
  - **Investigating** - Mark as under investigation
  - **Resolved** - Mark threat as handled
  - **False Positive** - Mark as incorrect detection
  - **Active** - Return to active state
  - Optional: Add notes explaining status change

- **Delete Button** - Permanently remove detection
  - Confirmation required
  - Action cannot be undone

- **Provide Feedback Button** - Help improve AI accuracy
  - Opens feedback form
  - Options: Helpful / Not Helpful
  - Select feedback type:
    - Detection only
    - Remediation only
    - Both
  - Add detailed comments (optional)

---

## Configuration

The Configuration section is the control center for all system settings, rules, feedback management, and AI agent configuration.

![alt text](<Documentation Screenshots/Configuration Landing Page.png>)

### Configuration Landing Page

Four main configuration areas are presented as cards:

1. **System Settings** - Detection modes, AI model configuration
2. **Rule Management** - Create and manage security detection rules
3. **Feedback History** - Review user feedback on detections
4. **AI Agent Instructions** - Configure AI agent prompts and behavior

Click any card to navigate to that configuration area.

---

## System Settings

Configure how the platform detects threats and which AI model to use.

![alt text](<Documentation Screenshots/System Settings.png>)

### Page Header

- **Back Button (←)** - Return to Configuration landing
- **Title & Description**
- **Actions**:
  - **Reset to Defaults** - Restore environment default settings
  - **Save Changes** - Apply and save modifications

**Save Changes** button is only enabled when you've made changes.

### Detection Configuration

![alt text](<Documentation Screenshots/Detection Configuration Card.png>)

#### Detection Mode

**Dropdown Options**:
- **Polling** - Checks for new logs at regular intervals (recommended for most use cases)
- **Realtime** - Instant detection as logs arrive (requires WebSocket support)
- **AI Only** - Uses only AI for detection, no rules (requires batch size ≤ 10)

![alt text](<Documentation Screenshots/Detection Mode Dropdown.png>)

**When to Use**:
- **Polling**: Best for most deployments, lower resource usage
- **Realtime**: When immediate detection is critical
- **AI Only**: Pure AI detection without rule-based filtering

#### Polling Interval (milliseconds)

- **When Visible**: Only shown for Polling and AI-only modes
- **Purpose**: How often to check for new unprocessed logs
- **Range**: Minimum 1000ms (1 second)
- **Default**: 600000ms (10 minutes)
- **Recommendation**: 
  - High-traffic: 60000ms (1 minute)
  - Normal traffic: 300000-600000ms (5-10 minutes)
  - Low-traffic: 600000ms+ (10+ minutes)

![alt text](<Documentation Screenshots/Polling Interval.png>)

#### Batch Size

- **Purpose**: Number of logs to process in each detection batch
- **Range**: 1-1000 (1-10 for AI-only mode)
- **Default**: 100
- **Important**: AI-only mode limited to 10 to manage AI token usage
- **Recommendation**:
  - Rule-based: 100-500 logs
  - AI-only: 5-10 logs (fewer = better quality)

![alt text](<Documentation Screenshots/Batch Size Config.png>)

#### Log Grouping Mode

![alt text](<Documentation Screenshots/Grouping Mode.png>)

- **Purpose**: How to group logs before detection analysis
- **Options**:
  - **Host Only** - All logs from same host are grouped together
  - **Host + User + Source IP** - Finer grouping requiring all three fields to match

**When to Use**:
- **Host Only**: When you want to see all activity from a specific machine
- **Host + User + Source IP**: When you want to separate activities by user and source

**Example**:
- Host Only: All logs from `SERVER-01` grouped together
- Host + User + Source IP: Separate groups for:
  - `SERVER-01 + alice + 192.168.1.50`
  - `SERVER-01 + bob + 192.168.1.51`

### AI Model Configuration

![alt text](<Documentation Screenshots/Ai Model Configuration.png>)

#### Model Provider

**Dropdown Options**:
- **OpenAI** - OpenAI GPT models (gpt-4o, gpt-4, gpt-3.5-turbo)
- **Azure OpenAI** - Microsoft Azure hosted OpenAI
- **Ollama** - Local LLM models
- **Mistral AI** - Mistral AI models

![alt text](<Documentation Screenshots/Modal Provider DropDown.png>)

#### Provider-Specific Fields

Fields change based on selected provider:

**OpenAI Configuration**:
- **Model** - Model name (e.g., gpt-4o, gpt-4-turbo)
- **API Key** - Your OpenAI API key
  - Shows `***Redacted***` if previously configured
  - Click **Unlock** to change existing key


**Azure OpenAI Configuration**:
- **Model** - Deployment name in Azure
- **Base URL** - Azure OpenAI endpoint (e.g., `https://your-resource.openai.azure.com`)
- **API Key** - Azure OpenAI API key
- **API Version** - API version (e.g., `2024-02-01`)


**Ollama Configuration**:
- **Model** - Ollama model name (e.g., `llama2`, `mistral`)
- **Base URL** - Ollama server URL (e.g., `http://localhost:11434`)


**Mistral AI Configuration**:
- **Model** - Model name (e.g., `mistral-large-latest`)
- **API Key** - Mistral AI API key
- **Base URL** - API endpoint (default: `https://api.mistral.ai`)

#### Field Features

- **Required Fields** - Marked with red asterisk (*)
- **Help Text** - Gray text below each field explains purpose
- **Locked API Keys** - Previously saved keys are masked for security
  - Click **Unlock** button to edit
- **Validation** - Save button disabled if required fields are missing


#### Saving Changes

1. Make your changes to any settings
2. **Save Changes** button becomes enabled (blue)
3. Click **Save Changes**
4. Success toast notification appears
5. Settings are immediately active


#### Resetting to Defaults

1. Click **Reset to Defaults** button
2. Confirmation dialog appears
3. Confirm to restore all settings to environment values
4. **Warning**: All custom configurations are lost


---

## Rule Management

Create, edit, and manage security detection rules using a visual editor or YAML upload.

![alt text](<Documentation Screenshots/Rule Management Page.png>)

### Page Header

- **Back Button (←)** - Return to Configuration
- **Title & Description**
- **Actions**:
  - **Upload YAML** - Import rules from YAML file
  - **Reload Rules** - Refresh rules from file system
  - **Create Rule** - Open rule editor to create new rule

![alt text](<Documentation Screenshots/Rule Management Header Actions.png>)

### Rules Table

#### Table Columns

| Column | Description |
|--------|-------------|
| **Rule ID** | Unique identifier |
| **Name** | Rule name/title |
| **Enabled** | Checkmark if active |
| **Severity** | Detection severity if triggered |
| **Category** | Threat category |
| **Actions** | Edit/Delete menu |

#### Table Features

- **Click Row** - Opens rule detail modal (read-only view)
- **Sort** - Click headers to sort
- **Pagination** - Navigate through rules
- **Search** (if many rules) - Search by name or ID

#### Row Actions

- **Edit** - Opens rule editor with current rule data
- **Delete** - Permanently remove rule (confirmation required)

### Creating a Rule

Click **Create Rule** to open the rule editor modal.

![alt text](<Documentation Screenshots/Create Rule screen.png>)

#### Rule Editor Fields

**Basic Information**:
- **Rule ID** - Unique identifier (auto-generated, can customize)
- **Name** - Human-readable rule name
- **Description** - What this rule detects
- **Enabled** - Toggle to activate/deactivate rule

![alt text](<Documentation Screenshots/Rules Basic Info Section.png>)

**Classification**:
- **Severity** - Dropdown: critical, high, medium, low, info
- **Category** - Dropdown: malware, intrusion, data_breach, etc.

**Detection Logic**:
- **Condition Builder** - Visual interface for building detection conditions
  - **Add Condition** - Click to add new condition
  - **Field** - Select log field to check (e.g., `normalized_data.event_id`)
  - **Operator** - Select comparison (equals, contains, greater than, etc.)
  - **Value** - Enter value to match
  - **Logic** - AND/OR between conditions

![alt text](<Documentation Screenshots/Rules Condition Builder.png>)

**Example Rule**:
```
Rule: Failed Login Attempts
Condition 1: event_id equals 4625
Condition 2: user_name not_equals "SYSTEM"
Logic: AND
Result: Detects failed Windows login attempts by real users
```

**Additional Settings**:
- **Tags** - Optional labels for organizing rules (comma-separated)

**Actions**:
- **Cancel** - Close without saving
- **Save Rule** - Create/update rule

### Uploading Rules (YAML)

Click **Upload YAML** to bulk import rules.

![alt text](<Documentation Screenshots/Rules YAML upload.png>)

1. **Upload Modal Opens**:
   - Drag & drop YAML file
   - Or click to browse files

2. **File Validation**:
   - System validates YAML syntax
   - Shows preview of rules to import
   - Displays count and any errors

3. **Confirm Import**:
   - Review rules to be imported
   - Click **Import** to add rules
   - Success message shows number of rules imported


### Rule Detail Modal

Click a rule row to view complete information in read-only mode.

![alt text](<Documentation Screenshots/Rule Details.png>)

**Modal Sections**:
- **Header** - Rule name and ID
- **Status** - Enabled/disabled, severity, category
- **Description** - Full rule description
- **Conditions** - Visual display of all conditions
- **Tags** - Associated tags
- **Metadata** - Created/updated timestamps

**Actions**:
- **Edit** - Switch to edit mode
- **Delete** - Remove rule
- **Close** - Close modal

---

## Feedback History

View and manage user feedback submitted on detections to improve AI accuracy.

![alt text](<Documentation Screenshots/Feedback Mgmt page.png>)

### Page Header

- **Back Button (←)** - Return to Configuration
- **Title & Description**

### Search & Filter


- **Search Box** - Search feedback comments
- **Filters Button** - Toggle filter panel
  - **Is Helpful** - Show only helpful/not helpful feedback
  - **Feedback Type** - Detection, Remediation, or Both
  - **Date Range** - Custom date filtering

### Feedback Table


#### Table Columns

| Column | Description |
|--------|-------------|
| **Detection** | Associated detection title |
| **Feedback Type** | Detection/Remediation/Both |
| **Is Helpful** | Thumbs up/down indicator |
| **Comments** | User feedback text (truncated) |
| **Created At** | When feedback was submitted |
| **Actions** | View/Delete menu |

#### Table Features

- **Click Row** - Opens feedback detail modal
- **Sort** - Click headers to sort
- **Bulk Selection** - Select multiple for bulk delete
- **Pagination** - Navigate pages

### Feedback Detail Modal

![alt text](<Documentation Screenshots/Feedback Details.png>)

**Information Shown**:
- **Detection Title** - Link to original detection
- **Feedback Type** - What was rated
- **Helpful Status** - User rating
- **Comments** - Full user feedback text
- **Timestamps** - When submitted

**Actions**:
- **View Detection** - Opens the associated detection
- **Delete** - Remove feedback
- **Close** - Close modal

### Using Feedback to Improve

Feedback helps improve AI detection quality:

1. **Review "Not Helpful" feedback** regularly
2. **Look for patterns** - Multiple users reporting same issue
3. **Update AI agent instructions** based on feedback
4. **Adjust detection thresholds** if needed
5. **Create rules** for consistently misdetected patterns

---

## AI Agent Instructions

Configure the prompts and behavior for the three AI agents that power detection, quality checking, and remediation.

![alt text](<Documentation Screenshots/AI agents Instruction Page.png>)

### Page Header

- **Back Button (←)** - Return to Configuration
- **Title & Description**

### Agent Tabs

Three tabs for the three AI agents:

![alt text](<Documentation Screenshots/Agents Tabs.png>)

#### 1. Detection Agent

**Purpose**: Analyzes logs to detect security threats

**Tab Content**:
- **System Instructions** - Base prompt defining agent's role and behavior
- **User Instructions** - Additional guidance for specific detection needs
- **Examples** - Sample detections to guide AI (optional)


**Configuration Options**:
- **Instructions Text Area** - Large text box for editing prompts
- **Reset to Defaults** - Restore original instructions
- **Save Changes** - Apply new instructions

**Best Practices**:
- Be specific about threat types to focus on
- Include your organization's security policies
- Add context about your environment
- Provide examples of past detections

#### 2. Quality Agent

**Purpose**: Validates detections and filters false positives

**Tab Content**:
- **System Instructions** - Prompt for quality validation role
- **User Instructions** - Criteria for false positive filtering
- **Validation Rules** - Specific checks to perform


**Configuration Options**:
- Same as Detection Agent
- **Sensitivity Slider** - Adjust false positive filtering (if available)

**Best Practices**:
- Define what constitutes a false positive
- Include known benign patterns
- Specify confidence thresholds
- List trusted processes/users

#### 3. Remediation Agent

**Purpose**: Generates remediation plans for confirmed threats

**Tab Content**:
- **System Instructions** - Prompt for remediation advisor role
- **User Instructions** - Organization-specific response procedures
- **Response Templates** - Structured remediation guidance


**Configuration Options**:
- Same as other agents
- **Response Format** - Template for remediation structure

**Best Practices**:
- Include your incident response procedures
- Reference specific security tools you use
- Mention compliance requirements
- Define escalation procedures
- Specify automated responses available

### Editing Agent Instructions


1. **Click into Text Area** - Cursor appears
2. **Edit Text** - Modify instructions as needed
3. **Preview** (if available) - See how changes affect output
4. **Save Changes** - Apply to agent
5. **Test** - Create test detection to verify behavior

**Saving Behavior**:
- Changes apply immediately to new detections
- Existing detections are not affected
- Can revert with "Reset to Defaults"

---

## Common Components

These components appear throughout the application.

### Command Palette

Press **Ctrl+K** (Windows/Linux) or **Cmd+K** (Mac), or click search icon in navbar.

![alt text](<Documentation Screenshots/Command Palette.png>)

**Features**:
- **Quick Navigation** - Type to filter pages
- **Keyboard Navigation** - Arrow keys to select, Enter to navigate
- **Recent Pages** - Shows your recent navigation
- **Actions** - Quick access to common actions

**Example Usage**:
- Type "rules" → Navigate to Rule Management
- Type "dashboard" → Return to Dashboard
- Type "critical" → Filter to critical detections

### Notification Sidebar

Click the bell icon in navbar to open.

![alt text](<Documentation Screenshots/Notification sidebar.png>)

**Features**:
- **Recent Notifications** - New detections, system alerts
- **Unread Count** - Red badge on bell icon
- **Mark as Read** - Click notification to mark read
- **Clear All** - Button to clear all notifications
- **Auto-Hide** - Click outside to close

**Notification Types**:
- **New Detection** - AI detected a threat
- **Rule Triggered** - Security rule matched
- **System Alert** - Configuration issues, errors
- **Status Update** - Detection status changed

### Modals

Modals are overlay windows that focus attention on specific content.


**Features**:
- **Backdrop** - Darkened background
- **Close Options**:
  - Click X button
  - Click outside modal
  - Press Escape key
- **Actions** - Usually at bottom (Cancel, Save, etc.)
- **Tabs** (in detail modals) - Switch between sections

### Tables

Tables are used throughout for displaying lists.


**Common Features**:
- **Sorting** - Click column headers
- **Pagination** - Navigate pages at bottom
- **Row Selection** - Checkboxes for bulk actions
- **Row Click** - Opens detail view
- **Action Menu** - Three-dot menu per row
- **Empty State** - Helpful message when no data

### Export Modals

Common pattern for exporting data.


**Options**:
- **Format Selection** - CSV, JSON, Markdown, HTML
- **Filter Options** - Use current filters or customize
- **Date Range** - Specific time period
- **Preview** - See what will be exported
- **Download** - Generate and download file

---

## Keyboard Shortcuts

Speed up your workflow with keyboard shortcuts:

| Shortcut | Action |
|----------|--------|
| **Ctrl/Cmd + K** | Open Command Palette |
| **Escape** | Close modal/panel |
| **Ctrl/Cmd + /** | Focus search box |
| **?** | Show help (if available) |
| **Arrow Keys** | Navigate tables and lists |
| **Enter** | Open selected item |
| **Tab** | Navigate form fields |

---

## Tips & Best Practices

### For Security Analysts

1. **Start Your Day on Dashboard**
   - Review the four key metrics
   - Check for critical alerts
   - Scan recent detections table

2. **Use Filters Effectively**
   - Save time with the Filters panel
   - Combine multiple filters (severity + status + date)
   - Use search for specific IOCs

3. **Provide Feedback**
   - Help improve detection accuracy
   - Always add comments explaining your feedback
   - Review feedback regularly in Feedback History

4. **Status Management**
   - Keep detections organized with proper statuses
   - Add notes when changing status
   - Mark false positives to train AI

5. **Export Reports**
   - Use Markdown format for documentation
   - Use CSV for spreadsheet analysis
   - Use JSON for tool integration

### For Administrators

1. **Configure AI Model First**
   - System Settings → AI Model Configuration
   - Test with small batch sizes initially
   - Monitor API costs

2. **Start with Polling Mode**
   - More stable than realtime for most deployments
   - Adjust polling interval based on log volume
   - Switch to realtime only if needed

3. **Use Log Grouping Wisely**
   - Host-only grouping for simpler deployments
   - Host+User+IP for multi-user systems
   - Affects AI context and detection quality

4. **Create Rules Gradually**
   - Start with high-confidence rules
   - Test rules individually before enabling
   - Use upload YAML for bulk imports

5. **Customize AI Agent Instructions**
   - Tailor to your environment and threats
   - Include your security policies
   - Update based on feedback patterns

6. **Monitor Performance**
   - Check detection latency
   - Review false positive rate
   - Adjust batch size if needed

### General Tips

1. **Use Date Range Picker**
   - Essential for focused analysis
   - Use presets (Today, Last 7 days) for speed
   - Custom ranges for specific incidents

2. **Click Around Safely**
   - All detail views are read-only unless explicitly editing
   - Delete actions always require confirmation
   - Unsaved changes show warning before leaving

3. **Bulk Actions Save Time**
   - Select multiple detections to update status
   - Bulk trigger detection on logs
   - Bulk delete feedback

4. **Keep Browser Updated**
   - Use modern browsers (Chrome, Firefox, Edge, Safari)
   - Enable JavaScript
   - Allow notifications for real-time alerts

5. **Mobile Usage**
   - Full functionality on tablets
   - Limited on phones (use desktop for detailed work)
   - Hamburger menu (☰) for navigation

---

## Troubleshooting

### Common Issues

#### "No data available"

**Cause**: No logs ingested or date range has no data

**Solution**:
- Check if logs are being ingested (ask admin)
- Adjust date range to include known log activity
- Verify data source connections

#### "Failed to load"

**Cause**: Backend API not responding

**Solution**:
- Check if backend is running
- Verify network connection
- Check browser console for errors
- Contact system administrator

#### Detection not appearing

**Cause**: AI hasn't processed yet or polling interval not reached

**Solution**:
- Wait for next polling cycle
- Check System Settings for polling interval
- Manually trigger detection on specific logs

#### Chart shows no data

**Cause**: No data for selected grouping or filters too restrictive

**Solution**:
- Change chart grouping (use three-dot menu)
- Adjust or clear filters
- Expand date range

### Getting Help

If you encounter issues:

1. **Check System Settings** - Ensure AI model is configured
2. **Review Notifications** - System may show error alerts
3. **Check Browser Console** - Press F12 to see errors
4. **Contact Administrator** - Provide error messages and screenshots

---

## Conclusion

The AI-Security Platform provides a powerful, intuitive interface for security log analysis and threat detection. By following this guide, you should be able to:

- Navigate all sections of the platform
- Analyze logs and detections effectively
- Configure rules and AI settings
- Provide feedback to improve detection accuracy
- Export reports and data as needed

For technical setup and deployment information, see the [Deployment Guide](./as-deployment-guide.md).

For API integration and development, see the [API Reference](./as-api-reference.md).

---

**Last Updated**: November 2024  
**Version**: 1.0.0
