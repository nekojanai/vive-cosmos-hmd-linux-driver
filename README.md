# HTC Vive Cosmos HMD for Linux

Using a base HTC Vive Cosmos  
( theinside-out tracking model)  
as an extended/mirrored desktop monitor on Linux.  

No official or open-source driver supports this headset on Linux  
the panel wake protocol below was reverse-engineered from USB captures.

### status
#### working
- [x] display
- [x] speakers

#### todo
- [ ] camera based inside-out 3dof tracking

### wake up mr freeman
#### Motivation
The display behind the lenses doesn't turn on after being plugged in  
because it's in sleep mode until receiving a wake signal

#### the secret sauce
The Cosmos panel is controlled over a vendor HID protocol  
located on interface 0 of `0bb4:0313`  
unnumbered 64-byte feature reports:

```
byte 0     : 0x00        (report-id placeholder)
byte 1..2  : command     (e.g. )
byte 3     : length
byte 4     : marker      (0xff / 0x80 / 0x40 depending on command)
byte 5..   : payload
``` 
#### The activation sequence 
a bunch of SET_FEATURE packets captured from Windows SteamVR reveal
```
   panel power command
   78 29 38 00 80 ... [byte47]=01
   commit
   78 29 38 00 00 02
```
#### The firmware wedges silently if responses are not drained
Every SET_FEATURE queues a response that must be read back with GET_FEATURE.  
Leave enough undrained and the firmware stops processing display commands entirely.  
This is why naive replays failed. 

**Fix:** USB-reset the device first `USBDEVFS_RESET`  
then drain after every SET.  

#### Keepalive
The panel auto-sleeps within a minute of host silence.  
Re-sending the power-on + commit pair every 3s keeps it awake and is idempotent.  
A lone heartbeat packet and a 30s full-window replay were both insufficient.

#### Useful commands
`0x16` - `0x19` are ackable status queries  
full calibration data, serial, mura correction data can be streamed out via:  
`0x10` = open named stream  
`0x11` = read chunk  

#### misc commands
`0x78 0x29` = display/power family  

#### kthxbye