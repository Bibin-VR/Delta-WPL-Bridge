# Delta-WPL-Bridge

Command-line programming for Delta DVP PLCs, without WPLSoft. WPLSoft is the
Windows program Delta ships for the DVP series, and it has no command line.
Delta-WPL-Bridge reads and writes WPLSoft's own `.dvp` program files and
talks to DVP PLCs and to Delta's PLC simulator directly. You can write a
program as text, turn it into a WPLSoft ladder file, download it, test it and
read it back, all from a terminal. Every step is one command, so an AI agent
such as Claude Code can do the same.

> Source lives in a private repository. This repo is a public overview.

## What it covers

It turns an instruction list into a WPLSoft 2.x `.dvp` file, with device
comments and with the ladder drawn the way WPLSoft draws it, and it reads
`.dvp` files back as instructions and ladder. The encoder handles contacts
and coils with rising and falling edges, compare contacts, branches (ANB,
ORB, MPS, MRD, MPP), timers, counters, SET and RST, E and F index registers,
and the API instructions in WPLSoft's own instruction table, such as MOV,
DMOV, ADD and ZRST. An instruction it can't encode stops the build with its
line number, and no file is written.

On the PLC side it downloads with read-back verification, uploads into a
ladder file, switches between RUN and STOP, and reads and writes X, Y, M, S,
T, C and D devices. It speaks Modbus ASCII, 9600 baud 7E1 by default, as
WPLSoft does. The same commands work with DVPSimulator, the PLC simulator
that ships with WPLSoft 2.52.

## A short session

```text
$ python3 dvp.py build start_stop_estop.il -o start_stop_estop.dvp
start_stop_estop.dvp: model 2, version 216, 9 steps, checksum ok
      0  LDP X0
      3  ANI M1002
      4  OR Y0
      5  AND X1
      6  AND X2
      7  OUT Y0
      8  END
ladder (rung starts: row 0 = step 0, row 2 = step 8)
    0    [PX0] [/M1002]     [X1]     [X2]    -----    -----    -----    -----    -----    -----    -----     (Y0)
    1     [Y0]    -----        |        .        .        .        .        .        .        .        .        .
    2    -----    -----    -----    -----    -----    -----    -----    -----    -----    -----    -----     [END

$ python3 dvp.py download start_stop_estop.dvp sim
the PLC is in RUN: add --stop to stop it for the download (this stops the machine)

$ python3 dvp.py download start_stop_estop.dvp sim --stop --run
PLC was in RUN: stopped it
downloaded and verified 9 steps
PLC set to RUN
```

The instruction list for that program, a VFD start, stop and E-stop:

```text
#X0 START PB (NO)
#X1 STOP PB (NC)
#X2 E-STOP (NC)
#Y0 VFD RUN (FWD)
LDP  X0      ; START, rising edge only
ANI  M1002   ; not on the first scan after power-up
OR   Y0      ; seal-in
AND  X1      ; STOP healthy (NC)
AND  X2      ; E-STOP healthy (NC)
OUT  Y0      ; VFD run
END
```

The E-STOP in this example is a control stop. A real emergency stop also
needs a hardwired safety circuit, such as the drive's STO input or a safety
relay.

## Commands

| Command | What it does |
|---|---|
| `build PROGRAM.il -o OUT.dvp` | instruction list to WPLSoft ladder file (`--model SV` or `SV2`) |
| `dump FILE.dvp`, `il FILE.dvp` | a `.dvp` file as instructions and ladder, or as an instruction list only |
| `check DIR...` | rebuild every WPLSoft file under DIR from its instructions and compare byte for byte |
| `download FILE LINK` | write a program to the PLC, read it back to verify; `--stop`, `--run` |
| `upload LINK -o OUT.dvp` | read the program from the PLC into a ladder file |
| `plc LINK status`, `run`, `stop` | RUN/STOP state, PLC ID, password flag |
| `plc LINK get D0 Y0`, `set M10=1` | read and write devices |

A LINK is one of:

- `sim://HOST:PORT` for Delta's simulator, which WPLSoft starts on TCP port
  10000 (plain `sim` means `127.0.0.1:10000`)
- `serial:/dev/cu.usbserial-X` for the PLC's programming port through a USB cable
- `ssh://PC/COM3` for a COM port on a Windows PC, through a one-shot PowerShell relay

## Safety

- A download refuses a PLC that is in RUN unless `--stop` is given, because
  stopping the PLC stops the machine.
- A password-protected PLC is refused before anything is changed, so it is
  never left stopped.
- Every download is read back and compared before the PLC goes to RUN.
- The SSH relay opens the COM port for one batch of requests and closes it
  right after.

## How it was checked

- Running `check` over 257 `.dvp` files from real projects rebuilt every rung
  from its instruction list and compared it with the ladder WPLSoft had
  saved. 3,492 of 3,493 rungs came out identical byte for byte, from files
  saved for five WPLSoft model codes. The one that differs is in a file whose saved ladder
  doesn't match its own instruction list. Ten more rungs use instruction
  words, or in one case a rung shape, that the tool doesn't handle yet, and
  `check` lists them. Two files that aren't WPLSoft 2.x programs were
  skipped, and one that decodes to invalid instructions was set aside.
- In Delta's DVPSimulator, two programs went through download, verification
  and upload, and both ran correctly. The start/stop/E-stop program passed
  15 behaviour checks, among them STOP, E-STOP, a broken STOP wire, and
  START held while the PLC goes to RUN. A 68-step test program passed 17
  checks covering a timer, a counter, MOV and DMOV, a compare contact,
  MPS/MPP branches, ORB, ANB, and ANDP with ADD.
- A self-test with a fake PLC covers download, the refusal cases and upload
  without any hardware.

## Status and limits

- It hasn't been run against a physical DVP PLC yet. The serial link has
  only been tested against a fake PLC on a pseudo-terminal.
- No file it wrote has been opened in the WPLSoft editor yet. So far the
  evidence is the byte-for-byte match with WPLSoft's own files.
- It writes files for two families: SV, EH and EH2 (WPLSoft model code 2),
  and SV2 and EH3 (model code 7). Each other DVP model needs an empty
  template file saved from WPLSoft.
- Programs are limited to 15,872 steps, since WPLSoft switches to a
  different transfer function above that.
- Direct serial needs macOS or Linux. A COM port on a Windows PC is reached
  through the SSH relay.
- Only the program is transferred. Device comments, retentive ranges,
  passwords and the PLC clock stay with WPLSoft.
- ISPSoft `.isp` projects aren't covered.

Python 3, standard library only.

---

<sub>Delta-WPL-Bridge is proprietary; the source is not public. It is not
affiliated with or endorsed by Delta Electronics, Inc. Delta, DVP, WPLSoft and
ISPSoft are trademarks of Delta Electronics.</sub>
