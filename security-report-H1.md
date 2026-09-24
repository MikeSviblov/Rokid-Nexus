**Title:** glasses-hub accepts unauthenticated RFCOMM clients on the RokidBus SPP service (can request a Wireless ADB pairing code)

Hi! I'm running a self-built Rokid Nexus (glasses-hub 1.4.11 / phone-hub 1.4.11, built from `398e7899`) and found an
issue in the phone↔glasses SPP channel that I think deserves a fix before it's widely known, so I'm reporting it privately.

### Summary
`SppServerManager` listens with `listenUsingInsecureRfcommWithServiceRecord("RokidBus", 0b005957-ec6d-4af5-bcba-6c786c46634e)`
and passes every accepted socket straight to `GlassesHub.onRemoteEnvelope` — no check of the peer address / bond state and no
session authentication in `FrameProtocol`. Remote envelopes are treated as coming from the trusted phone hub, including
navigation/IME/native-app control, self-arm/repair routes and `/debug/adb/request` (for which a syntactically valid `pluginId`
is enough). A new SPP client also replaces the shared output stream, and outbound traffic prefers SPP over CXR, so the
replies (e.g. the ADB pairing code) go to the newest client.

### Reproduction (verified on device, harmless variant)
Rokid Glasses (RV101, firmware 1.26), Nexus set up normally with a Pixel. From a Linux box with a Bluetooth adapter that has
**never been paired** with the glasses:

```
$ hcitool name AC:86:D1:5D:26:D3            # glasses are not discoverable, but connectable
Glasses_1682
$ sdptool browse AC:86:D1:5D:26:D3 | grep -A3 RokidBus
Service Name: RokidBus  …  Channel: 5
$ python3 -c 'import socket;s=socket.socket(socket.AF_BLUETOOTH,socket.SOCK_STREAM,socket.BTPROTO_RFCOMM);s.connect(("AC:86:D1:5D:26:D3",5));print("connected")'
connected
```
Glasses logcat, no pairing dialog, the peer never becomes bonded:
```
ROKIDBUS: SPP client accepted
ROKIDBUS: SPP client loop ended
bt_btif_config: bt_config remove section=<peer> reason=remove_unpaired_clone
```
I sent no frames, so I did not exercise the command routes — the rest is from reading the code
(`SppServerManager.kt:68,90,93`, `GlassesHub.kt:260,294,369,416`, `WirelessAdbController.kt:99,157`,
`GlassesOutboundTransportPolicy.kt:32`).

### Impact
Anyone within Bluetooth range who knows the glasses' BT address can talk to the glasses hub as if they were the phone
(UI navigation/input, intercepting hub replies). The BT address is easy to learn (on this device it is the Wi-Fi MAC + 1).
If self-arm is done and the attacker is on the same Wi-Fi, the chain `/debug/adb/request` → pairing code →
`adb pair`/`adb connect` gives an `adb shell` on the glasses. Not tested end-to-end.

### Suggested fix
- Authenticate the peer: e.g. a per-install shared secret established over the already-authenticated CXR channel during
  setup, then a challenge–response (HMAC over nonce) on every SPP connection before any envelope is dispatched; bind
  replies to that session and reject/close unauthenticated sockets.
- Don't let a new socket replace the output stream of an authenticated session.
- As an extra layer: `listenUsingRfcommWithServiceRecord` (secure) and/or accept only bonded peers.
- Treat all hub-only routes (IME/navigation/self-arm/ADB) as unavailable on the remote ingress until authenticated.

Happy to test a fix on my device, or to send a PR if you prefer.
