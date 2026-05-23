## 1. Przestrzeń nazw (namespace)
* **Wdrożony zasób:** `social-media-prod`
* **Uzasadnienie:**
Cały system został wdrożony w dedykowanej, wyizolowanej przestrzeni nazw. Praktyka ta zapobiega konfliktom nazw z innymi aplikacjami w klastrze (np. w przestrzeni `default` lub `kube-system`) oraz umożliwia nałożenie globalnych limitów zasobów (Resource Quota) wyłącznie na ten konkretny projekt. Zwiększa to bezpieczeństwo i ułatwia zarządzanie cyklem życia środowiska.

## 2. Zarządzanie obciążeniem
### A. Deployment
* **Wdrożone zasoby:** `app-deployment` (Backend - Spring Boot) oraz `frontend-deployment` (Frontend - Nginx)
* **Uzasadnienie:**
Dla warstwy logiki biznesowej oraz prezentacji wybrano obiekt `Deployment`, ponieważ są to aplikacje bezstanowe (stateless). Gwarantuje to wysoką dostępność dzięki zastosowaniu `RollingUpdate` (z parametrem `maxUnavailable: 25%`). Liczba replik: **1**. Jedna replika jest wystarczająca do obsługi ruchu i weryfikacji funkcjonalności, zachowując niskie wykorzystanie zasobów sprzętowych.
### B. StatefulSet
* **Wdrożony zasób:** `db-statefulset` (Baza danych PostgreSQL)
* **Uzasadnienie:**
Baza danych została wdrożona jako `StatefulSet`, ponieważ relacyjne bazy danych wymagają trwałego stanu, gwarancji kolejności uruchamiania, oraz stałego przypisania do wolumenu na dysku. Użycie Deploymentu groziłoby niespójnością danych w przypadku awarii i restartu poda.

## 3. Komunikacja wewnętrzna (Services)
* **Wdrożone zasoby:** `db-service`, `app-service`, `frontend-service`
* **Uzasadnienie:**
Wszystkie trzy mikroserwisy używają serwisu typu **ClusterIP**.
Zgodnie z zasadą najmniejszych uprawnień, żaden z komponentów nie został wystawiony bezpośrednio na zewnątrz klastra (zrezygnowanie z typu `NodePort`). Komunikacja odbywa się wyłącznie w izolowanej sieci wewnątrz K8s. Dostęp dla użytkownika końcowego realizowany jest bezpiecznie i w pełni kontrolowanie za pośrednictwem bramy Ingress.

## 4. Dostęp z zewnątrz (Ingress)
* **Wdrożony zasób:** `social-media-ingress`
* **Uzasadnienie:**
Zamiast przypisywać zewnętrzny adres IP do każdej usługi, zastosowano kontroler Ingress (port 80/443). Oferuje on pojedynczy punkt wejścia do systemu pod domeną `post-manager.local`. Realizuje routing oparty na ścieżkach: zapytania do ścieżki `/api` oraz `/swagger-ui` są kierowane do backendu, natomiast główny ruch `/` trafia do statycznego frontendu.

## 5. Przechowywanie danych (PV / PVC)
* **Wdrożony zasób:** `postgres-pvc`
* **Uzasadnienie:**
Zdefiniowano żądanie przestrzeni dyskowej o wielkości 1 GB z trybem dostępu `ReadWriteOnce`. Powiązanie PVC z StatefulSet gwarantuje, że dane nie ulegną zniszczeniu po zrestartowaniu lub usunięciu kontenera bazy danych. Cykl życia dysku został oddzielony od cyklu życia samego poda, co jest krytycznym wymogiem dla systemów produkcyjnych.

## 6. Wstrzykiwanie konfiguracji (ConfigMap / Secrets)
### A. ConfigMap
* **Wdrożony zasób:** `app-config`
* **Uzasadnienie:** Dane niewrażliwe zostały przeniesione z pliku `manifest.yaml` do niezależnego obiektu ConfigMap. Dzięki temu obraz aplikacji pozostaje uniwersalny, a zmianę konfiguracji bazy można wykonać bez przebudowywania obrazu Dockera.
### B. Secrets
* **Wdrożony zasób:** `db-credentials`
* **Uzasadnienie:** Hasło do bazy danych (dane wrażliwe) zostało zakodowane w Base64 i umieszczone w obiekcie typu Secret. Zapobiega to wyciekom poświadczeń, które mogłyby nastąpić w przypadku przechowywania ich otwartym tekstem w systemie kontroli wersji (Git).

## 7. Dodatkowe aspekty bezpieczeństwa
Infrastruktura rozszerza podstawowe wymogi poprzez implementację:
* **Security Context**: Wymuszono działanie jako użytkownicy `non-root` w celu zapobiegania potencjalnym atakom.
* **Liveness / Readiness Probes**: Dodano sondy kondycji HTTP oraz CLI, weryfikujące gotowość aplikacji do przyjmowania ruchu.