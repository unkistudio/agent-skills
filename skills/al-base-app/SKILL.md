---
name: al-base-app
description: Look up pages, actions, action categories, fields, and other objects from the Microsoft BC Base Application .app package. Use this skill when working on AL extensions that target standard BC pages or objects and you need to verify action group names, category names, field names, or page structure — for example to correctly hide or extend print/send groups.
---

The Base Application .app package is a NAVX-wrapped ZIP. Use the workflow below to extract and inspect it without any external tools.

## Package Location

The `.alpackages` folder is typically at the workspace root. Use the most recent Base Application version:

```powershell
Get-ChildItem "<workspace>\.alpackages" -Filter "Microsoft_Base Application_*.app" |
    Sort-Object Name -Descending |
    Select-Object -First 1 -ExpandProperty FullName
```

## Extraction Workflow

The NAVX format is a ZIP starting at byte offset 40. Extract with pure PowerShell:

```powershell
$appPath = "<path to .app>"
$bytes   = [System.IO.File]::ReadAllBytes($appPath)
$zipPath = "$env:TEMP\BaseApp_inspect.zip"
[System.IO.File]::WriteAllBytes($zipPath, $bytes[40..($bytes.Length - 1)])
```

## Reading a Specific File

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$archive = [System.IO.Compression.ZipFile]::OpenRead($zipPath)

# List entries matching a pattern
$archive.Entries | Where-Object { $_.Name -like "*PostedSalesCreditMemo*" } | Select-Object FullName

# Read a specific entry
$entry  = $archive.Entries | Where-Object { $_.FullName -eq "src/Sales/History/PostedSalesCreditMemo.Page.al" }
$reader = New-Object System.IO.StreamReader($entry.Open())
$content = $reader.ReadToEnd()
$reader.Dispose()
$archive.Dispose()
```

## Common Inspection Queries

### Find all action category group names
```powershell
$content -split "`n" | Select-String "group\(Category_" 
```

### Find the promoted Print/Send group
```powershell
$lines = $content -split "`n"
for ($i = 0; $i -lt $lines.Length; $i++) {
    if ($lines[$i] -match "Print/Send") {
        # Print surrounding context
        $lines[($i-5)..($i+5)] | ForEach-Object -Begin { $n=$i-4 } { "${n}: $_"; $n++ }
    }
}
```

### Show lines around a keyword with context
```powershell
$lines = $content -split "`n"
for ($i = 0; $i -lt $lines.Length; $i++) {
    if ($lines[$i] -match "Category_Category7|Print|Send") {
        Write-Host "$($i+1): $($lines[$i])"
    }
}
```

### Show a line range
```powershell
$lines[1204..1234] | ForEach-Object -Begin { $i = 1205 } { "${i}: $_"; $i++ }
```

## Common Base App Source Paths

| Object | Page ID | Path |
|---|---|---|
| Posted Sales Invoice (card) | 132 | `src/Sales/History/PostedSalesInvoice.Page.al` |
| Posted Sales Invoices (list) | 143 | `src/Sales/History/PostedSalesInvoices.Page.al` |
| Posted Sales Credit Memo | 134 | `src/Sales/History/PostedSalesCreditMemo.Page.al` |
| Sales Order | 42 | `src/Sales/Document/SalesOrder.Page.al` |
| Sales Invoice | 43 | `src/Sales/Document/SalesInvoice.Page.al` |
| Sales Quote | 41 | `src/Sales/Document/SalesQuote.Page.al` |
| Purchase Order | 50 | `src/Purchases/Document/PurchaseOrder.Page.al` |
| Posted Purchase Invoice | 138 | `src/Purchases/History/PostedPurchaseInvoice.Page.al` |

Paths follow the pattern `src/<Module>/<Subfolder>/<ObjectName>.<Type>.al`.  
Use `$archive.Entries | Where-Object { $_.Name -like "*<keyword>*" }` when unsure of the path.

## Key Rules for Page Extensions

### Category Group Names

Always verify the **exact** `group(Category_...)` name before using `modify(...)` or `addbefore(...)`/`addafter(...)` in a pageextension. Different pages use different category numbers for the same logical group (e.g. Print/Send is `Category_Category6` on page 132 but `Category_Category7` on page 134).

### Object IDs and Names

Object IDs must fall within the ranges defined in `app.json`'s `idRanges`. Always check `app.json` before picking a new ID — never assume a range extends further than declared. If the range is exhausted, bump the upper bound in `app.json`.

Object names (the second parameter after the ID) must not exceed **30 characters**.

### Layout: `moveafter` / `movebefore` Syntax

`moveafter` and `movebefore` are **bare statements** in the `layout` section of a page extension. They must NOT end with a trailing semicolon — adding one causes `AL0104: Syntax error, '}' expected`.

```al
// CORRECT
moveafter("Order No."; "Due Date")

// WRONG — trailing semicolon breaks compilation
moveafter("Order No."; "Due Date");
```

Control names with dots, spaces, or special characters must be quoted. Simple identifiers can be unquoted:

```al
moveafter(Active; Name)                          // unquoted identifiers
moveafter("CurrSalesCycleStage"; "Contact No.")  // quoted (spaces/dots)
```

### `moveafter`/`movebefore` on List Page Repeaters

`moveafter` and `movebefore` can be **unreliable** for repositioning existing fields in a list page repeater. If the fields end up at the wrong position (often at the end of the repeater), do NOT use `moveafter`/`movebefore`. Instead use this reliable `addafter` pattern:

1. Use `addafter(<target>)` to add new field controls with **unique** control names (not the same as existing field names, which would cause duplicate errors)
2. Set `ApplicationArea` on each new field so all users see them
3. Use `modify` to set `Visible = false` on the original field controls

```al
pageextension 50100 MyListPageExt extends "Posted Sales Invoices"
{
    layout
    {
        addafter("Sell-to Customer No.")
        {
            field(OrderNo; Rec."Order No.")
            {
                ApplicationArea = Basic, Suite;
            }
            field(ExtDocNo; Rec."External Document No.")
            {
                ApplicationArea = Basic, Suite;
            }
            field(DocDate; Rec."Document Date")
            {
                ApplicationArea = Basic, Suite;
            }
        }

        modify("Order No.")
        {
            Visible = false;
        }

        modify("External Document No.")
        {
            Visible = false;
        }

        modify("Document Date")
        {
            Visible = false;
        }
    }
}
```

### Extracting Full Repeater Field Order

To see all fields in a list page repeater in declaration order:

```powershell
$lines = $content -split '\n'
for ($i = 0; $i -lt $lines.Length; $i++) {
    $l = $lines[$i].Trim()
    if ($l -match '^field\(') { Write-Host "$($i+1): $l" }
}
```
