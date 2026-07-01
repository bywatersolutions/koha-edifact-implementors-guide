# Koha EDIFACT Enhanced Plugin Guide

This document describes the **EDIFACT Enhanced Plugin** for Koha, developed by ByWater Solutions. The plugin extends Koha's built-in EDIFACT implementation with additional configuration options for order generation, invoice processing, and message parsing. It is designed for libraries that need vendor-specific customizations beyond what standard Koha provides.

For the standard Koha EDIFACT specification, see the companion document [EDI Supplier Guide](EDI_Supplier_Guide.md).

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Installation and Requirements](#2-installation-and-requirements)
- [3. How the Plugin Integrates with Koha](#3-how-the-plugin-integrates-with-koha)
- [4. Configuration Reference](#4-configuration-reference)
  - [4.1 Buyer Identification](#41-buyer-identification)
  - [4.2 File Suffixes](#42-file-suffixes)
  - [4.3 LIN Segment — Line Item Identification](#43-lin-segment--line-item-identification)
  - [4.4 PIA Segments — Additional Product Identifiers](#44-pia-segments--additional-product-identifiers)
  - [4.5 GIR Segments — Copy-Level Data](#45-gir-segments--copy-level-data)
  - [4.6 Other Order Options](#46-other-order-options)
  - [4.7 Invoice Processing Options](#47-invoice-processing-options)
  - [4.8 Shipment Charge Configuration](#48-shipment-charge-configuration)
  - [4.9 Item Receipt Options](#49-item-receipt-options)
- [5. Changes from Standard Koha Behavior](#5-changes-from-standard-koha-behavior)
  - [5.1 Order Message (ORDERS) Differences](#51-order-message-orders-differences)
  - [5.2 Incoming Message Parsing Differences](#52-incoming-message-parsing-differences)
  - [5.3 Invoice Processing Differences](#53-invoice-processing-differences)
  - [5.4 Item Receipt Differences](#54-item-receipt-differences)
- [6. Vendor-Specific Features](#6-vendor-specific-features)
- [7. YAML Configuration Examples](#7-yaml-configuration-examples)
- [8. Troubleshooting](#8-troubleshooting)

---

## 1. Overview

The EDIFACT Enhanced Plugin replaces four key components of Koha's EDI pipeline when enabled on a vendor EDI account:

| Component | Standard Koha | Plugin Replacement |
|-----------|--------------|-------------------|
| Order generation | `Koha::Edifact::Order` | `EdifactEnhanced::Edifact::Order` |
| EDIFACT parsing | `Koha::Edifact` | `EdifactEnhanced::Edifact` |
| File transport | `Koha::Edifact::Transport` | `EdifactEnhanced::Edifact::Transport` |
| Invoice processing | `Koha::EDI::process_invoice` | `EdifactEnhanced::edifact_process_invoice` |

The plugin does **not** create any custom database tables. All settings are stored in Koha's standard `plugin_data` table.

---

## 2. Installation and Requirements

**Minimum Koha version:** 22.05.06

**Required Perl module:**
```
Business::Barcode::EAN13
```

This module is used for EAN-13 validation when resolving product identifiers for the LIN segment.

**Installation steps:**
1. Install the plugin via the Koha plugin manager (upload the `.kpz` file)
2. Navigate to the plugin's configuration page to set options
3. On each vendor EDI account that should use the plugin, set the **Plugin** field to `Koha::Plugin::Com::ByWaterSolutions::EdifactEnhanced`

The plugin is activated **per vendor EDI account**, not globally. Different vendors can use different configurations or no plugin at all.

---

## 3. How the Plugin Integrates with Koha

When a vendor EDI account has this plugin configured, Koha's standard EDI code checks for plugin methods at each stage of processing:

1. **Sending orders:** Koha calls the plugin's `edifact_order()` method instead of creating a standard `Koha::Edifact::Order` object.
2. **Downloading files:** Koha calls the plugin's `edifact_transport()` method instead of using the standard transport.
3. **Parsing incoming messages:** Koha calls the plugin's `edifact()` method instead of the standard parser.
4. **Processing invoices:** Koha calls the plugin's `edifact_process_invoice()` method, which **completely replaces** the standard `Koha::EDI::process_invoice` pipeline.

Quote and order response processing continue to use the standard Koha pipeline, but use the plugin's parser for reading the EDIFACT data.

---

## 4. Configuration Reference

All settings are accessible through the plugin's configuration page in the Koha staff interface (**Plugins > EDIFACT Enhanced > Configure**). Changes are logged to the Koha action log under the `EDIFACT` module with the `SETTINGS_UPDATED` action.

### 4.1 Buyer Identification

These settings control how the library identifies itself in outgoing ORDERS messages. Standard Koha always sends the Library EAN in both the UNB header and the NAD+BY segment. The plugin allows substituting or supplementing this with a SAN or other identifier.

#### UNB Interchange Header (Sender Identity)

| Setting | Description |
|---------|-------------|
| `buyer_san` | A static SAN value to send as the buyer identifier |
| `buyer_id_code_qualifier` | The qualifier for the buyer identifier: `14` (EAN), `31B` (US SAN), `91` (supplier-assigned), `92` (buyer-assigned) |
| `buyer_san_in_header` | Enable sending the buyer SAN in the UNB interchange header |
| `branch_ean_in_header` | Enable sending the Library EAN in the UNB header (standard behavior) |

When `buyer_san_in_header` is enabled, the plugin offers three alternative sources for the SAN value (checked in this priority order):

| Setting | Behavior |
|---------|----------|
| `buyer_san_extract_from_library_ean_description` | Extracts a SAN from the Library EAN's description field using the pattern `SAN:{value}` |
| `buyer_san_use_username` | Uses the vendor EDI account's username as the buyer identifier |
| `buyer_san_use_library_ean_split_first_part` | Splits the Library EAN on spaces and uses only the first part |

If none of these alternative sources are enabled, the static `buyer_san` value is used.

#### NAD+BY Segment (Buyer Name and Address)

| Setting | Description |
|---------|-------------|
| `buyer_san_in_nadby` | Include a NAD+BY segment with the buyer SAN |
| `branch_ean_in_nadby` | Include a NAD+BY segment with the Library EAN |

These are independent — you can have zero, one, or two NAD+BY segments. Standard Koha always sends exactly one.

**Example:** A library with SAN `1234567` and EAN `9876543210123` could produce:

```edifact
UNB+UNOC:3+1234567:31B+...
...
NAD+BY+1234567::31B'
NAD+BY+9876543210123::9'
```

### 4.2 File Suffixes

| Setting | Description | Default |
|---------|-------------|---------|
| `order_file_suffix` | File extension appended to outgoing order files | None (standard Koha uses `.CEP`) |
| `invoice_file_suffix` | File extension for matching incoming invoice files during download | None (standard Koha uses `.CEI`) |

If `order_file_suffix` is set to e.g. `edi`, outgoing files will be named `ordr{basketno}.edi`. If left blank, no extension is appended.

### 4.3 LIN Segment — Line Item Identification

The LIN segment identifies the product being ordered. Standard Koha uses only the order line's `line_item_id` field with a hardcoded `EN` (EAN-13) qualifier. The plugin provides a configurable priority cascade — the **first** source that produces a valid value is used.

#### Priority Order

| Priority | Setting | Source | Qualifier |
|----------|---------|--------|-----------|
| 1 | `lin_use_item_field` | Any column from the Koha `items` table | Configurable via `lin_use_item_field_qualifier` |
| 2 | *(default)* | `line_item_id` from the order line | `EN` |
| 3 | `lin_use_ean` | EAN from biblioitem data, validated with `Business::Barcode::EAN13` | `EN` |
| 4 | `lin_use_issn` | ISSN from biblioitem data | `IS` |
| 5 | `lin_use_isbn` | ISBN from biblioitem data (see ISBN options below) | `EN` |
| 6 | `lin_use_upc` | UPC from MARC field 024$a | `UP` |
| 7 | `lin_use_product_id` | Product ID from MARC field 028$a | `PI` |

#### ISBN Resolution Options

When `lin_use_isbn` is enabled, the plugin applies additional logic:

| Setting | Behavior |
|---------|----------|
| `lin_force_first_isbn` | Use only the first ISBN found (ignores others) |
| `lin_use_invalid_isbn13` | Accept any 13-character string as an ISBN-13, even if the check digit is invalid. Useful for vendors like Baker & Taylor that use proprietary 13-digit identifiers. |
| `lin_use_invalid_isbn_any` | Accept any ISBN string regardless of length or validity |

When multiple ISBNs exist, the plugin prefers a true ISBN-13 over an ISBN-10 converted to ISBN-13.

#### Item Field Side Effect

When `lin_use_item_field` is used and the value is successfully placed in the LIN segment, the plugin also injects that value into the MARC record's 020 (ISBN) field if it is not already present, and saves the record via `ModBiblio`. This ensures the identifier is searchable in the catalog.

### 4.4 PIA Segments — Additional Product Identifiers

Standard Koha generates at most one PIA segment per line item. The plugin can generate multiple PIA segments with different identifier types.

| Setting | Identifier Source | Qualifier |
|---------|------------------|-----------|
| `pia_send_lin` | Duplicate of the LIN value | Same as LIN |
| `pia_use_ean` | EAN from biblioitem | `EN` |
| `pia_use_issn` | ISSN from biblioitem | `IS` |
| `pia_use_isbn10` | All ISBN-10s from the record | `IB` |
| `pia_use_isbn13` | All ISBN-13s from the record | `EN` |
| `pia_use_upc` | UPC from MARC 024$a | `UP` |
| `pia_use_product_id` | Product ID from MARC 028$a | `PI` |

| Setting | Description | Default |
|---------|-------------|---------|
| `pia_limit` | Maximum number of PIA segments per order line | 25 |

These options are **not** mutually exclusive — enabling multiple options generates multiple PIA segments. The first PIA uses function code `5` (product identification) if no LIN identifier was found; subsequent PIAs use function code `1` (additional identification).

### 4.5 GIR Segments — Copy-Level Data

The GIR (Related Identification Numbers) segment carries item-level details such as branch, fund, and item type. The plugin provides three levels of customization.

#### Disabling GIR Entirely

| Setting | Description |
|---------|-------------|
| `gir_disable` | Globally disables all GIR segment generation |

You can also disable GIR per Library EAN by adding `NO_GIR:{True}` to the Library EAN's description field. This is useful when the same plugin configuration serves multiple accounts but only some require GIR data.

#### Custom GIR Mapping

| Setting | Description |
|---------|-------------|
| `gir_mapping` | YAML map that replaces the default GIR field mapping |

The default Koha GIR mapping is:

| Qualifier | Data Source |
|-----------|------------|
| `LLO` | `homebranch` (items table) |
| `LFN` | `budget_code` (from aqbudgets) |
| `LST` | `itype` (items table) |
| `LSQ` | Item field from `EdifactLSQ` syspref |
| `LSM` | `itemcallnumber` (items table) |
| `LVT` | `servicing_instruction` (vendor note) |

The custom mapping can reference:

| Value Type | Syntax | Example |
|------------|--------|---------|
| Items table column | `column_name` | `homebranch`, `itype`, `location` |
| Budget code | `budget_code` | `budget_code` |
| Vendor note | `servicing_instruction` | `servicing_instruction` |
| Any aqorders column | `aqorders.column` | `aqorders.sort1` |
| Literal string | `\value` | `\BOOK` |
| MARC field/subfield | `{field}${subfield}` | `082$a` |

Example YAML:
```yaml
LLO: homebranch
LFN: budget_code
LST: itype
LSQ: location
LSM: 082$a
```

#### GIR Value Replacements

| Setting | Description |
|---------|-------------|
| `gir_value_replacements_map` | YAML map for value substitution within GIR data |

This allows translating Koha internal codes to vendor-expected codes. Structure:

```yaml
GIR_QUALIFIER:
  koha_value: vendor_value
```

Example — translate branch codes:
```yaml
LLO:
  MAIN: 001
  BRANCH1: 002
  BRANCH2: 003
```

#### GIR Segment Splitting

| Setting | Description | Default |
|---------|-------------|---------|
| `split_gir` | Number of qualifier+value pairs per GIR segment line | 5 (standard Koha behavior) |

Standard EDIFACT limits a GIR segment to 5 data pairs. If a copy needs more, continuation GIR segments are emitted with the same copy sequence number. Setting this to `0` puts all data in a single GIR segment (no splitting). Values 1–20 are supported.

### 4.6 Other Order Options

| Setting | Description |
|---------|-------------|
| `send_basketname` | Send the basket name instead of the basket number in the BGM segment. Useful when the basket name carries a meaningful purchase order number. |
| `send_rff_bfn` | Include an `RFF+BFN:{budget_code}` segment per order line (the budget/fund code). Not generated by standard Koha. |
| `send_rff_bfn_biblionumber` | Include an `RFF+BFN:{biblionumber}` segment per order line (the Koha bibliographic record number). Not generated by standard Koha. |

### 4.7 Invoice Processing Options

| Setting | Description |
|---------|-------------|
| `set_bookseller_from_order_basket` | Determine the vendor from the order's basket rather than the EDI message's vendor ID. Useful when multiple vendor EDI accounts point to the same physical vendor. |
| `skip_nonmatching_san_suffix` | If the supplier SAN in the EDIFACT message does not match the vendor EDI account's SAN, skip processing and attempt to re-route the message to the correct account. |
| `ignore_duplicate_reciepts` | Skip order lines that are already fully received (`quantity == quantityreceived`) instead of attempting to re-receipt them. |
| `update_pricing_from_vendor_settings` | After setting prices from invoice data, recalculate using the vendor's tax configuration. Uses `populate_with_prices_for_ordering()` and `populate_with_prices_for_receiving()`. |
| `close_invoice_on_receipt` | Automatically close (set `closedate`) on the invoice when processing completes. |

### 4.8 Shipment Charge Configuration

Standard Koha sums all MOA (monetary amount) segments found in message-level ALC+C (charge) segments before the first LIN line item. The plugin provides granular control over which amounts are included in shipping costs.

| Setting | Description |
|---------|-------------|
| `shipment_charges_alc_dl` | Include ALC+DL (delivery charge) amounts. Standard Koha always includes these. |
| `shipment_charges_moa_8` | Include MOA qualifier 8 (allowance/charge amount — often used for value-added services like barcoding and lamination) |
| `shipment_charges_moa_79` | Include MOA qualifier 79 (total line items amount) |
| `shipment_charges_moa_124` | Include MOA qualifier 124 (tax amount) |
| `shipment_charges_moa_131` | Include MOA qualifier 131 |
| `shipment_charges_moa_304` | Include MOA qualifier 304 |
| `add_tax_to_shipping_costs` | Add the vendor's tax rate to the calculated shipping charge |
| `shipping_budget_id` | Static budget ID for shipping costs (overrides the vendor EDI account's `shipment_budget` setting) |
| `ship_budget_from_orderline` | Use the budget from the last processed order line as the shipping cost budget |

**Note:** Unlike standard Koha, the plugin scans the **entire** message for charge segments, not just those before the first LIN segment.

### 4.9 Item Receipt Options

These settings control how items are updated when they are received via an EDIFACT invoice.

| Setting | Values | Description |
|---------|--------|-------------|
| `no_update_item_price` | `update_both` (default), `update_neither`, `update_price`, `update_replacementprice` | Controls which price fields are updated on the item record when received |
| `set_nfl_on_receipt` | Authorized value from NOT_LOAN | Set the item's `notforloan` status to a specific value upon receipt |
| `add_itemnote_on_receipt` | boolean | Add "Received via EDIFACT" to the item's non-public notes field |
| `lin_use_item_field_clear_on_invoice` | boolean | Clear the item field that was used for the LIN identifier (from `lin_use_item_field`) when the invoice is processed |

---

## 5. Changes from Standard Koha Behavior

### 5.1 Order Message (ORDERS) Differences

| Feature | Standard Koha | Plugin |
|---------|--------------|--------|
| **File extension** | Always `.CEP` | Configurable via `order_file_suffix`, or no extension |
| **UNB sender ID** | Always Library EAN | Configurable: SAN, username, partial EAN, or extracted from description |
| **NAD+BY segments** | Exactly one (Library EAN) | Zero, one, or two (SAN and/or EAN independently) |
| **BGM document number** | Basket number (zero-padded to 11 digits) or basket name with `po_is_basketname` | Basket number or basket name via `send_basketname` |
| **LIN identifier** | `line_item_id` only, qualifier `EN` | Configurable priority cascade across 7 sources with per-source qualifiers |
| **PIA segments** | At most one, single ISBN | Multiple segments, multiple identifier types, configurable limit |
| **GIR mapping** | Fixed field mapping | Fully customizable via YAML, with value replacement |
| **GIR splitting** | Hardcoded at 5 pairs | Configurable 0–20 |
| **GIR disabling** | Not possible | Global toggle or per-account via EAN description |
| **FTX segments** | Sends vendor notes as `FTX+LIN` | Does **not** generate FTX segments |
| **RFF+BFN** | Not generated | Optional: budget code or biblionumber |
| **Smart quote handling** | Not handled | Converts Unicode right single quotes (U+2019) to ASCII apostrophes before escaping |

### 5.2 Incoming Message Parsing Differences

| Feature | Standard Koha | Plugin |
|---------|--------------|--------|
| **Missing UNA segment** | Fatal error (croak) | Warning only (carp), attempts to parse anyway |
| **Line breaks in messages** | Not handled | Strips leading whitespace and non-printable characters from segments |
| **Non-printable characters** | Preserved in element values | Stripped from all element values at access time |

These parsing improvements address a known issue where some vendors insert line break characters every 80 characters in their EDIFACT output. **If you are experiencing parse errors with vendor messages, ask the vendor to disable line-break insertion.** The plugin handles this gracefully, but clean messages are preferred.

### 5.3 Invoice Processing Differences

The plugin **completely replaces** standard Koha invoice processing. Key behavioral changes:

| Feature | Standard Koha | Plugin |
|---------|--------------|--------|
| **Duplicate invoices** | Creates a new invoice each time | Searches for existing invoice by number+vendor, updates if found |
| **Vendor routing** | Uses EDI message vendor ID | Can re-route based on SAN matching or order basket vendor |
| **Cross-vendor matching** | Rejects if order vendor ≠ invoice vendor | Allows if both vendor accounts use the same plugin class |
| **Tax on shipping** | Not calculated | Optional: adds vendor tax rate to shipping charge |
| **Shipping budget** | From vendor EDI account setting | Configurable: static ID, from order line, or vendor account setting |
| **Standing orders** | Not specifically handled | Creates partial receipt keeping `quantity_remaining = 1` to keep order open |
| **Duplicate receipts** | May fail or double-receive | Optional: skips already-received lines |
| **Price recalculation** | Uses raw invoice values | Optional: recalculates using vendor tax settings |
| **Invoice closing** | Manual | Optional: auto-close on receipt |

### 5.4 Item Receipt Differences

| Feature | Standard Koha | Plugin |
|---------|--------------|--------|
| **Date accessioned** | Set on receipt | Set on receipt |
| **Bookseller ID** | Not set on item | Set to the vendor's ID (source of acquisition) |
| **Price fields** | Both `price` and `replacementprice` updated | Configurable: update both, neither, or individually |
| **Not-for-loan status** | Not changed | Optional: set to a specific authorized value |
| **Item notes** | Not changed | Optional: add "Received via EDIFACT" note |
| **LIN field clearing** | Not applicable | Optional: clear the item field used as LIN identifier |

---

## 6. Vendor-Specific Features

### Baker & Taylor

Baker & Taylor uses proprietary 13-digit identifiers that are not valid ISBN-13s (they fail check-digit validation). Two settings were specifically designed for this:

- **`lin_use_invalid_isbn13`**: Accepts any 13-character string as an ISBN-13 identifier for the LIN segment, regardless of check digit validity.
- **`lin_use_invalid_isbn_any`**: Ultimate fallback that accepts any ISBN string regardless of format.

### Accounts Without GIR Requirements

Some vendors or account types (e.g., Baker & Taylor unprocessed accounts) do not need copy-level GIR data. Rather than creating a separate plugin configuration, add `NO_GIR:{True}` to the Library EAN's description field for that account.

### Multi-Account / Multi-Vendor Setups

Several features support configurations where a single physical vendor is represented by multiple Koha vendor records or EDI accounts:

- **`skip_nonmatching_san_suffix`**: Ensures each vendor EDI account only processes invoices matching its SAN. Mismatched invoices are re-routed to the correct account.
- **`set_bookseller_from_order_basket`**: Determines the vendor from the order's basket rather than the EDI message, ensuring the correct vendor record is used for the invoice.
- **Cross-plugin matching**: During invoice processing, if an order's vendor does not match the invoice's vendor but both vendor EDI accounts use the same plugin class, the receipt is allowed.

---

## 7. YAML Configuration Examples

### Custom GIR Mapping

Map GIR qualifiers to Koha data sources:

```yaml
LLO: homebranch
LFN: budget_code
LST: itype
LSQ: location
LSM: itemcallnumber
LVT: servicing_instruction
```

### GIR Mapping with MARC Fields

Use a MARC field/subfield as the shelfmark:

```yaml
LLO: homebranch
LFN: budget_code
LST: itype
LSM: 082$a
```

### GIR Mapping with Literal Values

Send a fixed value for all items:

```yaml
LLO: homebranch
LFN: budget_code
LST: \BOOK
```

### GIR Mapping with aqorders Fields

Include data from the order record:

```yaml
LLO: homebranch
LFN: budget_code
LST: itype
LSQ: aqorders.sort1
```

### GIR Value Replacements

Translate Koha branch codes to vendor-specific codes:

```yaml
LLO:
  MAIN: "001"
  NORTH: "002"
  SOUTH: "003"
  CHILD: "004"
LST:
  BK: BOOK
  DVD: DVDV
  CD: CDMU
```

### Combined Example

A complete GIR configuration for a vendor that uses numeric branch codes and simplified item types:

**gir_mapping:**
```yaml
LLO: homebranch
LFN: budget_code
LST: itype
LSQ: location
LSM: 082$a
```

**gir_value_replacements_map:**
```yaml
LLO:
  MAIN: "001"
  BRANCH_A: "002"
  BRANCH_B: "003"
LST:
  BK: BOOK
  CF: CDFM
  VM: VDEO
```

**Result:** An order line for branch `MAIN` with item type `BK` would produce:
```edifact
GIR+1+001:LLO+ADULT-NF:LFN+BOOK:LST+REFERENCE:LSQ+823.914:LSM'
```

---

## 8. Troubleshooting

### Vendor Messages Failing to Parse

**Symptom:** Incoming QUOTES, ORDRSP, or INVOIC messages fail with parse errors.

**Likely cause:** The vendor inserts line break characters every 80 characters in their EDIFACT output.

**Solution:** Ask the vendor to disable line-break insertion. The plugin handles this more gracefully than standard Koha (it strips whitespace and non-printable characters), but clean messages are preferred.

### LIN Identifier Not Matching Expected Value

**Symptom:** Orders are sent with unexpected product identifiers in the LIN segment.

**Explanation:** The plugin uses a priority cascade. The first source that returns a valid value wins. Check the priority order in [Section 4.3](#43-lin-segment--line-item-identification) and verify which source is matching first.

**Debugging:** Check the raw EDIFACT message in the Koha EDIFACT messages log (under the Acquisitions module) to see what was actually sent.

### Invoices Not Processing

**Symptom:** Invoice messages stay in "new" status and are not processed.

**Possible causes:**
1. The `EdifactInvoiceImport` system preference is set to `manual` (requires staff to trigger processing).
2. If `skip_nonmatching_san_suffix` is enabled, the supplier SAN in the message may not match the vendor EDI account's SAN.
3. The order lines referenced by `RFF+LI` may not exist or may belong to a different vendor.

### Duplicate Invoices

**Symptom:** The same invoice appears multiple times in Koha.

**Explanation:** The plugin checks for existing invoices by number and vendor before creating a new one, which prevents duplicates from retransmissions. If duplicates still appear, check whether the invoice number or vendor ID differs between transmissions.

### GIR Data Not Appearing

**Symptom:** Outgoing orders do not contain GIR segments.

**Check:**
1. Is `gir_disable` enabled in the plugin configuration?
2. Does the Library EAN description contain `NO_GIR:{True}`?
3. Is a custom `gir_mapping` configured with invalid field names?

### Shipping Charges Not Calculated Correctly

**Symptom:** Invoice shipping charges are zero or incorrect.

**Explanation:** The plugin requires explicit opt-in for each MOA qualifier that should be included in shipping charges. Verify that the appropriate `shipment_charges_*` options are enabled for the qualifiers your vendor uses. Check the raw EDIFACT message to identify which MOA qualifiers the vendor sends at the message level.

### Configuration Changes Not Logged

All configuration changes are recorded in the Koha action log under module `EDIFACT`, action `SETTINGS_UPDATED`. Both old and new values are stored. Check **Tools > Log viewer** filtered to the EDIFACT module.

---

*This document describes the EDIFACT Enhanced Plugin version 4.x for Koha 22.05.06+. For the plugin source code, see the [GitHub repository](https://github.com/bywatersolutions/koha-plugin-edifact-enhanced/). For the standard Koha EDIFACT specification, see [EDI Supplier Guide](EDI_Supplier_Guide.md).*
