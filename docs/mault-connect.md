# Hosted Mault Connect

Mault Connect is a companion Linux SBC agent. Hardware profiles and command
translation stay on the SBC. The hosted server only authenticates, bounds and
relays the existing sorter protocol. This hosted patch does not install the
agent, configure GPIO, or migrate an existing sorter service.

## Connection and camera

The paired agent connects outbound to `/sorter-link/device` using its device
bearer token. Never expose the loopback camera or inference helpers publicly.
Only one controller may own a physical sorter. Stop the previous station
controller before starting Connect, after independently verifying the GPIO map.
Do not re-pair unnecessarily: replacing a pairing token invalidates the old one.

Enable the camera in the SBC profile. Agents advertise
`X-Sorter-Pi-Camera: mjpeg-v1`. In the hosted scanner's camera selector, choose
**Mault Connect camera** for that station's selected device. Browser cameras,
phone cameras and document scanners remain separate choices. The selection is
saved through the existing camera/settings workflow.

The authenticated `/sorter-link/camera` WebSocket permits one subscriber from
the owning organization. JPEG frames use the `PIC1` device prefix, are limited
to 2 MiB and 20 relayed frames per second, and are dropped under backpressure.
The browser keeps one pending frame, releases decoded bitmaps, and reports an
error on disconnect or ten seconds without a decoded frame. Use Retry after
restoring the connection. Remote focus and zoom are not supported.

## Optional NPU callout

NPU use is opt-in through the companion agent's profile, not a global browser
setting. Enable it only when a compatible loopback Milo worker is installed
and tested. The agent advertises `X-Sorter-Npu-Protocol: milo-v1`.
Selecting that device's Mault Connect camera routes its Milo embeddings through
the SBC when NPU support is advertised. Other camera sources retain their
existing inference behavior. Card detection and OCR still use the existing
scanner paths; this callout only accelerates Milo embeddings.

`POST /cards/embedding` requires normal bearer authentication, organization
membership and device ownership, and rejects impersonation. Its multipart body
contains `deviceGuid` and `tensor`, exactly little-endian float32 NCHW
`[1,3,448,448]`, normalized with the shared ImageNet means and standard
deviations. Non-finite input is rejected. The pinned Milo source checksum is
sent with the request; the SBC worker must use a compatible model.

The relay permits two pending requests per device with a ten-second timeout.
Responses must contain a correlated request ID, shape `[1,128]`, and 128 finite
float32 values with a nonzero norm. The server L2-normalizes the result.
Inference failures are explicit, with no silent browser or server fallback
once an SBC callout has been requested.

Device, camera and inference state is in-process. All related requests must
reach the same server process. This requires affinity in multi-instance
deployments. Transmitting each input costs approximately 2.30 MiB, so network
latency can dominate NPU compute time.

## Release and safety gates

Do not enable native motion until the GPIO/channel mapping is independently
verified and a fail-safe motion-permit GPIO is wired and tested. An independent
servo-power cutoff is required but does not replace that software input.
Never move motors without the operator present.

Installing a Pi release requires approval of its exact release ID and manifest
SHA-256 through the normal idle-checked updater. Hosted changes must pass
review and the normal deployment workflow. Unit and local transport tests do
not establish physical or hosted end-to-end readiness.
