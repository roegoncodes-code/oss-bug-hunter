# Netty: a truncated gzip body breaks the next HTTP response on the connection

**Project:** netty/netty, branch 4.2, at `0945590e0bf92061ce8ad955fe80ce907d805463`
**Area:** `codec-http`, `HttpContentDecoder`
**Status:** verified on GitHub Actions — [run 44483855527](https://github.com/roegoncodes-code/oss-bug-hunter/actions/runs/44483855527)

## Problem

A response is correctly framed (chunked with a final `0\r\n\r\n`, or `Content-Length`), but its gzip body is truncated: the 8-byte CRC32/ISIZE footer is missing. Since upstream commit 0945590 ("Fail JdkZlibDecoder when the input ends inside a GZIP or ZLIB stream", #18445), `JdkZlibDecoder` throws `DecompressionException` for such input.

In `HttpContentDecoder`, that exception is thrown by `finishedDecoder.finish()` before the `LastHttpContent` is passed on. So the handlers after the decompressor never see the end of the message. With `HttpObjectAggregator` in the pipeline, the aggregator keeps the unfinished message, and the **next, valid response on the same keep-alive connection** fails with `MessageAggregationException` and is dropped.

This contradicts the goal stated in 0945590's commit message, that a message is not dropped because the body of the previous message was incomplete. The 4.1 port of that change (#18471, open) would carry the same problem to 4.1.

## Root cause

`HttpContentDecoder.decode`, `LastHttpContent` branch: `finish()` can now throw, and nothing after it runs, including the `fireChannelRead` of the last content.

## Solutions compared

1. **Always pass on a failed `LastHttpContent` and swallow the exception.** Fixes the next message, but breaks upstream's deliberate behaviour for bodies delimited by the end of the connection: `testTruncatedGzipResponseFails` asserts that such a truncated body is *not* completed. CI showed this conflict, so this option was rejected.
2. **Make `HttpObjectAggregator` tolerate a missing end of message.** Wrong layer; other handlers have the same problem.
3. **Chosen:** if the connection is already inactive (body delimited by the end of the connection), keep upstream's behaviour and rethrow without completing the message. If the connection is still active (the message itself was complete), pass on the end of the message with a failed `DecoderResult` (a framing failure wins over the decompression failure), then rethrow, so `exceptionCaught` still fires as before.

## The fix

See `fix.patch`: about 20 lines in `HttpContentDecoder.java`. Regression test in `test.patch`: `HttpContentDecoderTest.testTruncatedGzipChunkedResponseDoesNotBreakNextResponse` (decoder → decompressor → aggregator; a truncated chunked gzip response, then a valid `Content-Length: 2` response that must arrive intact).

## Proof

- **Without the fix:** the new test fails with `DecoderException: MessageAggregationException`, exactly the predicted failure.
- **With the fix:** the new test passes, and `HttpContentDecoderTest` (28), `HttpObjectAggregatorTest` (26) and `HttpContentDecompressorTest` (8) all pass: 62 tests.

## Behaviour change to call out upstream

For a framed message with a truncated compressed body, the handlers after the decompressor now receive a `LastHttpContent` whose `decoderResult()` is a failure. Before, they received nothing. Netty's AGENTS.md asks for such changes to be called out in the PR description.

## How to submit upstream

1. Netty requires a signed CLA, and asks that discussion with maintainers is in your own words; use `.github/PULL_REQUEST_TEMPLATE.md`. Setup guide: https://netty.io/wiki/developer-guide.html
2. Fork netty/netty, branch from `4.2`, apply `test.patch` then `fix.patch`.
3. Run `./mvnw verify -DskipTests` (checkstyle and revapi; not run in this hunt's CI) and the three test classes above.
4. Open the PR against `4.2`; mention #18445 and the open 4.1 port #18471.

Review this change yourself before submitting: you are accountable for it.
