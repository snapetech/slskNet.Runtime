# slskNet.Runtime Fork Changes

This document tracks protocol and runtime behavior added after the fork from Soulseek.NET `10.0.0`.

## Compatibility Position

The fork is intended to remain compatible with legacy Soulseek clients.

Default behavior preserves the upstream wire behavior. New outbound protocol messages are only sent when an application calls the new APIs. Peer-message obfuscation is disabled by default, and enabling it still requires the regular peer-message port to be advertised.

Compatibility rules used by the implementation:

- Do not replace the regular peer-message listener with an obfuscated-only listener.
- Do not obfuscate file-transfer or distributed-network connections.
- Do not attempt outbound obfuscated peer-message connections unless the remote peer has advertised a compatible type-1 endpoint.
- Keep regular direct and indirect peer-message connection attempts available as fallback paths.
- Fail closed on malformed new protocol responses rather than accepting impossible counts or partial repeated data.

## Type-1 Peer-message Obfuscation

Type-1 obfuscation support covers peer-message streams only. It is not encryption and does not change transfer payload transport.

Implementation pieces:

- `PeerObfuscationOptions` controls runtime behavior.
- `SetListenPortCommand` can append obfuscation type and obfuscated port metadata.
- `UserAddressResponse` and `ConnectToPeerResponse` parse advertised obfuscation metadata.
- `SoulseekClient` can start a dedicated obfuscated peer-message listener.
- `ListenerHandler` decodes the initial obfuscated peer-message frame on obfuscated listeners.
- `MessageConnection` reads and writes obfuscated peer-message frames when the connection is marked obfuscated.
- `PeerConnectionManager` can prefer a cached compatible obfuscated endpoint while racing regular direct and indirect paths.

Expected use:

```csharp
var options = new SoulseekClientOptions(
    listenPort: 2234,
    peerObfuscationOptions: new PeerObfuscationOptions(
        enabled: true,
        listenPort: 2235,
        preferOutbound: true));
```

Legacy impact:

- Legacy peers can still connect to the regular peer-message port.
- If a legacy peer ignores obfuscation metadata, no compatibility issue is introduced.
- If an obfuscated outbound attempt fails, regular direct or indirect connection setup can still succeed.

## Interest and Recommendation APIs

The fork exposes server protocol messages that were defined in Soulseek protocol references but not previously surfaced as client APIs.

Client APIs:

- `AddInterestAsync(string item)`
- `RemoveInterestAsync(string item)`
- `AddHatedInterestAsync(string item)`
- `RemoveHatedInterestAsync(string item)`
- `GetRecommendationsAsync()`
- `GetGlobalRecommendationsAsync()`
- `GetUserInterestsAsync(string username)`
- `GetSimilarUsersAsync()`
- `GetItemRecommendationsAsync(string item)`
- `GetItemSimilarUsersAsync(string item)`

Implementation pieces:

- Outgoing request messages encode the corresponding server message codes.
- Incoming response parsers materialize typed results: `RecommendationList`, `UserInterests`, `SimilarUser`, `ItemRecommendations`, and `ItemSimilarUsers`.
- Waiter keys for user and item responses are case-normalized so a server response with different casing can still complete the request.
- Repeated response counts are validated by `ProtocolCountReader` before allocation and parsing.

Legacy impact:

- These messages are sent only when called by the application.
- Existing search, browse, transfer, room, and private-message flows are unchanged.

## Multi-user Private Messages

The fork exposes the Soulseek `MessageUsers` command:

```csharp
await client.SendPrivateMessageAsync(new[] { "alice", "bob" }, "hello");
```

Implementation details:

- Null, empty, and whitespace-only usernames are rejected.
- Recipients are deduplicated case-insensitively.
- A call is capped at 100 unique recipients to avoid oversized packets and accidental mass sends.
- The single-user `SendPrivateMessageAsync(string, string, CancellationToken?)` path is unchanged.

Legacy impact:

- This command is sent only when the multi-user overload is called.
- The regular single-recipient private-message command remains available and unchanged.

## Room Creation Failure Handling

The fork handles server `CannotCreateRoom` responses.

Implementation details:

- The server response is parsed as the failed room name.
- The pending join-room waiter is completed with a `RoomException`.
- Callers get an explicit failure instead of waiting for timeout behavior.

Legacy impact:

- Join-room request encoding is unchanged.
- This is passive handling of a server response already present in the protocol.

## Parser Hardening

New parsers validate repeated collection counts before reading items.

Validation behavior:

- Negative counts throw `MessageException`.
- Counts that cannot physically fit in the remaining payload throw `MessageException`.
- Successful parses return read-only collections without extra list copies.

This protects the runtime from malformed server responses and avoids silent acceptance of partial or contradictory data.
