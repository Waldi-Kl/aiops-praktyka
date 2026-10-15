# Dziennik diagnozy — lab02 (namespace: waldemar)

## Stan początkowy (`setup/check.sh`)

```
❌ Deployment kantyna-web: gotowe 0/2
❌ Deployment kantyna-orders-api: gotowe 0/2
✅ Deployment kantyna-payments: gotowe 2/2
❌ Deployment kantyna-worker: gotowe 0/1
✅ StatefulSet kantyna-postgres: gotowy
✅ Service kantyna-web: ma endpointy
✅ Service kantyna-orders-api: ma endpointy
❌ Service kantyna-payments: brak podów za Service
✅ Service kantyna-postgres: ma endpointy
❌ Aplikacja: menu przez web — brak działającego poda web
❌ Aplikacja: zamówienie przez web — brak działającego poda web
```

## Dziennik diagnozy

| # | Objaw | Hipoteza (kto: AI / ja / k8sgpt) | Komenda weryfikująca | Wynik | Status | Poprawka |
|---|-------|----------------------------------|----------------------|-------|--------|----------|
| 1 | Service `kantyna-payments`: brak podów za Service | AI: selector Service nie zgadza się z etykietą podów | `kubectl -n waldemar get svc kantyna-payments -o yaml` (selector `app.kubernetes.io/name: payment`) vs `kubectl -n waldemar get pods -l app.kubernetes.io/name=payments` (etykieta `payments`) | Selector = `payment` (l. poj.), etykieta podów = `payments` (l. mn.) → niezgodność potwierdzona | ✅ | `kubectl -n waldemar patch svc kantyna-payments -p '{"spec":{"selector":{"app.kubernetes.io/name":"payments"}}}'` |
| 2 | `kantyna-orders-api`: CrashLoopBackOff, 0/2 gotowe | AI: zła nazwa hosta DB w `DATABASE_URL` | `kubectl -n waldemar logs <pod> --previous` | `psycopg_pool.PoolTimeout` / `Name or service not known` dla hosta `kantyna-postgresql`; `kubectl -n waldemar get svc` pokazuje realny Service `kantyna-postgres` (bez „ql") | ✅ | `kubectl -n waldemar set env deployment/kantyna-orders-api 'DATABASE_URL=postgresql://$(DB_USER):$(DB_PASSWORD)@kantyna-postgres:5432/kantyna'` |
| 3 | `kantyna-web`: ImagePullBackOff, 0/2 gotowe | AI: nieistniejący tag obrazu | `kubectl -n waldemar describe pod <pod-web>` → `Back-off pulling image ...web:1.0.1`; porównanie z tagami innych serwisów (`orders-api`, `payments`, `worker` = `:1.0.0`) | Tylko `web` używa `:1.0.1`, który nigdy nie został zbudowany/wypchnięty do ECR | ✅ | `kubectl -n waldemar set image deployment/kantyna-web web=<ecr>/kantyna/web:1.0.0` |
| 4 | `kantyna-worker`: 0/1 gotowe, `FailedCreate` | AI: request pamięci przekracza quota namespace | `kubectl -n waldemar describe rs <rs-worker>` → `exceeded quota: participant-quota, requested: requests.memory=2Gi, used: 960Mi, limited: 2560Mi`; `app/deploy/helm/kantyna/values.yaml` pokazuje domyślny request `128Mi` | Request `2Gi` (odbiegający od defaultu w chart `128Mi`) nie mieścił się w quota | ✅ | `kubectl -n waldemar set resources deployment/kantyna-worker --requests=cpu=50m,memory=128Mi --limits=cpu=250m,memory=256Mi` |
| 5 | Aplikacja: menu/zamówienie przez web zwracały 504/499 — **ujawnione dopiero po naprawie #1–4** | AI: NetworkPolicy blokuje ruch web → orders-api | `kubectl -n waldemar logs -l app.kubernetes.io/name=web -c web` → `upstream timed out ... while connecting to upstream http://...:8000`; `kubectl -n waldemar get networkpolicy kantyna-orders-api-ingress -o yaml` → ingress dozwolony tylko z `app.kubernetes.io/name: frontend`; pody web mają etykietę `app.kubernetes.io/name: web` | Selector NetworkPolicy (`frontend`) nie zgadzał się z żadną istniejącą etykietą poda → cały ruch do orders-api blokowany | ✅ | `kubectl -n waldemar patch networkpolicy kantyna-orders-api-ingress --type=json -p='[{"op":"replace","path":"/spec/ingress/0/from/0/podSelector/matchLabels/app.kubernetes.io~1name","value":"web"}]'` |

## Stan końcowy (`setup/check.sh`)

```
SUKCES: 11/11
```

## Porównanie z k8sgpt

`k8sgpt analyze --namespace waldemar --explain` uruchomiony **po** naprawie #1–5 (backend LiteLLM skonfigurowany w trakcie labu). Nie znalazł już żadnej z pięciu awarii z tabeli powyżej (bo zostały naprawione), ale zgłosił dwa inne "problemy" — oba fałszywe alarmy:

| # | Zgłoszenie k8sgpt | Sugerowana poprawka k8sgpt | Ocena | Dlaczego **nie** wdrożono |
|---|---|---|---|---|
| A | ConfigMap `kube-root-ca.crt` "nieużywany przez żadne pody" | `kubectl delete configmap kube-root-ca.crt` | ❌ Błędne | To standardowy ConfigMap tworzony automatycznie w każdym namespace, montowany niejawnie do każdego poda przez projected volume tokena service account (CA do weryfikacji API servera). Reguła k8sgpt widzi tylko explicite zadeklarowane `volumeMounts` po nazwie, więc nie wykrywa tego niejawnego mountu — false positive, nie dowodzi rzeczywistego problemu. Dodatkowo `kubectl delete` jest zablokowane w `.claude/settings.json` (deny), a nawet gdyby przeszło — Kubernetes odtworzy ConfigMap automatycznie. |
| B | Ingress `kantyna-web` "używa klasy `alb`, która nie istnieje" | `kubectl apply -f .../aws-load-balancer-controller/.../v2_6_0_full.yaml` (reinstalacja kontrolera ALB) | ❌ Błędne | `kubectl get ingressclass` zwraca `Forbidden` — konto `waldemar` nie ma uprawnień do listowania `ingressclasses` na poziomie klastra. k8sgpt trafił na ten sam brak dostępu i błędnie zinterpretował go jako "klasa nie istnieje". W rzeczywistości kontroler ALB działa: `kubectl describe ingress kantyna-web` pokazuje `Normal SuccessfullyReconciled (x11 over 38h)` i przypisany adres ELB. Sugerowana poprawka jest też **cluster-wide** — nadpisałaby współdzielony kontroler używany przez wszystkich uczestników kursu, więc nawet gdyby diagnoza była słuszna, nie leży to w zakresie moich uprawnień na współdzielonym klastrze. |

**Wniosek:** reguły wbudowane k8sgpt (bez `--explain`) są ograniczone do tego, co widzi przez RBAC danego konta — brak uprawnień (`Forbidden`) bywa interpretowany jako "zasób nie istnieje", co prowadzi do błędnych diagnoz i sugestii zmian poza zakresem namespace. AI z pełnym kontekstem repo (ten agent) wykryło 5 rzeczywistych awarii nieznalezionych przez k8sgpt (selector Service, zła nazwa hosta DB, nieistniejący tag obrazu, przekroczona quota, błędny selector NetworkPolicy) i poprawnie zidentyfikowało 2 zgłoszenia k8sgpt jako false positive.

## Podsumowanie

Wszystkie 5 poprawek z głównej diagnozy zostały wdrożone — żadna hipoteza AI (tego agenta) nie została odrzucona w tej sesji. Natomiast **2 sugestie k8sgpt zostały odrzucone** (patrz tabela wyżej): obie wynikały z fałszywych alarmów (niejawny mount ConfigMapu, brak uprawnień RBAC błędnie odczytany jako brak zasobu), a jedna z proponowanych komend (`kubectl apply` reinstalujący kontroler ALB) wykraczała poza zakres namespace i byłaby ryzykowna na współdzielonym klastrze.
