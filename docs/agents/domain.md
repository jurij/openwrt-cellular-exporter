# Domain documentation

This is a single-context repository.

Before exploring an area, read the root `GLOSSARY.md` and relevant records
under `docs/adr/`, when they exist. If absent, proceed silently.
Do not create empty documents merely to satisfy the layout.
Domain modeling creates them when terms or decisions are actually resolved.

Use glossary terms consistently in issues, tests and explanations.
Flag conflicts with an existing architecture decision record explicitly;
do not silently override it. Record a changed decision with its rationale.

Expected layout, created as needed:

- `GLOSSARY.md`: shared domain terminology.
- `docs/adr/`: numbered architecture decision records.

Product scope, tested hardware and supported data sources are separate
concepts. A successful fixture test does not establish hardware compatibility.
