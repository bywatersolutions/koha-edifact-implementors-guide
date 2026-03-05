# Koha EDIFACT Supplier Integration Guide

This document specifies how book suppliers should format EDIFACT messages sent to Koha, and how Koha formats messages sent to suppliers. It is intended for technical staff at supplier organizations who are implementing or maintaining an EDI connection with a Koha-based library.

Koha's EDIFACT implementation is based on the **UN/EDIFACT** standard (directory version **D:96A**), using the **EANCOM** subset and **EDItEUR** / **BIC** (Book Industry Communication) conventions for the book supply chain.

---

## Table of Contents

- [1. Supported Message Types](#1-supported-message-types)
- [2. General EDIFACT Envelope](#2-general-edifact-envelope)
  - [2.1 Service String Advice (UNA)](#21-service-string-advice-una)
  - [2.2 Interchange Header (UNB)](#22-interchange-header-unb)
  - [2.3 Interchange Trailer (UNZ)](#23-interchange-trailer-unz)
  - [2.4 Character Encoding](#24-character-encoding)
  - [2.5 Text Escaping](#25-text-escaping)
- [3. ORDERS — Purchase Order (Koha to Supplier)](#3-orders--purchase-order-koha-to-supplier)
  - [3.1 Message Structure](#31-message-structure)
  - [3.2 Segment Reference](#32-segment-reference)
- [4. QUOTES — Quotation (Supplier to Koha)](#4-quotes--quotation-supplier-to-koha)
  - [4.1 Message Structure](#41-message-structure)
  - [4.2 Segment Reference](#42-segment-reference)
  - [4.3 IMD-to-MARC Mapping](#43-imd-to-marc-mapping)
- [5. ORDRSP — Order Response (Supplier to Koha)](#5-ordrsp--order-response-supplier-to-koha)
  - [5.1 Message Structure](#51-message-structure)
  - [5.2 Segment Reference](#52-segment-reference)
  - [5.3 Availability Code Tables](#53-availability-code-tables)
- [6. INVOIC — Invoice (Supplier to Koha)](#6-invoic--invoice-supplier-to-koha)
  - [6.1 Message Structure](#61-message-structure)
  - [6.2 Segment Reference](#62-segment-reference)
- [7. GIR Qualifiers — Copy-Level Data](#7-gir-qualifiers--copy-level-data)
- [8. RFF Qualifiers — References](#8-rff-qualifiers--references)
- [9. File Naming and Transport](#9-file-naming-and-transport)
- [10. Identification and Qualifiers](#10-identification-and-qualifiers)
- [11. Configuration Prerequisites](#11-configuration-prerequisites)

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

## 2. General EDIFACT Envelope

All message types share a common envelope structure.

### 2.1 Service String Advice (UNA)

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

Non-standard separator characters are **not** supported.

### 2.2 Interchange Header (UNB)

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

See [Section 10](#10-identification-and-qualifiers) for ID code qualifier values.

### 2.3 Interchange Trailer (UNZ)

```
UNZ+<message_count>+<control_ref>'
```

| Element | Description |
|---------|-------------|
| Message count | Number of messages in this interchange |
| Control ref | Must match the UNB control reference |

### 2.4 Character Encoding

- The wire format is **ISO 8859-1** (Latin-1), per `UNOC:3` in the UNB segment.
- Koha converts incoming messages from ISO 8859-1 to UTF-8 on receipt.
- Koha converts outgoing messages by stripping diacritics to ASCII-safe equivalents via transliteration.

### 2.5 Text Escaping

Within any data element, the following characters **must** be escaped with the `?` release character:

| Character | Escaped Form |
|-----------|-------------|
| `?` | `??` |
| `'` | `?'` |
| `:` | `?:` |
| `+` | `?+` |

---

## 3. ORDERS — Purchase Order (Koha to Supplier)

This is the **only** message type generated by Koha. Each ORDERS message represents one acquisition basket.

### 3.1 Message Structure

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

### 3.2 Segment Reference

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

The document number is the Koha basket number (or basket name if the EDI account has `po_is_basketname` enabled).

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

The agency code in NAD uses `9` for EAN (note: this differs from the UNB segment where EAN uses `14`).

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

Format indicator `L` = free-form text. Text values are broken into 35-character chunks across sub-components.

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

Each GIR segment carries up to 5 data pairs. If a copy requires more, a continuation GIR with the same copy sequence number is emitted.

| Qualifier | Data |
|-----------|------|
| `LLO` | Branch/location code (library branch receiving the item) |
| `LFN` | Fund/budget code |
| `LST` | Stock category / item type |
| `LSQ` | Shelving sequence / collection code |
| `LSM` | Shelfmark / call number |
| `LVT` | Servicing instruction (free text) |

See [Section 7](#7-gir-qualifiers--copy-level-data) for the complete qualifier table.

#### FTX — Free Text

```
FTX+LIN+++<text>'
```

- Only included when the order line has a vendor note.

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

## 4. QUOTES — Quotation (Supplier to Koha)

QUOTES messages allow a supplier to propose items for a library to order. Koha creates acquisition baskets, bibliographic records, and (optionally) items from the quote data.

### 4.1 Message Structure

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

### 4.2 Segment Reference

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

#### LIN — Line Item

Same format as ORDERS. The `EN` qualifier denotes EAN-13.

#### PIA — Additional Product Identification

Same format as ORDERS. Use `IB` for ISBN-10 or `EN` for EAN-13.

#### IMD — Item Description

QUOTES messages use a **richer set** of IMD codes than ORDERS, particularly for author information:

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

**Important:** IMD text values are limited to 35 characters per sub-component. Split longer values across sub-components separated by `:`.

#### QTY — Quantity

```
QTY+1:<quantity>'
```

- Qualifier `1` = discrete quantity (note: QUOTES uses `1`, not `21` as in ORDERS)

#### GIR — Copy-Level Data

Same qualifier scheme as ORDERS. See [Section 7](#7-gir-qualifiers--copy-level-data).

The `LFN` (fund allocation) qualifier is particularly important — Koha uses it to determine which budget/fund to charge.

#### PRI — Price

```
PRI+AAE:<price>'
```

or with type indicator:

```
PRI+AAE:<price>:DI'
```

- `DI` = discount price indicator

#### RFF — Reference (Line Level)

```
RFF+QLI:<supplier_quote_line_reference>'
```

- `QLI` = quotation line item — the supplier's unique reference for this quoted item
- Koha stores this and echoes it back in subsequent ORDERS messages

### 4.3 IMD-to-MARC Mapping

When Koha cannot find an existing bibliographic record matching the ISBN/EAN in a quote line, it creates a new MARC record from the IMD segments. The full mapping is listed in the IMD table above.

Key points for suppliers:
- **At minimum**, include IMD codes `050` (title) and `010` (author) for Koha to create a usable bibliographic record.
- Publisher (`120`) and date (`170`) are strongly recommended.
- ISBN via PIA is the primary key used for duplicate detection against the Koha catalog.

---

## 5. ORDRSP — Order Response (Supplier to Koha)

ORDRSP messages allow the supplier to acknowledge or reject order lines.

### 5.1 Message Structure

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
  │ GIR+<copy>+<item_id>:LCO+<barcode>:LAC+<fund>:LFN+<branch>:LLO'
  │ FTX+LIN++<status_code>:<table>:<agency>'
  │ PRI+AAE:<price>:CA:SRP'
  │ RFF+LI:<koha_ordernumber>'
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
| `231` | Order response |
| `23C` | Order response with copy detail |

| Function Code | Meaning |
|---------------|---------|
| `9` | Original |
| `7` | Retransmission |
| `4` | No action / change request |
| `27` | Not accepted |

#### LIN — Line Item with Action Code

```
LIN+<line_number>+<action_code>'
```

The action code indicates what happened to the order line:

| Action Code | Meaning | Koha Behavior |
|-------------|---------|---------------|
| `2` | Cancelled | Order is cancelled in Koha |
| `3` | Change requested | Recorded; no automatic action |
| `4` | No action | Recorded; no automatic action |
| `5` | Accepted | Recorded; no automatic action |
| `10` | Not found | Recorded; no automatic action |
| `24` | Recorded (accepted with changes) | Recorded; no automatic action |

**Important:** When the action code is `2` (cancelled), Koha automatically cancels the order line. All other codes are recorded as a supplier report against the order but do not trigger automatic changes.

#### QTY — Quantity

| Qualifier | Meaning |
|-----------|---------|
| `21` | Ordered quantity (echo of original order) |
| `83` | Confirmed / backorder quantity |

#### DTM — Date/Time

| Qualifier | Meaning |
|-----------|---------|
| `137` | Response date |
| `44` | Availability date / expected ship date |

#### FTX — Free Text with Availability Codes

```
FTX+LIN++<coded_status>:<code_table>:<agency>'
```

The coded status uses EDItEUR code tables. Koha recognizes tables `8B`, `7B`, `12B`, `10B`, and `9B`.

#### RFF — Reference

```
RFF+LI:<koha_ordernumber>'
```

**Critical:** The `RFF+LI` segment **must** contain the Koha order number from the original ORDERS message. This is how Koha matches response lines to existing orders.

### 5.3 Availability Code Tables

#### Table 8B / 7B — EDItEUR Availability Codes

Used in `FTX+LIN` segments to report item availability status:

| Code | Meaning |
|------|---------|
| `AB` | Publication abandoned |
| `AD` | Apply direct to publisher |
| `AU` | Publisher address unknown |
| `CS` | Status uncertain |
| `FQ` | Only available abroad |
| `HK` | Paperback OP — hardback available |
| `IB` | In stock |
| `IP` | In print and in stock at publisher |
| `MD` | Manufactured on demand |
| `NK` | Item not known |
| `NN` | We do not supply this item |
| `NP` | Not yet published |
| `NQ` | Not stocked |
| `NS` | Not sold separately |
| `OB` | Temporarily out of stock |
| `OF` | Out of print in this format — other format available |
| `OP` | Out of print |
| `OR` | Out of print — new edition coming |
| `PK` | Hardback OP — paperback available |
| `PN` | Publisher no longer in business |
| `RE` | Awaiting reissue |
| `RF` | Refer to other publisher or distributor |
| `RM` | Remaindered |
| `RP` | Reprinting |
| `RR` | Rights restricted — cannot supply in this market |
| `SD` | Sold — second-hand or antiquarian only |
| `SN` | Our supplier cannot trace |
| `SO` | Pack/set not available — single items only |
| `ST` | Stocktaking — temporarily unavailable |
| `TO` | Only to order |
| `TU` | Temporarily unavailable |
| `UB` | Unobtainable from our suppliers |
| `UC` | Unavailable — reprint under consideration |

#### Table 12B — EDItEUR Response Codes

Used in `FTX+LIN` segments for detailed order response status:

| Code | Meaning |
|------|---------|
| `100` | Order line accepted |
| `101` | Price query — held awaiting customer response |
| `102` | Discount query — held |
| `103` | Minimum order value not reached — held |
| `104` | Firm order required |
| `110` | Accepted — substitute product will be supplied |
| `200` | Order line not accepted |
| `201` | ISBN not recognised |
| `202` | Duplicate order |
| `203` | Cancelled — publication abandoned |
| `204` | Out of print |
| `205` | Restricted — cannot supply in this market |
| `206` | Out of scope for this supplier |
| `207` | Edition not available — other edition supplied |
| `210` | Not accepted — substitute offered |
| `220` | Cancelled at customer request |
| `221` | Cancelled — order expired |
| `222` | Cancelled — unobtainable |
| `223` | Cancelled — not yet available |
| `300` | Passed to new supplier |
| `301` | Passed to secondhand department |
| `400` | Backordered — awaiting stock |
| `401` | On order from publisher |
| `402` | On order from abroad |
| `403` | Backordered — not yet published |
| `404` | Applying for rights |
| `405` | Awaiting reissue |
| `406` | Awaiting new edition |
| `407` | Waiting to reach minimum order level |
| `408` | Held — awaiting customer instruction |
| `409` | Being manufactured on demand |
| `410` | Part supply — remainder backordered |
| `411` | Sourcing from alternative supplier |
| `412` | Being reprinted |
| `500` | Order line on hold |
| `800` | Already despatched |
| `900` | Cannot trace |
| `901` | Held — see note |
| `902` | Awaiting customer authority |
| `903` | Held — refer to customer services |
| `999` | Temporary hold — processing |

---

## 6. INVOIC — Invoice (Supplier to Koha)

Invoice messages trigger the creation of invoices in Koha and can automatically receive ordered items.

### 6.1 Message Structure

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
  │ MOA+52:<discount_amount>'
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

### 6.2 Segment Reference

#### BGM — Beginning of Message

```
BGM+380+<invoice_number>+9'
```

- Document type `380` = commercial invoice
- Function code `9` = original

The invoice number becomes the Koha invoice number.

#### DTM — Date/Time

| Qualifier | Meaning |
|-----------|---------|
| `131` | Tax point date |
| `137` | Invoice date |

#### RFF — Reference (Message Level)

```
RFF+DQ:<despatch_note_number>'
```

- `DQ` = delivery note number / despatch reference

#### ALC — Allowance or Charge (Message Level)

Shipment-level charges appear **before** the first LIN segment:

```
ALC+C++++DL'
```

| Element | Values |
|---------|--------|
| `C` | Charge (as opposed to `A` for allowance) |
| `DL` | Delivery/freight charge |

The monetary amount follows in a subsequent MOA segment.

#### LIN — Line Item

Same as ORDERS format.

#### QTY — Quantity

```
QTY+47:<quantity>'
```

- Qualifier `47` = invoiced quantity

If `QTY+47` is absent, Koha falls back to the generic quantity.

#### MOA — Monetary Amount

| Qualifier | Description |
|-----------|-------------|
| `8` | Allowance or charge amount (line-level) |
| `52` | Discount amount |
| `113` | Prepayment amount |
| `124` | Tax amount |
| `125` | Taxable amount (summary section) |
| `128` | Total line amount including allowances and tax |
| `129` | Total invoice payable (summary section) |
| `9` | Total of line items amount (summary section) |
| `146` | Unit price in alternate currency |
| `203` | Line item amount (after allowances, excl. tax) |

#### PRI — Price

| Qualifier | Description |
|-----------|-------------|
| `AAA` | Net calculation price (excl. tax, incl. allowances/charges) |
| `AAB` | Gross calculation price (excl. all taxes, allowances, charges) |
| `AAE` | Information price (incl. tax, excl. allowances/charges) |
| `AAF` | Information price (incl. all) |

#### TAX — Tax Details

```
TAX+7+<type>+++:::<rate>+<category>'
```

| Element | Values |
|---------|--------|
| Function | `7` = tax |
| Type | `VAT`, `GST`, or `IMP` |
| Rate | Percentage as decimal (e.g. `20.00`) |

| Category Code | Meaning |
|---------------|---------|
| `E` | Exempt from tax |
| `G` | Export item, tax not charged |
| `H` | Higher rate |
| `L` | Lower rate |
| `S` | Standard rate |
| `Z` | Zero-rated |

#### ALC — Allowance (Line Level)

```
ALC+A++++DI::28'
```

- `A` = allowance
- `DI` = discount
- `28` = code list identifier

Followed by PCD (percentage) and MOA (amount) segments for the discount.

#### RFF — Reference (Line Level)

```
RFF+LI:<koha_ordernumber>'
```

**Critical:** Each invoice line **must** include `RFF+LI` with the Koha order number from the original ORDERS message. This is the primary mechanism for matching invoiced items to open orders.

#### GIR — Copy-Level Data in Invoices

The `LAC` (barcode) qualifier is especially important in INVOIC messages. When present, Koha assigns the barcode from the GIR segment directly to the received item:

```
GIR+1+<barcode>:LAC+<branch>:LLO'
```

---

## 7. GIR Qualifiers — Copy-Level Data

The GIR (Related Identification Numbers) segment carries copy-level detail. Each GIR segment can hold up to **5** qualifier+value pairs. If more data is needed for a single copy, a continuation GIR with the same copy sequence number is used.

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

For typical orders:

| Qualifier | Required? | Notes |
|-----------|-----------|-------|
| `LLO` | Recommended | Must match a Koha branch code |
| `LFN` | Recommended | Must match a Koha budget/fund code |
| `LST` | Optional | Must match a Koha item type code |
| `LSQ` | Optional | Shelving sequence or collection code |
| `LSM` | Optional | Call number |
| `LAC` | Optional (INVOIC) | Barcode — mainly used in invoice messages |

---

## 8. RFF Qualifiers — References

| Qualifier | Context | Description |
|-----------|---------|-------------|
| `LI` | All messages (line level) | Buyer's order line reference — the Koha order number |
| `QLI` | QUOTES, ORDERS (line level) | Supplier's quotation line item reference |
| `SLI` | Any (line level) | Supplier's order line reference number |
| `ON` | QUOTES, ORDRSP (message level) | Purchase order number |
| `DQ` | INVOIC (message level) | Despatch note / delivery note reference |

### Reference Matching

- **`RFF+LI`** is the primary key for matching incoming messages to Koha orders. It must accurately echo the Koha order number from the original ORDERS message.
- **`RFF+QLI`** links quotes to subsequent orders. When Koha generates an ORDERS message in response to a quote, it includes the supplier's `QLI` reference.
- **`RFF+ON`** at message level in QUOTES is used as the basket name (when `po_is_basketname` is configured).

---

## 9. File Naming and Transport

### Transport Protocols

Koha supports three transport methods for EDI file exchange:

| Protocol | Description |
|----------|-------------|
| **SFTP** | SSH File Transfer Protocol (recommended) |
| **FTP** | Standard FTP |
| **FILE** | Local filesystem (for testing or same-server setups) |

### Directory Structure

Each EDI account is configured with:
- A **download directory** — where the supplier places files for Koha to retrieve
- An **upload directory** — where Koha places outgoing ORDERS files for the supplier to retrieve

### File Extensions

| Extension | Message Type |
|-----------|-------------|
| `.CEQ` | QUOTES (incoming) |
| `.CEP` | ORDERS (outgoing) |
| `.CEA` | ORDRSP (incoming) |
| `.CEI` | INVOIC (incoming) |

### Processing Schedule

The EDI cron job (`edi_cron.pl`) is typically run every **15 minutes**. The cycle is:
1. Download new QUOTE files
2. Download new INVOICE files
3. Upload pending ORDERS files
4. Download new ORDRSP files
5. Process all downloaded messages

---

## 10. Identification and Qualifiers

### ID Code Qualifiers

These qualifiers identify the type of party identifier used in UNB and NAD segments:

| Qualifier | Standard | Description |
|-----------|----------|-------------|
| `14` | EAN | EAN International identifier |
| `31B` | SAN | US Standard Address Number |
| `91` | — | Assigned by supplier |
| `92` | — | Assigned by buyer |

**Note:** Within NAD segments, EAN uses agency code `9` (not `14`). The Koha code handles this conversion automatically for outgoing messages.

### Supplier Requirements

The supplier must provide:
- Their SAN or EAN identifier
- The qualifier type for their identifier
- FTP/SFTP connection credentials (host, username, password)
- The download and upload directory paths on the server

The library will provide:
- Their branch EAN/SAN identifiers (one per branch)
- Branch codes used in GIR `LLO` segments
- Fund/budget codes used in GIR `LFN` segments
- Item type codes used in GIR `LST` segments

---

## 11. Configuration Prerequisites

Before EDI messages can be exchanged, both parties need to agree on the following:

### Standard

Koha supports two EDIFACT standard profiles:
- **EDItEUR** (default) — International standard for the book trade
- **BIC** — Book Industry Communication (UK-focused)

The choice of standard affects BGM codes in ORDERS messages (e.g., `22V` for quote-response orders is BIC-only).

### Code Mappings

The supplier and library must share and agree on values for:

| Data Element | GIR Qualifier | Notes |
|--------------|--------------|-------|
| Branch codes | `LLO` | Must match Koha branch codes exactly |
| Fund codes | `LFN` | Must match Koha budget codes exactly |
| Item type codes | `LST` | Must match Koha item type codes exactly |
| Collection codes | `LSQ` | Must match values configured in Koha |

### Minimum Required Segments

For each message type, these segments are required for successful processing:

**QUOTES (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, NAD+SU, NAD+BY
- Per line: LIN or PIA (ISBN/EAN), IMD+050 (title), QTY
- UNS, UNT, UNZ

**ORDRSP (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, NAD+SU, NAD+BY
- Per line: LIN with action code, RFF+LI (Koha order number)
- UNS, UNT, UNZ

**INVOIC (Supplier &rarr; Koha):**
- UNA, UNB, UNH, BGM, DTM+137, NAD+SU, NAD+BY
- Per line: LIN, QTY+47, RFF+LI (Koha order number), at least one MOA or PRI segment
- UNS, UNT, UNZ

### Plugin Extension Points

Koha supports vendor-specific plugins that can customize message handling:

| Plugin Method | Purpose |
|--------------|---------|
| `edifact_order` | Replace the standard ORDERS message generator |
| `edifact_transport` | Replace the standard file transport |
| `edifact` | Replace the standard EDIFACT parser for incoming messages |
| `edifact_process_invoice` | Replace the standard invoice processing logic |

If a supplier's message format deviates from the standard described in this document, a Koha plugin can be developed to handle the differences.

---

*This document was generated from the Koha source code (version 25.11.x). For the most current information, consult the [Koha community manual](https://koha-community.org/manual/) and the source files in `Koha/Edifact/` and `Koha/EDI.pm`.*
