# ABAP Repository

Main workspace for all SAP ECC projects created via Hermes.
Local mirror: `E:\File\abapdownload\Test ABAP Hermes`

## Structure

Each SAP project gets its own folder:

```
<PROJECT_NAME>/
  <program>.txt      ABAP source (copied into SE38)
  notes.md           Purpose, SE38 errors encountered, fixes applied
  shdb/              BDC / batch input recordings (if any)
```

## Conventions

- All ABAP deliverables are `.txt` files for SE38 paste-in.
- Program structure: INCLUDE per section — `<program>_TOP` (TYPES),
  `<program>_CLS` (class DEFINITION + global object DATA + IMPLEMENTATION),
  `<program>_SCR` (selection screen); event blocks in main program.
- Main program header block: Title / Transaction / Author (Yosua Sirait) /
  Requester / Date / Description.
- Classic ABAP syntax only (SAP ECC compatible).
- Commit per milestone, not per edit.
