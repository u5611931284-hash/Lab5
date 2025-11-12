#  Laboratorium 5 — PFSwChO

Repozytorium zawiera **kompletne rozwiązanie zadania głównego z Laboratorium 5**.  
Celem było skonfigurowanie dwóch przestrzeni nazw (`ns-dev`, `ns-prod`) z różnymi politykami zasobów,  
a następnie przetestowanie działania **ResourceQuota** i **LimitRange** za pomocą trzech wdrożeń (`Deployment`):

- `no-test`  
- `yes-test`  
- `zero-test`

---

##  1. Konfiguracja środowiska

### Tworzenie przestrzeni nazw i polityk zasobów

####  Polecenia uruchomieniowe

```bash
# Krok 1: Utworzenie przestrzeni nazw
kubectl apply -f manifests/01-namespace-ns-dev.yaml
kubectl apply -f manifests/02-namespace-ns-prod.yaml

# Krok 2: Polityki dla ns-dev (Quota + LimitRange)
kubectl apply -f manifests/11-rq-ns-dev.yaml
kubectl apply -f manifests/12-lr-ns-dev.yaml

# Krok 3: Polityka dla ns-prod (Quota)
kubectl apply -f manifests/13-rq-ns-prod.yaml

##  . Weryfikacja konfiguracji

# Sprawdzenie przestrzeni nazw
kubectl get ns

# Quota w ns-dev (oczekiwany limit 10 podów)
kubectl get resourcequota -n ns-dev

# LimitRange w ns-dev (oczekiwane limity 200m/256Mi)
kubectl get limitrange -n ns-dev

# Quota w ns-prod
kubectl get resourcequota -n ns-prod

##  2. Testowanie Deploymentów w ns-dev

# Uruchomienie "złego" wdrożenia
kubectl apply -f manifests/21-deploy-no-test.yaml

# Sprawdzenie statusu (oczekiwane READY: 0/1)
kubectl get deploy -n ns-dev

# Brak utworzonych Podów
kubectl get pods -n ns-dev

# Szczegóły błędu (FailedCreate z powodu przekroczenia limitu)
kubectl describe deployment no-test -n ns-dev

##  Test 3: zero-test — Automatyczne przypisanie domyślnych wartości

# Uruchomienie poprawnego wdrożenia
kubectl apply -f manifests/22-deploy-yes-test.yaml

# Sprawdzenie statusu (oczekiwane READY: 1/1)
kubectl get deploy,po -n ns-dev

# Sprawdzenie przydzielonych zasobów (150m)
kubectl describe pod -l app=yes-test -n ns-dev

# Uruchomienie wdrożenia bez sekcji resources
kubectl apply -f manifests/23-deploy-zero-test.yaml

# Sprawdzenie statusu (oczekiwane READY: 1/1)
kubectl get deploy,po -n ns-dev

# Sprawdzenie przypisanych zasobów
kubectl describe pod -l app=zero-test -n ns-dev

# Wyciągnięcie przypisanych wartości
kubectl get pod -l app=zero-test -n ns-dev -o=jsonpath="{.items[0].spec.containers[0].resources}"

