# mqtt_host 반환값 수정 요청

**발신**: edge | **수신**: backend | **날짜**: 2026-04-07

---

## 문제

`POST /api/v1/edge/register` 응답의 `mqtt_host` 값이 `"localhost"`로 반환됩니다.

HA Custom Component는 Docker 컨테이너(`pcbang-ha`) 안에서 실행되므로, `"localhost"`는 HA 컨테이너 자신을 가리켜 MQTT 연결에 실패합니다.

## 현재 응답

```json
{
  "success": true,
  "data": {
    "mqtt_host": "localhost",
    "mqtt_port": 1883,
    ...
  }
}
```

## 요청

`mqtt_host`를 `"pcbang-mosquitto"`로 변경해 주세요.

- `pcbang-mosquitto`는 `pcbang` Docker 네트워크의 컨테이너명
- HA(`pcbang-ha`)와 Mosquitto(`pcbang-mosquitto`)가 같은 `pcbang` 네트워크에 있어 Docker DNS로 해석 가능

## 수정 후 기대 응답

```json
{
  "success": true,
  "data": {
    "mqtt_host": "pcbang-mosquitto",
    "mqtt_port": 1883,
    "username": "30584",
    "password": "..."
  }
}
```

## 참고

백엔드 docker-compose의 `MQTT_HOST: mosquitto`는 backend→mosquitto 내부 연결용 설정이고,
edge→mosquitto 연결은 외부에서 접근하므로 컨테이너명(`pcbang-mosquitto`)을 사용해야 합니다.
