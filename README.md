# 내 모델 Y 대시보드 (1단계)

아이폰 사파리에서 보는 개인용 Tesla 기록 대시보드입니다. 빌드 과정이나 설치할 것 없이 **파일 몇 개**로 되어 있습니다.

| 파일 | 역할 |
|---|---|
| `index.html` | 대시보드 전체 (화면 + 계산 + 차트) |
| `manifest.json`, `icon.svg` | 아이폰 "홈 화면에 추가" 시 앱처럼 열리게 함 |
| `.nojekyll` | GitHub Pages가 `.well-known` 폴더를 숨기지 않게 함 (지우지 마세요) |
| `data-example.json` | 데이터 형식 예시 |

지금은 **샘플(가짜) 데이터**로 화면이 채워져 있습니다. 실제 데이터는 2단계(서버 수집)에서 들어옵니다.

## 화면 구성
- **개요**: 현재 SOC·예상 주행거리, 기간별 주행거리·전비·충전량·흡혈 방전, 최근 7일 배터리 그래프
- **주행**: 주행별 전비(Wh/km), 주행 기록표
- **충전**: 주간 충전량, 완속/급속 구분 기록표
- **흡혈 방전**: 3시간 이상 주차한 구간의 %/일
- **배터리 열화**: 충전 완료 시 표시거리 ÷ SOC → 100% 환산거리 추세, 신차 기준 대비 %
- **설정**: 신차 기준거리(기본 511km)·배터리 용량(기본 75kWh), JSON 불러오기/내보내기

---

## 1. 무료로 인터넷에 올리기 (GitHub Pages)
이 주소가 그대로 Tesla에 등록할 **내 도메인** 역할도 합니다.

1. github.com 가입 → 오른쪽 위 **+** → **New repository** → 이름을 `내아이디.github.io` 로, **Public** 선택 → Create
2. 만든 저장소 화면에서 **Add file → Upload files** → 이 폴더의 파일을 모두 끌어다 놓기 → **Commit changes**
   - `.nojekyll`처럼 점으로 시작하는 파일은 윈도우 탐색기에서 숨겨질 수 있습니다. 안 올라가면 GitHub에서 **Add file → Create new file**로 이름만 `.nojekyll` 입력해 빈 파일을 만드세요.
3. 1~2분 뒤 `https://내아이디.github.io` 접속 → 아이폰 사파리 **공유 → 홈 화면에 추가**

> 저장소가 Public이라 **샘플 대시보드 화면은 누구나 볼 수 있습니다.** 실제 주행 데이터(위치 포함)를 이 공개 저장소에 올리지 마세요. 2단계에서 데이터는 비공개 서버에 둡니다.

## 2. Tesla 개발자 준비
### 2-1. 키 만들기 (PC에서 한 번)
윈도우 PowerShell에서 (Git for Windows를 설치하면 `openssl`이 함께 들어 있습니다):

```bash
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem
```

```bash
openssl ec -in private-key.pem -pubout -out com.tesla.3p.public-key.pem
```

- `private-key.pem` = **비밀키. 절대 GitHub에 올리지 말 것.** USB·비밀번호 관리자 등에 백업.
- `com.tesla.3p.public-key.pem` = 공개키. 저장소에 `.well-known/appspecific/` 폴더를 만들고 그 안에 올립니다.
  (GitHub에서 **Create new file** → 이름 칸에 `.well-known/appspecific/com.tesla.3p.public-key.pem` 입력 → 공개키 내용 붙여넣기)
- 확인: `https://내아이디.github.io/.well-known/appspecific/com.tesla.3p.public-key.pem` 이 브라우저에서 열리면 성공

### 2-2. 개발자 앱 등록
developer.tesla.com 에서 앱 등록 → Allowed Origin에 `https://내아이디.github.io` 입력 → 승인 대기(1~3일).
필요한 권한(scope)은 차량 정보·위치·충전 관련 읽기 권한 위주로 최소한만 고릅니다.

> `github.io` 하위 주소가 Tesla 등록에 그대로 통과되는지는 직접 확인이 필요합니다. 막히면 저렴한 개인 도메인(연 1~2만 원대)을 GitHub Pages에 연결하면 됩니다.

### 2-3. 차량에 가상 키 등록
아이폰 Tesla 앱이 로그인된 상태에서 사파리로 `https://tesla.com/_ak/내아이디.github.io` 접속 → 안내대로 차량 승인.
Fleet Telemetry 설정을 차에 보내려면 이 가상 키가 꼭 필요합니다.

---

## 3. 2단계(서버 수집)로 넘어갈 때
- Fleet Telemetry는 **차가 내 서버로 데이터를 밀어주는** 방식이라, 항상 켜져 있고 HTTPS가 되는 서버가 필요합니다 (무료 클라우드 VM 등). 폴링이 필요 없어 비용이 적게 듭니다.
- 텔레메트리는 "주행 1건" 같은 기록이 아니라 신호(SOC, 속도, 기어, 충전 상태, 주행거리계 등)의 흐름입니다. 서버가 이를 묶어서 주행·충전 기록으로 만듭니다.
- 서버가 `data/latest.json`을 아래 형식으로 만들어 두면, 대시보드가 **코드 수정 없이** 자동으로 읽습니다 (그때는 공개 GitHub Pages가 아니라 비공개 위치에서 대시보드를 함께 서빙).

### 데이터 형식 (`data-example.json` 참고)
```json
{
  "vehicle":   { "name": "내 모델 Y" },
  "drives":    [{ "start": "ISO시각", "end": "ISO시각", "start_soc": 78, "end_soc": 73.1, "distance_km": 21.4, "energy_kwh": 3.4 }],
  "charges":   [{ "start": "...", "end": "...", "type": "AC|DC", "location": "집", "start_soc": 42, "end_soc": 80, "energy_added_kwh": 30.8, "end_range_km": 385 }],
  "snapshots": [{ "t": "...", "soc": 73.1, "range_km": 352, "odometer_km": 41251 }]
}
```
- `energy_kwh`가 없으면 SOC 차이 × 배터리 용량으로 추정합니다.
- 흡혈 방전은 기록 사이의 3시간 이상 빈 구간에서 자동 계산되므로 따로 넣지 않아도 됩니다.
- 열화는 `end_soc`가 50% 이상인 충전의 `end_range_km`로 계산합니다.

## S3XY Commander
앱과 연동하지 않고 차 안에서 따로 쓰는 것으로 두었습니다. 대시보드는 Tesla 공식 API 데이터만 사용합니다.
