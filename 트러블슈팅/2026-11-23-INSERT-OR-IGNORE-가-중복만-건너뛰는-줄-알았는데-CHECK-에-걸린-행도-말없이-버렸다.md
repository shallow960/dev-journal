# 2026-11-23 INSERT OR IGNORE 가 중복만 건너뛰는 줄 알았는데, CHECK 에 걸린 행도 말없이 버렸다

> 태그: #트러블슈팅 #SQLite #조용한실패 #결제 #키오스크

## 무엇을 / 왜

사진 키오스크에 '원본 사진 결제' 를 만들고 있었습니다. 손님이 인쇄 화면에서 돈을 더 내면, 필터를 입히기 전의 카메라 원본을 묶음 파일 하나로 만들어 서버에 올리고, 손님은 이미 받은 QR 페이지에서 내려받는 기능이었습니다.

서버로 올리는 일은 키오스크에 원래 있던 업로드 대기열에 맡겼습니다. 대기열은 키오스크 PC 의 SQLite 표였고, 사진·영상처럼 올릴 것마다 종류를 적은 행을 넣어 두면 인터넷이 끊겨도 다시 이어서 올렸습니다. 코드의 종류 목록(타입)에 '원본' 을 더하고 행을 넣게 했는데, 원본 행이 대기열에 들어가지 않는 것을 찾았습니다. 오류는 하나도 없었습니다. 그대로 내보냈다면 돈을 받고도 원본은 올라가지 않을 뻔했습니다.

## 어떻게 풀었나

### 원인: 표의 CHECK 와 OR IGNORE

대기열 표를 만드는 SQL 에는 종류 칸이 받을 값이 CHECK 로 적혀 있었습니다. 코드의 타입에는 '원본' 을 더했지만 이 목록에는 없었습니다. 타입 검사는 코드 안에서만 하니, 표가 그 값을 거절한다는 것까지는 알려 주지 못했습니다.

행을 넣는 코드는 `INSERT OR IGNORE` 를 쓰고 있었습니다. 같은 QR·종류·파일의 행을 두 번 넣지 않으려고 고유 색인과 함께 쓴 것이었습니다. 찾아보니 SQLite 의 `OR IGNORE` 는 고유(UNIQUE) 위반만이 아니라 NOT NULL, CHECK, PRIMARY KEY 위반에도 똑같이 걸린 행만 건너뛰고, 문장은 오류 없이 끝낸다는 걸 알게 됐습니다. 남는 흔적은 "바뀐 행 0" 하나뿐이었습니다.

코드는 바뀐 행이 0 이면 "같은 행이 이미 있어 건너뜀" 으로 보고 디버그 로그 한 줄만 남겼습니다. 중복과 거절이 같은 모습이어서, 로그도 원인을 다르게 말하고 있었습니다.

### 고친 것

1. **새로 까는 PC:** 표를 처음 만드는 SQL 의 목록에 '원본' 을 넣었습니다.
2. **이미 쓰던 PC:** 찾아보니 SQLite 의 ALTER TABLE 은 표 이름 바꾸기와 칸 더하기·이름 바꾸기·지우기만 되고, CHECK 같은 제약은 바꿀 수 없었습니다. 그래서 키오스크가 켜질 때 표를 만든 SQL(sqlite_master 에 남아 있는 CREATE 문)을 읽어 '원본' 이 없으면, 한 트랜잭션 안에서 새 표를 만들고 → 옛 행을 옮기고 → 옛 표를 지우고 → 새 표의 이름을 바꾸고 → 색인을 다시 만들었습니다. 옛 표에 있던 칸과 값은 모두 옮겼고, 중간에 실패하면 되돌려 옛 표를 그대로 두었습니다. 사진과 영상은 예전처럼 올라가게 하기 위해서였습니다.
3. **넣은 뒤 다시 확인:** 원본 행을 넣었는데 바뀐 행이 0 이면 같은 행이 정말 있는지 다시 읽어 보고, 없으면 실패로 다뤘습니다. 표를 바꾸지 못한 PC 라면 원본을 준비할 때 "업로드 기록 표를 바꾸지 못함 - 프로그램을 다시 켜 주세요" 같은 사유로 실패를 남겼습니다. 어느 쪽이든 그 결제는 직원 화면의 '확인이 필요한 결제' 에 사유와 함께 올려, 다시 올리거나 환불할 수 있게 했습니다.
4. **목록은 한 곳에서:** 코드에 종류 목록을 배열 하나로 두고 다시 만드는 표의 CHECK 를 그 배열로 만들었습니다. 처음 만드는 SQL 에 적힌 목록이 그 배열과 같은지는 시험으로 맞춰 보게 했습니다.

## 코드로 보기 (개념 재현)

이름은 바꾼 더미입니다.

```sql
-- 처음 만든 표: 받을 종류 목록에 'full_res' 가 없다
CREATE TABLE send_job (
  id         INTEGER PRIMARY KEY,
  pass_code  TEXT NOT NULL,
  saved_file TEXT NOT NULL,
  job_type   TEXT NOT NULL CHECK (job_type IN ('image', 'clip'))
);
CREATE UNIQUE INDEX send_job_once ON send_job (pass_code, job_type, saved_file);

INSERT OR IGNORE INTO send_job (pass_code, saved_file, job_type) VALUES ('Q1', 'a.jpg', 'image');      -- 1행 들어감
INSERT OR IGNORE INTO send_job (pass_code, saved_file, job_type) VALUES ('Q1', 'a.jpg', 'image');      -- 0행: 중복이라 건너뜀
INSERT OR IGNORE INTO send_job (pass_code, saved_file, job_type) VALUES ('Q1', 'all.zip', 'full_res'); -- 0행: CHECK 위반인데 오류 없음

INSERT INTO send_job (pass_code, saved_file, job_type) VALUES ('Q1', 'all.zip', 'full_res');
-- 오류: CHECK constraint failed  (OR IGNORE 를 빼야 비로소 보인다)
```

```js
// 넣은 뒤 '바뀐 행 0' 을 중복이라고 단정하지 않는다
function pushJob(db, job) {
  const r = db
    .prepare('INSERT OR IGNORE INTO send_job (pass_code, saved_file, job_type) VALUES (?, ?, ?)')
    .run(job.passCode, job.file, job.type);
  if (r.changes > 0) return 'added';
  const same = db
    .prepare('SELECT 1 FROM send_job WHERE pass_code = ? AND saved_file = ? AND job_type = ?')
    .get(job.passCode, job.file, job.type);
  return same ? 'already-there' : 'not-added'; // not-added 면 그 결제를 '확인이 필요한 결제' 에 올린다
}

// 이미 쓰던 PC: CHECK 는 ALTER TABLE 로 못 바꾸니, 한 트랜잭션으로 표를 다시 만든다
const JOB_TYPES = ['image', 'clip', 'full_res']; // 종류 목록은 여기 한 곳

function widenJobTypes(db) {
  const row = db.prepare("SELECT sql FROM sqlite_master WHERE type = 'table' AND name = 'send_job'").get();
  if (!row || row.sql.includes("'full_res'")) return true; // 표가 없거나(처음 만들 때 넣는다) 이미 받는다
  const list = JOB_TYPES.map((t) => `'${t}'`).join(', ');
  try {
    db.exec('BEGIN IMMEDIATE');
    db.exec(`
      CREATE TABLE send_job_next (
        id         INTEGER PRIMARY KEY,
        pass_code  TEXT NOT NULL,
        saved_file TEXT NOT NULL,
        job_type   TEXT NOT NULL CHECK (job_type IN (${list}))
      );
      INSERT INTO send_job_next (id, pass_code, saved_file, job_type)
        SELECT id, pass_code, saved_file, job_type FROM send_job;
      DROP TABLE send_job;
      ALTER TABLE send_job_next RENAME TO send_job;
      CREATE UNIQUE INDEX send_job_once ON send_job (pass_code, job_type, saved_file);
    `);
    db.exec('COMMIT');
    return true;
  } catch (e) {
    try { db.exec('ROLLBACK'); } catch { /* 시작도 못 했으면 되돌릴 것도 없다 */ }
    return false; // 옛 표 그대로. 원본을 준비할 때 이 결과를 보고 사유와 함께 실패를 남긴다
  }
}
```

## 💡 배운 것

- SQLite 의 `OR IGNORE` 는 "중복이면 건너뛰기" 가 아니라 "제약에 걸리면 그 행만 건너뛰기" 였습니다. 바뀐 행 0 을 중복이라고 단정하면 거절을 놓쳤고, 0 이 나오면 같은 행이 정말 있는지 다시 확인해야 했습니다.
- 코드의 타입과 표의 CHECK 는 서로를 몰랐습니다. 타입에 값을 더해도 타입 검사는 통과했고, 표는 실행할 때에야 말없이 거절했습니다. 같은 목록이 두 곳에 있으면 한 곳에서 만들거나, 둘이 같은지 시험으로 묶어 둬야 했습니다.
- 말없이 건너뛰는 길은 원인을 감춘다는 걸 [2026-09-11 글](2026-09-11-음수를-막았는데-음수가-들어오지-않고-있었다.md)에 이어 다시 배웠습니다. 그때는 입력 칸이 '-' 를 말없이 버렸고, 이번에는 데이터베이스가 행을 말없이 버렸습니다.

## 소감

원본 사진 결제는 돈을 받은 뒤에 일이 이어지는 기능이라, 중간에 조용히 끊기는 곳이 하나만 있어도 손님은 돈만 내고 사진을 받지 못했을 겁니다. 내보내기 전에 찾아서 다행이었습니다. 앞으로 대기열에 새 종류를 더할 때는 표의 제약부터 보려고 합니다.
