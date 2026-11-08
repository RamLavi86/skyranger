# 11 Safety Note: Sky Ranger Is Not a Safety System

This is a normative boundary statement. It binds the product, the HMI text, the documentation, marketing, quotes and commissioning reports. Nothing in the other security files, and nothing a customer asks for, overrides it.

## 1. The statement

> **Sky Ranger is a monitoring and supervisory control system. It is not a safety system, not a safety instrumented system (SIS), not a protection system, and not a substitute for one. It must not be used to perform, or be relied on for, any function whose failure could injure a person, damage the environment, or damage equipment. Hardwired interlocks, independent safety devices and the plant's own protective functions remain the sole means of protection.**

The same text (shortened to one sentence) appears in: the HMI "About" page, the installation guide, the handover document, every commissioning report, quotes and the website footer.

## 2. Why this is a security document

Security and safety interact in both directions:
- A compromised or misbehaving SCADA must not be able to cause an unsafe state. The only reliable way to guarantee that is for **safety functions to be independent of it**: a PLC safety program, a relay circuit, a pressure relief valve, a float switch wired to a pump contactor.
- If a customer starts to depend on Sky Ranger for protection ("the leak alarm will shut the valve"), every outage, update, bug, or attack on it becomes a safety event. We prevent that by design and by contract.

## 3. What Sky Ranger can and cannot be

| Can be | Cannot be, and we say so |
|---|---|
| A display of process values with quality and age | A source of truth for a safety decision (values can be stale, uncertain, delayed, or wrong) |
| An alarm system for **operator awareness** (leak, device offline) | A protection function. Alarms are advisory, delivered best-effort to a screen that may not be watched |
| A way to write a setpoint or start/stop command to a device **that has its own limits and interlocks** | A way to open or close something in the absence of a local safe state. Writes can be delayed, lost, or have unknown outcome (api.md, driver-interface.md) |
| A historian for reports and evidence | A certified record for legal or billing purposes unless a customer-specific validation says so |
| A platform an integrator may place **next to** a safety system at a defined boundary | A component inside a safety loop |

## 4. Design rules that enforce the boundary

1. **No safety logic in Sky Ranger.** The tag engine has no interlock, permissive or trip feature. Derived tags (`counter_total`, `rate`, `liveness`, `continuous_activity`, `select`, `scale`) are measurement functions. Future scripting (Python) is also supervisory; the scripting design must not offer a "trip" API, and documentation must say so.
2. **Writes assume an independent device-side limiter.** `write_limits` on tags are a convenience to catch typos, not protection. The device or PLC must reject out-of-range commands itself. Integrators document the device-side limit for every writable tag in the handover.
3. **Fail to the device's safe state, never to "last command held" by us.** Sky Ranger does not keep sending commands. Loss of the gateway must leave the process in whatever state the PLC/device's own program defines. A device whose safe state depends on a continuous stream from Sky Ranger is not allowed (use the PLC's own timeout or a watchdog on the PLC side).
4. **Unknown outcomes are shown as unknown.** The HMI never reports a command as done until feedback confirms it (05-hmi-api/05 section 4). Writes are never auto-retried.
5. **Alarms are not acknowledged by silence.** Alarm audio and banner are operator aids; they are not a certified alarm annunciator. Where a customer needs a guaranteed annunciator (horn/beacon wired to a PLC output), the integrator wires it from the PLC, not from Sky Ranger.
6. **Security controls never trade away safety.** Lockouts, certificate expiry, license checks (future), audit-failure blocking (08: fail closed for changes) apply to Sky Ranger operations only. Because Sky Ranger is not in the protection path, a security failure cannot disable a safety function. This is also why login throttling never hard-locks accounts (02): an operator must still be able to see and act.
7. **No safety-related claims in naming.** Features are not called "safe", "safety", "fail-safe", "SIL", "interlock", or "emergency" (apart from the statement above). `emergency stop` buttons are never HMI widgets. The `control` component has no "e-stop" kind, and the screen validator warns (it does not block) when a widget label matches a configurable deny list such as "emergency" or "e-stop", prompting the engineer to review.

## 5. Boundary with a real safety system

If a site has a SIS or safety PLC (Purdue-style Z6 in [05-network-segmentation.md](05-network-segmentation.md) section 5):

- **Default: no connection.** No conduit from Sky Ranger to the SIS network.
- **Permitted, only with the site's functional safety engineer's written approval:** a **one-way** status read (SIS to Sky Ranger) through a standard data diode or a read-only gateway function owned by the safety engineer, for display only. Sky Ranger never writes to it, never holds its credentials, and its failure cannot affect it.
- Bypass and override of safety functions must never be exposed through Sky Ranger.
- If a customer asks for any of these, the answer is "outside our scope; engage the safety engineer".

## 6. Wording rules for sales, docs and the UI

Never write or say:
- "SIL 1/2/3", "safety rated", "IEC 61508 compliant" or "IEC 61511 compliant" (these are standards and certifications for safety functions and products; we hold none, and do not claim to).
- "Protects against", "prevents flooding/overflow/contamination". Say "alerts an operator to".
- "Guarantees", "never misses", "real-time" (we are soft real-time over best-effort networks; see 05-hmi-api/06 for latency numbers).
- "Secure" as an absolute. Say what controls exist and what residual risks remain (01 section 7).
- "IEC 62443 certified" until a certificate exists (12).

Always say:
- What the alarm does and does not do: "notifies operators on screen and optionally by [channel]. It does not stop the process."
- Which device-side protections the project relies on, and who owns them.

Quote and contract language (to be reviewed by a lawyer before first use): scope statement, limitation of liability, and an acknowledgement signed at handover that the customer's safety functions are independent of Sky Ranger.

## 7. Commissioning checklist (safety boundary items)

These are part of the pre-shipping checklist (14):
- [ ] The statement in section 1 appears in the handover document and on the HMI About page.
- [ ] For every writable tag, the device-side limit that makes a bad write harmless is recorded.
- [ ] The customer confirms in writing that no safety function depends on Sky Ranger.
- [ ] If a leak/overflow/low-pressure condition could harm people or property, an independent hardwired or PLC-based protection exists and does not use Sky Ranger.
- [ ] No Sky Ranger network connection to any safety system exists, or an approved one-way link is documented.
- [ ] Alarm acknowledgement and shelving rules were reviewed with the customer: shelved alarms are not protection.

## 8. The water meter, as an example

The `leak` alarm (continuous flow for 2 h) and the `health` alarm (meter not reporting) tell an operator that something looks wrong. They do not close a valve. If a burst pipe could flood a pump room, a float switch and a PLC-driven valve or pump trip are the protection; the Sky Ranger alarm is a courtesy that may arrive late, or not at all if the Wi-Fi is down.

## Decisions made
1. Sky Ranger contains no safety logic and makes no safety claims; this is a product-level boundary, not a configuration option.
2. Safety functions must be independent of Sky Ranger in hardware and power; connections to a SIS are default-off and, if ever approved, one-way read only.
3. Write limits, alarms and acknowledgements are operator aids; device-side limits and independent protections are mandatory for writable tags.
4. Wording rules forbid safety-rating, guarantee and absolute-security claims.
5. A commissioning checklist and a signed customer acknowledgement record the boundary at every site.

## Alternatives rejected and why
- **Aiming for SIL-rated functions later "as an upgrade"**: a SIL claim applies to a product, development process and architecture assessed as a whole; bolting it on a SCADA gateway is not realistic for a one-person team and would raise liability without need.
- **Offering "interlock" or "trip" features for convenience**: customers will rely on them, and the first outage becomes a safety incident.
- **Silent on the topic and left to contracts**: the UI and docs shape expectations; contracts alone are not enough.
- **Bidirectional bridge to a SIS "for convenience"**: it makes the SCADA a path into the safety network, which is exactly what segmentation is for.

## Open questions
- A lawyer must review the quote, limitation-of-liability and handover acknowledgement wording; jurisdictions differ.
- Does any target customer (water utility) have a regulatory expectation that alarm systems meet a standard (for example ISA-18.2 style management)? UNVERIFIED; 04-alarms owner to note what we do and do not claim.
- Product liability and professional indemnity insurance coverage for the integrator is a business question outside this directory.
- Should the HMI show the safety statement on first login each year, with an audited acknowledgement by the admin?
