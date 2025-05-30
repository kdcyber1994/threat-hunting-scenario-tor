<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/kdcyber1994/threat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

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

Searched the DeviceFileEvents table for ANY file that had the string “tor” in it and discovered what looks like the user “MdeTest” downloaded a tor installer, did something that resulted in many tor-related files being copied to the desktop and the creation of a file called “tor-shopping-list.txt” on the desktop at: 2025-05-06T00:34:21.8671095Z. These events began at:2025-05-06T00:08:53.2178449Z

**Query used to locate events:**

```kql
DeviceFileEvents
| where FileName startswith "tor"
| where DeviceName == "kevin-mde-test"
| where Timestamp >= datetime(2025-05-06T00:08:53.2178449Z)
| order by Timestamp desc
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
<img alt="image" src="https://github.com/user-attachments/assets/8b009085-7dff-4bef-980c-e9edde80b21d">

---

### 2. Searched the `DeviceProcessEvents` Table

Searched the DeviceProcessEvents table for any ProcessCommandLine that contained the string "tor-browser-windows-x86_64-portable-14.5.1.exe". Based on the logs returned at:2025-05-06T00:11:09.137328Z, an employee on the “kevin-mde-test” device ran the file tor-browser-windows-x86_64-portable-14.5.1.exe from their downloads folder, using a command that triggered a silent installation. 

**Query used to locate event:**

```kql

DeviceProcessEvents
| where DeviceName == "kevin-mde-test"
| where ProcessCommandLine contains "tor-browser-windows-x86_64-portable-14.5.1.exe"
| project Timestamp, DeviceName, AccountName, ActionType, FolderPath, SHA256, ProcessCommandLine
```
<img alt="image" src="https://github.com/user-attachments/assets/d879a8ae-7f93-41c0-9e59-452827318ba9">

---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched the DeviceProcessEvents table for any indication that user “MdeTest” actually opened the tor browser. There was evidence that they did open it at: 2025-05-06T00:12:24.5560725Z. There were several other instances of firefox.exe (Tor) as well as tor.exe spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents
| where FileName has_any("tor.exe","firefox.exe")
| where DeviceName == "kevin-mde-test"
| project Timestamp, DeviceName, AccountName, ActionType, FolderPath, SHA256, ProcessCommandLine
```
<img alt="image" src="https://github.com/user-attachments/assets/c507ae2f-9eeb-48a8-b366-9f5a9ffa9f40">

---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched the DeviceNetworksEvents table for any indication the tor browser was used to establish a connection using any of the known ports. At 2025-05-06T00:12:57.0548159Z, on the computer named "kevin-mde-test," the user "MdeTest" ran the program "tor.exe" from the folder "c:\users\mdetest\desktop\tor browser\browser\torbrowser\tor\" and successfully established a connection to the remote IP address 157.143.119.5 on port 9001. There were a few other connections.

**Query used to locate events:**

```kql
DeviceNetworkEvents
| where DeviceName == "kevin-mde-test"
| where InitiatingProcessAccountName != "system"
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150")
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath
| order by Timestamp desc
```
<img alt="image" src="https://github.com/user-attachments/assets/cb04f52a-0489-4f1f-94aa-64d532f546ac">

---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-05-06T00:08:53.2178449Z`
- **Event:** The user "MdeTest" downloaded a file named `tor-browser-windows-x86_64-portable-14.5.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\MdeTest\Downloads\tor-browser-windows-x86_64-portable-14.5.1.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-05-06T00:11:09.137328Z`
- **Event:** The user "MdeTest" executed the file `tor-browser-windows-x86_64-portable-14.5.1.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.5.1.exe /S`
- **File Path:** `C:\Users\MdeTest\Downloads\tor-browser-windows-x86_64-portable-14.5.1.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-05-06T00:12:24.5560725Z`
- **Event:** User "MdeTest" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\MdeTest\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-05-06T00:12:57.0548159Z`
- **Event:** A network connection to IP `157.143.119.5` on port `9001` by user "MdeTest" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `C:\Users\MdeTest\Desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-05-06T00:12:57.0548159Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "MdeTest" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-05-06T00:34:21.8671095Z`
- **Event:** The user "MdeTest" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\MdeTest\Desktop\tor-shopping-list.txt`

---

## Summary

The user "MdeTest" on the "kevin-mde-test" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `kevin-mde-test` by the user `MdeTest`. The device was isolated, and the user's direct manager was notified.

---
