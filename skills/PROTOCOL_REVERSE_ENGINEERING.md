# Protocol Reverse Engineering Guide

> How pump drivers get built — from BLE traffic capture to working AndroidAPS driver.
> Based on the Medtronic/OpenAPS precedent and current YpsoPump efforts.

## Historical Precedent: How Medtronic Was Reverse-Engineered

### Timeline (7 years total)

| Year | Milestone | Who |
|------|-----------|-----|
| 2012 | USB CareLink stick protocol decoded | Ben West |
| 2013 | RF protocol (916MHz) captured via SDR | Pete Schwamb |
| 2014 | First DIY closed loop (Raspberry Pi + CareLink) | Dana Lewis + Scott Leibrand |
| 2015 | OpenAPS launched; RileyLink hardware designed | Pete Schwamb |
| 2016 | Loop (iOS) built with RileyLink | Nate Racklyeft |
| 2018 | AndroidAPS Medtronic driver development | Andy Rozman |
| 2019 | Driver merged into AndroidAPS mainline | Community |

### Key Lessons

1. **Start with read-only operations** — status, history, glucose. Don't touch insulin delivery until protocol is well understood
2. **Correlate captures with known actions** — "I pressed bolus 2.0U at 14:32" matched to packet data
3. **Build outside main project first** — Andy Rozman built RileyLinkAAPS as standalone test app before AAPS integration
4. **Use milestone approach** — basic comm → command split → state machine → AAPS integration → full commands → history
5. **Frame as patient data access** — not "hacking". Patients accessing their own medical data.
6. **Open-source from the start** — transparency protects against legal action, enables safety review
7. **Expect manufacturer to respond technically** — Medtronic added encryption to newer models. Target currently-available hardware.
8. **No legal action was taken** by Medtronic against the community despite full protocol reversal

---

## Phase 1: Reconnaissance

### FCC Filings
Every wireless medical device must file with the FCC. These reveal:
- Operating frequency and modulation
- Internal photos showing chipsets
- Power levels and antenna specs

### Identify Communication Interface
| Interface | Tools | Difficulty |
|-----------|-------|------------|
| BLE GATT | nRF Connect app, Wireshark | Easiest |
| BLE RFCOMM | Wireshark HCI snoop | Medium |
| Sub-GHz RF | SDR (HackRF, RTL-SDR), RFCat | Hardest |
| USB Serial | Wireshark, pyserial | Easy |

### Check Existing Work
- Search GitHub for pump name + "reverse engineer"
- Check Black Hat / DEF CON presentations
- Search BSI (German Federal Security Office) ManiMed reports
- Check OpenAPS/AndroidAPS community forums and Discord
- Look for academic papers on the device

---

## Phase 2: BLE Traffic Capture

### Method A: Android HCI Snoop Log
```bash
# 1. Enable on phone: Developer Options → Enable Bluetooth HCI snoop log
# 2. Use the official pump app normally (pair, bolus, read status)
# 3. Retrieve log file:
adb pull /sdcard/btsnoop_hci.log
# 4. Open in Wireshark with BLE dissector
wireshark btsnoop_hci.log
# Filter: btatt || btle
```

### Method B: nRF Connect (Real-Time)
1. Install nRF Connect app on Android
2. Scan for the pump's BLE advertisement
3. Connect and enumerate all services/characteristics
4. Record UUIDs, properties (read/write/notify), and values
5. This gives you the service map without needing the official app

### Method C: Frida (Intercept Official App)
```bash
# For encrypted protocols where you need to see plaintext
# Requires rooted phone + Magisk
frida -U -l hook_script.js -f com.official.pump.app
```
Used by vicktor for YpsoPump key extraction.

### Method D: Ubertooth / btlejack (Passive Sniffing)
For capturing BLE traffic without being a participant in the connection:
- **Ubertooth One** — hardware BLE sniffer
- **btlejack** — open-source BLE sniffer using micro:bit ($15)

---

## Phase 3: Protocol Analysis

### Identify Packet Structure
```
Typical BLE pump packet:
┌─────────┬──────────┬─────────┬──────────┬─────────┐
│ Header  │ Command  │ Length  │ Payload  │ CRC/MAC │
│ (1-2B)  │ (1-2B)   │ (1-2B)  │ (var)    │ (2-4B)  │
└─────────┴──────────┴─────────┴──────────┴─────────┘

Example (Dana RS):
Start: 0xA5 | Command | Data... | End: 0x5A
Start: 0xAA | Command | Data... | End: 0xEE (BLE5 mode)
```

### Correlation Analysis
1. Perform known action on pump (via official app or physical buttons)
2. Note exact timestamp
3. Match to captured BLE write/notify in Wireshark
4. Repeat 5-10 times to confirm pattern
5. Identify changing bytes (timestamps, counters, checksums) vs static (command codes)

### Command Enumeration
Build a table:
| Action | Write Char | Bytes Written | Response | Notes |
|--------|-----------|---------------|----------|-------|
| Read status | UUID-xxx | `0x70 0x00` | `0x70 0x12 ...` | Always same prefix |
| Bolus 1.0U | UUID-xxx | `0x42 0x0A ...` | `0x42 0x01` | 0x0A = 10 = 1.0U * 10 |
| Set TBR 150% | UUID-xxx | `0x43 0x96 ...` | `0x43 0x01` | 0x96 = 150 |

### Encryption Analysis
If traffic is encrypted (most modern pumps):

| Encryption Level | Difficulty | Example |
|-----------------|------------|---------|
| None | Easy | Medtronic 512-554 |
| Simple XOR | Easy | Some older pumps |
| AES-128 with hardcoded key | Medium | Early YpsoPump (BSI finding) |
| Full key exchange (ECDH) | Hard | YpsoPump current, Medtronic 600+ |
| Certificate-pinned TLS | Very hard | Some newer pumps |

---

## Phase 4: Driver Development (AndroidAPS Integration)

### Milestone Plan (from Andy Rozman's Medtronic approach)

| Milestone | Goal | Risk Level |
|-----------|------|------------|
| M1 | BLE connection + service discovery | None |
| M2 | Send read-only commands (status, battery, reservoir) | None |
| M3 | Parse responses, build state machine | None |
| M4 | Integrate into AAPS as plugin (read-only) | Low |
| M5 | Implement write commands (TBR, bolus) with safety guards | **High** |
| M6 | History read, TDD, full PumpSync integration | Medium |

### Architecture Decision: Which Pattern?

Based on existing drivers (see `skills/PUMP_DRIVER_PATTERNS.md`):

| If pump uses... | Use pattern... | Example |
|-----------------|---------------|---------|
| Direct BLE GATT | **Pattern B: Manager-Based** | OmnipodDash |
| BLE via relay hardware | **Pattern C: RileyLink** | Medtronic, Eros |
| BLE requiring background service | **Pattern A: Service-Based** | DanaRS |

For YpsoPump (direct BLE GATT) → **Pattern B** is recommended.

### Module Structure
```
pump/ypsopump/
├── build.gradle.kts
└── src/main/kotlin/app/aaps/pump/ypsopump/
    ├── YpsoPumpPlugin.kt           # Main plugin (extends PumpPluginBase)
    ├── di/
    │   └── YpsoPumpModule.kt       # Dagger module
    ├── comm/
    │   ├── BleManager.kt           # BLE connection management
    │   ├── PacketEncoder.kt        # Packet encoding/decoding
    │   └── Encryption.kt           # Protocol encryption
    ├── data/
    │   ├── YpsoPumpState.kt        # Pump state tracking
    │   └── YpsoPumpHistory.kt      # History record parsing
    └── defs/
        └── YpsoPumpCommands.kt     # Command type enumeration
```

---

## Phase 5: Safety Verification

**Before any insulin delivery command is implemented:**

- [ ] Read-only operations tested for 2+ weeks without issues
- [ ] All commands go through AAPS CommandQueue (never direct)
- [ ] All delivery results reported via PumpSync
- [ ] ConstraintsChecker applied to all calculated values
- [ ] Edge cases tested: max bolus, max basal, zero temp, disconnect mid-command
- [ ] Error recovery tested: what happens if BLE drops during bolus?
- [ ] Multiple users testing on real hardware
- [ ] Code review by experienced pump driver developer

---

## Legal Context

### EU: Directive 2009/24/EC
Reverse engineering for interoperability is explicitly permitted. No authorization from rights holder required.

### US: DMCA § 1201(f)
Interoperability exception allows reverse engineering of computer programs. Medical device patients accessing own data has additional protections.

### Practical Reality
- Medtronic: No legal action taken despite full protocol reversal
- Omnipod: Insulet acknowledged DIY community, no legal action
- Ypsomed: BSI published vulnerabilities in 2021, no litigation followed
- Frame as patient data access and safety research

---

## Current Active Efforts (as of March 2026)

### YpsoPump (#55)
| Effort | Status | Repo |
|--------|--------|------|
| vicktor | **Working** — bolus, TBR functional | [GitHub](https://github.com/vicktor) |
| SandraK82 | **Research** — 19 RE docs, driver skeleton | [GitHub](https://github.com/SandraK82/ypsopump-research) |
| Encryption | XChaCha20-Poly1305 cracked via Frida | Requires rooted phone |
| Key exchange | 9-step protocol via Ypsomed gRPC backend | Keys valid ~28 days |

### Key Resources

| Resource | Description |
|----------|-------------|
| [decoding-carelink](https://github.com/bewest/decoding-carelink) | Original Medtronic RE (Python) |
| [minimed_rf](https://github.com/ps2/minimed_rf) | RF protocol decoding (Ruby) |
| [RileyLink](https://getrileylink.org/) | Open-source BLE-to-RF bridge |
| [SandraK82/ypsopump-research](https://github.com/SandraK82/ypsopump-research) | YpsoPump RE docs |
| [ManiMed BSI Report](https://insinuator.net/2021/07/manimed-ypsomed-ag-mylife-ypsopump-system-vulnerabilities/) | YpsoPump security findings |
| [CISA ICSMA-19-178-01](https://www.cisa.gov/news-events/ics-medical-advisories/icsma-19-178-01) | Medtronic vulnerability advisory |
