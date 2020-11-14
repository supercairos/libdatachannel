# libdatachannel - C API Documentation

### General considerations

Unlike stated otherwise, functions return `RTC_ERR_SUCCESS`, defined as `0`, on success.

All functions can return the following negative error codes:
- `RTC_ERR_INVALID`: an invalid argument was provided
- `RTC_ERR_FAILURE`: a runtime error happened
- `RTC_ERR_NOT_AVAIL`: an element is not available at the moment
- `RTC_ERR_TOO_SMALL`: a user-provided buffer is too small

All functions taking pointers as arguments need the memory to be accessible until the call returns, but it can be safely discarded afterwards.

### Common

#### rtcInitLogger

```
void rtcInitLogger(rtcLogLevel level, rtcLogCallbackFunc cb)
```

Arguments:
- `level`: the log level. It must be one of the following: `RTC_LOG_NONE`, `RTC_LOG_FATAL`, `RTC_LOG_ERROR`, `RTC_LOG_WARNING`, `RTC_LOG_INFO`, `RTC_LOG_DEBUG`, `RTC_LOG_VERBOSE`.
- `cb` (optional): the callback to pass the log lines to. If the callback is already set, it is replaced. If NULL after a callback is set, the callback is unset. If NULL on first call, the library will log to stdout instead.

`cb` must have the following signature:
`void myLogCallback(rtcLogLevel level, const char *message)`

#### rtcPreload

```
void rtcPreload(void)
```

An optional call to `rtcPreload` preloads the global resources used by the library. If it is not called, resources are lazy-loaded when they are required for the first time by a PeerConnection, which for instance prevents from properly timing connection establishment (as the first one will take way more time). The call blocks until preloading is finished. If resources are already loaded, the call has no effect.

#### rtcCleanup

```
void rtcCleanup(void)
```

An optional call to `rtcCleanup` requests unloading of the global resources used by the library. If all created PeerConnections are deleted, unloading will happen immediately and the call will block until unloading is done, otherwise unloading will happen as soon as the last PeerConnection is deleted. If resources are already unloaded, the call has no effect.

#### rtcSetUserPointer

```
void rtcSetUserPointer(int id, void *user_ptr)
```

Sets a opaque user pointer for a Peer Connection, Data Channel, Track, or WebSocket. The user pointer will be passed as last argument in each corresponding callback. It will never be accessed in any way. The initial user pointer of a Peer Connection or WebSocket is `NULL`, and the initial one of a Data Channel or Track is the one of the Peer Connection at the time of creation.

Arguments:
- `id`: the identifier of Peer Connection, Data Channel, Track, or WebSocket
- `user_ptr`: an opaque pointer whose meaning is up to the user

### PeerConnection

#### rtcCreatePeerConnection

```
int rtcCreatePeerConnection(const rtcConfiguration *config)

typedef struct {
	const char **iceServers;
	int iceServersCount;
	uint16_t portRangeBegin;
	uint16_t portRangeEnd;
} rtcConfiguration;
```

Creates a Peer Connection.

Arguments:
- `config`: the configuration structure, containing:
  - `iceServers` (optional): an array of pointers on null-terminated ice server URLs (NULL if unused)
  - `iceServersCount` (optional): number of URLs in the array pointed by `iceServers` (0 if unused)
  - `portRangeBegin` (optional): first port (included) of the allowed local port range (0 if unused)
  - `portRangeEnd` (optional): last port (included) of the allowed local port (0 if unused)

Return value: the identifier of the new Peer Connection or a negative error code.

The Peer Connection must be deleted with `rtcDeletePeerConnection`.

#### rtcDeletePeerConnection

```
int rtcDeletePeerConnection(int pc)
```

Deletes the specified Peer Connection.

Arguments:
- `pc`: the Peer Connection identifier

Return value: `RTC_ERR_SUCCESS` or a negative error code

After this function has been called, `pc` must not be used in a function call anymore. This function will block until all scheduled callbacks of `pc` return (except the one this function might be called in) and no other callback will be called for `pc` after it returns.

#### rtcSetXCallback

These functions set, change, or unset (if `cb` is `NULL`) the different callbacks of a Peer Connection.

```
int rtcSetLocalDescriptionCallback(int pc, rtcDescriptionCallbackFunc cb)*
```

`cb` must have the following signature: `void myDescriptionCallback(int pc, const char *sdp, const char *type, void *user_ptr)`

```
int rtcSetLocalCandidateCallback(int pc, rtcCandidateCallbackFunc cb)
```

`cb` must have the following signature: `void myCandidateCallback(int pc, const char *cand, const char *mid, void *user_ptr)`

```
int rtcSetStateChangeCallback(int pc, rtcStateChangeCallbackFunc cb)
```

`cb` must have the following signature: `void myStateChangeCallback(int pc, rtcState state, void *user_ptr)`

`state` will be one of the following: `RTC_CONNECTING`, `RTC_CONNECTED`, `RTC_DISCONNECTED`, `RTC_FAILED`, or `RTC_CLOSED`.

```
int rtcSetGatheringStateChangeCallback(int pc, rtcGatheringStateCallbackFunc cb)
```

`void myGatheringStateCallback(int pc, rtcGatheringState state, void *user_ptr)`

`state` will be `RTC_GATHERING_INPROGRESS` or `RTC_GATHERING_COMPLETE`.

```
int rtcSetDataChannelCallback(int pc, rtcDataChannelCallbackFunc cb)
```

`cb` must have the following signature: `void myDataChannelCallback(int pc, int dc, void *user_ptr)`

```
int rtcSetTrackCallback(int pc, rtcTrackCallbackFunc cb)
```

`cb` must have the following signature: `void myTrackCallback(int pc, int tr, void *user_ptr)`

#### rtcSetLocalDescription

```
int rtcSetLocalDescription(int pc, const char *type)
```

Initiates the handshake process. Following this call, the local description callback will be called with the local description, which must be sent to the remote peer by the user's method of choice.

Arguments:
- `pc`: the Peer Connection identifier
- `type` (optional): type of the description ("offer", "answer", "pranswer", or "rollback") or NULL for autodetection.

Calling this function is only necessary after `rtcAddDataChannel*` and `rtcAddTrack`. You should not call this function after `rtcCreateDataChannel*`, as it is done internally.

#### rtcSetRemoteDescription

```
int rtcSetRemoteDescription(int pc, const char *sdp, const char *type)
```

Sets the remote description received from the remote peer by the user's method of choice. The remote description may have candidates or not. Following this call, if the remote description is an offer the local description callback will be called with the local answer description.

Arguments:
- `pc`: the Peer Connection identifier
- `type` (optional): type of the description ("offer", "answer", "pranswer", or "rollback") or NULL for autodetection.

#### rtcAddRemoteCandidate

```
int rtcAddRemoteCandidate(int pc, const char *cand, const char *mid)
```

Adds a trickled remote candidate received from the remote peer by the user's method of choice.

Arguments:
- `pc`: the Peer Connection identifier
- `cand`: a null-terminated SDP string representing the candidate (with or without the `"a="` prefix)
- `mid` (optional): a null-terminated string representing the mid of the candidate in the remote SDP description or NULL for autodetection

The Peer Connection must have a remote description set.

Return value: `RTC_ERR_SUCCESS` or a negative error code

#### rtcGetLocalDescription

```
int rtcGetLocalDescription(int pc, char *buffer, int size)
```

Retrieves the current local description in SDP format.

Arguments:
- `pc`: the Peer Connection identifier
- `buffer`: a user-supplied buffer to store the description
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the description is not copied but the size is still returned.

#### rtcGetRemoteDescription

```
int rtcGetRemoteDescription(int pc, char *buffer, int size)
```

Retrieves the current remote description in SDP format.

Arguments:
- `pc`: the Peer Connection identifier
- `buffer`: a user-supplied buffer to store the description
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the description is not copied but the size is still returned.

#### rtcGetLocalAddress

```
int rtcGetLocalAddress(int pc, char *buffer, int size)
```

Retrieves the current local address, i.e. the network address of the currently selected local candidate. The address will have the format `"IP_ADDRESS:PORT"`, where `IP_ADDRESS` may be either IPv4 or IPv6. The call might fail if the PeerConnection is not in state `RTC_CONNECTED`, and the address might change if the state is not `RTC_COMPLETED`.

Arguments:
- `pc`: the Peer Connection identifier
- `buffer`: a user-supplied buffer to store the address
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the address is not copied but the size is still returned.

#### rtcGetRemoteAddress

```
int rtcGetRemoteAddress(int pc, char *buffer, int size)
```

Retrieves the current remote address, i.e. the network address of the currently selected remote candidate. The address will have the format `"IP_ADDRESS:PORT"`, where `IP_ADDRESS` may be either IPv4 or IPv6. The call may fail if the state is not `RTC_CONNECTED`, and the address might change if the state is not `RTC_COMPLETED`.

Arguments:
- `pc`: the Peer Connection identifier
- `buffer`: a user-supplied buffer to store the address
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the address is not copied but the size is still returned.

#### rtcGetSelectedCandidatePair

```
int rtcGetSelectedCandidatePair(int pc, char *local, int localSize, char *remote, int remoteSize)
```

Retrieve the currently selected candidate pair. The call may fail if the state is not `RTC_CONNECTED`, and the selected candidate pair might change if the state is not `RTC_COMPLETED`.

Arguments:
- `pc`: the Peer Connection identifier
- `local`: a user-supplied buffer to store the local candidate
- `localSize`: the size of `local`
- `remote`: a user-supplied buffer to store the remote candidate
- `remoteSize`: the size of `remote`

Return value: the maximun length of strings copied in buffers (including the terminating null character) or a negative error code

If `local`, `remote`, or both, are `NULL`, the corresponding candidate is not copied, but the maximum length is still returned.

### Data Channel

#### rtcAddDataChannel

```
int rtcAddDataChannel(int pc, const char *label)
int rtcAddDataChannelEx(int pc, const char *label, const rtcDataChannelInit *init)

typedef struct {
	bool unordered;
	bool unreliable;
	unsigned int maxPacketLifeTime;
	unsigned int maxRetransmits;
} rtcReliability;

typedef struct {
	rtcReliability reliability;
	const char *protocol;
	bool negotiated;
	bool manualStream;
	uint16_t stream;
} rtcDataChannelInit;
```

Adds a Data Channel on a Peer Connection. The Peer Connection does not need to be connected, however, the Data Channel will be open only when the Peer Connection is connected.

Arguments:
- `pc`: identifier of the PeerConnection on which to add a Data Channel
- `label`: a user-defined UTF-8 string representing the Data Channel name
- `init`: a structure of initialization settings containing:
  - `reliability`: a structure of reliability settings containing:
    - `bool unordered`: if `true`, the Data Channel will not enforce message ordering, else it will be ordered
    - `bool unreliable`: if `true`, the Data Channel will not enforce strict reliability, else it will be reliable
    - `unsigned int maxPacketLifeTime`: if unreliable, maximum packet life time in milliseconds
    - `unsigned int maxRetransmits`: if unreliable and maxPacketLifeTime is 0, maximum number of retransmissions (0 means no retransmission)
  - `protocol` (optional): a user-defined UTF-8 string representing the Data Channel protocol, empty if NULL
  - `negociated`: if `true`, the Data Channel is assumed to be negociated by the user and won't be negociated by the WebRTC layer
  - `manualStream`: if `true`, the Data Channel will use `stream` as stream ID, else an available id is automatically selected
  - `stream` (0-65534): if `manualStream` is `true`, the Data Channel will use it as stream ID, else it is ignored

`rtcDataChannel()` is equivalent to `rtcDataChannelEx()` with settings set to ordered, reliable, non-negociated, with automatic stream ID selection (all flags set to `false`), and `protocol` set to an empty string.

Return value: the identifier of the new Data Channel or a negative error code.

The Data Channel must be deleted with `rtcDeleteDataChannel`.

#### rtcCreateDataChannel

```
int rtcCreateDataChannel(int pc, const char *label)
int rtcCreateDataChannelEx(int pc, const char *label, const char *protocol, const rtcReliability *reliability)
```

These functions are respectively equivalent to respectively `rtcAddDataChannel` and `rtcAddDataChannelEx` with the same arguments followed by `rtcSetLocalDescription`.

Return value: the identifier of the new Data Channel or a negative error code.

The Data Channel must be deleted with `rtcDeleteDataChannel`.

#### rtcDeleteDataChannel

```
int rtcDeleteDataChannel(int dc)
```

Deletes a Data Channel.

Arguments:
- `dc`: the Data Channel identifier

After this function has been called, `dc` must not be used in a function call anymore. This function will block until all scheduled callbacks of `dc` return (except the one this function might be called in) and no other callback will be called for `dc` after it returns.

#### rtcGetDataChannelStream

```
int rtcGetDataChannelStream(int dc)
```

Retrieves the stream ID of the Data Channel.

Arguments:
- `dc`: the Data Channel identifier

Return value: the stream ID (0-65534) or a negative error code

#### rtcGetDataChannelLabel

```
int rtcGetDataChannelLabel(int dc, char *buffer, int size)
```

Retrieves the label of a Data Channel.

Arguments:
- `dc`: the Data Channel identifier
- `buffer`: a user-supplied buffer to store the label
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the label is not copied but the size is still returned.

#### rtcGetDataChannelProtocol

```
int rtcGetDataChannelProtocol(int dc, char *buffer, int size)
```

Retrieves the protocol of a Data Channel.

Arguments:
- `dc`: the Data Channel identifier
- `buffer`: a user-supplied buffer to store the protocol
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the protocol is not copied but the size is still returned.

#### rtcGetDataChannelReliability

```
int rtcGetDataChannelReliability(int dc, rtcReliability *reliability)
```

Retrieves the reliability settings of a Data Channel. The function may be called irrelevant of how the Data Channel was created.

Arguments:
- `dc`: the Data Channel identifier
- `reliability` a user-supplied structure to fill

Return value: `RTC_ERR_SUCCESS` or a negative error code

### Track

#### rtcAddTrack

```
int rtcAddTrack(int pc, const char *mediaDescriptionSdp)
```

Adds a new Track on a Peer Connection. The Peer Connection does not need to be connected, however, the Track will be open only when the Peer Connection is connected.

Arguments:
- `pc`: the Peer Connection identifier
- `mediaDescriptionSdp`: a null-terminated string specifying the corresponding media SDP. It must start with a m-line and include a mid parameter.

Return value: the identifier of the new Track or a negative error code

The new track must be deleted with `rtcDeleteTrack`.

#### rtcDeleteTrack

```
int rtcDeleteTrack(int tr)
```

Deletes a Track.

Arguments:
- `tr`: the Track identifier

After this function has been called, `tr` must not be used in a function call anymore. This function will block until all scheduled callbacks of `tr` return (except the one this function might be called in) and no other callback will be called for `tr` after it returns.

#### rtcGetTrackDescription

```
int rtcGetTrackDescription(int tr, char *buffer, int size)
```

Retrieves the SDP media description of a Track.

Arguments:
- `dc`: the Track identifier
- `buffer`: a user-supplied buffer to store the description
- `size`: the size of `buffer`

Return value: the length of the string copied in buffer (including the terminating null character) or a negative error code

If `buffer` is `NULL`, the description is not copied but the size is still returned.

### WebSocket

#### rtcCreateWebSocket

```
int rtcCreateWebSocket(const char *url)
int rtcCreateWebSocketEx(const char *url, const rtcWsConfiguration *config)
```

Creates a new client WebSocket.

Arguments:
- `url`: a null-terminated string representing the fully-qualified URL to open.
- `config`: a structure with the following parameters:
  - `bool disableTlsVerification`: if true, don't verify the TLS certificate, else try to verify it if possible

Return value: the identifier of the new WebSocket or a negative error code

The new WebSocket must be deleted with `rtcDeleteWebSocket`. The scheme of the URL must be either `ws` or `wss`.

#### rtcDeleteWebSocket

```
int rtcDeleteWebSocket(int ws)
```

Arguments:
- `ws`: the identifier of the WebSocket to delete

Return value: the identifier of the new WebSocket or a negative error code

After this function has been called, `ws` must not be used in a function call anymore. This function will block until all scheduled callbacks of `ws` return (except the one this function might be called in) and no other callback will be called for `ws` after it returns.

### Channel (Data Channel, Track, and WebSocket)

The following common functions might be called with a generic channel identifier. It may be the identifier of either a Data Channel, a Track, or a WebSocket.

#### rtcSetXCallback

These functions set, change, or unset (if `cb` is `NULL`) the different callbacks of a channel.

```
int rtcSetOpenCallback(int id, rtcOpenCallbackFunc cb)
```

`cb` must have the following signature: `void myOpenCallback(int id, void *user_ptr)`

```
int rtcSetClosedCallback(int id, rtcClosedCallbackFunc cb)
```

`cb` must have the following signature: `void myClosedCallback(int id, void *user_ptr)`

```
int rtcSetErrorCallback(int id, rtcErrorCallbackFunc cb)
```

`cb` must have the following signature: `void myErrorCallback(int id, const char *error, void *user_ptr)`

```
int rtcSetMessageCallback(int id, rtcMessageCallbackFunc cb)
```

`cb` must have the following signature: `void myMessageCallback(int id, const char *message, int size, void *user_ptr)`

```
int rtcSetBufferedAmountLowCallback(int id, rtcBufferedAmountLowCallbackFunc cb)
```

`cb` must have the following signature: `void myBufferedAmountLowCallback(int id, void *user_ptr)`

```
int rtcSetAvailableCallback(int id, rtcAvailableCallbackFunc cb)
```

`cb` must have the following signature: `void myAvailableCallback(int id, void *user_ptr)`

#### rtcSendMessage

```
int rtcSendMessage(int id, const char *data, int size)
```

Arguments:
- `id`: the channel identifier
- `data`: the message data
- `size`: if size >= 0, `data` is interpreted as a binary message of length `size`, otherwise it is interpreted as a null-terminated UTF-8 string.

Return value: `RTC_ERR_SUCCESS` or a negative error code

Sends a message immediately if possible.

Data Channel and WebSocket: If the message may not be sent immediately due to flow control or congestion control, it is buffered until it can actually be sent. You can retrieve the current buffered data size with `rtcGetBufferedAmount`.
Tracks are an exception: There is no flow or congestion control, messages are never buffered and `rtcGetBufferedAmount` always returns 0.

#### rtcGetBufferedAmount

```
int rtcGetBufferedAmount(int id)
```

Retrieves the current buffered amount, i.e. the total size of currently buffered messages waiting to be actually sent in the channel. This does not account for the data buffered at the transport level.

Return value: the buffered amount or a negative error code

#### rtcGetBufferedAmountLowThreshold

```
int rtcSetBufferedAmountLowThreshold(int id, int amount)
```

Changes the buffered amount threshold under which `BufferedAmountLowCallback` is called. The callback is called when the buffered amount was strictly superior and gets equal to or lower than the threshold when a message is sent. The initial threshold is 0, meaning the the callback is called each time the buffered amount goes back to zero after being non-zero.

Arguments:
- `id`: the channel identifier
- `amount`: the new buffer level threshold

Return value: the identifier of the new WebSocket or a negative error code

#### rtcReceiveMessage

```
int rtcReceiveMessage(int id, char *buffer, int *size)
```

Receives a pending message if possible. The function may only be called if `MessageCallback` is not set.

Arguments:
- `id`: the channel identifier
- `buffer`: a user-supplied buffer where to write the message data
- `size`: a pointer to a user-supplied int which must be initialized to the size of `buffer`. On success, the function will write the size of the message to it before returning.

Return value: `RTC_ERR_SUCCESS` or a negative error code (In particular, `RTC_ERR_NOT_AVAIL` is returned when there are no pending messages)

If `buffer` is `NULL`, the message is not copied and kept pending but the size is still written to `size`.

#### rtcGetAvailableAmount

```
int rtcGetAvailableAmount(int id)
```

Retrieves the available amount, i.e. the total size of messages pending reception with `rtcReceiveMessage`. The function may only be called if `MessageCallback` is not set.

Arguments:
- `id`: the channel identifier

Return value: the available amount or a negative error code


