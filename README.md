# Job Application Tracker (Claude Skill)

This repository contains the configuration files and operational instructions to enable the **Job Application Tracker** skill on your Claude assistant.

The skill allows you to completely automate updating a **Notion** database with your job applications by cross-referencing recent **Gmail** messages and **Google Calendar** events.

---

## 🚀 How to Download and Activate the Skill

### Option 1: Direct Use in Claude Chat (Manual)

1. Download this repo as a ZIP
2. Go to **claude.ai → Sidebar → Customize → Skills → Upload a Skill**
3. Upload or import the file `SKILL.md` into your Claude interface (or include its contents inside your *Project Instructions* / *Custom Instructions* if you prefer manual synchronization).
4. Activate the skill in your chat session by typing `/job_application_tracker`
5. Let the SKILL do the work.

### Option 2: Continuous Automation with Claude Cowork
If you want the background monitoring to run continuously without manual intervention:
1. Open **Claude Cowork**.
2. Set up a scheduled task using the `/schedule` command.
3. Reference or load the downloaded skill file directly into the schedule body, setting your preferred recurrence interval (e.g., every 12 hours or once a day).

---

## 📋 Prerequisites: MCP Connectors
The skill relies on the MCP (Model Context Protocol) framework. It requires access to three tools to perform fully. If any are missing, Claude will guide you through setup:
* 🛠️ **Notion Connector** (Required to store the data)
* 📧 **Gmail Connector** (To scan for replies and application receipts)
* 📅 **Google Calendar Connector** (To detect scheduled interviews and tests)

---

## 🛠️ Troubleshooting & Limitations
* **Notion Account:** If you haven't a Notion account where to store the data, the skill will create a PDF file to download with the data formatted in a table.
* **Notion Plans:** Certain advanced SQL query tools require a Notion Business plan. If a `validation_error` occurs, Claude will seamlessly switch to a standard indexed keyword search (`notion-search`).
* **Email Languages:** The skill is configured to scan for both English and Italian application terminology. If you receive communications in another language, just let Claude know during activation so it can expand its keyword criteria.
