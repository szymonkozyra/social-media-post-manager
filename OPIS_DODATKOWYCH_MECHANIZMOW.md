## Ograniczenie wykorzystywanych zasobów
1. **Na poziomie podów (Containers Resources):** Każdy z kontenerów (Postgres, Spring Boot, Nginx) posiada zdefiniowane bloki `requests` (gwarantowane minimum zasobów do startu) oraz `limits` (limit zużycia). Zapobiega to sytuacji, w której jeden uszkodzony proces wykorzystuje cały dostępny RAM (noisy neighbors).
2. **Na poziomie klastra (ResourceQuota):** Utworzono obiekt `project-quota` ograniczający całą przestrzeń nazw `social-media-prod` do maksymalnie 2 rdzeni CPU oraz 2 GB RAM. Stanowi to zabezpieczenie przed atakami DDoS oraz wyciekami pamięci.

## Zdefiniowanie polityki sieciowej
* Zastosowano podejście Zero Trust implementując obiekt `NetworkPolicy` o nazwie `db-network-policy`.
* **Uzasadnienie i działanie:**
Domyślnie Kubernetes pozwala na wzajemną komunikację każdego poda. Zmodyfikowano ten mechanizm wprowadzając zaporę (firewall). Wyodrębniono ruch przychodzący (Ingress) do podów bazy danych (etykieta `app: db`) i ustawiono rygorystyczną regułę przepuszczającą pakiety na porcie 5432 **wyłącznie** ze źródła posiadającego etykietę `app: backend`. W przypadku skompromitowania frontendu, atakujący nie uzyska dostępu sieciowego do bazy danych.