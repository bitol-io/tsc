# Drop datasetName

champion: jgp, simon

Keep ODCS small and free from vendor specific infos. 
`datasetName` is GCP specific. 
The documentation says so. 
It is confusing with the `dataset` field that is not GCP specific.

## Decision

Remove `datasetName` in ODCS v3.

## Consequences

- Breaking change
- Migration path: move to custom field (could be supported by an automated migration tool)

### Alternatives

- Rename to `gcpDatasetName`.
 
