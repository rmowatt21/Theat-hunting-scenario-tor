# Theat-hunting-scenario-tor

<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/rmowatt21/Theat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

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

Searched for any file that had the string `tor` in it and discovered what looks like the user `azurelab` downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-04-08T19:18:41.9897782Z`. These events began at `2025-04-08T18:20:49.4108717Z`.

**Query used to locate events:**

```kql
DeviceFileEvents
| where DeviceName == "tor-network"
| where FileName contains "tor"
| where Timestamp >= datetime(2025-04-08T18:20:49.4108717Z)
| order by Timestamp desc 
| project Timestamp, DeviceName, ActionType, FileName, FolderPath
```
![image](https://github.com/user-attachments/assets/5907a04d-0469-41b0-ac56-5240f837dd0b)

---

### 2. Searched the `DeviceProcessEvents` Table

Searched on the DeviceName `tor-network` in the FileName 'tor', 'tor.exe'. Based on the logs returned, at `2025-04-08T18:20:49.4108717Z`, a user on the `tor-network` device ran the file `tor-browser-windows-x86_64-portable-14.0.9.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents 
| where DeviceName == "tor-network"
| where AccountName == "azurelab"
| where ProcessCommandLine contains "" "tor-browser-windows-x86_64-portable-14.0.9.exe"
| project Timestamp, DeviceName, AccountName, FileName, FolderPath, ProcessCommandLine
```
![image](https://github.com/user-attachments/assets/180b3ca1-5482-41cd-bb5e-50637da15d67)


---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user 'azurelab' actually opened the TOR browser. There was evidence that they did open it at `2025-04-08T18:20:49.4108717Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents 
| where DeviceName == "tor-network"
| where FileName in ("tor", "tor.exe", "firefox.exe")
| project Timestamp, DeviceName, AccountName, FileName, FolderPath, ProcessCommandLine
| order by Timestamp desc 
```
![image](https://github.com/user-attachments/assets/cd53dfcb-edff-4e7c-903d-36f1d47f26e9)

---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `2025-04-08T18:29:48.3012912Z`, an employee on the "tor-network" device successfully established a connection to the remote IP address `109.120.157.121` on port `9001`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to sites over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents  
| where DeviceName == "tor-network"  
| where InitiatingProcessAccountName != "system"  
| where InitiatingProcessFileName in ("tor.exe", "firefox.exe")  
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150", "80", "443")  
| project Timestamp, DeviceName, InitiatingProcessAccountName, ActionType, RemoteIP, RemotePort, RemoteUrl, InitiatingProcessFileName, InitiatingProcessFolderPath  
| order by Timestamp desc
```
![image](https://github.com/user-attachments/assets/37039734-2189-4f88-bcf3-1f02b8435d41)


---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-04-08T18:21:52.3744615Z`
- **Event:** The user "azurelab" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.9.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\azurelab\Downloads\tor-browser-windows-x86_64-portable-14.0.9.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-04-08T18:21:52.3744615Z`
- **Event:** The user "azurelab" executed the file `tor-browser-windows-x86_64-portable-14.0.9.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.9.exe /S`
- **File Path:** `C:\Users\azurelab\Downloads\tor-browser-windows-x86_64-portable-14.0.9.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-04-08T18:21:52.3744615Z`
- **Event:** User "azurelab" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\azurelab\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `2025-04-08T18:21:52.3744615Z`
- **Event:** A network connection to IP `176.198.159.33` on port `9001` by user "azurelab" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `C:\Users\azurelab\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-04-08T18:29:50.1972793Z` - Connected to `109.120.157.121` on port `9001`.
  - `2025-04-08T18:21:52.3744615Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "azurelab" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-04-08T19:18:41.9897782Z`
- **Event:** The user "azurelab" created a file named `tor-shopping-list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\azurelab\Desktop\tor-shopping-list.txt`

---

## Summary

The user "azurelab" on the "tor-network" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor-shopping-list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `tor-network` by the user `azurelab`. The device was isolated, and the user's direct manager was notified.

---
