# 2026-11-08 Electron 앱이 지운 폴더에, .asar 파일 하나만 남아 있었다

> 태그: #트러블슈팅 #Electron #asar #Windows #파일잠금 #업데이트

## 무엇을 / 왜

키오스크 PC 에는 촬영 앱(Electron)과, 그 옆에서 앱을 지켜보고 업데이트를 받는 관리 프로그램(에이전트)이 함께 돌고 있었습니다. 에이전트도 Electron 으로 만든 프로그램이라, 설치 폴더 안에 `resources\app.asar` 파일이 들어 있었습니다.

에이전트를 새 버전으로 바꾸는 일은 촬영 앱이 맡고 있었고, 순서는 이랬습니다.

1. 지난번에 남은 백업 폴더(`agent.bak`)를 지웠습니다.
2. 지금 설치 폴더(`agent`)를 `agent.bak` 으로 옮겼습니다.
3. 새 버전을 `agent` 에 놓았습니다. 실패하면 `agent.bak` 을 되돌렸습니다.

2026-11-06 회사 키오스크에서 에이전트 업데이트가 같은 버전에서 계속 실패했습니다. 기록을 보니 2번 단계가 `EPERM` 으로 여러 번 실패했고, 실패가 쌓이자 그 버전이 아예 막혀 있었습니다. 그날은 촬영 앱과 에이전트를 모두 끈 뒤 남은 `agent.bak` 의 이름을 손으로 바꾸고서야 업데이트가 됐습니다. 프로그램을 켜 둔 채로는 그 폴더 이름조차 바꿀 수 없었습니다. 원인은 찾지 못하고 넘어갔다가, 2026-11-08 에 다시 파고들었습니다.

## 어떻게 풀었나

남은 `agent.bak` 안을 보니 `resources\app.asar` 하나만 남아 있었습니다. 폴더를 통째로 지우라고 했는데 파일 하나가 남은 게 이상했습니다.

찾아보니 Electron 은 안에 들어 있는 Node 의 `fs` 를 고쳐서, `.asar` 파일을 폴더처럼 열어 볼 수 있게 해 둔다는 걸 알게 됐습니다. 앱 코드를 asar 묶음 하나로 배포하기 때문에 있는 기능이었습니다. 그래서 Electron 안에서 `fs.rmSync(폴더, { recursive: true })` 를 부르자 `app.asar` 를 파일이 아니라 폴더로 보고 그 "안" 을 지우려다가, 결국 `resources` 폴더를 비우지 못하고 `ENOTEMPTY` 로 끝났던 것입니다.

남은 파일을 다시 지우면 될 것 같았지만 그것도 안 됐습니다. asar 를 한 번 열어 본 Electron 프로세스가 그 파일의 핸들을 쥐고 있어서, 같은 프로세스 안에서는 원래 Node 방식으로 지워도 `EBUSY` 가 났습니다. 핸들은 프로세스가 끝나야 풀렸습니다. 프로그램을 모두 끈 뒤에야 폴더 이름을 바꿀 수 있었던 이유가 이것이었습니다.

정리하면 "처음 한 번은 되고, 그다음부터 계속 실패" 하는 모양이었습니다.

- 첫 교체는 성공했지만, 성공 뒤 `agent.bak` 정리가 같은 이유로 실패해 `app.asar` 가 남았습니다.
- 다음 교체 때 1번(남은 백업 지우기)이 또 실패했고, 2번에서 설치 폴더를 이미 있는 `agent.bak` 자리로 옮기려다 `EPERM` 이 났습니다.
- 이 실패가 쌓이면서 그 버전이 영구 차단 목록에 올랐습니다.

고친 것은 세 가지였습니다.

- 다른 Electron 앱의 설치·백업 폴더를 지우고 옮길 때는, Electron 이 고치지 않은 원래 `fs`(`original-fs`)를 쓰게 했습니다.
- 남은 백업을 그래도 못 지우면 이름 뒤에 시각을 붙여 비켜 두고, 그것도 안 되면 새 이름으로 백업하게 했습니다.
- "백업 폴더를 옮기지 못해 아무것도 바꾸지 않은 실패" 는 환경 문제로 따로 세서, 그 버전을 영구 차단하는 횟수에 넣지 않고 잠시 쉬었다가 다시 시도하게 했습니다.

## 코드로 보기 (개념 재현)

Electron 을 메인 프로세스로 띄우고, 진짜 asar 파일(Electron 에 들어 있는 `default_app.asar` 사본)이 든 폴더로 같은 순서를 재현했습니다.

```js
// asar-demo.js - Electron 메인 프로세스로 실행: electron.exe asar-demo.js <데모 폴더> <asar 파일>
const { app } = require('electron');
const fs = require('fs');            // Electron 이 고친 fs (.asar 를 폴더처럼 봄)
const ofs = require('original-fs');  // Electron 이 고치지 않은 원래 Node fs
const path = require('path');

const [dir, asarSrc] = process.argv.slice(2);
const short = (m) => m.split(dir).join('.');   // 출력의 긴 경로를 데모 폴더 기준으로 줄임

// 설치 폴더 흉내: <이름>/app.exe + <이름>/resources/app.asar (진짜 asar 파일 사본)
function makeInstall(name) {
  const d = path.join(dir, name);
  ofs.mkdirSync(path.join(d, 'resources'), { recursive: true });
  ofs.writeFileSync(path.join(d, 'app.exe'), 'dummy');
  ofs.copyFileSync(asarSrc, path.join(d, 'resources', 'app.asar'));
  return d;
}
const tryIt = (label, fn) => {
  try { fn(); console.log(label, 'ok'); }
  catch (e) { console.log(label, e.code, short(e.message)); }
};

console.log(`Electron ${process.versions.electron} / Node ${process.versions.node}`);

// 1) 같은 파일을 두 fs 가 다르게 본다
const bak = makeInstall('agent.bak');
const asar = path.join(bak, 'resources', 'app.asar');
console.log('1) electron fs: app.asar 가 폴더?', fs.statSync(asar).isDirectory());
console.log('1) original-fs: app.asar 가 폴더?', ofs.statSync(asar).isDirectory());

// 2) 지난번 백업 폴더를 Electron fs 로 통째로 지운다
tryIt('2) electron fs 로 agent.bak 지우기:', () => fs.rmSync(bak, { recursive: true, force: true }));
console.log('   남은 것:', ofs.existsSync(asar) ? 'resources\\app.asar' : '없음');

// 3) 다음 교체: 설치 폴더를 백업 이름으로 옮긴다 (남은 폴더가 그 자리에 있음)
const inst = makeInstall('agent');
tryIt('3) agent -> agent.bak 옮기기:', () => fs.renameSync(inst, bak));

// 4) 같은 프로세스 안에서 원래 fs 로 다시 지워 본다
tryIt('4) original-fs 로 남은 것 지우기:', () => ofs.rmSync(bak, { recursive: true, force: true }));

// 5) Electron fs 가 한 번도 열어 보지 않은 폴더는 원래 fs 로 지우고 옮길 수 있다
tryIt('5) original-fs 로 새 폴더 지우기:', () => ofs.rmSync(makeInstall('fresh.bak'), { recursive: true, force: true }));
tryIt('5) original-fs 로 새 폴더 옮기기:', () => ofs.renameSync(makeInstall('fresh2'), path.join(dir, 'fresh2.bak')));

app.quit();
```

Windows 에서 실제로 돌린 출력입니다.

```text
Electron 33.4.11 / Node 20.18.3
1) electron fs: app.asar 가 폴더? true
1) original-fs: app.asar 가 폴더? false
2) electron fs 로 agent.bak 지우기: ENOTEMPTY ENOTEMPTY: directory not empty, rmdir '.\agent.bak\resources'
   남은 것: resources\app.asar
3) agent -> agent.bak 옮기기: EPERM EPERM: operation not permitted, rename '.\agent' -> '.\agent.bak'
4) original-fs 로 남은 것 지우기: EBUSY EBUSY: resource busy or locked, unlink '.\agent.bak\resources\app.asar'
5) original-fs 로 새 폴더 지우기: ok
5) original-fs 로 새 폴더 옮기기: ok
```

프로세스가 끝난 뒤 PowerShell 의 `Remove-Item -Recurse` 로 남은 `agent.bak` 을 지우자 바로 지워졌습니다. 키오스크에서 겪은 순서(지우기 실패 → 옮기기 `EPERM` → 프로그램을 꺼야 풀림)가 그대로 나왔습니다.

## 💡 배운 것

- Electron 안의 `fs` 는 순수한 Node `fs` 가 아니라는 걸 배웠습니다. `.asar` 파일을 폴더로 보기 때문에, asar 가 든 폴더를 지우거나 옮기는 코드는 일반 Node 에서와 결과가 달랐습니다. 다른 Electron 앱의 폴더처럼 asar 를 그냥 "파일" 로 다뤄야 하는 곳에서는 `original-fs` 를 써야 했습니다.
- 한 번 열어 본 asar 는 그 프로세스가 끝날 때까지 잠겨 있었습니다. 그래서 "나중에 다시 지우면 되겠지" 로는 풀리지 않았고, 처음부터 asar 를 열지 않는 방식으로 다뤄야 했습니다.
- 같은 "지우기" 라도 누가 어떤 방식으로 부르느냐에 따라 결과가 달라진다는 걸 다시 배웠습니다([2026-11-02, 읽기만 되게 잠근 로그인 파일을, PowerShell 은 지웠는데 Node 는 지우지 못했다](../트러블슈팅/2026-11-02-읽기만-되게-잠근-로그인-파일을-PowerShell-은-지웠는데-Node-는-지우지-못했다.md)). 그때는 권한 때문이었고, 이번에는 같은 Node 라도 Electron 이 고친 `fs` 였기 때문이었습니다.
- 환경 때문에 생긴 실패와 진짜 실패를 같은 횟수로 세면, 한 번 꼬인 환경이 그 버전을 계속 막을 수 있다는 것도 알게 됐습니다.

## 소감

처음 실패를 봤을 때는 "다른 프로그램이 폴더를 잡고 있겠지" 하고 프로그램을 끄고 넘어갔습니다. 그런데 그 폴더를 잡고 있던 건, 그 폴더를 지우려던 바로 그 앱이었습니다. 오류가 난 단계(옮기기)보다 한 단계 앞(지우기)에서 무엇이 남았는지를 먼저 봤다면 더 빨리 찾았을 것 같습니다.
