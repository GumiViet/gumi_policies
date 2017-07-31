# Contents
---

[TOC]

---

## Log Policy

Separate into AP and Action log as below.

|No|Classification|Category|Outline|
|:--|:--|:--|:--|
|1|System|AP Log|Output log when the application processing|
|2|User|Action Log|Output log during various operations|

## 1. AP Log

Outut log in case of screen drawing, button clicking, DB access, ...

### Log Level

|No|Name|Description|
|:--|:--|:--|
|1|FATAL|Output log when exception, abnormal termination occurring|
|2|ERROR|Output log when DB connection error, no responding service |
|3|WARN|Output log when displaying error page, invalidation input|
|4|INFO|Output processing information|
|4|DEBUG|Output debugging process information when developing|

### Log Format

Includes the following information. 
- Output datetime- Log level- Username- Log message- Original classIn addition, need to mask personal information such as e-mail address and password at the time of log output.
(Example) authentication log```2014/04/18 10:01:45.677;INFO;sapphire_user;User Login : name=[sapphire_user] password=[********];RegistrantsController
```

### Output Timing
- Web Service  - Screen display  - Button / link pressed  - Login / Logout  - DB Access  - Error Occurring
  - Exception
  - Abnormal termination- System function  - Process start/end  - DB Access  - Error Occurring
  - Exception
  - Abnormal termination

### RotationRotate by the file size.
File size threshold to rotate is set to 200MB, to keep a log of up to 10 generations.
## 2. Action Log

Collect the action log of the user for improving UI or additional functions.
The collection of the action log to use Google Analytics.