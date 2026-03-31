Task 8 Learnings (Appendix)
- Implemented bounds checking for sscanf usage in PdfFile Adaptors.cpp by introducing a width specifier (%6x) to prevent potential buffer overflows when parsing hex values into unsigned types.
- PdfAnnot.cpp already uses width-bounded formats for color components; no changes required there.
- Evidence generated: .sisyphus/evidence/task-8-sscanf-fix.txt
- Next steps: integrate clang-tidy checks and PDF parsing tests in CI; watch for any similar unsafe sscanf usage elsewhere.
