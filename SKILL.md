---
name: sonos-control
description: "Use when controlling Sonos speakers: play, skip, volume."
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [sonos, audio, upnp, home-automation]
---

## When to Use

User asks to play, pause, skip, change volume, or check status of a Sonos speaker or room ("skip back N tracks on sonos", "turn down the bedroom speaker"). Also the fallback when the Spotify Web API returns 403 `Restricted device` and playback is on a Sonos.

# Sonos control via plain UPnP/SOAP

## Preferred binary

`sonos` (steipete/sonoscli, `go install github.com/steipete/sonoscli/cmd/sonos@latest`):
`sonos discover|status|play|pause|stop|volume set N|group join|queue|favorites|smapi search --service Spotify ...`, target with `--name "Room"` or `--ip IP`. `sonos status` shows Title/Artist even over Spotify line-in. The curl/SOAP recipes below are the zero-dependency fallback (no Go, no install).

## Discover speakers

Sonos speaks unauthenticated UPnP/SOAP on port 1400. LAN scan (adjust the /24 prefix):

```bash
for i in $(seq 1 254); do (curl -s -m 1 "http://192.168.1.$i:1400/xml/device_description.xml" | grep -q Sonos && echo "SONOS: 192.168.1.$i") & done; wait
```

Room name: `curl -s http://IP:1400/xml/device_description.xml | grep -o '<roomName>[^<]*'`.

## Send a command

Every command is one POST. Template (swap action name; substitute your speaker's IP):

```bash
curl -s -m 3 -X POST "http://SPEAKER_IP:1400/MediaRenderer/AVTransport/Control" \
  -H 'Content-Type: text/xml; charset="utf-8"' \
  -H 'SOAPAction: "urn:schemas-upnp-org:service:AVTransport:1#ACTION"' \
  -d '<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"><s:Body><u:ACTION xmlns:u="urn:schemas-upnp-org:service:AVTransport:1"><InstanceID>0</InstanceID></u:ACTION></s:Body></s:Envelope>'
```

- Transport (AVTransport:1): `Play`/`Pause`/`Stop`/`Next`/`Previous` (no extra args), `GetPositionInfo` (read `<TrackDuration>`, `<RelTime>`, `<TrackURI>`), `GetTransportInfo` (`<CurrentTransportState>`).
- Volume: service `RenderingControl:1`, endpoint `/MediaRenderer/RenderingControl/Control` (bare `/RenderingControl/Control` returns 405), `GetVolume`/`SetVolume` need `<InstanceID>0</InstanceID><Channel>Master</Channel>` (SetVolume also `<DesiredVolume>N</DesiredVolume>` 0–100). Read current volume first, apply the delta, clamp to 0–100.
- Skip N tracks: loop the `Previous`/`Next` POST N times, then `GetPositionInfo` to verify.
- Success = empty SOAP response envelope; an HTML 405 page means the control URL is wrong.

## Pitfalls

- **Verify skips by duration change, not title.** Spotify line-in (VLI) playback hides Title/Creator from `/Status` and `GetPositionInfo` (Track reads 0); a changed `<TrackDuration>` + fresh `<RelTime>` is the proof the track advanced. The `sonos` binary decodes VLI metadata, so prefer it for "what's playing".
- **`Previous` near a track's start replays the same track** (standard player behavior), so N POSTs may land fewer than N back. Report the caveat and offer "again" rather than claiming exact N.
- **Spotify Web API 403 `Restricted device` on `/me/player/*` control endpoints** means the active Connect device refuses Web-API control — no retry or client-side fix works. Go straight to Sonos UPnP control of the speaker, which bypasses the Spotify API entirely; Play/Pause/Skip/Stop and volume all work over VLI.