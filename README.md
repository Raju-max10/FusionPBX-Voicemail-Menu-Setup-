# FusionPBX: Voicemail Menu Setup 

Setup and reporting documentation for an inbound line on FusionPBX. Callers hear a welcome message, press 1 or 2, and leave a voicemail that is emailed to the relevant team.

All names, numbers, domains and email addresses in this document are placeholders (ABC, XYZ, `example.com`). Replace them with the real values for each customer.

> **Do not commit credentials.** SMTP usernames, passwords and hostnames are intentionally left out. Store them in a password manager.

## Overview

| Item | Value |
|---|---|
| Customer | ABC Company |
| Brands | Brand ABC (option 1), Brand XYZ (option 2) |
| PBX domain | `pbx.example.com` |
| Inbound number (DDI) | `<DDI>` (country code +91, leading 0 dropped as per dialplan setup) |
| Purpose | Temporary maintenance line: capture voicemails for each brand |

## Call flow

```
Incoming call on ABC Main Line
        |
        v
Welcome message (IVR)
        |
        +-- Press 1 --> Voicemail box 7001 --> emailed to hello@abc.example --> end call
        |
        +-- Press 2 --> Voicemail box 7002 --> emailed to hello@xyz.example --> end call
        |
        +-- No key / invalid key --> fallback voicemail box --> end call
```

### Welcome message script

> "Hi, you've reached the Brand ABC and Brand XYZ Customer Help Desk. We are currently carrying out maintenance. Please leave your name, email address or telephone number, and a brief description of what you would like to discuss today, and we will get back to you as soon as possible. Thank you for contacting us. To leave a message for Brand ABC, press 1. For Brand XYZ, press 2."

The final sentence (the key prompts) must be in the recording, otherwise callers will not know to press anything.

## Components

| Component | FusionPBX location | Name / ID |
|---|---|---|
| Inbound route | Dialplan > Destinations | ABC Main Line |
| IVR menu | Applications > IVR Menus | Extension `500000` (confirm against the route's action) |
| Welcome recording | Applications > Recordings | Welcome message (WAV, 16-bit mono) |
| Voicemail: option 1 | Applications > Voicemail | `7001`, ABC Brand ABC |
| Voicemail: option 2 | Applications > Voicemail | `7002`, ABC Brand XYZ |
| Fallback voicemail | Applications > Voicemail | Recommended: separate general box (e.g. `7000`) |
| Email settings | Advanced > Default Settings > Email | Shared, platform-wide |

## Customer requirements

**Essential**
1. DDI and confirmation it is live with the carrier (if porting: current provider, account number, recent bill, signed Letter of Authority, port date)
2. Approved welcome message wording, including the key prompts
3. Recording preference: supplied audio file, or voice preference (e.g. UK accent)
4. Email recipients for each box, plus anyone to copy in
5. Fallback destination for no-key or invalid-key calls (a general mailbox is best)

**Email delivery**
6. Whitelisting of the PBX sending address and server IP by the customer's IT or email host
7. SPF/DKIM changes, only if sending from the customer's own domain

**Worth confirming**
8. How long the maintenance period lasts and what replaces it
9. Voicemail format (attachment or link) and retention period
10. Who holds the voicemail PINs
11. Data protection: any privacy notice or retention policy needed for captured names and contact details
12. Test contact and go-live sign-off

## Setup steps

Order matters: each step depends on the one before it.

### 1. Email settings

**Advanced > Default Settings > Email.** These settings usually already exist. Edit each one, set the value and make sure **Enabled** is on.

| Subcategory | Value |
|---|---|
| method | `smtp` |
| smtp_auth | `True` |
| smtp_host | SMTP server hostname |
| smtp_username | SMTP username |
| smtp_password | SMTP password |
| smtp_from | `voicemails@example.com` |
| smtp_from_name | `PBX Voicemails` |
| smtp_port | `587` (set **Enabled** to True, it defaults to disabled with value 0) |
| smtp_secure | `tls` (not "yes (StartTLS)": use the FusionPBX value) |

Click **Reload** after saving. A domain-level email setting (Advanced > Domain Settings) can override these defaults.

### 2. Welcome recording

**Applications > Recordings.** Upload the WAV file, name it clearly and play it back to check. FusionPBX resamples 24 kHz audio automatically. If it sounds odd on a call, convert to 8 kHz or 16 kHz mono.

### 3. Voicemail boxes

**Applications > Voicemail > Add**, once per box.

| Field | Option 1 | Option 2 |
|---|---|---|
| Voicemail ID | `7001` | `7002` |
| Password | set a PIN | set a PIN |
| Play Tutorial | Off | Off |
| Greeting | None | None |
| Recording Instructions | Off | Off |
| Recording Options | Off | Off |
| Mail To | hello@abc.example | hello@xyz.example |
| Voicemail File | Audio File Attachment | Audio File Attachment |
| Keep Local After Email | On | On |
| Enabled | On | On |
| Description | ABC Brand ABC | ABC Brand XYZ |

Notes:
- **Greeting = None** removes the stock "The person at extension 7001 is not available" message, so callers go straight to the tone after the welcome message.
- **Recording Instructions off** avoids the extra "at the tone, please record your message" prompt.
- **Recording Options off** avoids the "press 1 to listen, 2 to save" prompt after the message.
- **Keep Local After Email must stay on.** The monthly report counts messages stored on the PBX.
- Voicemail IDs share the number space with extensions in the domain. Make sure `7001` and `7002` are not already used.

### 4. IVR menu

**Applications > IVR Menus > Add.**

| Field | Value |
|---|---|
| Name | ABC Main IVR |
| Extension | `500000` |
| Greet Long | Welcome recording |
| Greet Short | Same welcome recording |
| Timeout | `5000` (milliseconds: 5 means the menu gives up almost instantly) |
| Exit Action | Voicemail box (fallback box, or `7001` if none) |
| Direct Dial | False |
| Context | The customer's domain (pre-filled) |
| Enabled | On |
| Description | ABC Main Line |

Options:

| Option | Destination | Order | Description | Enabled |
|---|---|---|---|---|
| 1 | Voicemail `7001` | 000 | Brand ABC | On |
| 2 | Voicemail `7002` | 010 | Brand XYZ | On |

**Exit Action must not be "Check Voicemail"**, which asks the caller to log in with an ID and PIN. It must also not be the welcome recording, which would replay the message and then end the call.

If the voicemail boxes do not appear in the Destination list, use a custom destination: `transfer *997001 XML <domain_name>` (and `*997002`).

### 5. Inbound route

**Dialplan > Destinations > Add.**

| Field | Value |
|---|---|
| Type | Inbound |
| Country Code | `+91` |
| Destination | DDI without the leading 0 as per dialplan setup|
| Context | `public` |
| Action | Transfer to the ABC IVR (extension `500000`) |
| Domain | `pbx.example.com` |
| Enabled | On |
| Description | ABC Main Line |

Check that the action points at the correct IVR extension (`500000`). If calls do not arrive, confirm in the carrier portal which number format they send.

## Testing checklist

- [ ] Call the DDI: the welcome message plays and includes the key prompts
- [ ] Press 1: no "person at extension" message, tone plays, message recorded
- [ ] Voicemail email arrives at hello@abc.example (check spam)
- [ ] Press 2: same checks for hello@xyz.example
- [ ] Press nothing: caller reaches the fallback box
- [ ] Press an invalid key: caller reaches the fallback box
- [ ] Each test call appears in the CDR; note what the last application and argument show for 1 and 2
- [ ] Test voicemails deleted or recorded before go-live, so customer figures start at zero

Direct voicemail test without the route: dial `*997001` or `*997002` from an extension on the domain.

## Reporting

A PBX domain often holds many customers' data, so always filter by DDI or voicemail ID. A domain filter alone inflates the totals.

> Table and column names below are from memory. Verify them on your install before relying on the numbers.

### Calls into the DDI (weekly)

**Quick method:** Status > Call Detail Records, filter on the DDI as destination, set the date range to the last 7 days and read the record count (CSV export is available).

**Query:**

```sql
SELECT
  count(*) AS total_calls,
  count(*) FILTER (WHERE last_app = 'voicemail' AND last_arg LIKE '%7001') AS press_1,
  count(*) FILTER (WHERE last_app = 'voicemail' AND last_arg LIKE '%7002') AS press_2
FROM v_xml_cdr
WHERE start_stamp >= now() - interval '7 days'
  AND direction = 'inbound'
  AND caller_destination = '<DDI>';
```

FusionPBX does not record keypresses as such. This counts where each call ended up, which is a good proxy once confirmed with test calls. If the fallback box is separate (`7000`), callers who press nothing are not counted under option 1.

### Voicemails left per option (monthly)

**Quick method:** Applications > Voicemail, read the **Messages (n)** count for 7001 and 7002, and compare with the previous month's figure.

**Query:**

```sql
SELECT
  v.voicemail_id,
  to_char(to_timestamp(m.created_epoch), 'YYYY-MM') AS month,
  count(*) AS messages
FROM v_voicemail_messages m
JOIN v_voicemails v ON v.voicemail_uuid = m.voicemail_uuid
JOIN v_domains d ON d.domain_uuid = v.domain_uuid
WHERE d.domain_name = 'pbx.example.com'
  AND v.voicemail_id IN ('7001', '7002')
GROUP BY v.voicemail_id, month
ORDER BY month, v.voicemail_id;
```

Run on the PBX server with `sudo -u postgres psql fusionpbx`.

### Calls versus messages

| Measure | Source | Notes |
|---|---|---|
| Calls into the DDI | CDR | All inbound calls |
| Key presses | CDR (last app and argument) | Includes callers who hang up before recording |
| Voicemails left | Voicemail message table | What the customer actually receives. Lower than key presses |

Do not delete messages from 7001 or 7002 without recording the counts first.

## Known pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Menu hangs up almost immediately | IVR Timeout is `5` (milliseconds) | Set to `5000` |
| Caller asked to log in to a mailbox | Exit Action set to Check Voicemail | Point it at a voicemail box |
| Welcome message replays, then call ends | Exit Action set to the recording | Point it at a voicemail box |
| "The person at extension 7001 is not available" | Voicemail Greeting set to Default | Set to None |
| Voicemail emails fail | `smtp_secure` set to "yes (StartTLS)" or `smtp_port` still disabled | Use `tls`, port `587`, Enabled on |
| Emails land in spam | Sending domain not authorised for the customer's mail server | Ask customer IT to whitelist; check SPF/DKIM |
| Callers do not know to press 1 or 2 | Key prompts missing from the recording | Re-record with the final sentence |
| Call does not reach the IVR | Destination number format mismatch, or route points at the wrong IVR | Check the carrier's format and the route's action |

## Open items

- [ ] Fallback voicemail box (`7000`) created and its email recipient agreed
- [ ] Voicemail box descriptions renamed to the customer's labelling
- [ ] Confirm the IVR extension in the inbound route matches the IVR menu
- [ ] Agree end date for the maintenance message and what replaces it
- [ ] Confirm CDR last app and argument values for 7001/7002 and update the call report query
- [ ] Rotate the SMTP credentials if they were shared in screenshots or tickets
- [ ] Automate the weekly call report and monthly voicemail report (cron + email)

 
