# One-off cancellation and cleanup corrections

Base: upstream Bento v1.21.1, commit
`008754ec1e4dbd700000314561e2b3e2fddbe6b8`.

This temporary fork contains four generic lifecycle corrections and their
regression tests:

1. Synchronize cancellation notifications in `autoretry.List.Shift` so a reader
   does not lose its cancellation wake-up.
2. Attempt cleanup of built resources when `service.Stream.Run` fails during
   pipeline construction. Preserve the original startup error when cleanup
   succeeds; retain both errors when cleanup fails. A Stream cannot be run again
   after a startup attempt. `Stop` can retry incomplete cleanup.
3. Stop and join InfluxDB's exporter polling loop before final flush/client close.
   Repeated and concurrent `Close` calls perform that cleanup once.
4. Make concurrent CloudWatch `Close` calls join one final upload and retain its
   result, rather than letting a second caller cancel the first upload.

These changes add no Tyk product feature, custom connector, processor, configuration
field, broker acknowledgement policy or timeout. Existing exporter upload-error
policies remain: InfluxDB logs errors and returns nil; CloudWatch retains its
existing flush result for every caller.

The cancellation issue is https://github.com/warpstreamlabs/bento/issues/1010.
That link concerns the cancellation correction, not all cleanup changes.

## Limits

Cleanup can wait for a stalled metrics endpoint. These corrections do not provide
a caller-deadline guarantee. InfluxDB's configured ping timeout does not bound its
HTTP writes. Optional InfluxDB runtime/GC collectors and CloudWatch's periodic
flush loop are not joined by these changes.

The startup correction is not a general fix for every partially constructed
pipeline or component. It reuses Stream's existing resource-cleanup path.

## Return to upstream

Return to an unmodified released upstream version when it includes equivalent
corrections and passes the retained regressions, Kafka replacement proofs and
affected build checks. Retain the regressions when removing the temporary
dependency replacement. Review removal at each dependency refresh and before
release readiness.
Do not add unrelated patches or treat this as a general extension mechanism.

Pin an exact revision when using this fork. It is not a general extension branch:
custom inputs, outputs and processors belong in their consuming applications.
The upstream license and notices remain unchanged.

Focused validation (Go 1.26.6):

```sh
go test -race ./public/service -run '^TestRetryReadCancellationDuringBackoff$' -count=100 -timeout=1m
go test -race ./public/service -run '^TestStream(FailedStartupClosesResources|StopBeforeRunPreservesResources|FailedStartupCleanupCanBeJoinedLater)$' -count=20 -timeout=1m
go test -race ./internal/impl/influxdb ./internal/impl/aws -run '^(TestInfluxClose|TestCloudWatchConcurrentClose|TestCloudWatchCloseRetains)' -count=20 -timeout=45s
go test -race ./internal/autoretry -count=1 -timeout=1m
```

The complete AWS race suite has separate baseline S3 test-fixture races in
`TestS3StreamingWriterTimerFlushLargeEnoughData`,
`TestS3StreamingWriterSlowStreamMultipleParts`, and
`TestS3StreamingWriterExactly5MBTimer`. They reproduce without these corrections
and are not fixed here. Focused exporter checks are not a claim that the full
AWS race suite passes.
