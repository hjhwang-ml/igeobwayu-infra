# 4. Redis 서버 (Valkey) — redis (.108)

Rocky 10 기본 저장소에는 Redis 대신 **Valkey**(Redis 호환 포크)가 있습니다. 설치 버전 `valkey-8.0.x` (Redis 7.2 호환). 프로토콜·명령어·`redis-py` 라이브러리를 그대로 씁니다. `valkey-compat-redis`는 저장소에 없어서 `redis-cli` 대신 `valkey-cli`를 씁니다.
발표 자료에서는 "Redis (Valkey)"로 표기합니다.

## 4-1. 설치

```bash
dnf -y install valkey
valkey-server --version
```

## 4-2. 설정 `/etc/valkey/valkey.conf`

```bash
sed -i 's/^bind .*/bind 0.0.0.0/' /etc/valkey/valkey.conf
sed -i 's/^protected-mode .*/protected-mode yes/' /etc/valkey/valkey.conf
sed -i 's/^# *requirepass .*/requirepass <PASSWORD>/' /etc/valkey/valkey.conf

# 캐시 용도: 메모리 상한 + 오래 안 쓴 키부터 지움
cat >> /etc/valkey/valkey.conf <<'EOF'
maxmemory 512mb
maxmemory-policy allkeys-lru
EOF

grep -E '^(bind|protected-mode|requirepass|maxmemory)' /etc/valkey/valkey.conf
```

- `bind 0.0.0.0`이라 모든 NIC에서 받지만, 방화벽(4-3)에서 내부망만 허용하고 비밀번호도 걸려 있습니다.
- `protected-mode yes`는 비밀번호가 있으면 외부 접속을 막지 않습니다.

## 4-3. 실행·방화벽

```bash
systemctl enable --now valkey
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" port port="6379" protocol="tcp" accept'
firewall-cmd --reload
```

## 4-4. 확인 (WAS에서)

```bash
dnf -y install valkey          # valkey-cli 만 쓰려고 (서버는 안 켬)
valkey-cli -h 192.168.56.108 -a '<PASSWORD>' ping                                # PONG
valkey-cli -h 192.168.56.108 -a '<PASSWORD>' --scan --pattern 'search:*' | head
valkey-cli -h 192.168.56.108 -a '<PASSWORD>' hgetall stats:cache                 # 캐시 적중 통계
```

## 4-5. 서비스에서 쓰는 키 (WAS `cache.py` 기준)

| 키 | 자료형 | TTL | 용도 |
|---|---|---|---|
| `search:{모드}:{종류}:{플랫폼}:{검색어}` | String(JSON) | 1시간 | 검색 결과 캐시 (검색어는 공백 정리·소문자) |
| `similar:*` | String(JSON) | 1시간 | 작품별 비슷한 작품 |
| `user:{세션}:recommend` | String(JSON) | 1시간 | 세션별 추천 (새 작품을 보면 바로 삭제 → 다시 계산) |
| `user:{세션}:recent_view` | List | 7일 | 최근 본 작품 (LREM → LPUSH → LTRIM → EXPIRE, 최대 10개) |
| `user:{세션}:recent_search` | List | 7일 | 최근 검색어 |
| `content:{tv/movie}:{id}` | Hash | 24시간 | 사이드 패널용 제목·포스터 |
| `rate:{IP}:{yyyyMMddHHmm}` | String(숫자) | 60초 | IP별 분당 요청 제한 |
| `stats:cache` | Hash | 없음 | 캐시 종류별 hit/miss 횟수 (발표용) |

- 세션은 로그인이 아니라 쿠키 `ott_sid`(첫 방문 후 30일)로 구분합니다.
- TTL 값은 WAS `config.ini`의 `[cache]` 절에서 바꿉니다 → [06-was.md](06-was.md)
- 카프카 적재 컨슈머가 한동안 새 메시지가 없으면 `clear_cache.py`로 `search:*`, `user:*:recommend`, `similar:*`를 지웁니다 (새 작품·바뀐 플랫폼이 바로 보이게). Redis가 죽어 있어도 적재는 계속되고, 캐시는 1시간 뒤 저절로 만료됩니다.
- 효과(실측): 검색 84ms → 4ms (캐시 MISS → HIT)
- Redis에는 캐시·세션만 있고 원본 데이터가 없어서 이중화·백업은 하지 않습니다. (※ 확인: Redis가 죽었을 때 WAS가 오류 없이 DB에서 바로 읽는지 — 질문이 나올 수 있음)
