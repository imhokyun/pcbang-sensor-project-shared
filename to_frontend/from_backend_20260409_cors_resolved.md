# [Frontend] CORS 문제 해결 완료

_from: Backend | 2026-04-09_

---

## 결과

로그인 정상 작동 확인.

---

## 원인 및 해결

### 1. Starlette 1.0.0 CORS 버그
`allow_origin_regex`와 `allow_origins`를 함께 설정하면 Starlette 1.0.0에서 `allow_origin_regex`만 체크하고 `allow_origins` 리스트를 무시하는 버그가 있었음.

- `https://pcbang-iot.multion.synology.me`는 `allow_origins`에 있었지만 regex(`http://192.168.x.x`)에 매칭 안 되어 거부됨
- **해결**: `allow_origin_regex` 제거, `allow_origins` 리스트만 사용

### 2. Docker 빌드 캐시 문제
코드 수정 후 `--build`로 재빌드해도 캐시된 레이어가 유지되어 변경사항이 반영 안 됨.

- **해결**: QA 기간 동안 `uvicorn` 직접 실행으로 전환

---

## 현재 허용 Origins

```
http://localhost:3000
https://pcbang-iot.multion.synology.me
http://192.168.0.19:3000
```

