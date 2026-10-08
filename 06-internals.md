# Istio 내부 동작: CNI chaining · Pod 생성 · Pod 통신 + 실무 Q&A

> Istio가 코드 수정 없이 트래픽을 가로채는 비결은 **Pod 네트워크 네임스페이스 안의 iptables/nftables 규칙**이다. Sidecar 모드에서는 UID 1337(Envoy)이 아닌 프로세스의 트래픽을 Envoy(15001/15006)로 돌린다. Ambient 모드에서는 패킷 mark `0x539`가 없는 트래픽을 노드 ztunnel이 Pod 네임스페이스 안에 열어 둔 소켓(15001/15006/15008)으로 돌린다. 이 규칙을 **언제, 누가** 까는지가 CNI chaining과 Pod 생성 과정이다. 그리고 그 규칙 때문에 생기는 경계(UID 1337 우회, waypoint가 없으면 L7 무시 등)가 실무 장애의 단골 원인이다.

이 문서의 "관찰" 표시는 2026-10-08에 kind v0.33.0(Kubernetes v1.37.0, kindnet CNI) + Istio 1.31.1(ambient 프로필, `amb`=ambient 네임스페이스, `sc`=sidecar 네임스페이스) 환경에서 직접 확인한 결과다. 다른 CNI나 버전에서는 세부가 다를 수 있다.

## 1. CNI chaining

### CNI 기초

- kubelet(실제로는 containerd 같은 컨테이너 런타임)은 Pod sandbox를 만들 때 `/etc/cni/net.d`에서 **사전순으로 첫 번째** 설정 파일을 읽는다. 그 `.conflist`의 `plugins` 배열을 **순서대로** 실행한다(`ADD` 명령). 플러그인 바이너리는 `/opt/cni/bin`에 있다([CNI spec](https://www.cni.dev/docs/spec/)).
- 앞 플러그인의 결과(인터페이스, IP)가 다음 플러그인에 `prevResult`로 넘어간다. 그래서 IP를 할당하는 **기본(primary) CNI 뒤에** 부가 기능 플러그인을 이어 붙일 수 있다. 이것이 chaining이다.

### istio-cni가 하는 일 (관찰)

Istio 설치 전의 kindnet 설정은 `ptp`(veth + IP 할당)와 `portmap` 두 플러그인뿐이다. `istio-cni-node` DaemonSet이 뜨면 다음이 일어난다.

| 단계 | 관찰 내용 |
| --- | --- |
| 1. 바이너리 설치 | `/opt/cni/bin/istio-cni`(약 33MB) 복사 |
| 2. 설정 수정 | 우선순위가 가장 높은 기존 파일 `10-kindnet.conflist`의 `plugins` **맨 끝에** `{"type": "istio-cni", ...}`를 추가(별도 파일을 만들지 않음) |
| 3. 감시 | "initial installation complete, start watching for re-installation". 다른 주체가 설정을 덮어쓰면 다시 쓴다 |
| 4. 노드 레벨 규칙 | 노드 네임스페이스에 `ISTIO_POSTRT` 체인 추가: kubelet 프로브(노드에서 출발한 소켓)가 ambient Pod로 갈 때 출발지를 `169.254.7.127`로 SNAT |
| 5. ztunnel 연결 | ztunnel이 `/var/run/ztunnel/ztunnel.sock`(UDS)으로 접속해 "received hello from ztunnel" |

추가된 플러그인 설정(발췌):

```json
{
  "type": "istio-cni",
  "name": "istio-cni",
  "ambient_enabled": true,
  "cni_agent_run_dir": "/var/run/istio-cni",
  "enablement_selectors": [
    { "podSelector": { "matchLabels": { "istio.io/dataplane-mode": "ambient" } } },
    { "namespaceSelector": { "matchLabels": { "istio.io/dataplane-mode": "ambient" } },
      "podSelector": { "matchExpressions": [ { "key": "istio.io/dataplane-mode", "operator": "NotIn", "values": ["none"] } ] } }
  ],
  "exclude_namespaces": ["kube-system"],
  "native_nftables": false
}
```

- `istio-cni` **플러그인**(바이너리)은 거의 아무것도 하지 않는다. Pod가 생겼다는 사실을 같은 노드의 **node agent**(`istio-cni-node` Pod)에게 `/var/run/istio-cni` 소켓으로 알릴 뿐이다. 실제 규칙 설치는 agent가 한다([Traffic redirection](https://istio.io/latest/docs/ambient/architecture/traffic-redirection/)).
- agent는 hostPath로 `/opt/cni/bin`, `/etc/cni/net.d`, 호스트 `/proc`, `/var/run/netns`, `/var/run/ztunnel`, `/var/run/istio-cni`를 마운트한다. 권한은 `NET_ADMIN`, `NET_RAW`, `SYS_ADMIN`, `SYS_PTRACE`, `DAC_OVERRIDE`이고 PriorityClass는 `system-node-critical`이다(관찰). 호스트 `/proc`를 통해 대상 Pod 프로세스의 네트워크 네임스페이스로 들어간다.
- 기본 CNI를 **대체하지 않는다**. "The Istio CNI node agent _does not_ replace your cluster's existing CNI." — *[istio.io, CNI](https://istio.io/latest/docs/setup/additional-setup/cni/)*

### 순서 경쟁(race)과 대응

| 문제 | 대응 | 출처 |
| --- | --- | --- |
| 노드가 schedulable이 된 뒤 istio-cni가 준비되기 전에 Pod가 뜨면 리다이렉션 없이 시작된다(sidecar 우회) | sidecar 주입 시 `istio-validation` init container가 규칙이 제대로 깔렸는지 검사하고, 아니면 Pod 시작을 막는다 | [CNI](https://istio.io/latest/docs/setup/additional-setup/cni/) |
| 위 상태로 멈춘 Pod | repair 컨트롤러가 자동 처리. 1.21+ 기본은 `repairPods`(재시작 없이 제자리 수리) | 같은 문서 |
| 오토스케일러로 새 노드가 추가될 때 | `cni.istio.io/not-ready` NoSchedule taint를 걸어 두고 agent가 준비되면 istiod가 제거(옵션, `pilot.taint.enabled`) | 같은 문서 |
| 다른 도구(CNI 재설치 등)가 conflist를 덮어씀 | agent가 감시하다가 다시 씀(관찰: "start watching for re-installation") | 관찰 |

## 2. Pod 생성 시 일어나는 일

### 2-1. Sidecar 모드

```
kubectl apply (Deployment)
  └▶ ReplicaSet → Pod 생성 요청
       └▶ API server: MutatingAdmissionWebhook ── istiod(:15017) "sidecar-injector"
            · 네임스페이스 istio-injection=enabled 확인
            · istio-sidecar-injector ConfigMap 템플릿으로 PodSpec 변형
            · initContainers 추가: istio-init(또는 istio-validation) + istio-proxy(native sidecar)
            · liveness/readiness probe를 :15020/app-health/... 로 재작성
  └▶ 스케줄 → kubelet → 런타임이 sandbox 생성 → CNI ADD (ptp → portmap → [istio-cni])
  └▶ init 1: istio-init       → Pod netns에 iptables 규칙 설치 (NET_ADMIN 필요)
       또는  istio-validation → 규칙이 이미 있는지 검증만 (CNI 사용 시)
  └▶ init 2: istio-proxy (restartPolicy: Always) → pilot-agent 시작
            · 키·CSR 생성 → istiod CA 서명 → SDS로 Envoy에 인증서 전달
            · Envoy 부트스트랩 → istiod와 xDS 연결(CDS/LDS/EDS/RDS)
            · ready가 되어야 다음 단계 진행
  └▶ app 컨테이너 시작
```

관찰한 `sc/httpbin` Pod 스펙은 다음과 같다(CNI가 설치된 클러스터).

| 위치 | 이름 | 비고 |
| --- | --- | --- |
| initContainers[0] | `istio-validation` | UID 1337. args `istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT ... --run-validation --skip-rule-apply`. 규칙을 깔지 않고(`--skip-rule-apply`) **검증만** 한다 |
| initContainers[1] | `istio-proxy` | `restartPolicy: Always`, UID 1337 → Kubernetes native sidecar |
| containers[0] | `httpbin` | 앱 |

- CNI가 없는 클러스터(04 실습의 `demo` 프로필)에서는 `istio-init`이 이 자리에 들어가 직접 규칙을 깐다. 그래서 `NET_ADMIN`·`NET_RAW`가 필요하고, Pod Security `baseline`/`restricted`와 충돌한다([Application requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/)).
- 주입은 **Pod 생성 시점에만** 일어난다. Deployment 객체 자체는 바뀌지 않는다([Sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)).
- HTTP/gRPC 프로브는 pilot-agent의 `:15020/app-health/<container>/livez|readyz`로 재작성된다. 원래 프로브는 `ISTIO_KUBE_APP_PROBERS` 환경변수에 보관된다([Health checking](https://istio.io/latest/docs/ops/configuration/mesh/app-health-check/)). mTLS STRICT여도 kubelet 프로브가 실패하지 않는 이유다.
- 인증서(관찰): `istioctl proxy-config secret`의 `default` 인증서는 `NOT BEFORE 07:55:57`, `NOT AFTER 다음날 07:57:57`이다. 즉 **유효기간이 24시간**이고 시계 오차를 감안해 2분 앞당겨 발급된다. `ROOTCA`의 유효기간은 10년이다(자체 서명 기본 CA).

#### Sidecar Pod netns 규칙 (관찰, nftables 백엔드)

```
chain OUTPUT      → ISTIO_OUTPUT
chain ISTIO_OUTPUT
  tcp dport 15020 return                                  # pilot-agent 자체
  ip saddr 127.0.0.6 oifname "lo" return                  # Envoy inbound → 앱 전달 경로
  daddr != 127.0.0.1 oifname "lo" ... skuid 1337 jump ISTIO_IN_REDIRECT
  oifname "lo" skuid != 1337 return
  skuid 1337 return                                       # ★ Envoy(UID 1337)가 보낸 건 그대로 나감
  skgid 1337 return
  ip daddr 127.0.0.1 return
  jump ISTIO_REDIRECT                                     # 나머지 = 앱 트래픽 → :15001
chain ISTIO_REDIRECT    redirect to :15001
chain PREROUTING  → ISTIO_INBOUND
chain ISTIO_INBOUND
  tcp dport 15008/15020/15021/15090 return                # Istio 관리 포트는 예외
  jump ISTIO_IN_REDIRECT                                  # 그 외 들어오는 TCP → :15006
chain ISTIO_IN_REDIRECT redirect to :15006
```

- 이 규칙은 iptables-nft로 쓰였다. 그래서 노드에서 `nsenter -t <pid> -n iptables-save`로는 보이지 않았고 `nft list ruleset`으로 보였다(관찰). 디버깅할 때 백엔드(legacy/nft)를 확인해야 한다.

### 2-2. Ambient 모드

```
Pod 생성 (주입 없음, PodSpec 그대로)
  └▶ 런타임 sandbox 생성 → CNI ADD: ptp(IP 할당) → portmap → istio-cni 플러그인
       └▶ 플러그인: "amb/httpbin 생성됨" → node agent에 통지
  └▶ node agent(istio-cni-node)
       · enablement_selectors 확인 (ns/pod 라벨 istio.io/dataplane-mode=ambient)
       · Pod netns에 진입해 iptables 규칙 설치
       · Pod IP를 노드 ipset(istio-inpod-probes-v4)에 등록 → 프로브 SNAT 대상
       · ztunnel에 UDS로 "이 Pod의 netns fd" 전달
  └▶ ztunnel: "pod received, starting proxy"
       · 그 netns 안에 리스닝 소켓 생성 15001 / 15006 / 15008 / 15053
       · 해당 ServiceAccount 인증서를 (자기 ID로 인증한 뒤) istiod에서 받아 둠
  └▶ app 컨테이너 시작 (이미 리다이렉션 준비 완료)
```

- 이미 떠 있는 Pod에 나중에 라벨을 붙인 경우에는, agent가 API server 이벤트를 보고 같은 과정을 수행한다. 그래서 **재시작이 필요 없다**([Traffic redirection](https://istio.io/latest/docs/ambient/architecture/traffic-redirection/)).
- 관찰한 ambient Pod netns 리스너: 앱(`go-httpbin`)은 `*:8080`을 열고, 같은 netns에서 **ztunnel 프로세스**(노드의 ztunnel PID)가 `*:15001`, `*:15006`, `*:15008`, `127.0.0.1:15053`을 열고 있다. 다른 netns의 프로세스가 이 netns에 소켓을 만든 것이다.
- ztunnel은 한 프로세스 안에서 Pod마다 논리 프록시를 따로 띄운다("within the same process"). Pod별 워크로드 인증서는 ztunnel이 **자기 ID로 CA에 인증한 뒤 대신 요청**한다. CA는 그 노드에 없는 ID에 대한 요청을 거절한다([Ambient data plane](https://istio.io/latest/docs/ambient/architecture/data-plane/)).

#### Ambient Pod netns 규칙 (관찰)

```
*nat
ISTIO_OUTPUT
  -d 169.254.7.127/32 -p tcp -j ACCEPT                            # 프로브 응답
  ! -o lo udp/tcp dport 53, mark != 0x539 → REDIRECT :15053        # DNS 캡처
  -p tcp -m mark --mark 0x111/0xfff -j ACCEPT                      # 이미 처리된 커넥션
  ! -d 127.0.0.1 -o lo -j ACCEPT
  ! -d 127.0.0.1 -p tcp, mark != 0x539 → REDIRECT :15001           # ★ 앱 outbound → ztunnel
ISTIO_PRERT
  -s 169.254.7.127/32 -p tcp -j ACCEPT                             # kubelet 프로브는 통과
  ! -d 127.0.0.1 -p tcp ! --dport 15008, mark != 0x539 → REDIRECT :15006   # 평문 inbound
*mangle
  ISTIO_PRERT: mark 0x539 → CONNMARK 0x111 / ISTIO_OUTPUT: connmark 0x111 복원
```

- 판별 기준이 **UID가 아니라 패킷 mark `0x539`(=1337)** 이다. ztunnel은 자기 소켓에 `SO_MARK`를 설정해 재캡처를 피한다. 앱 프로세스가 mark를 붙이려면 `CAP_NET_ADMIN`이 필요하다.
- 공식 문서 예시는 inbound에 `mangle` 테이블의 `TPROXY`를 쓴다. 관찰한 1.31.1에서는 `nat` 테이블의 `REDIRECT`였다. 버전이나 설정(`inpodPortReuse` 등)에 따라 달라지는 것으로 보인다(**확인 필요**).
- 15008(HBONE)로 들어오는 트래픽은 리다이렉트하지 않는다. ztunnel이 그 포트를 직접 listen하고 있기 때문이다.

### 2-3. 비교 요약

| 항목 | Sidecar | Ambient |
| --- | --- | --- |
| PodSpec 변경 | webhook이 변형(init + sidecar 추가) | 없음 |
| 규칙 설치 주체 | `istio-init`(Pod 내 특권 init) 또는 istio-cni agent | istio-cni agent(필수) |
| 가로채기 판별 | 소켓 UID/GID 1337 | 패킷 mark 0x539 |
| 프록시 프로세스 위치 | Pod 안(istio-proxy 컨테이너) | 노드의 ztunnel이 Pod netns에 소켓만 생성 |
| 인증서 보유 | Pod 안 Envoy(자기 것만) | 노드 ztunnel(그 노드의 모든 ambient 워크로드 것) |
| 메시 편입 | Pod 재생성 필요 | 라벨만(재시작 불필요) |
| 프로브 처리 | pilot-agent `:15020`으로 재작성 | 노드에서 출발지 SNAT → `169.254.7.127` 허용 |

## 3. Pod 통신 시 일어나는 일

예: `curl` Pod → `http://httpbin:8000` (Service ClusterIP `10.96.x.x:8000` → Pod `:8080`)

### 3-1. Sidecar ↔ Sidecar

```
[curl Pod netns]
 ① curl(UID≠1337)이 DNS로 httpbin → ClusterIP:8000 으로 connect()
 ② nat OUTPUT → ISTIO_OUTPUT → ISTIO_REDIRECT → 127.0.0.1:15001 (원래 목적지는 SO_ORIGINAL_DST로 보존)
 ③ Envoy virtualOutbound(15001) → 원래 목적지 0.0.0.0:8000 리스너로 넘김(use_original_dst)
 ④ HTTP 파싱 → Route "8000"(RDS) → VirtualService 규칙(가중치·재시도·타임아웃)
 ⑤ Cluster outbound|8000||httpbin.sc.svc.cluster.local (CDS) → 엔드포인트 선택(EDS, LB)
     → 목적지 = Pod IP 10.244.0.11:8080  ※ kube-proxy의 ClusterIP DNAT를 쓰지 않음
 ⑥ Envoy(UID 1337)가 mTLS(ISTIO_MUTUAL, ALPN istio-*)로 접속 → skuid 1337 return → 그대로 나감
[노드 네트워크] veth → (ptp 라우팅) → 목적지 veth
[httpbin Pod netns]
 ⑦ PREROUTING → ISTIO_INBOUND → :15006 (virtualInbound)
 ⑧ Envoy가 mTLS 종단, 상대 SPIFFE ID 검증, AuthorizationPolicy 평가
 ⑨ 127.0.0.6을 출발지로 localhost:8080 앱에 전달(규칙 ip saddr 127.0.0.6 → return)
 ⑩ 앱이 받은 헤더: X-Forwarded-Client-Cert: By=spiffe://.../sa/httpbin;URI=spiffe://.../sa/curl
```

관찰 근거:
- `istioctl proxy-config listeners`: `0.0.0.0:15001`(outbound), `0.0.0.0:15006`(inbound, `Trans: tls; App: istio-http/1.1...` 등 필터 체인), `0.0.0.0:8000`(Route: 8000)
- `proxy-config clusters`: `httpbin.sc.svc.cluster.local 8000 outbound EDS`
- `proxy-config endpoints`: `10.244.0.11:8080 HEALTHY` ← Service의 targetPort와 Pod IP
- 앱이 받은 `X-Forwarded-Client-Cert`에 양쪽 SPIFFE ID가 담김

L7 기능(재시도·타임아웃·가중치)은 **출발지** Envoy에서, 인가는 **목적지** Envoy에서 처리된다. 그래서 L7 처리가 두 번 일어난다([Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/)).

### 3-2. Ambient (ztunnel만)

```
[curl Pod netns]
 ① curl → ClusterIP:8000
 ② nat ISTIO_OUTPUT: mark≠0x539 → REDIRECT :15001 (ztunnel이 이 netns에 연 소켓)
 ③ ztunnel(curl용 논리 프록시): 원래 목적지 확인 → Service → 엔드포인트 선택(L4 LB)
 ④ curl의 SPIFFE ID 인증서로 mTLS + HTTP/2 CONNECT 터널(HBONE) 생성
     → 목적지 10.244.0.9:15008, CONNECT 대상(hbone_addr) 10.244.0.9:8080
     ※ "the tunnels are in fact between the source and destination pods" — 암호화가 출발 Pod netns 안에서 일어남
[httpbin Pod netns]
 ⑤ :15008은 리다이렉트 예외 → ztunnel(httpbin용) 소켓이 바로 수신
 ⑥ mTLS 종단, L4 AuthorizationPolicy 평가 → 통과하면 localhost:8080으로 전달
```

ztunnel 접근 로그(관찰, 출발·도착 각각 한 줄):

```
info access connection complete src.workload="curl-..." src.identity="spiffe://cluster.local/ns/amb/sa/curl"
  dst.addr=10.244.0.9:15008 dst.hbone_addr=10.244.0.9:8080 dst.service="httpbin.amb.svc.cluster.local"
  dst.identity="spiffe://cluster.local/ns/amb/sa/httpbin" direction="outbound" bytes_sent=87 bytes_recv=346 duration="2ms"
```

- 같은 노드 안의 통신도 ztunnel을 거친다. 그래서 L4 정책과 텔레메트리가 노드 경계와 무관하게 동일하게 적용된다([Ambient data plane](https://istio.io/latest/docs/ambient/architecture/data-plane/)).
- ztunnel은 HTTP를 파싱하지 않는다. 그래서 로그에 HTTP 메서드나 상태 코드가 없고 바이트 수만 있다.

### 3-3. Ambient + waypoint

```
curl → ztunnel(출발) ──HBONE #1──▶ waypoint Pod :15008 (CONNECT 대상 = Service ClusterIP:8000)
                                     · HTTPRoute / L7 AuthorizationPolicy / 재시도 / 메트릭
                                     · 엔드포인트 선택
       waypoint ──HBONE #2──▶ 목적지 Pod :15008 → ztunnel(도착) → 앱
```

관찰(waypoint 적용 후 ztunnel 로그):

```
dst.addr=10.244.0.13:15008 dst.hbone_addr=10.96.226.117:8000 dst.workload="waypoint-..."
dst.identity="spiffe://cluster.local/ns/amb/sa/waypoint" direction="outbound"
```

- waypoint가 없을 때는 `hbone_addr`가 **Pod IP:8080**이다(ztunnel이 엔드포인트를 직접 고름). waypoint가 있으면 **Service ClusterIP:8000**이다(엔드포인트 선택을 waypoint에 맡김).
- waypoint 이후 목적지 ztunnel이 보는 출처는 waypoint ID다. 그래서 L4 정책에 waypoint SA를 허용해야 한다([04-usage.md](./04-usage.md)).

## 4. 실무 상황 Q&A

### Q1. VirtualService/HTTPRoute로 90:10 카나리를 걸었는데 비율이 안 맞거나 한쪽으로 몰린다

**결론**: "프록시가 새 설정을 아직 안 받았다", "트래픽이 그 프록시를 안 지난다", "측정이 틀렸다" 중 하나다. 순서대로 확인한다.

| 원인 | 확인 방법 | 비고 |
| --- | --- | --- |
| xDS 미동기화(istiod → 프록시) | `istioctl proxy-status`에서 해당 프록시가 SYNCED인지, `istioctl proxy-config routes <pod> -o json`에 가중치 90/10이 있는지 | 실습에서 적용 후 수 초간 이전 동작이 유지되는 것을 관찰 |
| 출발지 Pod에 sidecar가 없음 | `kubectl get pod`의 READY(2/2인지), 라벨 붙이기 전에 만든 Pod인지 | sidecar 모드의 가중치는 **출발지 Envoy**가 적용한다. 출발지에 Envoy가 없으면 kube-proxy 기본 분산(버전 무관)이 된다 |
| 가로채기 실패 | Pod netns의 규칙(`nft list ruleset` / `iptables-save`), istio-cni 로그, `istio-validation` 실패 여부 | CNI race로 규칙 없이 뜬 Pod |
| Pod IP·headless 직접 호출 | 호출 대상이 Service 이름인지 | Service를 거치지 않으면 라우팅 규칙이 매칭되지 않는다([Application requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/)) |
| **Ambient인데 waypoint가 없음** | `kubectl get httproute -o yaml`의 status | 관찰: `ResolvedWaypoints: istio.io/use-waypoint label missing ...; in ambient mode, route will not be respected`. 헤더 추가 필터가 waypoint를 붙인 뒤에야 적용됨 |
| **Ambient ingress 경유** | 요청 경로가 gateway → 서비스인지 | ingress 트래픽은 기본적으로 waypoint를 우회한다(04 실습에서 관찰). `istio.io/ingress-use-waypoint` 필요 |
| DestinationRule subset 누락 / 동시 적용 | `istioctl analyze` | subset이 없으면 503. make-before-break([05](./05-issues.md)) |
| 측정 오류 | 표본 수, 집계 방법 | 실습에서 응답 HTML에 Pod 이름이 두 번 나와 두 배로 집계되는 버그가 있었다. 200회 기준 179:21(≈90:10)이 정상 범위 |
| 커넥션 단위 분산 | 프로토콜이 TCP인지 | L4(TCP)·ztunnel만 쓰는 경우는 **커넥션 단위**로 분산된다. 커넥션을 오래 재사용하는 클라이언트는 한쪽으로 쏠려 보일 수 있다(**확인 필요**: 환경별 정량 검증 안 함) |

### Q2. Pod를 장악한 악성 프로세스가 Envoy를 우회해 평문으로 직접 보내면?

**결론**: Sidecar 모드에서 **우회는 가능하다**. 가로채기 규칙이 "UID 1337이 아닌 트래픽만" 잡기 때문에, UID 1337로 실행된 프로세스는 Envoy를 거치지 않는다. 막는 것은 **받는 쪽의 STRICT mTLS**(와 NetworkPolicy)다. 흔히 퍼진 "커널이 UID를 검사하니 우회가 원천적으로 불가능하다"는 설명은 틀렸다.

실험(관찰): `sc/curl` Pod에 UID를 바꾼 ephemeral container를 붙여 `httpbin`을 호출했다.

| 설정 | 앱 UID 1000 (일반) | UID 1337 (Envoy와 같은 UID) |
| --- | --- | --- |
| Sidecar, PERMISSIVE | 200, 응답에 `X-Forwarded-Client-Cert`(양쪽 SPIFFE ID) 있음 → mTLS 경유 | **200, XFCC 없음 → Envoy를 우회한 평문이 그대로 도착** |
| Sidecar, STRICT | 200 (mTLS) | **연결 실패(curl exit 56)** → 목적지가 평문을 거부 |
| Ambient | 200 (HBONE) | **200이지만 ztunnel이 가로채 HBONE으로 전송**(로그에 `src.identity=.../sa/curl`, `dst.addr=...:15008`) → 우회되지 않음 |

해설:
- 규칙은 **노드가 아니라 Pod 네트워크 네임스페이스에** 있다. 앱 컨테이너와 Envoy는 같은 netns를 공유한다. 규칙의 `skuid 1337 return`은 Envoy 자신의 트래픽을 다시 가로채지 않기 위한 예외인데, 같은 UID를 쓰는 다른 프로세스도 이 예외에 걸린다. 공식 문서가 "앱을 UID 1337로 실행하지 말라"고 하는 이유다([Application requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/)).
- 앱 컨테이너에 `NET_ADMIN`이 있으면 규칙 자체를 지울 수도 있다. 그래서 앱에는 이 권한을 주지 않는다(CNI를 쓰면 `istio-init`의 특권도 필요 없다).
- Ambient는 판별 기준이 **mark 0x539**이므로 UID를 바꿔도 우회되지 않았다. mark를 붙이려면 `CAP_NET_ADMIN`이 필요하다.
- 방어 원칙: (1) STRICT mTLS로 목적지가 평문을 거부, (2) 앱 컨테이너 UID ≠ 1337, `NET_ADMIN` 미부여(Pod Security `restricted`), (3) NetworkPolicy로 L3/L4 이중 방어, (4) egress 통제는 `REGISTRY_ONLY` + egress gateway([05](./05-issues.md)).

### Q3. Ambient(노드당 ztunnel 하나)의 설계상 약점은?

**결론**: 장애·자원·키 보관이 **노드 단위로 묶인다**는 것이 약점이다. 다만 "키 탈취" 측면은 sidecar보다 나은 점도 있어 일방적인 단점은 아니다.

| 관점 | Ambient의 약점 | 반대편 사실 / 완화 |
| --- | --- | --- |
| 장애 반경 | ztunnel이 크래시·버그·OOM이면 그 노드의 **모든 ambient Pod** 통신이 영향을 받는다 | 관찰: ztunnel Pod를 삭제했더니 새 ztunnel이 istio-cni로부터 snapshot을 받아 약 1초 안에 Pod별 프록시를 복구했다("received snapshot sent", "pod received, starting proxy"). 0.2초 간격 요청 134건 중 1건 실패. 공식 문서도 "briefly disrupt all ambient mesh traffic on the node"이며 주로 **long-running 커넥션**에 영향을 준다고 설명한다([Ambient Helm upgrade](https://istio.io/latest/docs/ambient/upgrade/helm/)) |
| 자원 경합(noisy neighbor) | 한 노드의 모든 Pod가 ztunnel의 worker 스레드(기본 2개)를 공유한다 | ztunnel은 L4만 처리해 가볍다. 무거운 L7은 waypoint로 분리되고, waypoint는 Deployment라 독립 스케일 가능([ztunnel ARCHITECTURE.md](https://github.com/istio/ztunnel/blob/master/ARCHITECTURE.md)) |
| 키 집중 | ztunnel 프로세스가 노드에 있는 모든 ambient 워크로드의 인증서·키를 메모리에 보유한다. ztunnel이 뚫리면 그 노드 워크로드 ID 전부가 노출된다 | CA는 그 노드에 없는 ID 요청을 거절하므로 노출 범위는 **그 노드**로 한정된다. 반대로 sidecar는 **앱 Pod가 뚫리면 그 Pod 키가 바로 노출**되지만, ambient에서는 앱 Pod가 뚫려도 메시 키에 접근할 수 없다([Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/), [Ambient data plane](https://istio.io/latest/docs/ambient/architecture/data-plane/)) |
| 업그레이드 | ztunnel 업그레이드는 revision을 써도 그 노드의 ambient 트래픽을 잠깐 끊는다("even with the use of revisions") | 업그레이드 전 노드 cordon/drain, 운영에서는 blue/green 노드 풀 권장([Ambient Helm upgrade](https://istio.io/latest/docs/ambient/upgrade/helm/)) |
| 기능 | `EnvoyFilter` 미지원, VM 미지원, L7은 waypoint 필수 | [03](./03-dataplane-modes.md) 참조 |

운영 포인트: ztunnel의 메모리·CPU·재시작 횟수와 `connection complete ... error=` 로그 비율을 노드별로 모니터링한다.

### Q4. Ambient에서 "GET만 허용" 정책을 걸었더니 GET까지 전부 막힌다

**결론**: waypoint 없이 `selector`로 붙인 정책에 HTTP 조건(`methods`, `paths` 등)을 넣었기 때문이다. ztunnel은 HTTP 조건을 집행할 수 없어서, ALLOW 정책에서 **그 규칙을 통째로 빼 버린다**. 그 결과 "ALLOW 정책은 있는데 매칭되는 규칙이 없음"이 되어 전부 거부된다.

관찰:
- 정책 status: `ztunnel does not support HTTP attributes (found: methods). In ambient mode you must use a waypoint proxy to enforce HTTP rules. Within an ALLOW policy, rules matching HTTP attributes are omitted. This will be more restrictive than requested.`
- ztunnel 로그: `error="connection closed due to policy rejection: allow policies exist, but none allowed"`, curl은 exit 56(연결 리셋)
- 해결: waypoint를 배포하고 정책을 `targetRefs`(Service 또는 waypoint Gateway)로 붙인다([04-usage.md](./04-usage.md)). DENY 정책에서는 반대로 HTTP 조건이 빠져 더 넓게 거부될 수 있으므로 status를 반드시 확인한다(**확인 필요**: DENY 동작은 직접 검증하지 않음).

### Q5. Ambient로 편입했더니 NetworkPolicy가 있는 Pod의 헬스체크가 실패한다

**결론**: ambient는 kubelet 프로브의 출발지를 `169.254.7.127`로 SNAT한다. 그래서 ingress를 막는 NetworkPolicy가 있으면 이 주소를 허용해야 한다.

- 관찰: 노드 netns에 `-A ISTIO_POSTRT -p tcp -m owner --socket-exists -m set --match-set istio-inpod-probes-v4 dst -j SNAT --to-source 169.254.7.127` 규칙이 있다. Pod netns에는 `-A ISTIO_PRERT -s 169.254.7.127/32 -p tcp -j ACCEPT` 규칙이 있다.
- 공식 문서는 IPv4 `169.254.7.127`과 IPv6 `fd16:9254:7127:1337:ffff:ffff:ffff:ffff`를 모두 허용하라고 안내한다([Ambient and NetworkPolicy](https://istio.io/latest/docs/ambient/usage/networkpolicy/)).
- 또 NetworkPolicy가 HBONE 포트 **15008**을 막으면 메시 트래픽이 끊긴다. 모든 앱 Pod에 대해 15008 inbound/outbound를 허용해야 한다([ztunnel ARCHITECTURE.md](https://github.com/istio/ztunnel/blob/master/ARCHITECTURE.md)).

### Q6. Sidecar 모드에서 init container의 네트워크 호출(DB 마이그레이션, 설정 다운로드)이 실패한다

**결론**: Istio CNI를 쓰면 리다이렉션 규칙이 **Pod 네트워크 설정 시점(init container보다 먼저)** 에 깔린다. 그런데 Envoy(istio-proxy)는 아직 준비되지 않았으므로 init container의 outbound가 갈 곳이 없다.

- 대응([CNI](https://istio.io/latest/docs/setup/additional-setup/cni/)): (1) 해당 init container를 UID 1337로 실행(규칙 예외 적용), (2) `traffic.sidecar.istio.io/excludeOutboundIPRanges` 또는 `excludeOutboundPorts` 어노테이션으로 대상 제외. 둘 다 그 트래픽이 메시 보호(mTLS·정책) 밖으로 나간다는 뜻이므로 범위를 최소화한다.
- native sidecar(1.27+ 기본)에서는 istio-proxy가 initContainers 목록에 들어간다. 그래서 **istio-proxy보다 뒤에 오는** init container는 Envoy가 준비된 뒤 실행된다(관찰한 스펙에서 istio-proxy는 initContainers[1]). 앞에 있는 init container에만 위 문제가 남는다(**확인 필요**: 사용자 init container가 주입 후 어느 위치에 놓이는지는 직접 검증하지 않음).

### Q7. Pod가 메시에 들어갔는지 어떻게 빠르게 확인하나

| 모드 | 확인 |
| --- | --- |
| Sidecar | READY `2/2`, `kubectl get pod -o jsonpath='{.spec.initContainers[*].name}'`에 `istio-proxy`, `istioctl proxy-status`에 등장 |
| Ambient | `istioctl ztunnel-config workloads`의 PROTOCOL이 `HBONE`, ztunnel 로그 `pod received, starting proxy`, istio-cni 로그 |
| 공통 | 목적지 앱이 받은 헤더에 `X-Forwarded-Client-Cert`(sidecar/waypoint 경유 HTTP), ztunnel `access` 로그의 `src.identity` |

## 출처

- https://www.cni.dev/docs/spec/
- https://istio.io/latest/docs/setup/additional-setup/cni/
- https://istio.io/latest/docs/ambient/architecture/traffic-redirection/
- https://istio.io/latest/docs/ambient/architecture/data-plane/
- https://istio.io/latest/docs/ambient/architecture/hbone/
- https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/
- https://istio.io/latest/docs/ops/configuration/mesh/app-health-check/
- https://istio.io/latest/docs/ops/deployment/application-requirements/
- https://istio.io/latest/docs/ambient/usage/networkpolicy/
- https://istio.io/latest/docs/ambient/usage/waypoint/
- https://istio.io/latest/docs/overview/dataplane-modes/
- https://github.com/istio/ztunnel/blob/master/ARCHITECTURE.md
- https://istio.io/latest/docs/ambient/upgrade/helm/

---
*조사 기준일: 2026-10-08. "관찰"은 kind v0.33.0 / Kubernetes v1.37.0 / kindnet / Istio 1.31.1에서 직접 확인한 결과이며 CNI·커널·버전에 따라 규칙 형태(iptables vs nftables, REDIRECT vs TPROXY)가 다를 수 있다. Q&A의 비교 실험은 단일 노드, 짧은 요청 기준.*
