Check if documentation is in sync with the actual workflow files.

For each workflow in `workflows/`:
1. Count the actual number of nodes in the JSON
2. Compare with what `docs/BLUEPRINT.md` says
3. Check that every node mentioned in BLUEPRINT.md exists in the JSON (by name)
4. Check that every node in the JSON is documented in BLUEPRINT.md
5. Verify that `CLAUDE.md` Architecture section and Repository Structure are accurate

Report mismatches as a list with specific file:section references.
