## 0.3.0 (Unreleased)

FEATURES:

* Add optional `consumer.sub_id` record identities and compact, shard-ready search indexes.
* Add a dry-run-first migration command for replacing legacy reverse records.
* Add an index-only backfill command and repair missing search references during record updates.
* Add a dry-run-first command for removing legacy search-index items.

## 0.1.0 (Unreleased)

FEATURES:

* Initial usageregistry provider with `usageregistry_record`.
* Add optional `id_regex` constraints and custom `id_regex_error_message` diagnostics to target and consumer types.
