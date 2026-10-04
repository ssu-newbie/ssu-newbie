# 업로드 소스 검토

## 확인 기준

| 원본 | SHA-256 |
| --- | --- |
| `LobbyScaling_9.6.0.cs` | `8b6537f3e22fa9037deea47f9e8190aabd4439c61c518c21dad8999c312cae3c` |
| `DamageMeter_1.1.1.cs` | `66db3229210971758fe08b5c219abfd7da26c8866bed4eb7928ecac9ca4b0b5d` |

두 파일의 BepInPlugin 버전은 파일명과 일치한다. 아래 원본 위치는 업로드된 단일 파일 기준이다. 게임 어셈블리와 과거 로그를 직접 분석했다는 뜻은 아니다.

## 인수인계와 대조한 구현

- `SpawnerPatch`는 `Dictionary<Type, FieldInfo[]>`를 사용한다. 예외 시 한 번 경고하지만 다음 호출을 영구 중단하지 않는다. 원본 LobbyScaling 1031~1183행.
- 이 실패 국소화가 모드 전체에 적용된 것은 아니다. `SpeedPatch`, `BagPatch`, `StatPatch`에는 실패를 기억하고 이후 호출을 건너뛰는 플래그가 남아 있다.
- 활성 보스 능력치 경로는 `HookSpeed` → `UnitAvatar.OnStartServer` Postfix → `SpeedPatch.ApplyBossStats`다. 원본 484~512행과 1402~1446행에서 확인했다. `HookBoss`와 `BossPatch`도 파일에 남아 있지만 `Awake`의 패치 등록 목록에는 `HookBoss` 호출이 없다. 두 경로가 동시에 활성이라고 해석하지 않는다.
- `DamageMeter.Stats`는 리플렉션으로 게임의 `dealsStatistics`와 `dealsStatistics_LastLocation`을 읽는다. 이 코드에는 별도의 피해 집계 이벤트 훅이나 네트워크 송신 구현이 없다. 원본 DamageMeter 135~286행.
- `Rect.x`, `Rect.width` 등의 접근은 실제 소스에 있으나, 과거 대역 DLL과 그 예외 로그는 받지 못했다. 과거 `MissingFieldException`의 전체 재현 근거는 아직 인수인계 기록이다.

## 후속 수정 후보

이번 파일 분리에서는 아래 동작을 수정하지 않았다. 게임 환경에서 재현할 조건을 먼저 남긴다.

| ID | 소스에서 확인한 문제 | 확인할 조건 |
| --- | --- | --- |
| LS-01 | `ApplyConcurrent`는 전역 multiplier≤1, bonus≤0이면 스테이지 규칙을 읽기 전에 반환한다. 반면 훅 등록은 스테이지 `concurrent` 규칙도 고려한다. | 전역 1·0과 스테이지 `concurrent=2`만 설정. 해당 스테이지에서 상한이 바뀌는지 확인 |
| LS-02 | `HookSpeed` 활성 조건에 PlayerCriticalBonus, PlayerCriticalDamageBonus, PlayerIgnoreDefenseBonus, PlanetDamageBonus가 없다. | 다른 활성 조건을 모두 기본 중립값으로 놓고 치명타 보너스만 켠 경우. 현재 기본 luck·steal 값에서는 가려질 수 있음 |
| LS-03 | `Stages.Parse(StageRules.Value)` 호출은 `Awake`에만 있고 Rules 변경 이벤트가 없다. | 실행 중 Rules를 바꾼 뒤 재시작 전·후 적용 차이를 기록. 단순히 Apply가 모두 실시간 반영된다고 안내하지 않음 |
| LS-04 | Boss 설정의 설명 문자열은 스포너 필드 가산 방식을 설명하지만 활성 코드는 유닛에 적용한다. | 코드와 설정 설명을 맞추고, 기존 cfg의 값·단위도 함께 검토 |
| DM-01 | `Stats.Read`의 예외 처리에서 `broken=true`로 고정되어 이후 읽기가 빈 목록으로 종료된다. | 일시적 조회 예외 뒤 다음 프레임/스테이지에서 복구되는지 확인. 게임 타입 변경과 일시 오류를 구분 |

LS-01과 LS-02는 조건식과 실행 경로의 불일치를 소스에서 확인한 것이다. 해당 사용자 설정에서 실제 게임 증상을 재현한 결과는 아직 없다. 기본 설정으로 모든 기능이 실패한다고 확대 해석하지 않는다.

## 다음 변경 단위

1. 실제 참조 DLL로 현재 정리본을 빌드하고 기존 동작 기준 확보
2. LS-01 또는 DM-01 중 하나만 골라 재현 로그 확보
3. 그 결함에 해당하는 최소 변경과 실패·복구 검증
4. 플러그인 버전을 올린 별도 릴리스 후보로 정리

[프로젝트 소개](sephiria-mods.md) · [실행 검증 절차](sephiria-validation.md)
