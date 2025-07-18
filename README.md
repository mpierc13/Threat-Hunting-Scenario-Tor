<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/mpierc13/Threat-Hunting-Scenario-Tor/blob/main/threat-hunting-scenario-tor-event-creation.md) 

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "mpierc13" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-04-09T22:07:10.6246828Z`. These events began at `2025-04-09T21:43:11.3191574Z`.

**Query used to locate events:**

```kql
DeviceFileEvents
| where DeviceName == "marcels-vm"
| where InitiatingProcessAccountName =="mpierc13"
| where FileName startswith "tor"
| where Timestamp >= datetime(2025-04-09T21:43:11.3191574Z)
| order by Timestamp desc 
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
![image](https://github.com/user-attachments/assets/e9060804-fce9-4018-943d-b7049f28be8d)


---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser". Based on the logs returned, at `2025-04-09T21:43:11.3191574Z`, an employee on the "mpierc13" device ran the file `tor-browser-windows-x86_64-portable-14.0.9.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql
DeviceProcessEvents
| where DeviceName == "marcels-vm"
| where ProcessCommandLine contains "tor-browser"
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine

```
![image](https://github.com/user-attachments/assets/621cc633-2ef3-4beb-b843-65476ddcad83)


---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "mpierc13" actually opened the TOR browser. There was evidence that they did open it at `2025-04-09T21:46:18.3259421Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents
| where DeviceName == "marcels-vm"
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")
| project Timestamp, DeviceName, AccountName, ActionType, FileName, FolderPath, SHA256, ProcessCommandLine
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/c2f7d6e3-d878-4735-a81a-c91c83a17fdd)


---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-04-09T21:46:51.0439209Z`, an employee on the "marcels-vm" device successfully established a connection to the remote IP address `127.0.0.1` on port `9150`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\mpierc13\desktop\tor browser\browser\firefox.exe`. There were a couple of other connections to sites over port `443` and port `80`.

**Query used to locate events:**

```kql
DeviceNetworkEvents
| where DeviceName == "marcels-vm"
| where InitiatingProcessAccountName != "system"
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath
```
![image](https://github.com/user-attachments/assets/bb3dc0b5-a32d-45d7-82f4-38969ad391aa)


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-04-09T21:43:11.3191574Z`
- **Event:** The user "mpierc13" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.9.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\mpierc13\Downloads\tor-browser-windows-x86_64-portable-14.0.9.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-04-09T21:43:11.3191574Z`
- **Event:** The user "mpierc13" executed the file `tor-browser-windows-x86_64-portable-14.0.9.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.9.exe /S`
- **File Path:** `C:\Users\mpierc13\Downloads\tor-browser-windows-x86_64-portable-14.0.9.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-04-09T21:46:18.3259421Z`
- **Event:** User "mpierc13" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\mpierc13\Desktop\Tor Browser\Browser\firefox.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-04-09T21:46:51.0439209Z`
- **Event:** A network connection to IP `127.0.0.1` on port `9150` by user "mpierc13" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\mpierc13\desktop\tor browser\browser\firefox.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-04-09T21:46:51.7963679Z` - Connected to `192.42.116.188` on port `80`.
  - `2025-04-09T21:46:49.7403936Z` - Local connection to `45.77.112.107` on port `443`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "mpierc13" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-04-09T22:07:10.4584787Z`
- **Event:** The user "mpierc13" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\mpierc13\Desktop\tor-shopping-list.txt`

---

## Summary

The user "mpierc13" on the "marcels-vm" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `marcels-vm` by the user `mpierc13`. The device was isolated, and the user's direct manager was notified.

---
