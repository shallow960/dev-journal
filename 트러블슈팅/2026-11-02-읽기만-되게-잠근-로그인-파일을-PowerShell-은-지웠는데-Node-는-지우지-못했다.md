# 2026-11-02 읽기만 되게 잠근 로그인 파일을, PowerShell 은 지웠는데 Node 는 지우지 못했다

> 태그: #트러블슈팅 #Windows #파일권한 #ACL #Node #Electron #PowerShell

## 무엇을 / 왜
사진 키오스크 PC 에서는 키오스크 앱과, 그 앱을 설치하고 업데이트하는 관리 프로그램(에이전트)이 함께 돌고 있었습니다. 둘 다 Electron 으로 만든 앱이었습니다. 에이전트가 서버에 로그인하면 받은 자격 정보를 로그인 파일 하나에 써 두었고, 키오스크 앱도 그 파일을 함께 썼습니다.

- 이 로그인 파일은 다른 프로그램이 고치지 못하게, 쓰고 나서 Windows 파일 권한(ACL)을 바꿔 잠가 두었습니다. 상속을 끊고 SYSTEM, Administrators, 그리고 나에게 모두 읽기(R)만 남기는 방식이었습니다.
- 매장 오픈을 앞두고 키오스크를 개발 서버에서 운영 서버로 옮기는 리허설을 했습니다. 에이전트의 '서버 변경'은 이 로그인 파일을 지워 로그아웃한 뒤, 새 서버 주소로 다시 로그인하게 하는 기능이었습니다.
- 회사 시험 키오스크에서 서버 변경을 누르자 `EPERM: operation not permitted, unlink '<로그인 파일 경로>'` 가 뜨고 아무것도 바뀌지 않았습니다.
- 그런데 같은 PC 에서 PowerShell 로 `Remove-Item -LiteralPath <로그인 파일 경로>` 를 하자, 그 파일은 아무 말 없이 지워졌습니다. 같은 Windows 계정, 같은 파일이었는데 한쪽은 지웠고 한쪽은 지우지 못했습니다.

## 어떻게 풀었나
- 처음에는 읽기만 남겼으니 못 지우는 게 당연하다고 생각했습니다. 그런데 PowerShell 이 지우는 걸 보고 나니 "삭제 권한이 없어서" 로는 설명이 되지 않았습니다.
- 찾아보니 Windows 에서는 파일에 삭제(D) 권한이 없어도, 그 파일이 든 폴더에 '하위 항목 삭제' 권한이 있으면 파일을 지울 수 있다는 걸 알게 됐습니다. 로그인 파일은 내 프로필 아래 폴더에 있었고, 그 폴더에는 나에게 모든 권한이 있었습니다.
- 그렇다면 Node 는 무엇이 달랐는지 보려고, 파일 권한을 하나씩 바꿔 가며 같은 파일을 Node, PowerShell, PowerShell `-Force` 세 가지로 지워 봤습니다(아래 코드).
  - 내 권한에 삭제(D)를 더해도 Node 는 여전히 EPERM 이었습니다.
  - 속성 쓰기(WA)를 더하자 Node 도 지웠습니다. Node 를 막은 것은 삭제 권한이 아니라 속성 쓰기 권한이었습니다.
  - PowerShell 도 `-Force` 를 붙이자, Node 처럼 읽기만 남긴 파일을 지우지 못했습니다.
- 이유를 찾아보니, Node 가 쓰는 libuv 는 파일을 지울 때 삭제 권한과 함께 속성 쓰기 권한까지 달라고 파일을 연다는 걸 알게 됐습니다. 파일에 '읽기 전용' 속성이 붙어 있으면 그 속성을 먼저 떼고 지우려는 것이었습니다. PowerShell 의 `-Force` 도 지우기 전에 읽기 전용·숨김 속성부터 떼려 했습니다. 속성 쓰기 권한이 없으니 둘 다 지우는 단계까지 가지 못하고 거부됐습니다.
- 실험에서도 그대로 나왔습니다. 권한은 그대로 두고 읽기 전용 속성만 붙인 파일은 거꾸로, Node 와 `-Force` 는 속성을 떼고 지웠고 `-Force` 없는 Remove-Item 은 지우지 못했습니다.
- 폴더 권한에서 '하위 항목 삭제'를 빼고 다시 해 보니, 읽기만 남긴 파일을 PowerShell 도 지우지 못했습니다. PowerShell 이 지울 수 있었던 건 폴더 권한 덕분이었습니다. 이 폴더에서는 속성 쓰기만 더한 파일(R,WA)도 지워지지 않았고, 삭제와 속성 쓰기를 함께 준 파일(R,D,WA)만 지워졌습니다.
- 2026-10-30 에 회사 키오스크를 다시 설치할 때도 이 파일 하나가 '액세스 거부'로 지워지지 않아, 폴더 이름을 바꿔 치우고 넘어간 적이 있었습니다. 그때 쓴 명령에는 `-Force` 가 붙어 있었고, 오류도 이번 실험의 `-Force` 칸과 같은 ArgumentException 이었습니다. 같은 원인을 그때 한 번 지나친 셈이었습니다.
- 고친 방법은 이렇습니다.
  - 새로 쓰는 로그인 파일은 내 권한을 읽기·삭제·속성 쓰기(R,D,WA)로 두게 했습니다. 삭제(D)까지 줘서 폴더 권한과 상관없이 지워지게 했고, 다른 계정에는 전처럼 읽기만 남겼습니다.
  - 예전 판이 읽기만 남겨 둔 파일은, 지우다 EPERM 이 나면 내 권한을 (R,D,WA)로 고친 뒤 다시 지우고, 그래도 안 되면 `-Force` 없는 Remove-Item 으로 지우게 했습니다. 지워졌는지는 명령의 종료 코드가 아니라 파일이 실제로 없어졌는지로 확인했습니다.
  - 키오스크 앱에도 같은 파일을 지우는 곳이 있어서 같은 규칙으로 맞추고, 두 프로그램의 권한 규칙이 같은지 대조하는 시험을 넣었습니다.
- 리허설 날에는 PowerShell 로 파일을 먼저 지운 뒤 서버 변경을 누르는 우회로 진행했고, 그 순서를 운영 절차서에 적었습니다. 고친 판은 회사 키오스크에서 다시 확인하기로 했습니다.

## 코드로 보기 (개념 재현)
폴더 두 개를 만들어 실험했습니다. 하나는 내 프로필 아래 폴더처럼 나에게 모든 권한이 있는 폴더이고, 다른 하나는 '하위 항목 삭제' 권한을 뺀 폴더입니다. 그 안에 로그인 파일을 쓰고 권한을 바꿔 잠근 뒤, Node 의 `fs.unlinkSync`, Windows PowerShell 5.1 의 `Remove-Item`, `Remove-Item -Force` 로 지워 봤습니다. `attrib +R` 줄은 권한은 그대로 두고 '읽기 전용' 속성만 붙인 파일입니다. 이름은 모두 예시입니다.

```javascript
// 로그인 파일을 "읽기만" 되게 잠갔을 때, 누가 그 파일을 지울 수 있는지 본다.
// 파일 권한(ACL)을 바꿔 가며 Node, PowerShell, PowerShell -Force 로 지워 본다. 이름은 모두 예시다.
const fs = require('fs');
const path = require('path');
const { execFileSync, spawnSync } = require('child_process');

const me = process.env.USERNAME;
const icacls = (...args) => execFileSync('icacls', args, { windowsHide: true });
let n = 0;

// 파일을 하나 쓰고 잠근다: 상속을 끊고 SYSTEM·Administrators 는 읽기만, 나는 perm 만 남긴다.
// perm 이 없으면 권한은 그대로 두고 '읽기 전용' 속성만 붙인다.
function makeLocked(dir, perm) {
  const file = path.join(dir, `login-token-${++n}.dat`);
  fs.writeFileSync(file, 'token=dummy');
  if (perm) icacls(file, '/inheritance:r', '/grant:r', 'SYSTEM:R', '/grant:r', 'Administrators:R',
                   '/grant:r', `${me}:${perm}`, '/c', '/q');
  else execFileSync('attrib', ['+R', file]);
  return file;
}

// Node 의 삭제(Electron 앱의 삭제도 같다)
function nodeDelete(file) {
  try { fs.unlinkSync(file); return 'OK'; } catch (e) { return e.code; }
}

// Windows PowerShell 5.1 의 Remove-Item. 경로는 환경변수로 넘긴다.
function psDelete(file, force) {
  const cmd = `try { Remove-Item -LiteralPath $env:TARGET ${force ? '-Force ' : ''}-ErrorAction Stop; 'OK' } ` +
              'catch { $_.CategoryInfo.Reason }';
  const r = spawnSync('powershell.exe', ['-NoProfile', '-Command', cmd],
    { env: { ...process.env, TARGET: file }, encoding: 'utf8', windowsHide: true });
  return r.stdout.trim();
}

function run(title, dir, cases) {
  console.log(title);
  const cols = (xs) => xs.map((x) => x.padEnd(28)).join(' ').trimEnd();
  console.log('내 권한     ' + cols(['Node unlink', 'Remove-Item', 'Remove-Item -Force']));
  for (const [label, perm] of cases) {
    const results = [nodeDelete(makeLocked(dir, perm)), psDelete(makeLocked(dir, perm), false),
                     psDelete(makeLocked(dir, perm), true)];
    console.log(`${label.padEnd(11)} ${cols(results)}`);
  }
}

function freshDir(name) {
  const dir = path.join(__dirname, name);
  if (fs.existsSync(dir)) icacls(dir, '/reset', '/t', '/q');
  fs.rmSync(dir, { recursive: true, force: true });
  fs.mkdirSync(dir);
  return dir;
}

console.log(`Node ${process.version}`);

// 1) 내 프로필 아래 폴더처럼, 폴더에는 내 권한이 모두 있을 때
const open = freshDir('acl-open');
run('[폴더에 내 권한이 모두 있을 때]', open,
  [['R', 'R'], ['R,D', '(R,D)'], ['R,WA', '(R,WA)'], ['R,D,WA', '(R,D,WA)'], ['attrib +R', null]]);

// 2) 폴더 권한에서 '하위 항목 삭제'를 뺐을 때: 나에게는 읽기·실행(RX)과 쓰기(W)만 준다.
const closed = freshDir('acl-closed');
icacls(closed, '/inheritance:r', '/grant:r', `${me}:(RX,W)`, '/q');
run('[폴더에 하위 항목 삭제 권한이 없을 때]', closed, [['R', 'R'], ['R,WA', '(R,WA)'], ['R,D,WA', '(R,D,WA)']]);

// 정리: 권한과 속성을 되돌리고 지운다. 파일과 폴더의 주인이라 권한을 바꿀 수 있다.
for (const dir of [open, closed]) {
  icacls(dir, '/reset', '/t', '/q');
  for (const name of fs.readdirSync(dir)) execFileSync('attrib', ['-R', path.join(dir, name)]);
  fs.rmSync(dir, { recursive: true, force: true });
}
```

Windows 11, Node v22.11.0, Windows PowerShell 5.1 에서 돌린 결과입니다.

```
Node v22.11.0
[폴더에 내 권한이 모두 있을 때]
내 권한     Node unlink                  Remove-Item                  Remove-Item -Force
R           EPERM                        OK                           ArgumentException
R,D         EPERM                        OK                           ArgumentException
R,WA        OK                           OK                           OK
R,D,WA      OK                           OK                           OK
attrib +R   OK                           IOException                  OK
[폴더에 하위 항목 삭제 권한이 없을 때]
내 권한     Node unlink                  Remove-Item                  Remove-Item -Force
R           EPERM                        UnauthorizedAccessException  ArgumentException
R,WA        EPERM                        UnauthorizedAccessException  UnauthorizedAccessException
R,D,WA      OK                           OK                           OK
```

- 폴더에 내 권한이 모두 있을 때, 읽기만 남긴 파일(R)은 Node 와 `-Force` 가 지우지 못했고 `-Force` 없는 Remove-Item 만 지웠습니다. 리허설에서 본 것과 같았습니다.
- 삭제(D)를 더해도 결과는 같았고, 속성 쓰기(WA)를 더하자 셋 다 지웠습니다.
- 읽기 전용 속성만 붙인 파일은 반대로 Node 와 `-Force` 만 지웠습니다. 둘 다 지우기 전에 속성부터 떼는 쪽이었습니다.
- 폴더에서 하위 항목 삭제 권한을 빼자, 읽기만 남긴 파일(R)과 속성 쓰기만 더한 파일(R,WA)은 셋 다 지우지 못했습니다. 삭제와 속성 쓰기를 함께 준 파일(R,D,WA)만 두 폴더 모두에서 지워졌습니다.

## 💡 배운 것
- Windows 에서 파일을 지울 수 있는지는 파일 권한 하나로 정해지지 않았습니다. 파일 권한, 폴더 권한, 그리고 지우는 쪽이 함께 달라고 하는 권한이 모두 맞아야 했습니다. 그래서 같은 파일을 PowerShell 은 지우고 Node 는 못 지울 수 있다는 걸 배웠습니다.
- Node(libuv)의 삭제는 `Remove-Item -Force` 처럼 속성부터 바꾸려 해서, 속성 쓰기 권한이 없는 파일은 지우지 못한다는 걸 배웠습니다. 파일을 잠글 때는 나중에 그 파일을 지울 내 코드가 어떤 권한을 쓰는지까지 같이 봐야 했습니다.
- 한쪽 방향만 시험하면 반대 방향에서 막힌다는 걸 다시 배웠습니다([2026-10-19, 되돌린 비밀번호가, 계속 통하고 있었다](../트러블슈팅/2026-10-19-되돌린-비밀번호가-계속-통하고-있었다.md)). 그때는 바꾼 값을 되돌리는 순서였고, 이번에는 잠근 파일을 지우는 순서였습니다. 파일을 잠그는 코드는 로그인할 때마다 돌았지만, 잠긴 파일을 지우는 서버 변경은 이번 리허설에서 처음 실제 키오스크로 해 봤습니다.

## 소감
처음에는 "읽기만 남겼으니 못 지우는 게 당연하다" 고 넘어갈 뻔했습니다. PowerShell 이 그 파일을 지우는 걸 보고서야 질문이 "왜 못 지우나" 에서 "누가 무슨 권한으로 지우려 하나" 로 바뀌었습니다. 2026-10-30 에 폴더 이름을 바꿔 넘어갔던 일도, 같은 원인이 먼저 보낸 신호였다는 걸 뒤늦게 알았습니다.
