# Koha EDIFACT Integration Guide

This document specifies how book suppliers should format EDIFACT messages exchanged with
Koha, **and** what Koha actually does with the inbound messages it receives. It combines a
wire-format specification (how to structure each message) with a behavior specification
(which elements are required, what each one means, and what Koha does with it at
processing time). It is intended for technical staff at supplier organizations and for
Koha implementers.

Koha's EDIFACT implementation is based on the **UN/EDIFACT** standard (directory version
**D:96A**), using the **EANCOM** subset and **EDItEUR** / **BIC** (Book Industry
Communication) conventions for the book supply chain.

It is based on:

- **Koha 25.11** — `Koha/EDI.pm`, `Koha/Edifact/Message.pm`, `Koha/Edifact/Line.pm`,
  `Koha/Edifact/Order.pm`, `misc/cronjobs/edi_cron.pl`.
- **EDIFACT Enhanced plugin 4.3.3** (ByWater Solutions), which replaces order generation,
  transport, parsing and invoice processing when enabled on a vendor EDI account. For
  plugin configuration options see the companion
  [EDI Enhanced Plugin Guide](EDI_Enhanced_Plugin_Guide.md).

---

## Table of Contents

- [1. Supported Message Types](#1-supported-message-types)
- [2. How Koha Consumes Inbound Messages](#2-how-koha-consumes-inbound-messages)
- [3. General EDIFACT Envelope](#3-general-edifact-envelope)
  - [3.1 Service String Advice (UNA)](#31-service-string-advice-una)
  - [3.2 Interchange Header (UNB)](#32-interchange-header-unb)
  - [3.3 Interchange Trailer (UNZ)](#33-interchange-trailer-unz)
  - [3.4 Character Encoding](#34-character-encoding)
  - [3.5 Text Escaping](#35-text-escaping)
  - [3.6 Reading the Element Notation](#36-reading-the-element-notation)
- [4. ORDERS — Purchase Order (Koha to Supplier)](#4-orders--purchase-order-koha-to-supplier)
  - [4.1 Message Structure](#41-message-structure)
  - [4.2 Segment Reference](#42-segment-reference)
- [5. QUOTES — Quotation (Supplier to Koha)](#5-quotes--quotation-supplier-to-koha)
  - [5.1 Message Structure](#51-message-structure)
  - [5.2 Segment Reference](#52-segment-reference)
  - [5.3 IMD-to-MARC Mapping](#53-imd-to-marc-mapping)
- [6. ORDRSP — Order Response (Supplier to Koha)](#6-ordrsp--order-response-supplier-to-koha)
  - [6.1 What an order response does in Koha](#61-what-an-order-response-does-in-koha)
  - [6.2 Message Structure](#62-message-structure)
  - [6.3 Message-level (header) elements](#63-message-level-header-elements)
  - [6.4 Line-level elements](#64-line-level-elements)
  - [6.5 Action codes and behavior](#65-action-codes-and-behavior)
  - [6.6 Coded status text and availability tables](#66-coded-status-text-and-availability-tables)
  - [6.7 Full worked example](#67-full-worked-example)
- [7. INVOIC — Invoice (Supplier to Koha)](#7-invoic--invoice-supplier-to-koha)
  - [7.1 What an invoice does in Koha](#71-what-an-invoice-does-in-koha)
  - [7.2 Message Structure](#72-message-structure)
  - [7.3 Message-level (header) elements](#73-message-level-header-elements)
  - [7.4 Line-level elements](#74-line-level-elements)
  - [7.5 How price and tax are derived](#75-how-price-and-tax-are-derived)
  - [7.6 How items are received](#76-how-items-are-received)
  - [7.7 Full worked example](#77-full-worked-example)
  - [7.8 Plugin behavior differences](#78-plugin-behavior-differences)
- [8. GIR Qualifiers — Copy-Level Data](#8-gir-qualifiers--copy-level-data)
- [9. RFF Qualifiers — References](#9-rff-qualifiers--references)
- [10. File Naming and Transport](#10-file-naming-and-transport)
- [11. Identification and Qualifiers](#11-identification-and-qualifiers)
- [12. Configuration Prerequisites](#12-configuration-prerequisites)
- [13. Required-Element Summary](#13-required-element-summary)
- [14. Common Failure Modes](#14-common-failure-modes)
- [15. Plugin Extension Points](#15-plugin-extension-points)

---

## 1. Supported Message Types

| Message | UN Type | Direction | File Extension | Description |
|---------|---------|-----------|---------------|-------------|
| **ORDERS** | `ORDERS` | Koha &rarr; Supplier | `.CEP` | Library sends a purchase order |
| **QUOTES** | `QUOTES` | Supplier &rarr; Koha | `.CEQ` | Supplier sends a quotation / proposed selection |
| **ORDRSP** | `ORDRSP` | Supplier &rarr; Koha | `.CEA` | Supplier sends an order response / acknowledgment |
| **INVOIC** | `INVOIC` | Supplier &rarr; Koha | `.CEI` | Supplier sends an invoice |

**Note:** DESADV (Despatch Advice) and RECADV (Receiving Advice) are **not** supported.

### File Extension Convention

The three-character extension follows this scheme:

| Position | Meaning | Values |
|----------|---------|--------|
| 1st | Status | `C` = Ready for pickup, `A` = Completed, `E` = Extracted |
| 2nd | Standard | `E` = EDIFACT |
| 3rd | Type | `P` = Purchase order, `Q` = Quote, `A` = Answer (order response), `I` = Invoice |

---

## 2. How Koha Consumes Inbound Messages

The three inbound message types (QUOTES, ORDRSP, INVOIC) are pulled from the vendor by the
EDI cron job (`misc/cronjobs/edi_cron.pl`, normally run every 15 minutes) and stored as
rows in the `edifact_messages` table before they are processed:

| File type | Downloaded as `message_type` | Default extension | Processor |
|-----------|------------------------------|-------------------|-----------|
| Quote | `QUOTE` | `.CEQ` | `Koha::EDI::process_quote` |
| Invoice | `INVOICE` | `.CEI` | `Koha::EDI::process_invoice` (or the plugin's `edifact_process_invoice`) |
| Order response | `ORDRSP` | `.CEA` | `Koha::EDI::process_ordrsp` |

Each downloaded message moves through three statuses: `new` → `processing` → `received`.

The cron cycle per EDI account is:

1. Download new QUOTE files
2. Download new INVOICE files
3. Upload pending ORDERS files
4. Download new ORDRSP files
5. Process all downloaded messages

Two important control points govern inbound processing:

- **Invoices only auto-process when the `EdifactInvoiceImport` system preference is set to
  `automatic`.** When it is `manual`, invoice rows stay at status `new` until a staff
  member triggers them from the acquisitions EDI messages page. Quotes and order responses
  are **always** processed by the cron run regardless of this preference.
- **Order responses always use the core Koha parser and core processor.**
  `process_ordrsp` instantiates `Koha::Edifact` directly and does not call any plugin
  hook. Only invoice processing (and, separately, order generation and transport) is
  replaceable by a plugin. So the ORDRSP behavior in [Section 6](#6-ordrsp--order-response-supplier-to-koha)
  is identical whether or not a vendor account has a plugin configured.

---

## 3. General EDIFACT Envelope

All message types share a common envelope structure.

### 3.1 Service String Advice (UNA)

```
UNA:+.? '
```

This **must** be present and use the standard separators exactly as shown:

| Character | Position | Function |
|-----------|----------|----------|
| `:` | 1 | Component data element separator |
| `+` | 2 | Data element separator |
| `.` | 3 | Decimal notation (period) |
| `?` | 4 | Release (escape) character |
| ` ` | 5 | Reserved (space) |
| `'` | 6 | Segment terminator |

Non-standard separator characters are **not** supported. (The Enhanced plugin's parser
tolerates a missing `UNA` and strips stray whitespace / non-printable characters; core
Koha does not.)

### 3.2 Interchange Header (UNB)

```
UNB+UNOC:3+<sender_id>:<qualifier>+<recipient_id>:<qualifier>+YYMMDD:HHMM+<control_ref>++<application_ref>'
```

| Element | Description |
|---------|-------------|
| `UNOC:3` | Syntax identifier — UN/ECE level C, version 3 (ISO 8859-1) |
| Sender ID | Sender's EAN or SAN with code qualifier |
| Recipient ID | Recipient's EAN or SAN with code qualifier |
| Date/Time | Preparation date and time (`YYMMDD:HHMM`) |
| Control Ref | Interchange control reference (unique identifier) |
| Application Ref | Message type: `ORDERS`, `QUOTES`, `ORDRSP`, or `INVOIC` |

See [Section 11](#11-identification-and-qualifiers) for ID code qualifier values.

### 3.3 Interchange Trailer (UNZ)

```
UNZ+<message_count>+<control_ref>'
```

| Element | Description |
|---------|-------------|
| Message count | Number of messages in this interchange |
| Control ref | Must match the UNB control reference |

### 3.4 Character Encoding

- The wire format is **ISO 8859-1** (Latin-1), per `UNOC:3` in the UNB segment.
- Koha converts incoming messages from ISO 8859-1 to UTF-8 on receipt.
- Koha converts outgoing messages by stripping diacritics to ASCII-safe equivalents via
  transliteration.

### 3.5 Text Escaping

Within any data element, the following characters **must** be escaped with the `?` release
character:

| Character | Escaped Form |
|-----------|-------------|
| `?` | `??` |
| `'` | `?'` |
| `:` | `?:` |
| `+` | `?+` |

### 3.6 Reading the Element Notation

Throughout this document, "element N" means the Nth `+`-delimited field after the segment
tag (the first field after the tag is element 0), and "component M" means the Mth
`:`-delimited part of that field. For example, in:

```edifact
QTY+47:2'
```

element 0 is `47:2`; component 0 of element 0 is `47` (the qualifier) and component 1 is
`2` (the value). This is exactly how Koha's parser addresses the data, so the positions
described in the segment references below are the ones that matter.

---

## 4. ORDERS — Purchase Order (Koha to Supplier)

This is the **only** message type generated by Koha. Each ORDERS message represents one
acquisition basket.

### 4.1 Message Structure

```edifact
UNA:+.? '
UNB+UNOC:3+<buyer_id>:<qual>+<supplier_id>:<qual>+YYMMDD:HHMM+<control_ref>++ORDERS'
UNH+<msg_ref>+ORDERS:D:96A:UN:EAN008'
BGM+220+<basket_no>+9'
DTM+137:<YYYYMMDD>:102'
NAD+BY+<buyer_ean>::<agency>'
NAD+SU+<supplier_san>::<agency>'
  ┌─── Repeated per line item ─────────────────────────────────
  │ LIN+<line_no>++<ean_13>:EN'
  │ PIA+5+<isbn>:<code>'
  │ IMD+L+009+:::<author>'
  │ IMD+L+050+:::<title_part1>:<title_part2>'
  │ IMD+L+109+:::<publisher>'
  │ IMD+L+170+:::<year>'
  │ IMD+L+220+:::<binding>'
  │ QTY+21:<quantity>'
  │ GIR+<copy_no>+<branch>:LLO+<fund>:LFN+<itype>:LST+<seq>:LSQ+<shelfmark>:LSM'
  │ FTX+LIN+++<vendor_note>'
  │ PRI+AAE:<price>:CA'
  │ RFF+LI:<ordernumber>'
  │ RFF+QLI:<supplier_quote_ref>'
  └────────────────────────────────────────────────────────────
UNS+S'
CNT+2:<line_count>'
UNT+<segment_count>+<msg_ref>'
UNZ+<message_count>+<control_ref>'
```

### 4.2 Segment Reference

#### UNH — Message Header

```
UNH+<msg_ref>+ORDERS:D:96A:UN:EAN008'
```

- Message reference number: unique within the interchange
- Standard: ORDERS, directory D:96A, agency UN, association code EAN008

#### BGM — Beginning of Message

```
BGM+<document_type>+<document_number>+<function>'
```

| Document Type Code | Meaning |
|--------------------|---------|
| `220` | Standard order |
| `22V` | Order in response to a prior quotation (BIC standard only) |

| Function Code | Meaning |
|---------------|---------|
| `9` | Original message |
| `7` | Retransmission |

The document number is the Koha basket number (or basket name if the EDI account has
`po_is_basketname` enabled, or the plugin's `send_basketname` is set).

#### DTM — Date/Time

```
DTM+137:<YYYYMMDD>:102'
```

- Qualifier `137` = document/message date/time
- Format qualifier `102` = CCYYMMDD

#### NAD — Name and Address

```
NAD+<qualifier>+<id_code>::<agency>'
```

| Qualifier | Party |
|-----------|-------|
| `BY` | Buyer (the library) |
| `SU` | Supplier (the vendor) |

The agency code in NAD uses `9` for EAN (note: this differs from the UNB segment where EAN
uses `14`).

#### LIN — Line Item

```
LIN+<line_number>++<ean>:EN'
```

- Line number: sequential within the message
- Product code type `EN` = EAN-13

#### PIA — Additional Product Identification

```
PIA+5+<product_id>:<code>'
```

- Function code `5` = product identification
- Code `IB` = ISBN (10-digit), `EN` = EAN-13 (13-digit ISBN)
- Only included when an ISBN/EAN is available on the order line

#### IMD — Item Description

```
IMD+L+<code>+:::<text>'
```

Format indicator `L` = free-form text. Text values are broken into 35-character chunks
across sub-components.

| IMD Code | Content |
|----------|---------|
| `009` | Author |
| `050` | Title |
| `080` | Volume / part number |
| `100` | Edition statement |
| `109` | Publisher |
| `110` | Place of publication |
| `170` | Date of publication |
| `180` | Physical description |
| `190` | Series |
| `220` | Binding (e.g. `Pbk`, `Hbk`) |
| `230` | Dewey classification |
| `240` | LC classification |
| `250` | Type of item / reading level |
| `260` | Subject (personal) |
| `270` | Subject (topical) |
| `280` | Genre |
| `300` | Notes |
| `310` | Summary |
| `320` | Target audience |

Text values longer than 35 characters are split across component elements:

```
IMD+L+050+:::A Very Long Title That Ex:ceeds Thirty Five Chars'
```

#### QTY — Quantity

```
QTY+21:<quantity>'
```

- Qualifier `21` = ordered quantity

#### GIR — Related Identification Numbers (Copy-Level Data)

```
GIR+<copy_sequence>+<value>:<qualifier>+<value>:<qualifier>...'
```

Each GIR segment carries up to 5 data pairs. If a copy requires more, a continuation GIR
with the same copy sequence number is emitted.

| Qualifier | Data |
|-----------|------|
| `LLO` | Branch/location code (library branch receiving the item) |
| `LFN` | Fund/budget code |
| `LST` | Stock category / item type |
| `LSQ` | Shelving sequence / collection code |
| `LSM` | Shelfmark / call number |
| `LVT` | Servicing instruction (free text) |

See [Section 8](#8-gir-qualifiers--copy-level-data) for the complete qualifier table.

#### FTX — Free Text

```
FTX+LIN+++<text>'
```

- Only included when the order line has a vendor note. (The Enhanced plugin does **not**
  emit FTX segments.)

#### PRI — Price

```
PRI+AAE:<price>:CA'
```

| Element | Value |
|---------|-------|
| `AAE` | Information price (incl. tax, excl. allowances/charges) |
| `CA` | Catalogue price |

Only included when a list price is present on the order line.

#### RFF — Reference

```
RFF+LI:<ordernumber>'
```

- `LI` = buyer's line item reference (the Koha order number)

If the order originated from a supplier quote:

```
RFF+QLI:<supplier_reference>'
```

- `QLI` = quotation line item reference (links back to the original quote)

---

## 5. QUOTES — Quotation (Supplier to Koha)

QUOTES messages allow a supplier to propose items for a library to order. Koha creates
acquisition baskets, bibliographic records, and (optionally) items from the quote data.

### 5.1 Message Structure

```edifact
UNA:+.? '
UNB+UNOC:3+<supplier_id>:<qual>+<buyer_id>:<qual>+YYMMDD:HHMM+<control_ref>++QUOTES'
UNH+<msg_ref>+QUOTES:D:96A:UN:EAN002'
BGM+31C+<quote_number>+9'
DTM+137:<YYYYMMDD>:102'
DTM+36:<YYYYMMDD>:102'
CUX+2:<currency_code>:12'
NAD+BY+<buyer_ean>::9'
NAD+SU+<supplier_ean>::9'
RFF+ON:<purchase_order_number>'
  ┌─── Repeated per line item ─────────────────────────────────
  │ LIN+<line_no>++<ean_13>:EN'
  │ PIA+5+<isbn>:IB'
  │ IMD+L+010+:::<author_surname>'
  │ IMD+L+050+:::<title>'
  │ IMD+L+060+:::<subtitle>'
  │ IMD+L+100+:::<edition>'
  │ IMD+L+110+:::<place_of_publication>'
  │ IMD+L+120+:::<publisher>'
  │ IMD+L+170+:::<date_of_publication>'
  │ IMD+L+180+:::<physical_description>'
  │ IMD+L+190+:::<series>'
  │ IMD+L+220+:::<binding>'
  │ IMD+L+230+:::<dewey>'
  │ QTY+1:<quantity>'
  │ GIR+<copy_no>+<branch>:LLO+<fund>:LFN+<itype>:LST+<seq>:LSQ'
  │ FTX+LIN++<coded_ref>:<table>:<agency>+<text>'
  │ PRI+AAE:<price>'
  │ RFF+QLI:<supplier_quote_line_ref>'
  └────────────────────────────────────────────────────────────
UNS+S'
CNT+2:<line_count>'
UNT+<segment_count>+<msg_ref>'
UNZ+<message_count>+<control_ref>'
```

### 5.2 Segment Reference

#### BGM — Beginning of Message

| Document Type Code | Meaning |
|--------------------|---------|
| `31C` | Quotation with copy detail (preferred) |
| `310` | Basic quotation |

#### DTM — Date/Time

| Qualifier | Meaning |
|-----------|---------|
| `137` | Document/message date |
| `36` | Expiry date (the date after which the quote is no longer valid) |

Format qualifier is always `102` (CCYYMMDD).

#### CUX — Currency

```
CUX+2:<currency_code>:12'
```

- Qualifier `2` = reference currency
- ISO 4217 currency code (e.g. `GBP`, `USD`, `EUR`)
- `12` = order currency

#### RFF — Reference (Message Level)

```
RFF+ON:<purchase_order_number>'
```

- Included **before** the first LIN segment
- Used by Koha as the basket name when `po_is_basketname` is enabled on the EDI account

#### LIN / PIA — Line Item and Product Identification

Same format as ORDERS. The `EN` qualifier denotes EAN-13; use `IB` for ISBN-10.

#### IMD — Item Description

QUOTES messages use a **richer set** of IMD codes than ORDERS, particularly for author
information:

| IMD Code | Content | MARC Mapping |
|----------|---------|-------------|
| `010` | Author surname | 100$a |
| `011` | Author forename | 100$c |
| `012` | Author name qualifier | 100$b |
| `013` | Author dates | 100$d |
| `014` | Author role | 100$e |
| `020` | Second author surname | 700$a |
| `021` | Second author forename | 700$c |
| `030` | Corporate author | 110$a or 111$a |
| `040` | Corporate added entry | 710$a or 711$a |
| `050` | Title | 245$a |
| `060` | Subtitle | 245$b |
| `065` | Statement of responsibility | 245$c |
| `080` | Volume / part number | — |
| `100` | Edition statement | 250$a |
| `101` | Edition additional | 250$b |
| `110` | Place of publication | 260$a |
| `120` | Publisher name | 260$b |
| `170` | Date of publication | 260$c |
| `180` | Physical description (extent) | 300$a |
| `181` | Physical description (details) | 300$b |
| `182` | Physical description (dimensions) | 300$c |
| `183` | Physical description (material) | 300$e |
| `190` | Series title | 490$a |
| `200` | Series title (additional) | 490$a |
| `210` | Series title (additional) | 490$a |
| `220` | Binding type | — |
| `230` | Dewey classification | 082$a |
| `240` | LC classification | 084$a |
| `250` | Type / reading level | — |
| `260` | Subject (personal name) | 600$a |
| `270` | Subject (topical) | 650$a |
| `280` | Genre | 655$a |
| `300` | Notes | 500$a |
| `310` | Summary | 520$a |
| `320` | Target audience | 521$a |

**Important:** IMD text values are limited to 35 characters per sub-component. Split longer
values across sub-components separated by `:`.

#### QTY — Quantity

```
QTY+1:<quantity>'
```

- Qualifier `1` = discrete quantity (note: QUOTES uses `1`, not `21` as in ORDERS)

#### GIR — Copy-Level Data

Same qualifier scheme as ORDERS. See [Section 8](#8-gir-qualifiers--copy-level-data). The
`LFN` (fund allocation) qualifier is particularly important — Koha uses it to determine
which budget/fund to charge.

#### PRI — Price

```
PRI+AAE:<price>'
```

or with type indicator `PRI+AAE:<price>:DI'` where `DI` = discount price indicator.

#### RFF — Reference (Line Level)

```
RFF+QLI:<supplier_quote_line_reference>'
```

- `QLI` = quotation line item — the supplier's unique reference for this quoted item
- Koha stores this and echoes it back in subsequent ORDERS messages

### 5.3 IMD-to-MARC Mapping

When Koha cannot find an existing bibliographic record matching the ISBN/EAN in a quote
line, it creates a new MARC record from the IMD segments. The full mapping is listed in the
IMD table above.

Key points for suppliers:
- **At minimum**, include IMD codes `050` (title) and `010` (author) for Koha to create a
  usable bibliographic record.
- Publisher (`120`) and date (`170`) are strongly recommended.
- ISBN via PIA is the primary key used for duplicate detection against the Koha catalog.

---

## 6. ORDRSP — Order Response (Supplier to Koha)

### 6.1 What an order response does in Koha

An `ORDRSP` lets the supplier tell the library what happened to each ordered line before it
ships or invoices. Koha uses it for exactly two outcomes per line:

1. **Cancel the order line** in Koha (when the supplier says the line is cancelled), or
2. **Record a supplier report** on the order line (for every other status), storing the
   supplier's coded status text and any availability date in the order's `suppliers_report`
   field.

An order response never receives items and never touches money — it only updates order
status/notes. It is always processed by core Koha (`process_ordrsp`); plugins do not
override it.

### 6.2 Message Structure

```edifact
UNA:+.? '
UNB+UNOC:3+<supplier_id>:<qual>+<buyer_id>:<qual>+YYMMDD:HHMM+<control_ref>++ORDRSP'
UNH+<msg_ref>+ORDRSP:D:96A:UN:EAN005'
BGM+231+<response_number>+9'
DTM+137:<YYYYMMDD>:102'
RFF+ON:<original_order_number>'
NAD+BY+<buyer_ean>::9'
NAD+SU+<supplier_ean>::9'
CUX+2:<currency_code>:9'
  ┌─── Repeated per line item ─────────────────────────────────
  │ LIN+<line_no>+<action_code>'
  │ PIA+5+<isbn>:IB'
  │ QTY+21:<ordered_quantity>'
  │ QTY+83:<confirmed_quantity>'
  │ DTM+44:<YYYYMMDD>:102'
  │ FTX+LIN+++<status_code>:<table>:<agency>'
  │ RFF+LI:<koha_ordernumber>'
  └────────────────────────────────────────────────────────────
UNS+S'
CNT+2:<line_count>'
UNT+<segment_count>+<msg_ref>'
UNZ+<message_count>+<control_ref>'
```

### 6.3 Message-level (header) elements

The header (`UNB`, `UNH`, `BGM`, `DTM`, `NAD`, `RFF+ON`, `CUX`) is parsed, but
`process_ordrsp` acts purely on the line items. The buyer/supplier `NAD` parties, the `BGM`
response number (`231` order response / `23C` with copy detail), the `RFF+ON` purchase
order number and the message dates are **informational** for order responses — none of
them are required for Koha to act, because matching is done per line via `RFF+LI`. Only the
BGM function code `9` (original) / `7` (retransmission) is interpreted. A valid envelope
(`UNB`/`UNH` declaring message type `ORDRSP`, closed by `UNT`/`UNZ`) is required for the
file to parse.

### 6.4 Line-level elements

A line begins at a `LIN` segment and runs until the next `LIN` (or the closing
`UNS`/`CNT`/`UNT`).

#### LIN — line item with action code (required)

```edifact
LIN+1+5'
```

- Element 0 = line sequence number.
- **Element 1 component 0 = the action/response code.** This is what determines Koha's
  behavior for the line — see [6.5](#65-action-codes-and-behavior).
- Element 2 component 0 (product id) may be present but is informational on a response.

#### RFF+LI — Koha order number (**REQUIRED**)

```edifact
RFF+LI:4123'
```

- Element 0 component 0 = `LI`; component 1 = the value Koha put in the original ORDERS
  message: the **`aqorders.ordernumber`**. **This is how Koha finds the order to update.**
  Without it, Koha cannot match the response line to an order and the update is a no-op.

#### FTX+LIN — coded status text (optional)

```edifact
FTX+LIN+++OP:8B:28'
```

- Element 0 = `LIN` marks it as order-line free text.
- Element 2 holds the coded status: component 0 is the code, component 1 is the code table.
- Koha expands it to human-readable text using the EDItEUR code tables (see
  [6.6](#66-coded-status-text-and-availability-tables)). The expanded text becomes the
  `cancellationreason` (for cancellations) or part of the `suppliers_report` (for
  everything else). Text with no recognized table falls through as the raw code.

#### DTM+44 — availability date (optional)

```edifact
DTM+44:20250301:102'
```

- Qualifier `44` = expected availability / ship date.
- **Behavior:** for non-cancelled lines, Koha appends ` Available: <date>` to the
  `suppliers_report`.

#### QTY — quantities (parsed, not acted on)

`QTY` qualifiers such as `21` (ordered) and `83` (confirmed/backorder) are parsed into the
line object, but `process_ordrsp` does not change Koha order quantities from a response — it
only cancels or records a report. Treat them as informational.

### 6.5 Action codes and behavior

The `LIN` action code (element 1) drives the behavior:

| Action Code | Meaning | Koha behavior |
|-------------|---------|---------------|
| `2` | Cancelled | **The order line is cancelled.** `orderstatus` set to `cancelled`, `datecancellationprinted` set to today, and `cancellationreason` set to the coded FTX text. |
| `3` | Change requested | Recorded only — coded text (+ availability date) written to `suppliers_report`. |
| `4` | No action | Recorded only. |
| `5` | Accepted | Recorded only. |
| `10` | Not found | Recorded only. |
| `24` | Recorded (accepted, change notified) | Recorded only. |
| *(absent / other)* | passed through | Treated as non-cancelled → recorded only. |

**Only code `2` triggers an automatic change** (cancellation). Every other code — and a
missing code — results in the supplier's status being stored on the order line as a report
for staff to read; the order otherwise stays as it was.

### 6.6 Coded status text and availability tables

Koha expands the `FTX+LIN` code according to the code table given in its second component.
The tables below list the meanings from `Koha::Edifact::Line` (`translate_8B` /
`translate_12B`) — the text Koha assigns when it recognizes a code. A few obvious typos in
the source strings (e.g. "Out pf print", "Oustanding", "orderline", "Unavailable@") are
normalized here for readability; the codes themselves are verbatim.

#### Tables 8B / 7B — EDItEUR availability codes (`7B` is a subset of `8B`)

| Code | Meaning |
|------|---------|
| `AB` | Publication abandoned |
| `AD` | Apply direct |
| `AU` | Publisher address unknown |
| `CS` | Status uncertain |
| `FQ` | Only available abroad |
| `HK` | Paperback OP: hardback available |
| `IB` | In stock |
| `IP` | In print and in stock at publisher |
| `MD` | Manufactured on demand |
| `NK` | Item not known |
| `NN` | We do not supply this item |
| `NP` | Not yet published |
| `NQ` | Not stocked |
| `NS` | Not sold separately |
| `OB` | Temporarily out of stock |
| `OF` | This format out of print: other format available |
| `OP` | Out of print |
| `OR` | Out of print; new edition coming |
| `PK` | Hardback out of print: paperback available |
| `PN` | Publisher no longer in business |
| `RE` | Awaiting reissue |
| `RF` | Refer to other publisher or distributor |
| `RM` | Remaindered |
| `RP` | Reprinting |
| `RR` | Rights restricted: cannot supply in this market |
| `SD` | Sold |
| `SN` | Our supplier cannot trace |
| `SO` | Pack or set not available: single items only |
| `ST` | Stocktaking: temporarily unavailable |
| `TO` | Only to order |
| `TU` | Temporarily unavailable |
| `UB` | Item unobtainable from our suppliers |
| `UC` | Unavailable; reprint under consideration |

#### Table 12B — EDItEUR order-response status codes

| Code | Meaning |
|------|---------|
| `100` | Order line accepted |
| `101` | Price query: order line will be held awaiting customer response |
| `102` | Discount query: order line will be held awaiting customer response |
| `103` | Minimum order value not reached: order line will be held |
| `104` | Firm order required: order line will be held awaiting customer response |
| `110` | Order line accepted, substitute product will be supplied |
| `200` | Order line not accepted |
| `201` | Price query: order line not accepted |
| `202` | Discount query: order line not accepted |
| `203` | Minimum order value not reached: order line not accepted |
| `205` | Order line not accepted: quoted promotion is invalid |
| `206` | Order line not accepted: quoted promotion has ended |
| `207` | Order line not accepted: customer ineligible for quoted promotion |
| `210` | Order line not accepted: substitute product is offered |
| `220` | Outstanding order line cancelled: reason unspecified |
| `221` | Outstanding order line cancelled: past order expiry date |
| `222` | Outstanding order line cancelled by customer request |
| `223` | Outstanding order line cancelled: unable to supply |
| `300` | Order line passed to new supplier |
| `301` | Order line passed to secondhand department |
| `400` | Backordered - awaiting supply |
| `401` | On order from our supplier |
| `402` | On order from abroad |
| `403` | Backordered, waiting to reach minimum order value |
| `404` | Despatched from our supplier, awaiting delivery |
| `405` | Our supplier sent wrong item(s), re-ordered |
| `406` | Our supplier sent short, re-ordered |
| `407` | Our supplier sent damaged item(s), re-ordered |
| `408` | Our supplier sent imperfect item(s), re-ordered |
| `409` | Our supplier cannot trace order, re-ordered |
| `410` | Ordered item(s) being processed by bookseller |
| `411` | Ordered item(s) being processed by bookseller, awaiting customer action |
| `412` | Order line held awaiting customer instruction |
| `500` | Order line on hold - contact customer service |
| `800` | Order line already despatched |
| `900` | Cannot trace order line |
| `901` | Order line held: note title change |
| `902` | Order line held: note availability date delay |
| `903` | Order line held: note price change |
| `999` | Temporary hold: order action not yet determined |

If the table is not `8B`/`7B`/`12B`, or the code is not in the table, the raw code is
stored as-is.

### 6.7 Full worked example

A response acknowledging one line, cancelling another, and backordering a third:

```edifact
UNA:+.? '
UNB+UNOC:3+5013546027856:14+9876543210123:14+250116:0902+RSP0001++ORDRSP'
UNH+1+ORDRSP:D:96A:UN:EAN005'
BGM+231+RESP-778+9'
DTM+137:20250116:102'
RFF+ON+B-2025-88'
NAD+BY+9876543210123::9'
NAD+SU+5013546027856::9'
CUX+2:USD:9'
LIN+1+5'
QTY+21:2'
FTX+LIN+++IB:8B:28'
RFF+LI:4123'
LIN+2+2'
FTX+LIN+++OP:8B:28'
RFF+LI:4124'
LIN+3+5'
QTY+83:1'
DTM+44:20250301:102'
FTX+LIN+++400:12B:28'
RFF+LI:4125'
UNS+S'
CNT+2:3'
UNT+18+1'
UNZ+1+RSP0001'
```

What Koha does with it:

- **Line 1 (order 4123, action `5` accepted):** records a supplier report of "In stock"
  (`IB` via table `8B`). Order status unchanged.
- **Line 2 (order 4124, action `2` cancelled):** cancels the order line. `orderstatus`
  becomes `cancelled`, cancellation date is today, and the cancellation reason is stored as
  "Out of print" (`OP` via `8B`).
- **Line 3 (order 4125, action `5` accepted):** records a supplier report of "Backordered -
  awaiting supply" (`400` via `12B`) with " Available: 20250301" appended.

---

## 7. INVOIC — Invoice (Supplier to Koha)

### 7.1 What an invoice does in Koha

Processing one `INVOIC` message causes Koha to:

1. Create (or, under the plugin, reuse) an **aqinvoice** record keyed on the invoice
   number, with a shipment date, billing date and shipment cost.
2. For each invoice line, find the **Koha order** it refers to and **receive** the stated
   quantity against it — either a full receipt or a partial receipt that splits the order.
3. Set the received order's unit price and tax from the invoice amounts.
4. Assign barcodes to received items and set the date accessioned.
5. Mark any linked purchase suggestion as `AVAILABLE`.

The invoice is the message that moves money and moves items from "on order" to "received".

### 7.2 Message Structure

```edifact
UNA:+.? '
UNB+UNOC:3+<supplier_id>:<qual>+<buyer_id>:<qual>+YYMMDD:HHMM+<control_ref>++INVOIC'
UNH+<msg_ref>+INVOIC:D:96A:UN:EAN008'
BGM+380+<invoice_number>+9'
DTM+131:<YYYYMMDD>:102'
DTM+137:<YYYYMMDD>:102'
RFF+DQ:<despatch_note_number>'
NAD+BY+<buyer_ean>::9'
NAD+SU+<supplier_ean>::9'
CUX+2:<currency_code>:4'
PAT+1++5:3:D:<days>'
ALC+C++++DL'
MOA+<qualifier>:<amount>'
  ┌─── Repeated per line item ─────────────────────────────────
  │ LIN+<line_no>++<ean_13>:EN'
  │ PIA+5+<isbn>:IB'
  │ IMD+L+009+:::<author>'
  │ IMD+L+050+:::<title>'
  │ QTY+47:<invoiced_quantity>'
  │ GIR+<copy>+<barcode>:LAC+<branch>:LLO+<seq>:LSQ'
  │ MOA+203:<line_amount_excl_tax>'
  │ MOA+128:<total_incl_tax>'
  │ MOA+124:<tax_amount>'
  │ PRI+AAA:<net_price>'
  │ PRI+AAB:<gross_price>'
  │ RFF+LI:<koha_ordernumber>'
  │ TAX+7+VAT+++:::<tax_rate>+<tax_category>'
  │ ALC+A++++DI::28'
  │ PCD+3:<discount_percentage>'
  │ MOA+8:<allowance_amount>'
  └────────────────────────────────────────────────────────────
UNS+S'
CNT+2:<line_count>'
MOA+129:<total_payable>'
MOA+9:<total_line_items>'
TAX+7+VAT+++:::<rate>+<category>'
MOA+125:<taxable_amount>'
MOA+124:<tax_total>'
UNT+<segment_count>+<msg_ref>'
UNZ+<message_count>+<control_ref>'
```

### 7.3 Message-level (header) elements

#### BGM — invoice number (required)

```edifact
BGM+380+INV-2025-0042+9'
```

- Element 0 = `380` (commercial invoice) — document code, informational.
- **Element 1 = the invoice number.** Stored as `aqinvoice.invoicenumber`. This is the
  human-facing invoice identifier throughout acquisitions, and (in the plugin) the key used
  to detect and reuse an already-imported invoice. Treat it as required.
- Element 2 = message function (`9` original, `7` retransmission), informational.

#### NAD+SU — supplier identifier (required in core Koha)

```edifact
NAD+SU+5013546027856::9'
```

- Component 0 of element 1 is the supplier's SAN/EAN.
- **In core Koha this is how the vendor is matched.** `process_invoice` looks up a
  `VendorEdiAccount` whose `san` equals this value. If no account matches, the whole invoice
  is skipped and an entry is written to `edifact_errors`:
  `Skipped invoice <n> with unmatched vendor san: <ean>`. Under core Koha the SAN here
  **must** exactly match the SAN configured on the receiving vendor EDI account.
- Under the plugin this element is only consulted when `skip_nonmatching_san_suffix` is
  enabled (see [7.8](#78-plugin-behavior-differences)); otherwise the vendor is taken from
  the message row's own EDI account.

`NAD+BY` (buyer) is parsed but not used during invoice receipt.

#### DTM — dates

```edifact
DTM+137:20250115:102'
DTM+131:20250114:102'
```

| Qualifier | Meaning | Behavior |
|-----------|---------|----------|
| `137` | Invoice / document date | Stored as `aqinvoice.shipmentdate`; also used as `datereceived` on every received order line. Effectively required — without it the shipment and receipt dates are blank. |
| `131` | Tax point date | Stored as `aqinvoice.billingdate`. Optional: if absent or not 8 digits, Koha falls back to the `137` date. |

#### RFF+DQ — despatch reference (optional)

```
RFF+DQ:<despatch_note_number>'
```

Delivery/despatch note reference; informational.

#### ALC + MOA — shipment charge

```edifact
ALC+C++++DL'
MOA+8:12.50'
```

- The header shipment charge is stored as `aqinvoice.shipmentcost`, charged against the
  vendor EDI account's shipment budget.
- In **core Koha**, the logic scans only the segments before the first `LIN`, notes an
  `ALC+C` charge whose element 4 is `DL` (delivery), and sums **every** `MOA` amount it
  finds there.
- The **plugin** changes this substantially — see [7.8](#78-plugin-behavior-differences). It
  sums only the specific `MOA` qualifiers you opt into (8, 79, 124, 131, 304) and scans the
  whole message rather than just the header.

### 7.4 Line-level elements

#### LIN — line item (required to form a line)

```edifact
LIN+1++9780306406157:EN'
```

- Element 0 = line sequence number; element 2 component 0 = product identifier (EAN/ISBN),
  component 1 = `EN`. On an invoice this identifier is informational — Koha matches lines to
  orders by the order number in `RFF+LI`, not by this identifier.

#### RFF+LI — Koha order number (**REQUIRED**)

```edifact
RFF+LI:4123'
```

- Element 0 component 0 = `LI`; component 1 = the **`aqorders.ordernumber`** from the
  original ORDERS message.
- **This is the single most important element in the whole message.** It is how the invoice
  line is matched to an open order. If it is missing, the line is skipped with
  `Skipped invoice line <n>, missing ordernumber`. If it points to an order that does not
  exist, the line is skipped with `cannot find order with ordernumber <n>`.
- The related qualifiers `QLI` (supplier quote-line reference) and `SLI` (supplier
  order-line reference) are parsed but not used for invoice matching.

#### QTY — invoiced quantity (required)

```edifact
QTY+47:2'
```

- Qualifier `47` = quantity invoiced. Any other qualifier is stored as the generic quantity.
- **Behavior:** Koha uses the `47` quantity if present, otherwise the generic quantity. This
  number decides how many copies are received:
  - **Less than** the ordered quantity → a **partial receipt** (see [7.6](#76-how-items-are-received)).
  - **Equal to** the ordered quantity → the line is received in full and marked `complete`.

#### MOA — monetary amounts (required for correct pricing)

```edifact
MOA+203:31.98'
MOA+128:34.54'
MOA+124:2.56'
```

Qualifier is component 0, amount is component 1. These are per-order-line totals.

| Qualifier | Meaning |
|-----------|---------|
| `203` | Line amount after allowances, **excluding** tax |
| `128` | Line total **including** allowances and tax |
| `124` | Tax amount on the line (summed if repeated) |
| `52` | Discount amount |
| `113` | Prepayment amount |
| `146` | Unit price in an alternate currency |

See [7.5](#75-how-price-and-tax-are-derived) for how these become the stored price.

#### PRI — price (informational on invoices)

| Qualifier | Description |
|-----------|-------------|
| `AAA` | Net calculation price (excl. tax, incl. allowances/charges) |
| `AAB` | Gross calculation price (excl. all taxes, allowances, charges) |
| `AAE` | Information price (incl. tax, excl. allowances/charges) |
| `AAF` | Information price (incl. all) |

**`PRI` is parsed but not used to set the received price** — Koha derives the stored price
from `MOA` (see [7.5](#75-how-price-and-tax-are-derived)). A vendor that sends only `PRI`
and no `MOA+203`/`MOA+128` will produce zero-priced receipts.

#### TAX — tax rate (optional)

```edifact
TAX+7+VAT+++:::20.00+S'
```

- Element 0 = `7` (duty/tax). Element 1 component 0 = tax type (`VAT`, `GST`, `IMP`). Element
  4 component 3 = the percentage rate. Element 5 component 0 = category:

| Category Code | Meaning |
|---------------|---------|
| `E` | Exempt from tax |
| `G` | Export item, tax not charged |
| `H` | Higher rate |
| `L` | Lower rate |
| `S` | Standard rate |
| `Z` | Zero-rated |

- **Behavior:** the rate populates `tax_rate_on_receiving`, and `tax_value_on_receiving` is
  computed as `quantity × excl-tax price × rate`. If category is `Z` and no rate is given,
  the rate is taken as 0. No `TAX` segment → received tax rate is 0.

#### GIR — copy-level data (optional, drives item receipt)

```edifact
GIR+1+9781234567890:LAC+MPL:LLO'
```

- Element 0 component 0 = copy number. Each following element is `value:qualifier`. On
  invoices the two qualifiers Koha acts on are:

| Qualifier | Field | Behavior on receipt |
|-----------|-------|---------------------|
| `LLO` | branch | Used to match which of the order's items (grouped by home branch) this invoiced copy corresponds to. |
| `LAC` | barcode | Assigned to the received item **if** the item has no barcode yet and the barcode is not already in use. A duplicate barcode is logged to `edifact_errors` and skipped. |

Other GIR qualifiers (fund `LFN`, item type `LST`, shelfmark `LSM`, etc.) are recognized by
the parser but are primarily used on outbound orders; on invoices only `LLO` and `LAC`
change item data. `FTX` and `IMD` are parsed on invoice lines but do not drive receipt
behavior.

#### RFF+DQ / ALC / PCD (line level)

Line-level allowance/charge (`ALC+A ... DI`), percentage (`PCD`), and their `MOA` are parsed
as part of the amount handling; the effective received price still comes from the `MOA`
totals described in [7.5](#75-how-price-and-tax-are-derived).

### 7.5 How price and tax are derived

Koha computes the stored unit prices from `MOA`, **not** from `PRI`
(`Koha::EDI::_get_invoiced_price`). For a line with received quantity `q`:

```
line total (tax incl.) = MOA+128            # if present
                       = MOA+203 + MOA+124   # otherwise (tax treated as 0 if absent)
excl-tax total         = MOA+203
unitprice / unitprice_tax_included = line total / q
unitprice_tax_excluded             = excl-tax total / q
tax_rate_on_receiving  = TAX rate / 100      # 0 if no TAX segment
tax_value_on_receiving = q × unitprice_tax_excluded × tax_rate
```

So a line needs at least `MOA+203` (and ideally `MOA+128`) for the received price to be
correct.

### 7.6 How items are received

- **Full receipt** (`invoiced == ordered`): the order line is updated in place —
  `quantityreceived` set, `orderstatus` → `complete`, prices and dates set, and items
  received (barcodes assigned, records updated).
- **Partial receipt** (`invoiced < ordered`): the original order is reduced by the invoiced
  quantity and set to `partial`; a **copy** of the order is created for the received copies
  with `orderstatus = complete`; items are transferred to the new order line by matching
  branch (`LLO`), then received.
- **Item matching:** received copies are grouped by home branch; for each invoiced copy
  (walking the `GIR` occurrences) an item is pulled from the matching branch bucket. If the
  invoiced branch has no unreceived item, it logs
  `No matching item found for invoice line <n>:<copy> at branch <branch>`.
- **Suggestions:** if the order's biblio is linked to a purchase suggestion, its status is
  set to `AVAILABLE`.
- **`AcqItemSetSubfieldsWhenReceived`** is honored when `AcqCreateItem` is `ordering`.

### 7.7 Full worked example

An invoice for order `4123` (2 copies ordered, 2 invoiced) and order `4124` (2 ordered,
only 1 invoiced — a partial receipt), from supplier SAN `5013546027856`:

```edifact
UNA:+.? '
UNB+UNOC:3+5013546027856:14+9876543210123:14+250115:1013+INV0001++INVOIC'
UNH+1+INVOIC:D:96A:UN:EAN008'
BGM+380+INV-2025-0042+9'
DTM+137:20250115:102'
DTM+131:20250114:102'
NAD+BY+9876543210123::9'
NAD+SU+5013546027856::9'
CUX+2:USD:4'
ALC+C++++DL'
MOA+8:9.00'
LIN+1++9780306406157:EN'
IMD+L+050+:::Programming Perl'
QTY+47:2'
MOA+203:63.98'
MOA+128:69.10'
MOA+124:5.12'
PRI+AAB:31.99'
TAX+7+VAT+++:::8.00+S'
RFF+LI:4123'
GIR+1+11223344:LAC+MPL:LLO'
GIR+2+11223345:LAC+MPL:LLO'
LIN+2++9781491910740:EN'
IMD+L+050+:::Head First Java'
QTY+47:1'
MOA+203:38.00'
MOA+128:41.04'
MOA+124:3.04'
TAX+7+VAT+++:::8.00+S'
RFF+LI:4124'
GIR+1+11223346:LAC+MPL:LLO'
UNS+S'
CNT+2:2'
MOA+9:101.98'
UNT+30+1'
UNZ+1+INV0001'
```

What Koha does with it:

- Creates invoice `INV-2025-0042`, shipment date `2025-01-15`, billing date `2025-01-14`,
  shipment cost from the header charge (see the plugin note below on which `MOA` counts).
- **Line 1 (order 4123):** receives 2 of 2 → full receipt, order marked `complete`. Per-copy
  price `69.10 / 2 = 34.55` incl. tax, `63.98 / 2 = 31.99` excl. tax, tax rate 8%. Barcodes
  `11223344` and `11223345` assigned to the two `MPL` items.
- **Line 2 (order 4124):** receives 1 of 2 → partial receipt. The original order drops to 1
  remaining and becomes `partial`; a new completed line is split off for the 1 received copy,
  priced `41.04` incl. / `38.00` excl., barcode `11223346`.

### 7.8 Plugin behavior differences

When a vendor EDI account has the EDIFACT Enhanced plugin set, `process_invoice` hands the
entire job to the plugin's `edifact_process_invoice`, which reads the same elements above but
changes these behaviors:

- **Vendor selection.** The vendor comes from the message row's own EDI account, not from
  `NAD+SU`. `NAD+SU` is only checked when `skip_nonmatching_san_suffix` is on; a mismatch
  then re-routes the message to the correct account (or sets it back to `new` if none
  matches). This makes core Koha's "unmatched vendor san → skip" failure far less common.
- **Idempotent invoices.** Before creating an `aqinvoice`, the plugin searches for an
  existing one by invoice number + bookseller (+ message id). A retransmitted invoice updates
  the existing Koha invoice instead of duplicating it.
- **Cancelled orders are skipped** — a line whose Koha order is already `cancelled` is not
  received.
- **Cross-vendor tolerance.** If the order's basket vendor differs from the invoice vendor,
  the line is still received when both vendor accounts use the same plugin class; otherwise
  the message is handed back for the correct account.
- **Standing orders** always receive as a partial (keeping one copy remaining) so the
  standing order stays open.
- **Shipment charge is opt-in per `MOA` qualifier** (`shipment_charges_moa_8` / `_79` /
  `_124` / `_131` / `_304`, plus `shipment_charges_alc_dl`), and the plugin scans the whole
  message, not just the header. `add_tax_to_shipping_costs` adds the vendor tax rate to the
  total. In the example above, the header `MOA+8:9.00` is only included in shipment cost if
  `shipment_charges_moa_8` is enabled.
- **Invoice adjustments from MOA.** If `invoice_adjustment_rules` are configured, the plugin
  reads every message-level `MOA` and creates `aqinvoice_adjustments` for the qualifiers
  named in the rules.
- **Item receipt** is done by the plugin's own `_receipt_items`, which additionally sets the
  item's `booksellerid` (source of acquisition), `dateaccessioned`, and (per config)
  `price`/`replacementprice`, a not-for-loan value (`set_nfl_on_receipt`), an
  `itemnotes_nonpublic` of "Received via EDIFACT", and can clear the LIN-source item field.
- **Optional close.** `close_invoice_on_receipt` sets the invoice `closedate` when done.

The **required** elements are the same as core Koha — `RFF+LI`, a quantity, and line amounts
are needed regardless of plugin. The plugin mainly changes vendor matching, duplicate
handling, and item-update side effects.

---

## 8. GIR Qualifiers — Copy-Level Data

The GIR (Related Identification Numbers) segment carries copy-level detail. Each GIR segment
can hold up to **5** qualifier+value pairs. If more data is needed for a single copy, a
continuation GIR with the same copy sequence number is used.

| Qualifier | Field | Description |
|-----------|-------|-------------|
| `LAC` | Barcode | Item barcode |
| `LAF` | First accession number | First accession number in range |
| `LAL` | Last accession number | Last accession number in range |
| `LCL` | Classification | Classification number |
| `LCO` | Item unique ID | Unique item identifier |
| `LCV` | Copy value | Copy value/price |
| `LFH` | Feature heading | Feature heading |
| `LFN` | Fund allocation | Budget/fund code |
| `LFS` | Filing suffix | Filing suffix |
| `LHC` | Library holding code | Library holding code |
| `LLN` | Loan category | Loan category |
| `LLO` | Branch | Library branch code |
| `LLS` | Label sublocation | Label sublocation |
| `LQT` | Part order quantity | Partial order quantity |
| `LRP` | Rotation plan | Library rotation plan |
| `LRS` | Record sublocation | Record sublocation |
| `LSC` | Statistical category | Statistical category |
| `LSM` | Shelfmark | Call number / shelfmark |
| `LSQ` | Sequence code | Shelving sequence / collection code |
| `LST` | Stock category | Item type code |
| `LSZ` | Size code | Physical size code |
| `LVC` | Coded servicing instruction | Coded servicing instruction |
| `LVT` | Servicing instruction | Free-text servicing instruction |
| `RIC` | Reader interest category | Reader interest category |

### Most Commonly Used Qualifiers

| Qualifier | Required? | Notes |
|-----------|-----------|-------|
| `LLO` | Recommended | Must match a Koha branch code |
| `LFN` | Recommended | Must match a Koha budget/fund code |
| `LST` | Optional | Must match a Koha item type code |
| `LSQ` | Optional | Shelving sequence or collection code |
| `LSM` | Optional | Call number |
| `LAC` | Optional (INVOIC) | Barcode — assigned to received items |

---

## 9. RFF Qualifiers — References

| Qualifier | Context | Description |
|-----------|---------|-------------|
| `LI` | All messages (line level) | Buyer's order line reference — the Koha order number |
| `QLI` | QUOTES, ORDERS (line level) | Supplier's quotation line item reference |
| `SLI` | Any (line level) | Supplier's order line reference number |
| `ON` | QUOTES, ORDRSP (message level) | Purchase order number |
| `DQ` | INVOIC (message level) | Despatch note / delivery note reference |

### Reference Matching

- **`RFF+LI`** is the primary key for matching incoming INVOIC and ORDRSP lines to Koha
  orders. It must accurately echo the Koha order number (`aqorders.ordernumber`) from the
  original ORDERS message. This is the single most common source of EDI problems.
- **`RFF+QLI`** links quotes to subsequent orders. When Koha generates an ORDERS message in
  response to a quote, it includes the supplier's `QLI` reference.
- **`RFF+ON`** at message level in QUOTES is used as the basket name (when `po_is_basketname`
  is configured).

---

## 10. File Naming and Transport

### Transport Protocols

| Protocol | Description |
|----------|-------------|
| **SFTP** | SSH File Transfer Protocol (recommended) |
| **FTP** | Standard FTP |
| **FILE** | Local filesystem (for testing or same-server setups) |

### Directory Structure

Each EDI account is configured with:
- A **download directory** — where the supplier places files for Koha to retrieve
- An **upload directory** — where Koha places outgoing ORDERS files for the supplier

### File Extensions

| Extension | Message Type |
|-----------|-------------|
| `.CEQ` | QUOTES (incoming) |
| `.CEP` | ORDERS (outgoing) |
| `.CEA` | ORDRSP (incoming) |
| `.CEI` | INVOIC (incoming) |

The Enhanced plugin can override the incoming invoice match suffix (`invoice_file_suffix`)
and the outgoing order suffix (`order_file_suffix`).

### Processing Schedule

The EDI cron job (`edi_cron.pl`) is typically run every **15 minutes**; see
[Section 2](#2-how-koha-consumes-inbound-messages) for the full cycle and the
`EdifactInvoiceImport` gate.

---

## 11. Identification and Qualifiers

### ID Code Qualifiers

These qualifiers identify the type of party identifier used in UNB and NAD segments:

| Qualifier | Standard | Description |
|-----------|----------|-------------|
| `14` | EAN | EAN International identifier |
| `31B` | SAN | US Standard Address Number |
| `91` | — | Assigned by supplier |
| `92` | — | Assigned by buyer |

**Note:** Within NAD segments, EAN uses agency code `9` (not `14`). Koha handles this
conversion automatically for outgoing messages.

### Supplier Requirements

The supplier must provide:
- Their SAN or EAN identifier and its qualifier type
- FTP/SFTP connection credentials (host, username, password)
- The download and upload directory paths on the server

The library will provide:
- Their branch EAN/SAN identifiers (one per branch)
- Branch codes used in GIR `LLO` segments
- Fund/budget codes used in GIR `LFN` segments
- Item type codes used in GIR `LST` segments

---

## 12. Configuration Prerequisites

Before EDI messages can be exchanged, both parties need to agree on the following.

### Standard

Koha supports two EDIFACT standard profiles:
- **EDItEUR** (default) — International standard for the book trade
- **BIC** — Book Industry Communication (UK-focused)

The choice of standard affects BGM codes in ORDERS messages (e.g., `22V` for quote-response
orders is BIC-only).

### Code Mappings

The supplier and library must share and agree on values for:

| Data Element | GIR Qualifier | Notes |
|--------------|--------------|-------|
| Branch codes | `LLO` | Must match Koha branch codes exactly |
| Fund codes | `LFN` | Must match Koha budget codes exactly |
| Item type codes | `LST` | Must match Koha item type codes exactly |
| Collection codes | `LSQ` | Must match values configured in Koha |

### Minimum Required Segments (quick checklist)

For each inbound message type, these segments are required for successful processing. See
[Section 13](#13-required-element-summary) for the per-element detail and the behavior when
one is missing.

**QUOTES (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, NAD+SU, NAD+BY
- Per line: LIN or PIA (ISBN/EAN), IMD+050 (title), QTY
- UNS, UNT, UNZ

**ORDRSP (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, UNS, UNT, UNZ
- Per line: LIN with action code, RFF+LI (Koha order number)

**INVOIC (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, DTM+137, NAD+SU, NAD+BY
- Per line: LIN, QTY+47, RFF+LI (Koha order number), MOA+203 (and ideally MOA+128)
- UNS, UNT, UNZ

---

## 13. Required-Element Summary

"Required" here means: without it, Koha either skips the line/message or cannot take the
intended action.

### INVOIC

| Level | Element | Required | Purpose |
|-------|---------|----------|---------|
| Envelope | `UNB`/`UNH` (type `INVOIC`), `UNT`/`UNZ` | Yes | Parseable message |
| Header | `BGM` invoice number | Yes (in practice) | `aqinvoice.invoicenumber` |
| Header | `NAD+SU` supplier SAN | Yes in core Koha | Vendor match (plugin: only with `skip_nonmatching_san_suffix`) |
| Header | `DTM+137` date | Yes (in practice) | Shipment date + receipt date |
| Header | `DTM+131` tax point | No | Billing date (falls back to `137`) |
| Line | `LIN` | Yes | Delimits the line |
| Line | `RFF+LI` order number | **Yes** | Matches line to Koha order |
| Line | `QTY` (ideally `47`) | Yes | Quantity to receive; full vs partial |
| Line | `MOA+203` (and `128`) | Yes for pricing | Net / total amounts → unit price |
| Line | `MOA+124` / `TAX` | No | Tax amount / rate on receipt |
| Line | `GIR` `LLO`/`LAC` | No | Branch match + barcode assignment |

### ORDRSP

| Level | Element | Required | Purpose |
|-------|---------|----------|---------|
| Envelope | `UNB`/`UNH` (type `ORDRSP`), `UNT`/`UNZ` | Yes | Parseable message |
| Line | `LIN` action code | Yes | `2` cancels; anything else records a report |
| Line | `RFF+LI` order number | **Yes** | Matches line to Koha order |
| Line | `FTX+LIN` coded status | No | Report / cancellation reason text |
| Line | `DTM+44` availability date | No | Appended to supplier report |

---

## 14. Common Failure Modes

- **`RFF+LI` carries the wrong value.** By far the most common EDI problem. The value must be
  the exact `ordernumber` Koha sent in `RFF+LI` on the original ORDERS message. If the vendor
  echoes their own line reference instead, invoice lines are skipped (`missing ordernumber` /
  `cannot find order`) and order responses silently do nothing.
- **Invoice SAN does not match the vendor account (core Koha).** The whole invoice is skipped
  with an unmatched-vendor-san error. Fix the SAN on the vendor EDI account, or use the
  plugin with `skip_nonmatching_san_suffix`.
- **Invoices never process at all.** Check that `EdifactInvoiceImport` is `automatic`; when it
  is `manual`, invoices wait for a staff member to run them.
- **Prices come out as zero or wrong.** Koha derives the received price from `MOA` (203 / 128
  / 124), not from `PRI`. A vendor that only sends `PRI` will produce zero-priced receipts.
  Ensure `MOA+203` (and ideally `MOA+128`) is present per line.
- **Barcode not assigned / "duplicate found".** A `GIR ... LAC` barcode is only applied when
  the item has no barcode and the barcode is not already used elsewhere; otherwise it is
  logged to `edifact_errors` and skipped.
- **"No matching item found ... at branch."** The invoiced `GIR` branch (`LLO`) does not match
  the home branch of any unreceived item on the order. Confirm the branch codes the vendor
  sends match the codes Koha sent on the order.
- **Line-break characters every 80 chars.** Some vendors wrap EDIFACT output. The Enhanced
  plugin parser strips leading whitespace and non-printable characters to cope; core Koha does
  not. Prefer asking the vendor to disable line wrapping.

---

## 15. Plugin Extension Points

Koha supports vendor-specific plugins that can customize message handling per vendor EDI
account:

| Plugin Method | Purpose |
|--------------|---------|
| `edifact_order` | Replace the standard ORDERS message generator |
| `edifact_transport` | Replace the standard file transport |
| `edifact` | Replace the standard EDIFACT parser for incoming QUOTES / INVOIC messages |
| `edifact_process_invoice` | Replace the standard invoice processing pipeline |

Note that **ORDRSP processing has no plugin hook** — order responses are always parsed and
processed by core Koha. If a supplier's message format deviates from the standard described
here, a Koha plugin can be developed to handle the differences; see the
[EDI Enhanced Plugin Guide](EDI_Enhanced_Plugin_Guide.md).

---

*Generated from the Koha 25.11 source (`Koha/EDI.pm`, `Koha/Edifact/Message.pm`,
`Koha/Edifact/Line.pm`, `Koha/Edifact/Order.pm`, `misc/cronjobs/edi_cron.pl`) and the EDIFACT
Enhanced plugin 4.3.3. For plugin configuration options, see the
[EDI Enhanced Plugin Guide](EDI_Enhanced_Plugin_Guide.md). For the most current information,
consult the [Koha community manual](https://koha-community.org/manual/).*
