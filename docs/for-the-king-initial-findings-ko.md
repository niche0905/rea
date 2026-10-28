# For The King: 첫 정적 분석 결과

2026-10-29, 설치된 REA 6.8.0의 관리 코드 CLI로 조사했습니다. 게임 실행·DLL 로드·패치·세이브 열람은 하지 않았습니다.

## 분석 준비 확인

사용자 PowerShell에서 Node.js `24.22.1`, npm/npx `11.19.1`, REA `6.8.0` 실행을 확인했습니다. npm의 Frida 설치 스크립트 경고는 이 관리 코드 분석을 막지 않았습니다. Ghidra나 MCP 연결 없이 실제 분석이 성공했습니다.

사용자가 저장한 `C:\Users\yudle\Documents\REA용\ForTheKing-analysis\assembly-artifact-utf8.json`은 정상 JSON으로 파싱됐습니다. 검사 범위는 `complete`, 문제 목록은 비어 있었습니다.

- 대상: `C:\Users\yudle\Documents\REA용\For The King\FTK_Data\Managed\Assembly-CSharp.dll`
- 크기: 10,446,336바이트
- SHA-256: `8096783ec9d1dcd8635c99ba8d73b133d5b5afad440ba4ea3053cc86e96a0dac`
- MVID: `b43c761a-f516-43f7-a466-5a7f39b1f370`
- REA 분류: `managed`, `unity-mono`, `cil`, `anycpu`

PE 헤더의 `x86` 표기만으로 게임 전체를 32비트라고 단정하지 않습니다. 이 관리 어셈블리의 아키텍처는 CLI 플래그와 함께 `anycpu`로 도출됐습니다.

## 전체 조사보다 범위를 좁혀야 합니다

전체 멤버 조사는 64 MiB 표현 예산에 도달해 `partial`로 끝났습니다. TypeDef 목록 4,740건은 보존됐지만 전체 메서드·필드의 빈 결과를 부재로 해석할 수 없습니다. 메타데이터 테이블은 MethodDef 29,046행, Field 35,382행을 선언합니다. 이는 선언된 행 수이며 전체 본문을 검증한 개수가 아닙니다.

첫 호출은 추가로 Evidence bundle 저장에서 접근 오류가 발생했습니다. 반환된 오류의 `details.partial_observation`에 타입 목록과 용량 제한 근거가 남아 있어 이를 재사용했습니다. 완성된 bundle 저장 성공으로 보고하지 않습니다.

타입 목록에서 다음 조사 후보를 확인했습니다. 역할은 이름에 따른 후보이며, 아직 각 구현을 검증한 것은 아닙니다.

| 실제 타입 이름 | 선언된 메서드 수 | 조사 후보 |
| --- | ---: | --- |
| `DamageCalculator` | 9 | 공격·피해량 처리 |
| `CharacterStats` | 159 | 캐릭터 능력치 |
| `CharacterSkills` | 20 | 캐릭터 스킬 |
| `ItemDrops` | 8 | 드롭 처리 |
| `GridEditor.FTK_itembase` | 22 | 아이템 데이터 |
| `GridEditor.FTK_itemsDB` | 9 | 아이템 데이터베이스 |

## DamageCalculator 선택 검사는 성공

실제로 실행한 작업은 `inspect-managed-members <대상 DLL> --type DamageCalculator --format json`입니다. 설치 패키지의 `scripts/rea.mjs`를 기존 Node로 호출했습니다.

REA Evidence ID: `ev_1e71c963f66402fdb0bf703f953e4e763adbdd4bd9a829703abcf8d6d3550b29`

선택 범위의 coverage는 `complete`, issues는 비어 있으며, 9개 메서드 모두 본문이 존재하고 잘린 명령은 0개로 보고됐습니다.

| 메서드 | 메타데이터 토큰 | 디코딩된 CIL 명령 수 |
| --- | --- | ---: |
| `get__userPickAttackTarget` | `0x0600042d` | 2 |
| `set__userPickAttackTarget` | `0x0600042e` | 3 |
| `CacheUserPickedTarget` | `0x0600042f` | 3 |
| `StartEngageAttack` | `0x06000430` | 275 |
| `_finishEngageAttack` | `0x06000431` | 127 |
| `_waitForUserPickTarget` | `0x06000432` | 28 |
| `_calcDamage` | `0x06000433` | 512 |
| `_playAttackSequence` | `0x06000434` | 31 |
| `_generateAttackAttempt` | `0x06000435` | 72 |

이 결과는 이름 목록뿐 아니라 CIL 본문도 정적으로 읽을 수 있음을 확인한 것입니다. 아직 피해량 계산식을 해석하거나 원래 C#을 복원하거나 실행 중 동작을 확인한 단계는 아닙니다. 토큰은 위 SHA-256과 MVID로 식별한 빌드 안에서만 의미를 가집니다.

## 로컬 결과 저장 위치

에이전트가 파일을 쓸 수 있는 Git 관리 밖의 로컬 작업 폴더에 저장했습니다. `.cache`는 임시 영역이므로 장기 보관하려면 별도 분석 폴더로 복사하세요. 게임 코드와 원본 분석 결과는 커밋하지 않습니다.

- `E:\GitHub\rea\.cache\ftk-analysis\damage-calculator.json`: 성공한 선택 검사의 전체 Evidence 기록입니다.
- `E:\GitHub\rea\.cache\ftk-analysis\members-summary.json`: 첫 호출의 오류와 부분 Evidence입니다. 정상 완료된 요약 결과는 아닙니다.

다음에는 알고 싶은 기능을 정하고 관련 메서드와 호출 대상을 추적합니다. 받는 피해량이나 공격 피해량을 조사하려면 `_calcDamage`가 후보입니다. C# 형태로 읽고 싶다면 ILSpy를 추가로 사용할 수 있습니다.

## PowerShell에서 JSON을 저장할 때의 주의 사항

처음 저장한 파일에서는 한글 경로의 문자가 깨지고 JSON 이스케이프가 손상됐습니다. 아래 설정을 적용해 다시 저장한 결과는 정상적으로 읽혔습니다. 정확한 근본 원인까지는 조사하지 않았습니다.

```powershell
$utf8 = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = $utf8
$OutputEncoding = $utf8
```

설치된 CLI에서 JSON 출력을 지정할 때는 `--format json`을 사용합니다. [준비 가이드](for-the-king-static-analysis-setup-ko.md)에도 실제 도움말에 맞는 옵션과 UTF-8 설정을 반영했습니다.
