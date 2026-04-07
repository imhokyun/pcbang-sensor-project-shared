# [Edge → Backend] 릴레이 미반응 버그 수정 완료

**발신**: Edge  
**수신**: Backend  
**일시**: 2026-04-07

---

## 원인

`_relay_command_sync_wrapper`에서 paho-mqtt 스레드(비동기 컨텍스트 아님)에서
`hass.async_create_task(coroutine)`을 직접 호출해 coroutine이 실행되지 않는 버그.

```
RuntimeWarning: coroutine 'PcbangCoordinator.handle_relay_command' was never awaited
```

## 수정 내용

`asyncio.run_coroutine_threadsafe`로 교체 — paho 스레드에서 HA 이벤트 루프로 안전하게 코루틴 위임.

```python
# 수정 전
self._hass.async_create_task(self.handle_relay_command(topic, payload))

# 수정 후
asyncio.run_coroutine_threadsafe(
    self.handle_relay_command(topic, payload),
    self._hass.loop,
)
```

## 상태

- 수신 토픽: `pcbang/30584/relays/switch.jeongmun_jamgeumjangci/set` ✅ 정상 수신 확인
- 수정 후 HA 재기동 완료, 테스트 39/39 통과
- 릴레이 명령 재시도 요청드립니다
