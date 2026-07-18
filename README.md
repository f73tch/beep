# BEEP
Broadcast Emergency Encoding Parser System Synopsis
# What BEEP Does
BEEP is a real-time emergency pager monitoring system that receives, decodes, and displays Victorian emergency services pager traffic on a live web dashboard accessible from any device on your local network.
# Hardware
Component	Details

Raspberry Pi 400	Main computer — runs all software, hosts the web dashboard

RTL-SDR Dongle	Realtek RTL2838UHIDIR USB software defined radio receiver

Antenna	Connected to the RTL-SDR dongle to receive RF signal

SD Card	Stores the Raspberry Pi OS and all software

Local Network	WiFi or ethernet connecting the Pi to other viewing devices
# Software Stack
Software	Purpose

Raspberry Pi OS Lite (32-bit, Bookworm)	Operating system — headless, no desktop required

rtl-sdr	Linux driver and tools for the RTL-SDR dongle

rtl_fm	Command line tool that tunes the dongle to 148.9125 MHz and demodulates the FM signal to raw audio

multimon-ng	Decodes the raw audio stream and extracts POCSAG512 pager messages

Python 3	Runs the main decoder and web server application

Flask	Python web framework that serves the live dashboard

systemd	Linux service manager that auto-starts BEEP on every boot
# Radio Configuration

Setting	Value

Frequency	148.9125 MHz

Protocol	POCSAG512 (512 baud)

Gain	36.4 dB

Squelch	0 (no squelch)

Sample rate	22050 Hz
________________________________________
# What the Python Code Does
The pager_decoder.py file does everything in one script — it has four main sections:
________________________________________
# 1. Configuration and Data
At the top of the file all settings are defined — frequency, gain, squelch, protocol — along with three large lookup dictionaries built from your capcodes spreadsheet:

•	USERS — maps every capcode to its unit name, group, district, mnemonic and agency (721 entries, stored both with and without leading zeros for reliable lookup)

•	MNEMONIC_TO_UNIT — maps brigade/station mnemonics to unit names for decoding responding units in message text (407 entries)

•	FRV_STATION_TO_NAME — maps FRV station numbers to station names, groups and districts (86 entries)

•	BLACKLIST — capcodes to silently ignore completely (16 entries)

•	DISTRICTS and GROUPS — sorted lists for the filter dropdowns (34 districts, 169 groups)

Also stored is the BEEP logo as a base64 encoded string so it can be embedded directly in the web page without needing a separate file.
________________________________________
# 2. Decoder Thread
This runs continuously in the background on a separate thread:

•	Launches rtl_fm as a subprocess, tuned to 148.9125 MHz, which captures raw FM audio from the RTL-SDR dongle

•	Pipes that audio directly into multimon-ng which decodes POCSAG512 pager messages

•	Reads the output of multimon-ng line by line, extracting the capcode and message text using regular expressions

•	Checks each capcode against the blacklist — silently drops blacklisted messages

•	Looks up the capcode in the USERS dictionary to identify the unit, agency, group and district

•	Determines the message type — ALERT (@@ALERT prefix), NON-EMERG (Hb prefix) or ADMIN (QD prefix)

•	For ALERT messages containing a job call number (F, S or E followed by digits), groups them into a job card: 

o	New job — creates a new job entry, determines card colour from the area code after @@ALERT (FRV if 5 digits starting with 0, CFA/other if mnemonic + number), extracts responding units from the message text

o	Existing job — adds the new message to the job, updates responding units, moves the job to the top of the feed

•	For messages without a call number — stores as individual message cards

•	Broadcasts every new or updated message to all connected browsers via Server-Sent Events

•	Appends every message to pager_messages.txt as a permanent log
________________________________________
# 3. Message and Unit Decoding
Several helper functions handle intelligent decoding:

•	Area code decoding — determine_job_agency() reads the area code immediately after @@ALERT. A 5-digit code starting with 0 is an FRV area (digits 2-3 = station number, digits 4-5 = area number). A mnemonic + number is a CFA area. Both are looked up to produce the Unit — Group — District badge on the job card

•	Responding unit extraction — extract_responding_units() scans message text for: 

o	CFA standard response: C + mnemonic (e.g. CLANW → Langwarrin, red chip)

o	CFA specific appliance: mnemonic + appliance code + number (e.g. LANWT1 → Langwarrin Tanker 1, red chip)

o	FRV appliances: appliance code + station number (e.g. P57, AP91, R87, yellow chip)

o	SES units: mnemonic + 1 (e.g. KNOX1 → Knox Unit, orange chip)

o	Fire ground channels: FDG + numbers (e.g. FDG20, grey chip)
________________________________________
# 4. Flask Web Dashboard
A Flask web server runs on port 5000 serving the BEEP dashboard:

•	/ — main dashboard page, rendered with all current jobs and messages

•	/stream — Server-Sent Events endpoint that pushes live updates to all connected browsers instantly without page refresh. 
Uses a per-client queue so each browser gets its own message stream. Sends a keepalive every 30 seconds to maintain the connection

•	/search — searches the permanent pager_messages.txt log file and returns matching lines as plain text

•	/download — downloads the complete log file
The dashboard HTML/CSS/JavaScript is embedded directly in the Python file as a template string. It includes:

•	BEEP logo and branding in the header

•	"Beep Happens" status indicator (green when connected, red when reconnecting)

•	Agency filter pills — CFA, FRV, SES, MSAR, LANW, TZV, WATCH, DMO, Unknown

•	Alert type filter pills — Alert, Non-emerg, Admin

•	District dropdown (34 districts)

•	Group dropdown (169 groups, with search box)

•	7 stat counters — Messages, Alerts, Non-emerg, Admin, Known capcodes, Unknown capcodes, Last message time

•	Job cards — colour coded by agency, showing all messages inline with responding unit chips at the bottom, moving to top when updated

•	Individual message cards for non-job messages

•	Auto-reconnecting SSE client that updates the page live
________________________________________
# Auto-Start
A systemd service file at /etc/systemd/system/pager-decoder.service ensures BEEP starts automatically every time the Pi powers on, restarts if it crashes, and logs all output to the system journal viewable with journalctl -u pager-decoder.
________________________________________
# Agencies Supported
Agency	Full Name	Colour

CFA	Country Fire Authority	Fire truck red

FRV	Fire Rescue Victoria	Yellow

SES	State Emergency Service	Orange

MSAR	Marine Search and Rescue	Blue

LANW	Langwarrin CFA	Maroon

TZV	Triple Zero Victoria	Green

WATCH	Watch Office / Control	Purple

DMO	District Management Officer	Black / white
________________________________________
# Data Flow Summary
Antenna → RTL-SDR dongle → rtl_fm (tune + demodulate)

→ multimon-ng (decode POCSAG512)

→ Python decoder (lookup + classify + group)

→ Flask (serve dashboard + SSE)

→ Browser (live display on any device)

→ pager_messages.txt (permanent log)
