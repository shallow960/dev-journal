# 2026-10-22 VS Code 에서 띄운 Electron 앱이, Node 로 떠서 죽었다

> 태그: #트러블슈팅 #Electron #환경변수 #VSCode

## 무엇을 / 왜

키오스크 앱(Electron)을 개발 모드로 띄우자 시작하자마자 TypeError 로 죽었습니다. 앱 코드 맨 위의 `import { app } from 'electron'` 에서 가져온 `app` 이 `undefined` 였습니다. 같은 명령을 다른 터미널 창에서 실행하면 잘 떴습니다. 코드는 같은데 **어디서 실행하느냐**에 따라 결과가 달랐습니다.

## 어떻게 풀었나

실행 위치에 따라 달라지는 건 대개 물려받은 환경변수입니다. 문제가 나는 터미널(VS Code 확장이 띄운 터미널)의 환경변수를 보니 `ELECTRON_RUN_AS_NODE=1` 이 들어 있었습니다.

이 변수는 Electron 실행 파일을 **Electron 이 아니라 평범한 Node 로** 돌게 만드는 스위치입니다. VS Code 같은 Electron 기반 도구가 자기 안에서 Node 스크립트를 돌릴 때 쓰는데, 그 아래에서 연 터미널에 그대로 남아 있었습니다.

Node 로 돌면 `require('electron')` 은 Electron 의 API 가 아니라 npm 의 `electron` 패키지를 읽습니다. 이 패키지는 실행 파일의 **경로 문자열**만 내보냅니다. 그러니 `app` 이 없었던 것입니다.

해결은 앱을 띄우는 스크립트에서 이 변수를 지우고 실행하는 것이었습니다.

## 코드로 보기 (개념 재현)

```bash
# 앱 폴더(의존성에 electron 패키지가 설치된 곳)에서 실행해 본다
ELECTRON_RUN_AS_NODE=1 ./node_modules/electron/dist/electron -e "
  const e = require('electron');      // Node 로 돌면 npm 의 electron 패키지가 잡힌다
  console.log(typeof e);              // 'string' -> Electron API 객체가 아니다
  console.log(String(e).slice(-40));  // 실행 파일 경로 문자열
  console.log(e.app);                 // undefined -> app 을 쓰는 첫 줄에서 TypeError
"
# 실제 출력:
# string
# \node_modules\electron\dist\electron.exe
# undefined
```

```bat
@echo off
rem 앱을 띄우는 배치 파일: 물려받았을 수 있는 스위치 변수를 먼저 비운다
set "ELECTRON_RUN_AS_NODE="
call npm run dev
```

```bash
# bash 라면
unset ELECTRON_RUN_AS_NODE
npm run dev
```

## 💡 배운 것

- 같은 명령이 실행한 곳에 따라 다르게 동작하면, 코드보다 먼저 **물려받은 환경변수**를 확인한다는 걸 배웠습니다.
- Electron 에는 실행 파일을 Node 로 바꾸는 스위치 변수가 있고, Electron 으로 만든 편집기 안에서 연 터미널에는 그 값이 남아 있을 수 있다는 걸 배웠습니다.
- `require('electron')` 이 문자열을 돌려주면 "Electron 이 아니라 Node 로 돌고 있다" 는 신호입니다.

## 소감

오류 메시지만 보면 코드가 잘못된 것처럼 보였는데, 실제 원인은 코드 밖 한 줄짜리 설정이었습니다. "다른 창에서는 된다" 는 단서를 일찍 붙잡은 게 시간을 많이 아꼈습니다.
