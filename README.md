# BEEP
Broadcast Emergency Encoding Parser<br>
*System Synopsis*
## What BEEP Does
BEEP is a real-time emergency pager monitoring system that receives, decodes, and displays Victorian emergency services pager traffic on a live web dashboard accessible from any device on your local network.<br>
# Hardware
*Component	Details*<br>
* Raspberry Pi 400:	Main computer — runs all software, hosts the web dashboard<br>
* RTL-SDR Dongle:	Realtek RTL2838UHIDIR USB software defined radio receiver<br>
* Antenna:	Connected to the RTL-SDR dongle to receive RF signal<br>
* SD Card:	Stores the Raspberry Pi OS and all software<br>
* Local Network:	WiFi or ethernet connecting the Pi to other viewing devices<br>
# Software Stack
*Software	Purpose*<br>
* Raspberry Pi OS Lite (32-bit, Bookworm):	Operating system — headless, no desktop required<br>
* rtl-sdr:	Linux driver and tools for the RTL-SDR dongle<br>
* rtl_fm:	Command line tool that tunes the dongle to 148.9125 MHz and demodulates the FM signal to raw audio<br>
* multimon-ng:	Decodes the raw audio stream and extracts POCSAG512 pager messages<br>
* Python 3:	Runs the main decoder and web server application<br>
* Flask	Python: web framework that serves the live dashboard<br>
* systemd:	Linux service manager that auto-starts BEEP on every boot<br>
# Radio Configuration
*Setting	Value*<br>
Frequency:	148.9125 MHz<br>
Protocol:	POCSAG512 (512 baud)<br>
Gain:	36.4 dB<br>
Squelch:	0 (no squelch)<br>
Sample rate:	22050 Hz<br>
________________________________________
# What the Python Code Does
The pager_decoder.py file does everything in one script — it has four main sections:<br>
________________________________________
## 1. Configuration and Data
At the top of the file all settings are defined — frequency, gain, squelch, protocol — along with three large lookup dictionaries built from your capcodes spreadsheet:<br>
•	USERS — maps every capcode to its unit name, group, district, mnemonic and agency (721 entries, stored both with and without leading zeros for reliable lookup)<br>
•	MNEMONIC_TO_UNIT — maps brigade/station mnemonics to unit names for decoding responding units in message text (407 entries)<br>
•	FRV_STATION_TO_NAME — maps FRV station numbers to station names, groups and districts (86 entries)<br>
•	BLACKLIST — capcodes to silently ignore completely (16 entries)<br>
•	DISTRICTS and GROUPS — sorted lists for the filter dropdowns (34 districts, 169 groups)<br>
Also stored is the BEEP logo as a base64 encoded string so it can be embedded directly in the web page without needing a separate file.<br>
________________________________________
## 2. Decoder Thread
This runs continuously in the background on a separate thread:<br>
*	Launches rtl_fm as a subprocess, tuned to 148.9125 MHz, which captures raw FM audio from the RTL-SDR dongle<br>
* Pipes that audio directly into multimon-ng which decodes POCSAG512 pager messages<br>
*	Reads the output of multimon-ng line by line, extracting the capcode and message text using regular expressions<br>
*	Checks each capcode against the blacklist — silently drops blacklisted messages<br>
*	Looks up the capcode in the USERS dictionary to identify the unit, agency, group and district<br>
*	Determines the message type — ALERT (@@ALERT prefix), NON-EMERG (Hb prefix) or ADMIN (QD prefix)<br>
*	For ALERT messages containing a job call number (F, S or E followed by digits), groups them into a job card:<br>
o	New job — creates a new job entry, determines card colour from the area code after @@ALERT (FRV if 5 digits starting with 0, CFA/other if mnemonic + number), extracts responding units from the message text<br>
o	Existing job — adds the new message to the job, updates responding units, moves the job to the top of the feed<br>
*	For messages without a call number — stores as individual message cards<br>
*	Broadcasts every new or updated message to all connected browsers via Server-Sent Events<br>
*	Appends every message to pager_messages.txt as a permanent log<br>
________________________________________
## 3. Message and Unit Decoding
Several helper functions handle intelligent decoding:<br>
*	Area code decoding — determine_job_agency() reads the area code immediately after @@ALERT. A 5-digit code starting with 0 is an FRV area (digits 2-3 = station number, digits 4-5 = area number). A mnemonic + number is a CFA area. Both are looked up to produce the Unit — Group — District badge on the job card<br>
*	Responding unit extraction — extract_responding_units() scans message text for:<br>
o	CFA standard response: C + mnemonic (e.g. CLANW → Langwarrin, red chip)<br>
o	CFA specific appliance: mnemonic + appliance code + number (e.g. LANWT1 → Langwarrin Tanker 1, red chip)<br>
o	FRV appliances: appliance code + station number (e.g. P57, AP91, R87, yellow chip)<br>
o	SES units: mnemonic + 1 (e.g. KNOX1 → Knox Unit, orange chip)<br>
o	Fire ground channels: FDG + numbers (e.g. FDG20, grey chip)<br>
________________________________________
## 4. Flask Web Dashboard
A Flask web server runs on port 5000 serving the BEEP dashboard:<br>
*	/ — main dashboard page, rendered with all current jobs and messages<br>
*	/stream — Server-Sent Events endpoint that pushes live updates to all connected browsers instantly without page refresh. 
Uses a per-client queue so each browser gets its own message stream. Sends a keepalive every 30 seconds to maintain the connection
*	/search — searches the permanent pager_messages.txt log file and returns matching lines as plain text<br>
*	/download — downloads the complete log file<br>
The dashboard HTML/CSS/JavaScript is embedded directly in the Python file as a template string. It includes:<br>
*	BEEP logo and branding in the header<br>
*	"Beep Happens" status indicator (green when connected, red when reconnecting)<br>
*	Agency filter pills — CFA, FRV, SES, MSAR, LANW, TZV, WATCH, DMO, Unknown<br>
*	Alert type filter pills — Alert, Non-emerg, Admin<br>
*	District dropdown (34 districts)<br>
*	Group dropdown (169 groups, with search box)<br>
*	7 stat counters — Messages, Alerts, Non-emerg, Admin, Known capcodes, Unknown capcodes, Last message time<br>
*	Job cards — colour coded by agency, showing all messages inline with responding unit chips at the bottom, moving to top when updated<br>
*	Individual message cards for non-job messages<br>
*	Auto-reconnecting SSE client that updates the page live<br>
________________________________________
# Auto-Start
A systemd service file at /etc/systemd/system/pager-decoder.service ensures BEEP starts automatically every time the Pi powers on, restarts if it crashes, and logs all output to the system journal viewable with journalctl -u pager-decoder.<br>
________________________________________
# Agencies Supported
## Agency	Full Name	Colour
* CFA:	Country Fire Authority	Fire truck red<br>
* FRV:	Fire Rescue Victoria	Yellow<br>
* SES:	State Emergency Service	Orange<br>
* MSAR:	Marine Search and Rescue	Blue<br>
* LANW:	Langwarrin CFA	Maroon<br>
* TZV:	Triple Zero Victoria	Green<br>
* WATCH:	Unknown / Trying to figure out Purple<br>
* DMO:	District Management Officer	Black / white<br>
________________________________________
# Data Flow Summary
→ Antenna → RTL-SDR dongle → rtl_fm (tune + demodulate)<br>
→ multimon-ng (decode POCSAG512)<br>
→ Python decoder (lookup + classify + group)<br>
→ Flask (serve dashboard + SSE)<br>
→ Browser (live display on any device)<br>
→ pager_messages.txt (permanent log)<br>
