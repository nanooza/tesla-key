# Tesla Tasker Plugin 설정 상세 가이드 (한글)

이 저장소는 **Android Tasker로 테슬라 차량을 제어**하기 위한 Tesla Fleet API 연동용
OAuth 중계(relay) 사이트입니다. GitHub Pages 위에서 동작하며, 두 가지 역할을 합니다.

1. **OAuth 콜백 중계** — 테슬라 로그인 후 돌아오는 인증 코드(`code`)를
   `teslatasker://callback` 로 넘겨 Tasker 플러그인 앱으로 전달합니다.
   (`callback/index.html`, `callback.html`)
2. **3rd‑party 공개키 호스팅** — 차량에 등록할 가상 키(virtual key)의 공개키를
   `.well-known/appspecific/com.tesla.3p.public-key.pem` 경로로 공개합니다.

> ⚠️ **사전 안내**
> - 이 가이드는 동영상(<https://youtu.be/GeJd0dAw3uw>)에 대응하는 **글 설명서**입니다.
>   동영상이 접근 차단(403)되어 자동 추출이 안 되었기 때문에, 이 저장소의 실제 코드와
>   Tesla Fleet API 공식 절차를 기준으로 작성했습니다. 동영상 화면과 메뉴 위치가
>   조금 다를 수 있으니 함께 참고하세요.
> - 가상 키(차량 명령) 기능은 **테슬라 계정 / 차량 / 지역 정책**에 따라 제한될 수 있습니다.

---

## 0. 준비물 (Checklist)

| 항목 | 설명 |
|------|------|
| 테슬라 계정 | 차량이 등록되어 있고 모바일 앱으로 로그인되는 계정 |
| 테슬라 차량 | 가상 키(Fleet API 명령)를 지원하는 차량 |
| Android 휴대폰 | **Tasker** 앱 + **Tesla Tasker Plugin** 설치 |
| 테슬라 모바일 앱 | 가상 키 페어링 시 필요 (차량 옆에서 진행) |
| GitHub 계정 | 이 저장소를 본인 계정으로 배포하기 위함 |
| PC (선택) | `openssl`, `curl` 사용 (키 생성·등록) |

---

## 1단계 — 저장소를 내 GitHub Pages로 배포

> 이미 `nanooza/tesla-key` 가 배포되어 있다면, 본인 차량/계정에 맞게
> **공개키만 교체**해서 그대로 쓰면 됩니다. 처음부터 직접 만들려면 아래대로 Fork 하세요.

1. 이 저장소를 본인 계정으로 **Fork** 합니다.
2. 저장소 → **Settings → Pages** 로 이동합니다.
3. **Build and deployment → Source** 를 **GitHub Actions** 로 설정합니다.
   (이 저장소에는 `.github/workflows/pages.yml` 배포 워크플로가 이미 들어 있습니다.)
4. `main` 브랜치에 푸시하면 자동으로 배포됩니다. 배포 후 주소는 보통:

   - 사이트: `https://<내아이디>.github.io/tesla-key/`
   - 콜백:   `https://<내아이디>.github.io/tesla-key/callback`
   - 공개키: `https://<내아이디>.github.io/tesla-key/.well-known/appspecific/com.tesla.3p.public-key.pem`

5. 브라우저로 위 **공개키 주소**를 직접 열어 `-----BEGIN PUBLIC KEY-----` 가 보이는지
   꼭 확인하세요. (404 나면 다음 단계 진행 불가)

> 📌 **중요 — 도메인/경로 주의**
> 테슬라는 가상 키 공개키를
> `https://<등록도메인>/.well-known/appspecific/com.tesla.3p.public-key.pem`
> **정확히 그 경로**에서 가져갑니다.
> GitHub *프로젝트* 페이지는 주소에 `/tesla-key/` 가 붙기 때문에, 테슬라에 도메인을
> `<내아이디>.github.io` 로 등록하면 키 경로가 맞지 않습니다.
> 다음 중 하나로 해결하세요.
> - **(권장)** 별도의 사용자 페이지 저장소 `<내아이디>.github.io` 를 만들어 공개키를
>   **루트**의 `.well-known/...` 에 두고, 그 도메인을 테슬라에 등록.
> - 또는 이 저장소에 **커스텀 도메인(CNAME)** 을 연결해 루트에서 `.well-known` 이
>   서비스되게 한 뒤 그 도메인을 등록.

---

## 2단계 — Tesla 개발자(앱) 등록

1. <https://developer.tesla.com> 접속 → 테슬라 계정으로 로그인.
2. **새 애플리케이션(App) 생성**.
3. 다음 값을 입력/선택합니다.

   | 항목 | 입력 값 |
   |------|---------|
   | **Allowed Origin** | `https://<내아이디>.github.io` |
   | **Redirect URI / Callback** | `https://<내아이디>.github.io/tesla-key/callback` |
   | **Scopes (권한)** | 차량 제어용: `openid`, `offline_access`, `vehicle_device_data`, `vehicle_cmds`, `vehicle_charging_cmds` 등 필요한 항목 |
   | **Domain (가상 키용)** | 공개키가 `.well-known` 루트에서 보이는 도메인 (1단계 📌 참고) |

4. 생성 후 화면에 표시되는 **Client ID** 와 **Client Secret** 을 안전하게 메모합니다.
   (Tasker 플러그인 설정에 입력)

> Redirect URI는 **글자 하나까지 정확히** 일치해야 합니다. 이 저장소의 콜백 페이지는
> 받은 `code` 와 `state` 를 `teslatasker://callback?code=...&state=...` 로
> 다시 던져 앱이 받도록 만들어져 있습니다.

---

## 3단계 — 가상 키(키 페어) 생성

PC 터미널에서 **EC P-256(prime256v1)** 키 쌍을 만듭니다.

```bash
# 비공개 키 (절대 외부 노출 금지 — 앱/안전한 곳에 보관)
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem

# 공개 키 (저장소에 올릴 파일)
openssl ec -in private-key.pem -pubout -out com.tesla.3p.public-key.pem
```

만든 **공개키 파일**의 내용을 저장소의

```
.well-known/appspecific/com.tesla.3p.public-key.pem
```

에 덮어쓰고 `main` 에 커밋·푸시합니다. 배포 후 1단계의 공개키 주소로 다시 확인합니다.

> 🔐 **비공개 키(private-key.pem)** 는 Tasker 플러그인 앱에서 차량 명령을 서명할 때
> 사용됩니다. 앱 안내에 따라 입력/보관하고, 깃 저장소에는 **절대 올리지 마세요.**

---

## 4단계 — 파트너 계정(Partner Account) 등록

테슬라가 내 도메인을 신뢰하도록 한 번 등록하는 과정입니다. (PC에서 `curl` 사용)

1. **파트너 토큰** 발급 (client_credentials):

   ```bash
   curl --request POST \
     'https://fleet-auth.tesla.com/oauth2/v3/token' \
     --header 'Content-Type: application/x-www-form-urlencoded' \
     --data-urlencode 'grant_type=client_credentials' \
     --data-urlencode 'client_id=<CLIENT_ID>' \
     --data-urlencode 'client_secret=<CLIENT_SECRET>' \
     --data-urlencode 'scope=openid vehicle_device_data vehicle_cmds vehicle_charging_cmds' \
     --data-urlencode 'audience=https://fleet-api.prd.na.vn.cloud.tesla.com'
   ```

   > `audience` 는 **지역**에 맞춰 바꾸세요.
   > - 북미/아태: `https://fleet-api.prd.na.vn.cloud.tesla.com`
   > - 유럽/중동/아프리카: `https://fleet-api.prd.eu.vn.cloud.tesla.com`
   > 한국은 보통 **na**(아태 포함) 서버를 사용합니다.

2. 응답의 `access_token` 으로 **도메인 등록**:

   ```bash
   curl --request POST \
     'https://fleet-api.prd.na.vn.cloud.tesla.com/api/1/partner_accounts' \
     --header 'Authorization: Bearer <PARTNER_ACCESS_TOKEN>' \
     --header 'Content-Type: application/json' \
     --data '{"domain": "<공개키가-루트에-있는-도메인>"}'
   ```

   성공하면 등록된 공개키 정보가 응답으로 돌아옵니다.

---

## 5단계 — 차량에 가상 키 등록(페어링)

> 이 단계는 **차량 근처에서**, 휴대폰에 **테슬라 앱이 설치된 상태**로 진행합니다.

1. 휴대폰 브라우저(또는 테슬라 앱)에서 아래 주소를 엽니다.
   (`<도메인>` 은 4단계에서 등록한 도메인)

   ```
   https://tesla.com/_ak/<도메인>
   ```

   예: `https://tesla.com/_ak/example.github.io`

2. 테슬라 앱이 열리며 **“가상 키 추가/페어링”** 화면이 나옵니다.
3. 대상 차량을 선택하고 안내에 따라 **승인** 합니다.
4. 차량의 **잠금/제어** 명령이 동작하려면 가상 키가 차량에 등록되어 있어야 합니다.

---

## 6단계 — Tasker 플러그인에서 OAuth 로그인

1. Android에 **Tasker** 와 **Tesla Tasker Plugin** 을 설치합니다.
2. 플러그인 설정 화면에 다음을 입력합니다.
   - **Client ID**, **Client Secret** (2단계)
   - 필요 시 **Redirect URI**: `https://<내아이디>.github.io/tesla-key/callback`
   - **비공개 키(private-key.pem)** (3단계, 명령 서명용)
3. **로그인(Authorize)** 을 누르면 브라우저(크롬 커스텀 탭)로 테슬라 로그인 창이 뜹니다.
4. 로그인·권한 동의 후, 이 저장소의 콜백 페이지가
   `code`/`state` 를 받아 `teslatasker://callback` 으로 앱에 되돌려줍니다.
   → 앱이 자동으로 토큰을 교환하고 로그인 완료됩니다.

> 💡 안드로이드 크롬에서는 콜백이 `intent://...;scheme=teslatasker;...` 형태로
> 처리되어, 새 화면을 만들지 않고 **기존 앱 화면(onNewIntent)** 으로 코드가 전달됩니다.
> (`callback.html` 의 `launchFlags=0x24000000` 주석 참고)

---

## 7단계 — Tasker 동작(태스크) 만들기

로그인이 끝나면 플러그인의 **액션(Action)** 을 Tasker 태스크에 추가해 차량을 제어합니다.
일반적으로 제공되는 명령 예시:

- 차량 깨우기(wake), 도어 잠금/해제
- 공조(에어컨) 켜기/끄기, 온도 설정
- 충전 시작/중지, 충전 한도 설정
- 트렁크/프렁크, 경적/라이트 등

> 명령마다 2단계에서 부여한 **scope(권한)** 가 필요합니다. 권한이 부족하면 다시
> 로그인하여 동의 범위를 넓혀야 할 수 있습니다.

---

## 문제 해결 (Troubleshooting)

| 증상 | 원인 / 해결 |
|------|-------------|
| 공개키 주소가 **404** | Pages 배포 미완료 또는 경로 오류. 1단계 주소 다시 확인, `_config.yml` 의 `.well-known` 포함 확인 |
| 파트너 등록/가상 키 추가 실패 | 테슬라가 보는 도메인 루트의 `.well-known` 경로가 안 맞음 → 1단계 📌 (사용자 페이지 또는 커스텀 도메인) |
| 로그인 후 앱으로 안 돌아옴 | Redirect URI 불일치, 또는 콜백 페이지가 `teslatasker://` 로 못 넘김. 2단계 URI 정확히 일치 확인 |
| “인증 코드를 찾을 수 없습니다” | 콜백에 `code` 파라미터가 없음. 로그인 과정 중단/취소 여부 확인 후 재시도 |
| 차량 명령은 되는데 데이터만 안 옴(또는 반대) | scope 부족 → 필요한 권한 추가 후 재로그인 |
| 명령 시 서명 오류 | 비공개 키(3단계)와 차량에 등록된 공개키(5단계)가 **같은 쌍**인지 확인 |

---

## 참고 링크

- Tesla Fleet API 개요: <https://developer.tesla.com/docs/fleet-api>
- 가상 키 개발자 가이드: <https://developer.tesla.com/docs/fleet-api/virtual-keys/developer-guide>
- 서드파티 토큰(인증): <https://developer.tesla.com/docs/fleet-api/authentication/third-party-tokens>
- 차량 명령 엔드포인트: <https://developer.tesla.com/docs/fleet-api/endpoints/vehicle-commands>
