# For The King 정적 분석 준비 가이드 — Windows

작성일: 2026-10-29. REA 로컬 커밋 `e79ad6ca8`, 버전 `6.8.0` 문서 기준.

목표는 복사한 게임을 **실행하지 않고**, 코드·데이터 구조를 조사하고 관심 있는 기능의 구현을 추적하는 것입니다. 아래 설치·분석 명령은 따라 할 절차이며, 이번 준비 점검에서 실행한 명령과 구분합니다.

## 1. 현재 확인한 상태

대상 경로:

```text
C:\Users\yudle\Documents\REA용\For The King
```

| 항목 | 직접 확인한 내용 | 의미 |
| --- | --- | --- |
| 게임 구조 | `FTK.exe`, `UnityPlayer.dll`, `FTK_Data/Managed`, `FTK_Data/Mono` 존재 | Unity Mono 계열 게임으로 판단할 근거 |
| 주 분석 후보 | `FTK_Data/Managed/Assembly-CSharp.dll`, 10,446,336바이트 | 게임 로직 조사 시작점. 실제 기능별 위치는 아직 미확인 |
| 관리 코드 헤더 | 해당 DLL의 PE CLI 디렉터리 RVA `0x2008`, 크기 72 | .NET 관리 코드 분석 경로 선택 근거 |
| 추가 코드 후보 | `Assembly-CSharp-firstpass.dll`, `Assembly-UnityScript.dll` 등 | 필요할 때 함께 조사 |
| 데이터 후보 | `StreamingAssets/db`, `mods`, `input.json`, Unity `.assets` 파일 | 이름만 확인. 내부 형식과 내용은 아직 미확인 |
| Node.js | 현재 에이전트 셸에서 `v24.22.1` 사용 가능 | 로컬 `.nvmrc`와 일치 |
| npm·REA·ILSpy CLI | 현재 셸 PATH에서 명령을 찾지 못함 | PC 전체에 미설치됐다는 뜻은 아님. 사용자 PowerShell에서도 확인 필요 |
| .NET SDK | `9.0.304`, `10.0.401` 확인 | 선택적 .NET 도구 사용 시 활용 가능. REA 자체 관리 코드 검사에는 불필요 |
| REA 클론 | `node_modules`, `dist/cli-runtime/cli.js` 없음 | 소스에서 바로 CLI를 실행할 준비는 안 됨 |
| REA MCP | 현재 대화의 도구 목록에 연결된 REA 도구 없음 | 우선 CLI로 시작 가능 |

확인한 `Assembly-CSharp.dll`의 SHA-256:

```text
8096783EC9D1DCD8635C99BA8D73B133D5B5AFAD440BA4EA3053CC86E96A0DAC
```

위 관찰은 PowerShell 파일 목록·헤더 읽기·해시 계산 결과입니다. 아직 REA Evidence를 생성하거나 CIL·메서드 본문을 검사하지 않았고, 게임도 실행하지 않았습니다. 복사본의 Steam 배포 출처와 빌드 버전도 검증하지 않았습니다.

## 2. 사용할 도구를 정하기

**첫 단계에 필요한 것은 Node.js·npm과 실행 가능한 REA입니다.** Ghidra·Java는 이 게임의 관리 코드 조사부터 시작할 때 필요하지 않습니다.

- **REA**: .NET 메타데이터, 타입·메서드, CIL 중간 언어, 호출·필드 접근 관계를 근거와 함께 조사합니다. 게임 DLL을 로드하여 실행하지 않습니다.
- **ILSpy — 선택 사항**: CIL을 읽기 쉬운 C# 형태로 디컴파일할 때 사용합니다. REA의 관리 코드 도구가 C# 전체 소스를 자동 생성하는 것은 아닙니다.
- **Ghidra — 나중에 필요할 때**: 관리 코드에서 호출하는 네이티브 플러그인 등의 내부 구현까지 조사해야 할 때 검토합니다.

복원된 C#은 원래 소스와 동일하다고 볼 수 없으며, 정적 호출 관계가 실제 플레이 중 실행을 증명하지는 않습니다.

## 3. PowerShell에서 Node.js·npm 확인

일반 PowerShell을 열고 확인합니다. 에이전트 전용 Node 경로는 일반 터미널 PATH와 다를 수 있습니다.

```powershell
Get-Command node.exe,npm.cmd,npx.cmd -ErrorAction SilentlyContinue
node --version
npm.cmd --version
```

로컬 소스 개발 기준은 Node.js `24.22.1`과 `npm@11.16.0`입니다. 명령을 찾지 못하면 기존 Node 설치와 PATH부터 확인하세요. 설치가 필요하면 [Node.js 공식 배포](https://nodejs.org/en/download)에서 저장소 요구 버전에 맞는 Node.js와 npm을 준비하고 PowerShell을 다시 엽니다. REA setup이 이를 대신 설치하지는 않습니다.

## 4. REA 준비 — 두 방법 중 하나 선택

### A. 패키지로 시작하기

로컬 REA 소스를 수정할 목적이 없다면 이 방법이 간단합니다. 게임 디렉터리가 아닌 신뢰하는 작업 위치에서 실행합니다.

```powershell
Set-Location 'E:\GitHub\rea'
npm.cmd install --global rea-agents
rea.cmd --help
rea.cmd inspect-managed-artifact --help
rea.cmd inspect-managed-members --help
```

패키지는 실행 시점의 배포 버전입니다. 이 문서의 로컬 소스 버전과 다를 수 있으므로 실제 `--help`를 기준으로 옵션을 확인합니다. 설치 후 명령을 못 찾으면 새 PowerShell에서 `npm.cmd config get prefix`로 설치 위치와 PATH를 확인합니다.

### B. 현재 클론을 빌드해서 사용하기

로컬 커밋을 기준으로 분석하고 싶을 때 선택합니다. 위의 고정 Node/npm 버전을 준비한 뒤:

```powershell
Set-Location 'E:\GitHub\rea'
npm.cmd ci
npm.cmd run build:cached
node '.\scripts\rea.mjs' --help
node '.\scripts\rea.mjs' inspect-managed-artifact --help
```

이 경로에서는 아래 예제의 `rea.cmd`를 `node 'E:\GitHub\rea\scripts\rea.mjs'`로 바꿉니다. 모든 서브모듈의 개별 엔진을 빌드할 필요는 없습니다.

## 5. 첫 분석: 핵심 DLL의 정체 확인

결과는 게임 복사본과 REA Git 저장소 밖에 둡니다. 아래 경로는 제안이며, 원하면 다른 위치로 바꿔도 됩니다. JSON 저장에 사용할 PowerShell 7의 `utf8`은 UTF-8이며, Windows PowerShell 5.1에서는 BOM이 포함될 수 있습니다.

```powershell
$utf8 = [System.Text.UTF8Encoding]::new($false)
[Console]::OutputEncoding = $utf8
$OutputEncoding = $utf8

$gameRoot = 'C:\Users\yudle\Documents\REA용\For The King'
$managedRoot = Join-Path $gameRoot 'FTK_Data\Managed'
$assemblyPath = Join-Path $managedRoot 'Assembly-CSharp.dll'
$reportRoot = 'C:\Users\yudle\Documents\REA용\ForTheKing-analysis'
New-Item -ItemType Directory -Path $reportRoot -Force | Out-Null

Get-FileHash -LiteralPath $assemblyPath -Algorithm SHA256

$artifactReport = Join-Path $reportRoot 'assembly-artifact.json'
if (Test-Path -LiteralPath $artifactReport) { throw '결과 파일이 이미 있습니다. 새 이름을 지정하세요.' }
rea.cmd inspect-managed-artifact $assemblyPath --format json |
    Out-File -LiteralPath $artifactReport -Encoding utf8
if ($LASTEXITCODE -ne 0) { throw 'REA 오류 결과를 확인하세요.' }
```

결과에서 어셈블리 식별 정보, 메타데이터 검사 범위, 오류·한계를 확인합니다. 이 단계가 성공하면 관리 코드 분석의 기본 준비가 된 것입니다. 실패하면 오류와 준비 상태를 먼저 확인하고, 바로 Ghidra 설치로 넘어가지 않습니다.

## 6. 읽기 쉬운 C#이 필요하면 ILSpy 추가

[ILSpy 공식 저장소](https://github.com/icsharpcode/ILSpy)의 배포 안내에서 Windows용 GUI를 준비합니다. 해당 배포판의 .NET 요구 사항을 확인하세요.

1. ILSpy에서 복사본의 `FTK_Data\Managed\Assembly-CSharp.dll`을 엽니다.
2. 필요하면 같은 `Managed` 폴더의 참조 DLL도 함께 엽니다.
3. 타입·메서드 이름을 살펴 첫 조사 주제를 좁힙니다.
4. 디컴파일 결과를 저장한다면 게임 폴더 밖의 분석 결과 폴더에 둡니다.

게임 EXE 실행, 디버거 연결, DLL 패치 없이 시작할 수 있습니다. ILSpy 열람·C# 복원 가능 여부는 이번 점검에서는 검증하지 않았습니다.

## 7. 관심 기능 하나를 선택해 REA로 추적

첫 주제는 예를 들어 전투 피해량, 명중 판정, 아이템 효과, 턴 순서 중 하나로 좁힙니다. 아직 실제 타입 이름은 조사하지 않았으므로 추측한 이름을 그대로 명령에 넣지 않습니다.

ILSpy 또는 REA 타입 목록에서 확인한 **정확한 전체 타입 이름**을 사용하면 범위를 줄일 수 있습니다. 아래 코드는 아직 실행할 수 있는 완성 명령이 아니라, 이름을 확인한 뒤 사용하는 예제입니다.

```powershell
$typeFullName = '여기에_확인한_전체_타입_이름'
$memberReport = Join-Path $reportRoot 'selected-members-summary.json'
$memberBundle = Join-Path $reportRoot 'selected-members.evidence.json'
if ((Test-Path -LiteralPath $memberReport) -or (Test-Path -LiteralPath $memberBundle)) {
    throw '결과 파일이 이미 있습니다. 새 이름을 지정하세요.'
}
rea.cmd inspect-managed-members $assemblyPath --type $typeFullName `
    --detail summary --evidence-bundle $memberBundle --format json |
    Out-File -LiteralPath $memberReport -Encoding utf8
if ($LASTEXITCODE -ne 0) { throw 'REA 오류 결과를 확인하세요.' }
```

요약만 AI에 전달하고, 전체 근거는 Evidence bundle에 보관합니다. 필요한 메서드·호출·필드만 후속 조회하는 절차는 [관리 코드 가이드](managed-code-analysis.md#cli-summary-and-saved-member-rows)를 따릅니다. 전체 DLL의 코드와 모든 결과를 한 번에 대화에 넣을 필요는 없습니다.

## 8. AI에서 직접 호출하고 싶다면 MCP 연결 — 선택 사항

CLI만으로도 분석할 수 있습니다. MCP는 AI가 REA를 직접 호출하게 만드는 추가 연결입니다.

REA를 준비한 뒤 `rea.cmd setup` 또는 소스 빌드의 `node 'E:\GitHub\rea\scripts\rea.mjs' setup`을 실행하고, Codex를 선택해 **변경할 설정 경로와 백업 계획을 검토한 후 승인**합니다. 이후 클라이언트를 재시작하고 REA 도구가 나타나는지 확인합니다. 이미 REA 플러그인으로 연결했다면 setup 등록을 중복해서 추가하지 않습니다.

## 준비 완료 체크리스트

- [x] 게임 복사본 경로와 Unity·관리 코드 파일 구조 확인
- [x] 핵심 DLL의 크기·PE CLI 헤더·SHA-256 기록
- [x] 일반 PowerShell에서 사용할 Node.js·npm 확인
- [x] 패키지 설치 또는 소스 빌드 중 하나로 REA 준비
- [x] `inspect-managed-artifact`로 실제 REA 분석 성공 확인
- [ ] 필요하면 ILSpy로 타입와 C# 코드 열람
- [ ] 첫 조사 기능 하나 선택
- [ ] 필요한 경우에만 MCP 연결

설치·빌드·MCP 설정 변경·REA 분석은 이번 준비 점검에서 수행하지 않았습니다. 게임 파일과 세이브 내용도 변경하지 않았습니다. 게임 바이너리·세이브·디컴파일 산출물은 REA 저장소에 커밋하지 않고 별도 분석 폴더에 보관합니다.

## 참고

- [REA .NET 관리 코드 분석](managed-code-analysis.md)
- [REA 설치와 설정](installation.md)
- [REA 디렉터리 조사](desktop-applications.md)
- [REA 원리 요약](rea-how-it-works-ko.md)
- [ILSpy 공식 프로젝트](https://github.com/icsharpcode/ILSpy)

후속 진행: REA 6.8.0 설치와 실제 분석이 완료됐습니다. 위 현황 표는 최초 점검 당시의 기록입니다. 현재 결과는 [첫 정적 분석 결과](for-the-king-initial-findings-ko.md)를 참고하세요.
