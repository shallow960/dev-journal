# 2026-08-26 OBS 없이 브라우저로 방송, WHIP/WebRTC 붙인 기록

> 태그: #새기술 #WebRTC #WHIP #라이브 #Cloudflare

## 무엇을 / 왜

라이브 방송을 하려면 셀러가 OBS 같은 방송 프로그램을 따로 깔아야 했다. 일반 판매자한테는
진입장벽이 높음. "설치 없이 브라우저에서 바로 방송" 이 되면 좋겠다 싶었다.

알아보니 우리가 쓰는 영상 서비스(Cloudflare)가 라이브 인풋을 만들 때 응답에 이걸 다 준다.

```json
{
  "rtmps": { "url": "...", "streamKey": "..." },   // OBS 용 (우리가 쓰던 것)
  "webRTC": { "url": "..." },                        // 브라우저 송출(WHIP)
  "webRTCPlayback": { "url": "..." }                 // 초저지연 시청(WHEP)
}
```

그런데 우리 코드는 `rtmps` 만 꺼내 쓰고 아래 둘을 버리고 있었다. 이미 받고 있는데 안 쓰던 값.

## WHIP / WHEP 이 뭔가 (내가 이해한 만큼)

- **WHIP** = 브라우저가 카메라 영상을 서버로 올리는 표준. getUserMedia 로 카메라를 잡고,
  WebRTC 연결의 제안(SDP offer)을 만들어서 그 URL에 그냥 POST 하면 된다. 답(SDP answer)을
  받아서 연결에 넣으면 송출 시작.
- **WHEP** = 반대로 시청자가 받는 쪽. 똑같이 offer 를 만들어 재생 URL에 POST, answer 받아서
  붙이면 영상이 온다. 이게 되면 지연이 1초 미만.

둘 다 라이브러리 없이 브라우저 기본 기능(RTCPeerConnection)과 fetch 로 된다는 게 신기했다.

## 어떻게 붙였나

송출(WHIP) 쪽 뼈대:

```
const pc = new RTCPeerConnection({ iceServers: [{ urls: "stun:stun.cloudflare.com:3478" }] });
media.getTracks().forEach(t => pc.addTrack(t, media));   // 카메라·마이크
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
await ICE_수집_대기(pc);                                    // 후보 다 모은 뒤 한 번에
const res = await fetch(whipUrl, {
  method: "POST",
  headers: { "Content-Type": "application/sdp" },
  body: pc.localDescription.sdp
});
await pc.setRemoteDescription({ type: "answer", sdp: await res.text() });
```

시청(WHEP)은 방향만 반대. `addTransceiver("video", {direction:"recvonly"})` 로 받기만 하고,
`ontrack` 으로 온 스트림을 video 에 꽂는다.

셀러 화면에는 방송 방식을 고르게 했다. 간편 방송(브라우저) / 전문가 방송(OBS). 둘 다 같은
라이브 인풋으로 들어가서 과금도 똑같다. 시청 쪽은 방송 중이면 WebRTC 우선, 실패하거나 없으면
기존 HLS 로 떨어지게.

## 막혔던 것

카메라 권한 창이 아예 안 떴다. 정책(Permissions-Policy) 헤더가 `camera=()` 로 카메라를
통째로 막고 있어서였다. `camera=(self)` 로 풀어 줘야 getUserMedia 가 물어보기라도 한다.

그리고 마이크 없는 기기에서 카메라까지 안 열렸는데, video+audio 를 한 번에 요청해서 마이크가
없으면 통째로 실패한 거였다. 영상만이라도 되게 폴백을 넣었다. (이건 따로 기록)

## 💡 배운 것

- 새 걸 만들기 전에 **이미 받고 있는 응답에 안 쓰던 값이 있는지** 보자. WHIP/WHEP URL 은
  처음부터 응답에 있었는데 우리가 안 꺼내 쓴 것뿐이었다. 없는 걸 만드는 것보다 빨랐다.
- WebRTC 송출/재생이 라이브러리 없이 fetch + RTCPeerConnection 으로 된다는 것. WHIP/WHEP 이
  "SDP 를 HTTP로 한 번 주고받는" 단순한 약속이라 그렇다.
- 브라우저 미디어는 정책 헤더에 막힐 수 있다. 코드가 맞아도 헤더 한 줄에 카메라가 안 열린다.

## 소감

라이브를 "인프라 결정 대기" 라고만 생각했는데, 열어 보니 이미 다 켜져 있었고 값도 다 오고
있었다. 결정이 안 된 게 아니라 내가 현황을 안 보고 있던 거였다. WHIP 이라는 단어도 이번에
처음 봤는데, 막상 해 보니 offer 하나 POST 하는 거라 겁먹을 게 아니었다.
