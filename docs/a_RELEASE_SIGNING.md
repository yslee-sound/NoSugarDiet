# Release Signing & Play App Signing Guide

최소/반복 방지를 위한 릴리즈 서명 운영 요약.

## 1. 환경 변수 (CI/로컬 동일)
| 변수 | 용도 |
|------|------|
| KEYSTORE_PATH | 업로드 키 JKS 절대 경로 |
| KEYSTORE_STORE_PW | Keystore 비밀번호 |
| KEY_ALIAS | alcoholictimeruploadkey |
| KEY_PASSWORD | 키 비밀번호 (store 비번과 동일해도 됨) |

---
```
1.1. Keytool 확인
PowerShell:
& "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -list -v -keystore "G:\secure\alcoholic-timer-upload.jks"

1.2. PowerShell:
- 제일 먼저 버전 변경할 것
app > build.gradle.kts
    versionCode = 2025100800 // YYYYMMDDxx
    versionName = "1.0.2"
    
- powershell.exe (관리자 아님) 실행
먼저 secure 폴더에 해당앱 폴더 생성
& "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -genkeypair -v -keystore "G:\secure\NoSugarDiet\nosugardiet-key.jks" -alias nosugardiet-alias -keyalg RSA -keysize 4096 -sigalg SHA256withRSA -validity 36500

- 드라이브 & 디렉터리 이동
G:
cd G:\Workspace\NoSugarDiet
(확인)
Get-ChildItem gradlew.bat
파일 목록에 gradlew.bat 보이면 OK.

- 서명 환경변수 (네 값으로 넣어, 따옴표 포함)
$env:KEYSTORE_PATH="G:/secure/NoSugarDiet/nosugardiet-key.jks"
$env:KEYSTORE_STORE_PW="****"
$env:KEY_ALIAS="nosugardiet-alias"
$env:KEY_PASSWORD="****"

- 서명 리포트 재확인 (선택이지만 3초)
.\gradlew.bat :app:signingReport

- 릴리즈 번들 빌드
.\gradlew.bat clean :app:bundleRelease

-산출물
app\build\outputs\bundle\release\app-release.aab

- Play Console 업로드

```

### 1.3 signingReport 오류(구성 캐시/데몬) 트러블슈팅
증상 예:
```
Could not load the value of field `dslSigningConfig` ... Class 'com.android.build.gradle.internal.dsl.SigningConfig$AgpDecorated' not found ...
```
조치 순서(Windows PowerShell):
```
G:
cd G:\Workspace\AlcoholicTimer

# 1) Gradle 데몬 중지
.\gradlew.bat --stop

# 2) 구성 캐시 정리(헬퍼 태스크)
.\gradlew.bat purgeConfigCache

# 3) 구성 캐시 비활성화로 1회 실행
.\gradlew.bat --no-configuration-cache :app:signingReport

# (대안) 환경 변수만 빠르게 확인
.\gradlew.bat :app:printReleaseSigningEnv
```
참고: 프로젝트는 기본적으로 구성 캐시를 사용(org.gradle.configuration-cache=true). signingReport는 AGP 내부 구현 이슈로 캐시 재사용 시 간헐적으로 실패할 수 있어 위 절차로 1회 강제 재구성/확인한다.


## 2. 새 업로드 키 생성 (최초 1회)
```
keytool -genkeypair -v -keystore upload-key.jks -storetype JKS -keyalg RSA -keysize 2048 -validity 9125 -alias upload
```
입력 항목: 비번, CN(이름), OU/Org(선택), City, State, Country Code(2자). 

보관: VCS 밖 안전한 경로(최소 2중 백업 + 비밀번호 메모).

## 3. 서명 강제 동작 (build.gradle.kts)
release 관련 태스크 (`bundleRelease`, `assembleRelease`, `publishRelease`) 실행 시 환경변수 없으면 즉시 실패 → 미서명 번들 업로드 사고 방지.

## 4. 빌드 & 검증
```
gradlew.bat :app:bundleRelease
```
(환경변수 세팅 없으면 실패해야 정상.)

### 서명 리포트
```
gradlew.bat :app:signingReport
```
출력 예 (요약):
```
Variant: release
Config: release
Store: G:\\SecureKeys\\upload-key.jks
Alias: upload
SHA1:  11:22:33:...:AA
SHA-256:  AA:BB:CC:...:FF
```
※ 출력이 안 나오면: Android Gradle Plugin 캐시/동기화 문제 → `gradlew.bat --stop` 후 `gradlew.bat purgeConfigCache` → `--no-configuration-cache` 로 1회 재실행.

## 5. Fingerprint 추출 (직접 keytool)
```
keytool -list -v -keystore G:\\SecureKeys\\upload-key.jks -alias upload -storepass <STORE_PW>
```
결과에서 SHA-1 / SHA-256 값을 복사 (Firebase / API Console 등록용).

## 6. Play App Signing 개념
| 구분 | 역할 |
|------|------|
| App Signing Key | Google 서버가 최종 배포에 사용하는 키 (Google 보관) |
| Upload Key | 개발자가 업로드하는 번들을 서명하는 키 (손상 시 교체 가능) |

### 활성화 (최초)
1. Play Console > 앱 > 릴리스 > 설정 > App Integrity (또는 App Signing) 진입
2. 안내에 따라 Play App Signing 활성화 (신규 앱이면 기본 활성 상태일 수 있음)
3. 업로드 키로 서명한 AAB 업로드 → Google 이 내부적으로 App Signing Key로 재서명 후 배포

### 업로드 키 교체 (유실/노출)
1. 새 업로드 키 생성
2. Play Console > App Integrity > 업로드 키 교체 요청 (등록된 개발자 계정 권한 필요)
3. 제공된 절차에 따라 새 키 업로드

## 7. CI 권장 체크
- 릴리즈 빌드 Job 시작 시 KEYSTORE_PATH 존재 확인 (없으면 fail)
- `gradlew :app:signingReport` SHA-256 를 로그에 (마스킹 해도 무방) 출력 → 추후 추적 용이
- AAB 산출 후 크기/해시 기록 (무결성 회귀 체크)

## 8. 비상 복구 시나리오
| 상황 | 대응 |
|------|------|
| Upload Key 유실 | 새 키 생성 후 업로드 키 교체 프로세스 진행 |
| App Signing Key 유실 | (Google 보관) 직접 손실 불가. 노출 의심 시 Google Play 지원문의 |
| 서명 안 된 번들 업로드 오류 | 환경변수 세팅 + 재빌드 |

## 9. 보안 Quick Tips
- Keystore는 절대 git add 금지 (.gitignore 이미 설정)
- 비밀번호 공용 채팅 공유 금지 → Secret Manager / CI Secret 사용
- 로컬 PC 교체 전 키 백업 확인 (클라우드 암호화 저장)

## 10. 최소 릴리즈 절차 (서명 포함)
1. 환경변수 세팅
2. `gradlew.bat :app:signingReport` 로 서명 정보 눈으로 확인
3. `gradlew.bat clean :app:bundleRelease`
4. 산출물: `app/build/outputs/bundle/release/app-release.aab`
5. Play Console 업로드 (versionCode 증가 확인)
6. 내부 테스트 → 프로덕션 승격

## 11. 아티팩트 네이밍 & 아카이빙 규칙
일관성 + 추적성 확보. 파일명 자체는 Play Console 동작과 무관하지만 내부 관리 표준 정의.

### 11.1 최종 릴리즈(배포용) 이름 패턴
- AAB: `alcoholic-timer-v<versionName>-<versionCode>.aab`
- mapping: `mapping-v<versionName>-<versionCode>.txt`

예) versionName=1.0.2, versionCode=2025100800 →
```
alcoholic-timer-v1.0.2-2025100800.aab
mapping-v1.0.2-2025100800.txt
```

### 11.2 후보/임시 빌드(Timestamp 포함)
- AAB: `alcoholic-timer-v<versionName>-<versionCode>-<yyyyMMdd-HHmm>.aab`
- mapping: `mapping-v<versionName>-<versionCode>-<yyyyMMdd-HHmm>.txt`

승인된 빌드 1개만 최종 패턴으로 rename 후 보관.

### 11.3 금지 / 비권장 패턴
- `app-release-...-signed.aab` : 모든 release 는 서명됨 → `-signed` 불필요
- `mapping.txt` : 버전 구분 불가 → 항상 versionName + versionCode 포함
- `final.aab`, `latest.aab` : 의미 소실 / 과거 추적 불가

### 11.4 Keystore 파일
- 운용 키: `upload-key.jks` (또는 `alcoholic-timer-upload-key.jks`)
- 로테이션/백업: `upload-key-YYYYMMDD.jks` (VCS 외부 Secure 저장소)
- `.bak`, `.old` 는 동일 디렉터리에 두지 말고 `SecureKeysArchive/` 로 이동

### 11.5 무결성 점검(선택)
빌드 산출물 중복 여부 확인:
```
Get-FileHash alcoholic-timer-v1.0.2-2025100800.aab -Algorithm SHA256
Get-FileHash mapping-v1.0.2-2025100800.txt -Algorithm SHA256
```
같은 버전의 다른 이름 파일이 존재하면 해시 비교 후 하나만 유지.

### 11.6 PowerShell Rename 예시
```
# AAB / mapping 표준화
Rename-Item "app-release-1.0.2-signed.aab" "alcoholic-timer-v1.0.2-2025100800.aab"
Rename-Item "mapping-1.0.2-2025100800.txt" "mapping-v1.0.2-2025100800.txt"

# Keystore 백업 분리
New-Item -ItemType Directory -Name "keystore-backup" -ErrorAction SilentlyContinue | Out-Null
Move-Item "alcoholic-timer-upload.jks.old" "keystore-backup" -ErrorAction SilentlyContinue
Move-Item "alcoholic-timer-upload.jks.bak" "keystore-backup" -ErrorAction SilentlyContinue
```

### 11.7 체크리스트 (릴리즈 아카이브 저장 시)
- [ ] AAB 이름 패턴 준수
- [ ] mapping 파일 동일 versionCode 대응 이름
- [ ] 해시(SHA-256) 기록(optional)
- [ ] Keystore 백업/운용 파일 구분

---

## UI 차별화 및 개선안 제안

앱의 UI가 기존 앱들과 유사하다는 피드백을 바탕으로, 다음과 같은 개선 방향을 제안합니다.

1. **컬러 팔레트 차별화**
   - 브랜드 아이덴티티를 강화할 수 있는 독특한 색상 조합을 선정합니다.
   - 기존 앱들과 겹치지 않는 메인/서브 컬러를 적용합니다.

2. **타이포그래피 개선**
   - 폰트 스타일, 크기, 자간 등을 조정하여 개성을 부여합니다.
   - 브랜드에 맞는 전용 서체 또는 변형 서체를 활용합니다.

3. **아이콘 및 일러스트 커스터마이징**
   - 앱 내 아이콘, 일러스트를 직접 제작하거나, 브랜드에 맞게 커스터마이징합니다.
   - 일관된 그래픽 스타일을 적용해 시각적 통일감을 줍니다.

4. **인터랙션 및 애니메이션 추가**
   - 버튼, 화면 전환 등에 부드러운 애니메이션을 적용해 사용자 경험을 개선합니다.
   - 터치 시 반응, 로딩 애니메이션 등 미묘한 인터랙션 효과로 앱의 완성도를 높입니다.

5. **레이아웃 및 컴포넌트 재구성**
   - 기존 앱들과 다른 레이아웃 구조(카드형, 그리드형, 리스트형 등)를 시도합니다.
   - 주요 기능의 배치, 접근 방식을 새롭게 설계합니다.

6. **브랜드 요소 강화**
   - 로고, 슬로건, 브랜드 컬러 등 브랜드 정체성을 UI 곳곳에 녹여냅니다.
   - 앱 내에서 브랜드 스토리나 메시지를 전달할 수 있는 공간을 마련합니다.

7. **접근성 및 사용성 개선**
   - 색약, 시각장애 등 다양한 사용자를 고려한 접근성 옵션을 제공합니다.
   - 직관적인 네비게이션과 명확한 피드백으로 사용성을 높입니다.

---

위 개선안을 바탕으로 UI 디자인을 리뉴얼하면, 기존 앱들과 차별화된 사용자 경험을 제공할 수 있습니다.

문의/확장 필요 시: CI 예시 스크립트, 다중 flavor 서명 등 추가 가능.
