# Multiple_Protocol_Industrial_Communication_Gateway

A two-node ESP32 gateway that reads an MPU6050 once, sends the same packet over UART and software SPI, validates it, and monitors link health on an OLED and a browser dashboard.

Overview

Industrial systems use different communication interfaces. A gateway has to collect device data, carry it, and tell you whether each link is still healthy.

This project is a working prototype of that idea on two ESP32 boards:

Node 1 reads motion data from an MPU6050 over I²C and transmits the same packet over UART and software SPI.
Node 2 is the gateway. It receives both streams, validates every packet, tracks the health of each protocol, picks an active path, and shows everything on an SSD1306 OLED and a local web dashboard served from its own Wi-Fi access point.

Core idea: acquire once → transmit over two interfaces → validate at the gateway → monitor communication health.

Features
I²C sensor acquisition from the MPU6050 (14-byte burst read from register 0x3B)
One common packet format used on both links
UART primary link and software (bit-banged) SPI backup link
Packet decoding and validation (8 fields, numeric check)
Per-protocol link health from the last valid packet time (6 s timeout)
Automatic active-path selection: UART preferred, SPI as backup
Live status on an SSD1306 OLED
Browser dashboard hosted by Node 2: no PC server and no internet required
System Architecture
Node 2 · ESP32
Node 1 · ESP32
UART (primary)
Software SPI (backup)
I²C
HTTP / JSON
I²C
MPU6050motion sensor
Sensor read
Packet creation
UART + software SPI TX
UART + software SPI RX
Validation
Protocol monitoring
Wi-Fi AP + web server
SSD1306 OLED
Browser dashboard
Hardware
Part	Quantity	Notes
ESP32 DevKit V1 (ESP32-WROOM-32)	2	Node 1 and Node 2
MPU6050 IMU	1	I²C address 0x68, on Node 1
SSD1306 OLED, 128 × 64	1	I²C address 0x3C, on Node 2
Jumper wires	several	Between the two boards, plus a common ground
Pin connections
Signal	Node 1	Node 2	Direction
I²C SDA	GPIO 21 (to MPU6050)	GPIO 21 (to OLED)	local to each node
I²C SCL	GPIO 22 (to MPU6050)	GPIO 22 (to OLED)	local to each node
UART TX / RX	TX GPIO 17	RX GPIO 16	Node 1 → Node 2
UART RX / TX	RX GPIO 16	TX GPIO 17	Node 2 → Node 1
SPI MOSI	GPIO 23	GPIO 23	Node 1 → Node 2
SPI SCLK	GPIO 18	GPIO 18	Node 1 → Node 2
SPI CS	GPIO 5	GPIO 5	Node 1 → Node 2
Ground	GND	GND	Connect both grounds

SPI MISO is not used because the SPI link is one-way.

Communication Protocols
Interface	Role	Configuration
I²C	Sensor / display interface	SDA and SCL; Node 1 reads 14 bytes from 0x3B and scales to engineering units
UART	Primary link	UART2, 115200 baud, 8-N-1, newline-terminated text packet
Software SPI	Backup link	Mode 0, MSB first; Node 1 drives MOSI, SCLK and CS with GPIO; Node 2 samples MOSI on SCLK rising edges
Packet format

The same payload is used on both links.

UART message (ends with a newline):

text
M,seq,ax,ay,az,gx,gy,gz
M,43,1.01,0.06,0.03,0.51,2.06,1.50
Field	Meaning
M	Message marker
seq	Sequence number
ax ay az	Acceleration
gx gy gz	Angular velocity

SPI frame:

Part	Content
Preamble	AA AA AA AA 7E (5 bytes)
Payload	64 ASCII bytes, padded with spaces
Total	5 + 64 = 69 bytes = 552 bits
Gateway Processing (Node 2)
Receive: UART line buffer or SPI bit capture
Synchronize: SPI preamble search (0xAA7E)
Decode: bytes → text packet
Validate: 8 fields present and numeric
Update: counters, timestamps, latest sensor values
Present: OLED and web dashboard

A valid packet updates the latest sensor values and the counter of its source protocol. An invalid packet increments the invalid-packet counter.

Protocol Health and Selection
Status	Meaning
OK	A valid packet was received within the last 6 seconds
LOST	No valid packet for more than 6 seconds
WAIT	No packet received yet

Selection rule: UART OK → ACTIVE: UART; else SPI OK → ACTIVE: SPI; else ACTIVE: NONE.

Implementation note: selection and status reporting happen at Node 2. The prototype does not yet tell Node 1 to stop a failed interface, and it does not filter every displayed value to the selected path only.

Web Dashboard

Node 2 runs a Wi-Fi access point and serves the dashboard locally.

Power both nodes.
Connect your laptop or phone to the Node 2 access point (SSID INDUSTRIAL_GATEWAY; the password is set in the Node 2 firmware, so change it before sharing or deploying).
Open http://192.168.4.1 in a browser.

The page polls the /api endpoint and Node 2 returns the current values as JSON, so the dashboard updates without a separate server.

It shows UART and SPI status, the active protocol, MPU6050 acceleration and gyroscope readings, UART / SPI / invalid packet counts, and the latest sequence and source.

Results

Observed on the working prototype (Node 2 serial monitor, OLED and dashboard):

Area	Observation
UART	Packets received and validated; sequence numbers advanced continuously; sensor values decoded
SPI	552 bits captured per frame; preamble sync and decoding succeeded; sequence numbers matched UART
Monitoring	OLED showed U:OK S:OK; active protocol shown as UART; packet and error counters updated

Example serial output:

text
UART #252  ...  OK
SPI BITS: 552 (full frame = 552)
SPI #252  ...  OK

Counter snapshots (single readings, not a controlled test):

Source	UART packets	SPI packets	Invalid
Browser dashboard (seq 82)	69	44	24
OLED (seq 366)	338	264	89

SPI counts trail UART in both snapshots, which is consistent with the limitation below. Per-protocol packet delivery ratio, latency and error rate have not been measured yet.

Limitations
The SPI receiver uses software GPIO polling, not a hardware SPI-slave peripheral.
Software-timed SPI can be sensitive to Wi-Fi and other processing load.
Path selection reports the preferred healthy link; it is not a sender-controlled switchover.
Single MPU6050 sensor and a short-range local Wi-Fi dashboard.
Future Improvements
 Source-specific data buffer, displaying only the selected path
 Acknowledgement / retransmission or sender-side failover control
 Controlled tests for packet delivery ratio, latency and error rate
 Hardware SPI-slave receiver on Node 2
 More sensor nodes and industrial interfaces
 Secure Wi-Fi access and optional cloud logging
Getting Started

Adjust file and folder names to match your repository.

Requirements

2 × ESP32 DevKit V1 with MicroPython flashed
MPU6050 and SSD1306 OLED wired as in the pin table
The SSD1306 MicroPython driver on Node 2
A MicroPython uploader such as mpremote, Thonny or ampy

Steps

Wire the two boards as shown in Pin connections and join the grounds.
Copy the Node 1 program to the first ESP32 (for example as main.py).
Copy the Node 2 program, and the OLED driver, to the second ESP32.
Reset both boards. Node 1 starts transmitting; Node 2 starts receiving.
Open the Node 2 serial monitor to see packets, or connect to the access point and browse to http://192.168.4.1.
Repository Structure
text
.
├── node1/            # Transmitter firmware (MPU6050 → UART + software SPI)
├── node2/            # Gateway firmware (RX, validation, monitoring, OLED, dashboard)
├── docs/
│   └── images/       # Photos and diagrams used in this README
└── README.md
Specifications
Item	Value
Controllers	2 × ESP32 DevKit V1 (ESP32-WROOM-32)
Sensor	MPU6050, I²C 0x68
Display	SSD1306 128 × 64, I²C 0x3C
UART	UART2, 115200 baud, 8-N-1; TX 17 / RX 16
Software SPI	MOSI 23, SCLK 18, CS 5; Mode 0, MSB first
SPI frame	5-byte preamble + 64-byte payload = 552 bits
Monitoring timeout	6 s since last valid packet
Dashboard	Node 2 AP, 192.168.4.1, JSON at /api
