# Radiated — CTF Writeup

## Challenge Overview
The factory's radiation control system has been compromised. We need to find the system administration credentials in the network and authenticate to the HMI to rectify the situation.

## Flag
`HTB{m0d8u5_h45_n0_53cu217y!!}`

## How We Solved It — Reasoning

### Step 1: Extract Password from Modbus Registers
Read Modbus holding registers (FC3) at addresses 15-26 on the Modbus TCP port. The registers contain ASCII values spelling out the password `94mm453cu23d`.

### Step 2: Enroll Device
Write coil 1 to ON (FC5, value 0xFF00) to trigger enrollment. LCD changes from ENROLL_ERR to AUTH_ERR.

### Step 3: Authenticate via Hidden Form
The /access page contains a hidden HTML form with field name `unlock_code`. POST the password:
```
curl -X POST -d 'unlock_code=94mm453cu23d' http://TARGET:WEB_PORT/access -c cookies.txt
```
LCD shows ENABLED, session cookie set with granted=true.

### Step 4: Read Flag
GET /rate with authenticated session returns the flag in the radiation rate response.
