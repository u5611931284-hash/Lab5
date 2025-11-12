# Sprawozdanie: Laboratorium 5 (PFSwChO)
Repozytorium zawiera kompletne rozwiązanie zadania głównego z Laboratorium 5 . Celem było skonfigurowanie dwóch przestrzeni nazw (ns-dev, ns-prod) z różnymi politykami zasobów, a następnie przetestowanie działania ResourceQuota i LimitRange za pomocą trzech wdrożeń (Deployment): no-test, yes-test oraz zero-test.

1. Konfiguracja Środowiska
Pierwszym krokiem jest utworzenie wymaganych przestrzeni nazw oraz zastosowanie polityk ResourceQuota (limitów dla całego namespace) i LimitRange (domyślnych limitów dla Podów) zgodnie z wymaganiami zadania .

Polecenia Uruchomieniowe
Poniższe polecenia tworzą namespaces oraz wdrażają polityki zasobów.

Bash

# Krok 1: Utworzenie przestrzeni nazw
kubectl apply -f manifests/01-namespace-ns-dev.yaml
kubectl apply -f manifests/02-namespace-ns-prod.yaml

# Krok 2: Zastosowanie polityk dla ns-dev (Quota + LimitRange)
kubectl apply -f manifests/11-rq-ns-dev.yaml
kubectl apply -f manifests/12-lr-ns-dev.yaml

# Krok 3: Zastosowanie polityki dla ns-prod (Quota)
kubectl apply -f manifests/13-rq-ns-prod.yaml
Weryfikacja Konfiguracji
Sprawdzenie, czy wszystkie zasady zostały poprawnie zastosowane w klastrze.

Bash

# Sprawdzenie, czy obie przestrzenie nazw istnieją
kubectl get ns

# Sprawdzenie Quota w ns-dev (oczekiwany limit 10 podów) 
kubectl get resourcequota -n ns-dev

# Sprawdzenie LimitRange w ns-dev (oczekiwane limity 200m/256Mi) 
kubectl get limitrange -n ns-dev

# Sprawdzenie Quota w ns-prod
kubectl get resourcequota -n ns-prod
2. Testowanie Deploymentów w ns-dev
Testy polegają na próbie uruchomienia trzech różnych wdrożeń w ns-dev, aby zweryfikować działanie polityki LimitRange .

Test 1: Deployment no-test (Oczekiwany błąd) 

Ten Deployment (plik 21-deploy-no-test.yaml) jawnie żąda zasobów (cpu: 300m), które przekraczają limit ustalony w LimitRange (200m). Oczekujemy, że Pod nie zostanie utworzony.

Bash

# Uruchomienie "złego" wdrożenia
kubectl apply -f manifests/21-deploy-no-test.yaml

# Sprawdzenie statusu wdrożenia (oczekiwane READY: 0/1)
kubectl get deploy -n ns-dev

# Sprawdzenie, czy Pody nie zostały utworzone
kubectl get pods -n ns-dev

# Diagnoza: Sprawdzenie zdarzeń (Events) w ReplicaSet lub Deployment
# pokaże błąd "FailedCreate" z powodu przekroczenia limitów (300m > 200m).
kubectl describe deployment no-test -n ns-dev
Test 2: Deployment yes-test (Oczekiwany sukces) 

Ten Deployment (plik 22-deploy-yes-test.yaml) jawnie żąda zasobów (cpu: 150m), które mieszczą się w limicie LimitRange (200m). Oczekujemy, że Pod uruchomi się poprawnie.

Bash

# Uruchomienie "dobrego" wdrożenia
kubectl apply -f manifests/22-deploy-yes-test.yaml

# Sprawdzenie statusu (oczekiwane READY: 1/1 i status Running)
kubectl get deploy,po -n ns-dev

# Sprawdzenie zasobów Poda (powinny odpowiadać żądaniu 150m)
kubectl describe pod -l app=yes-test -n ns-dev
Test 3: Deployment zero-test (Domyślne wartości) 

Ten Deployment (plik 23-deploy-zero-test.yaml) nie definiuje żadnych zasobów. Oczekujemy, że LimitRange automatycznie przypisze mu domyślne wartości (requests i limits).

Bash

# Uruchomienie wdrożenia "zero"
kubectl apply -f manifests/23-deploy-zero-test.yaml

# Sprawdzenie statusu (oczekiwane READY: 1/1 i status Running)
kubectl get deploy,po -n ns-dev

# Sprawdzenie zasobów Poda (powinny pojawić się domyślne limity)
kubectl describe pod -l app=zero-test -n ns-dev

# Precyzyjne wyciągnięcie przypisanych zasobów (twardy dowód)
kubectl get pod -l app=zero-test -n ns-dev -o=jsonpath="{.items[0].spec.containers[0].resources}"
Oczekiwany wynik: Polecenie powinno zwrócić zasoby przypisane przez LimitRange (np. limits: cpu: 200m, memory: 256Mi).

3. Zawartość Plików Manifestów (manifests/)
Poniżej znajduje się zawartość plików YAML użytych do wykonania zadania, umieszczonych w katalogu manifests/.

manifests/01-namespace-ns-dev.yaml
cat <<EOF > manifests/01-namespace-ns-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-dev
EOF

cat <<EOF > manifests/02-namespace-ns-prod.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns-prod
EOF

cat <<EOF > manifests/11-rq-ns-dev.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-pod-quota
  namespace: ns-dev
spec:
  hard:
    pods: "10"
EOF

cat <<EOF > manifests/12-lr-ns-dev.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-default-limits
  namespace: ns-dev
spec:
  limits:
  - default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
EOF

cat <<EOF > manifests/13-rq-ns-prod.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-main-quota
  namespace: ns-prod
spec:
  hard:
    pods: "20"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
EOF

cat <<EOF > manifests/21-deploy-no-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-test
  template:
    metadata:
      labels:
        app: no-test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "300m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
EOF

cat <<EOF > manifests/22-deploy-yes-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yes-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yes-test
  template:
    metadata:
      labels:
        app: yes-test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "150m"
            memory: "128Mi"
          limits:
            cpu: "150m"
            memory: "256Mi"
EOF

cat <<EOF > manifests/23-deploy-zero-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zero-test
  template:
    metadata:
      labels:
        app: zero-test
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
