# Majestrix database overrides

Files here override the YAML databases in `db/` and `db/pre-re/`.

Each machine copies the empty templates with `cp -n db/import-tmpl/* db/import/`
(see `doc/majestrix_setup.md`). Those copies are git-ignored. When you
customize one, add a `!/db/import/<file>` line to `.gitignore` so it gets
committed.

No database overrides yet: the baseline uses the stock Pre-Renewal tables,
including the stock level cap.
