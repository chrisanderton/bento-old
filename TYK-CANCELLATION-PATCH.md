# One-off cancellation correction

Base: upstream Bento v1.21.1, commit
`008754ec1e4dbd700000314561e2b3e2fddbe6b8`.

This one-off fork corrects a lost cancellation wake-up in `autoretry.List.Shift`.
It contains only synchronization around the existing broadcast, its explanatory
comment, and a public-API regression test. It adds no Tyk product feature,
connector policy, retry setting, acknowledgement change, or timeout.

Upstream issue: https://github.com/warpstreamlabs/bento/issues/1010.

Return to an unmodified released upstream version when it contains the fix and
passes the retained cancellation and Kafka replacement proofs plus affected
build checks. Retain this regression test when removing the temporary dependency
replacement. Review removal at each dependency refresh and before release readiness.
Do not add unrelated patches or treat this as a general extension mechanism.

Pin an exact revision when using this fork. It is not a general extension branch:
custom inputs, outputs and processors belong in their consuming applications.
The upstream license and notices remain unchanged.

Focused validation (Go 1.26.6):

```sh
go test -race ./public/service -run '^TestRetryReadCancellationDuringBackoff$' -count=100 -timeout=1m
go test -race ./internal/autoretry -count=1 -timeout=1m
```
