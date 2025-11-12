# Laboratorium 5: Deployment, Quota i LimitRange w Kubernetes

Repozytorium zawiera pliki konfiguracyjne YAML oraz dokumentację zadań wykonanych w ramach Laboratorium nr 5 (PFSwChO).

Celem ćwiczeń było zapoznanie się z zaawansowanymi metodami zarządzania zasobami (`ResourceQuota`, `LimitRange`) oraz sposobem działania obiektów `Deployment`.

## 🏁 Część 1: Ćwiczenia z Instrukcji

### [cite_start]Zarządzanie zasobami (`appresources.yaml`) [cite: 28-33]
* **Krok 1:** Poprawne uruchomienie Poda `myapp` (MySQL + WordPress) z domyślnymi zasobami.
* [cite_start]**Krok 2 (Błąd):** Modyfikacja pliku `appresources.yaml` i zmiana `requests: memory` na `64Gi`[cite: 31].
* **Diagnoza:** Pod "utknął" w stanie `Pending`. Polecenie `kubectl describe pod myapp` wykazało błąd `FailedScheduling` z powodu `Insufficient memory`. Klaster nie mógł zagwarantować 64GB RAM.

### [cite_start]Quota i Deployment (`limiter`) [cite: 53-59, 81, 101]
* [cite_start]**Krok 1:** Utworzono `namespace restricted` oraz `ResourceQuota` o nazwie `xlimits` (limit `cpu=1`, `memory=500M`, `pods=3`) [cite: 53-59].
* [cite_start]**Krok 2 (Błąd):** Próba uruchomienia `Deployment` o nazwie `limiter` (`kubectl create deploy limiter...`) [cite: 74-76] nie powiodła się (`READY: 0/3`). [cite_start]Diagnoza: `Quota` zablokowała tworzenie Podów, ponieważ Deployment *nie definiował* `requests` ani `limits` dla zasobów[cite: 39].
* [cite_start]**Krok 3 (Naprawa):** Limity i żądania zostały dodane "w locie" za pomocą polecenia `kubectl set resources deployment limiter --requests=... --limits=...` [cite: 92-93, 101-104].
* **Krok 4 (Sukces):** Deployment poprawnie uruchomił 3 Pody. Polecenie `kubectl describe quota xlimits` pokazało zużycie zasobów (np. `cpu: 300m/1`).

### Monitorowanie i Edycja (`kubectl top`, `kubectl edit`)
* **Problem:** Polecenie `kubectl top` zwróciło błąd `Metrics API not available`.
* [cite_start]**Rozwiązanie:** Włączenie dodatku `minikube addons enable metrics-server` rozwiązało problem i pozwoliło monitorować bieżące zużycie CPU/RAM przez Pody [cite: 156-161].
* [cite_start]**Edycja:** Za pomocą `kubectl edit quota xlimits` [cite: 177-179] limit CPU został obniżony do `100m`. [cite_start]Spowodowało to, że zużycie przekroczyło limit (`300m/100m`), jednak Pody **nadal działały**, co dowodzi, że Quota jest sprawdzana *tylko w momencie tworzenia* Poda [cite: 239-240].

### [cite_start]LimitRange (`limitrangetest.yaml`) [cite: 244-246, 258]
* [cite_start]**Krok 1:** Utworzono `namespace galaxy` [cite: 253] [cite_start]oraz `LimitRange` [cite: 258-262][cite_start], który ustawiał domyślne zasoby (Request: 256Mi, Limit: 512Mi) dla kontenerów[cite: 260].
* [cite_start]**Krok 2 (Test):** Uruchomiono Poda `nginx` (`kubectl run lmemory...`) *bez* definiowania zasobów [cite: 269-270].
* **Diagnoza:** `kubectl describe pod lmemory` pokazało, że Pod **automatycznie otrzymał** `Requests: memory: 256Mi` oraz `Limits: memory: 512Mi`, zgodnie z definicją w `LimitRange`.

### [cite_start]Obiekt Deployment (Właściwości) [cite: 273, 280, 287-292]
* [cite_start]**Samo-naprawianie:** Po ręcznym usunięciu Poda z Deploymentu (`kubectl delete pod...`), `ReplicaSet` natychmiast wykrył brak i uruchomił nowy Pod, aby utrzymać żądaną liczbę replik [cite: 280-281].
* [cite_start]**Skalowanie:** Pokazano dwie metody skalowania Deploymentu `dredis`: za pomocą `kubectl scale deploy dredis --replicas=4` [cite: 288] [cite_start]oraz `kubectl edit deploy dredis` [cite: 291-292].

---

## 🏆 Część 2: Zadanie Główne (ns-dev, ns-prod)

[cite_start]Zgodnie z zadaniem [cite: 296-306], skonfigurowano dwa środowiska.

* **`ns-prod`:** Otrzymał `ResourceQuota` z wysokimi limitami zasobów (plik `prod-quota.yaml`).
* [cite_start]**`ns-dev`:** Otrzymał `ResourceQuota` (limit `pods=10`) oraz `LimitRange` (max `cpu=200m`, `memory=256Mi`) (plik `dev-setup.yaml`) [cite: 298-299].

### Testowanie `ns-dev`:

1.  **`zero-test` (Test 4):**
    * [cite_start]**Akcja:** `kubectl create deploy zero-test --image=nginx -n ns-dev`[cite: 306].
    * **Wynik:** **Sukces**. Pod uruchomił się poprawnie.
    * **Dowód:** `kubectl describe pod...` wykazał, że Pod automatycznie otrzymał domyślne limity (200m CPU / 256Mi RAM) z `LimitRange`.

2.  **`no-test` (Test 2):**
    * [cite_start]**Akcja:** Próba ustawienia limitów `cpu=300m` (`kubectl set resources...`), co przekracza limit `LimitRange` (200m)[cite: 304].
    * **Wynik:** **Błąd**. Deployment nie był w stanie utworzyć Poda (`READY: 0/1`).
    * **Dowód:** `kubectl describe replicaset...` pokazał błąd `FailedCreate` z komunikatem, że żądane zasoby (`300m`) przekraczają maksymalny limit `LimitRange` (`200m`).

3.  **`yes-test` (Test 3):**
    * [cite_start]**Akcja:** Ustawienie limitów `cpu=150m`, czyli *poniżej* limitu `LimitRange`[cite: 305].
    * **Wynik:** **Sukces**. Pod uruchomił się poprawnie, ponieważ mieścił się w dozwolonych limitach.
