# REQ-0448-2 — Lead Processes / Record Types / Lead Status / Lead Path / Convert Button

## Environment limitation (read this first)

This run had **no shell/CLI access** (filesystem tools only — no `bash`, no `sf`, no org
connection). Therefore:

- The ticket's mandatory "**Check first**" step (current Lead Status values, existing Lead
  Processes, existing Lead Record Types, existing Lead Path, Convert button on the layout)
  **could not be executed**.
- `sf project deploy start --dry-run --test-level NoTestRun` **could not be run**, so nothing
  below is deploy-validated.
- All artifacts are authored from the ticket specification, **not** from observed org state.

## Artifacts produced

- id: lead-status
  path: force-app/main/default/standardValueSets/LeadStatus.standardValueSet-meta.xml
  kind: standard-valueset
- id: lead-process
  path: force-app/main/default/objects/Lead/businessProcesses/Standard_Lead_Process.businessProcess-meta.xml
  kind: business-process
- id: lead-recordtype
  path: force-app/main/default/objects/Lead/recordTypes/Standard_Lead_Process.recordType-meta.xml
  kind: record-type
- id: lead-path
  path: force-app/main/default/pathAssistants/Lead_Status_Path.pathAssistant-meta.xml
  kind: path-assistant

## Wiring (cross-references agree byte-for-byte)

- from: lead-recordtype → to: lead-process
  detail: RecordType `<businessProcess>` = `Standard_Lead_Process`
- from: lead-path → to: lead-recordtype
  detail: PathAssistant `<recordTypeName>` = `Standard_Lead_Process`
- from: lead-path → to: lead-status
  detail: `<fieldName>Status`; step `<picklistValueName>` = Open, Contacted, Qualified, Unqualified, Converted (in that order)
- from: lead-process → to: lead-status
  detail: `<values>` = Open (default), Contacted, Qualified, Unqualified, Converted
- from: lead-recordtype → to: lead-status
  detail: `<picklistValues>` picklist `Status` lists the same five values, Open default

## Ticket step coverage

| Ticket step | Artifact | Status |
|---|---|---|
| Status values Open/Contacted/Qualified/Unqualified/Converted, default Open | lead-status | authored — **org not reconciled** |
| Only one status marked Converted; `Converted` is it | lead-status (`converted=true` on `Converted`, `converted=false` on `Closed - Converted`) | authored — **could not confirm** the org's current converted status |
| Lead Process with the five values in order, ending Converted | lead-process | authored |
| Record Type: use existing if present, else one standard type | lead-recordtype | **could not check** — created `Standard_Lead_Process` assuming none exists beyond Master |
| Lead Path with the five stages | lead-path | authored, bound to `Standard_Lead_Process` |
| Convert button on the Lead page layout | none | **could not check** — needs layout retrieve; see below |

### Two deliberate risk decisions

1. **Legacy status values retained.** `StandardValueSet` deploys as a *full replace* of the
   value set, so any value omitted from the file is removed. Because the ticket says
   "don't remove others in use" and the org could not be inspected, the classic defaults
   `Open - Not Contacted`, `Working - Contacted`, `Closed - Converted`,
   `Closed - Not Converted` are retained alongside the five required values. Any *custom*
   lead status values in use in the org are **not** in this file and would need to be added
   before deploy (query listed below).
2. **`converted=false` set explicitly on `Closed - Converted`.** If the org currently marks
   that value as the converted status, this flips it to `Converted` — which is what the ticket
   asks for, but it is a change to existing configuration that must be eyeballed first.
3. **`<converted>` element name is unverified.** The converted-status flag on a
   `StandardValue` was written as `<converted>true</converted>` because the ticket requires
   exactly one status be marked Converted. This could not be confirmed against a retrieved
   sample. If the dry-run reports it as invalid for the type, delete the `<converted>` lines
   and instead set the converted status once in the target org's UI (Lead Settings) — or
   retrieve the org's own `LeadStatus` value set and mirror its element name.

## Convert button (intentionally not changed)

`Convert` is a standard Lead button and shows unless the Lead page layout lists it in
`<excludeButtons>`. No Lead layout exists in this repo, and a layout `-meta.xml` is a
whole-file replace — deploying a hand-written partial layout would wipe the org's existing
sections/fields. So no layout was authored. To check:

    sf project retrieve start --metadata "Layout:Lead-Lead Layout" --output-dir ./orgmeta

Then look for `<excludeButtons>Convert</excludeButtons>`. Only if present should it be removed
from the *retrieved* (complete) layout and that file deployed.

## Check-first / reconciliation commands (run in a CLI+org-enabled session)

    sf org list --json
    sf org list metadata --metadata-type BusinessProcess --json
    sf org list metadata --metadata-type PathAssistant --json
    sf project retrieve start --metadata "StandardValueSet:LeadStatus" \
        --metadata "BusinessProcess" --metadata "PathAssistant" \
        --metadata "Layout:Lead-Lead Layout" --output-dir ./orgmeta
    sf data query -q "SELECT Id, DeveloperName, Name, SobjectType, IsActive, BusinessProcessId \
        FROM RecordType WHERE SobjectType='Lead'" --use-tooling-api
    sf data query -q "SELECT Status, COUNT(Id) FROM Lead GROUP BY Status"   # values actually in use

Reconciliation rules:
- If a Lead **Record Type** already exists beyond Master, delete `lead-recordtype` and repoint
  `lead-path`'s `<recordTypeName>` at the existing record type (ticket: reuse existing).
- If a Lead **Business Process** already exists, delete `lead-process` and point the record
  type's `<businessProcess>` at the existing one.
- If a Lead **Path** already exists, delete `lead-path` and update the existing one (one Path
  per record type/field).
- Add any custom Lead Status values found in use to `lead-status` before deploying.

## Verification performed

**No org verification.** `sf project deploy start --dry-run --test-level NoTestRun` could not
be run — no CLI/org in this environment. Static self-check only: all four files are well-formed
XML, placed at the correct SFDX source path for their type, and the cross-references
(`businessProcess`, `recordTypeName`, `fieldName`, step `picklistValueName`s) agree
byte-for-byte across files.
