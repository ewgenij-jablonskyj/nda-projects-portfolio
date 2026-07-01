# 🏢 System Ewidencji Aktów Notarialnych i Nieruchomości

> ⚠️ **Informacja:** Ze względu na umowę o zachowaniu poufności (NDA), niniejsze repozytorium 
nie zawiera kodu źródłowego (stanowi część zamkniętego monorepo). Poniższa dokumentacja prezentuje 
przyjęte rozwiązania technologiczne oraz złożoną logikę biznesową, którą zaimplementowałem w ramach 
tego modułu.

## 📌 Przegląd projektu
Moduł backendowy stworzony w celu cyfryzacji, zautomatyzowanego zarządzania i pełnego audytowania 
setek aktów notarialnych, umów kupna-sprzedaży nieruchomości oraz ewidencji działek i ksiąg 
wieczystych (KW). 

Dostarcza on wydajne, autoryzowane API w architekturze **GraphQL**, pozwalając aplikacjom klienckim 
na bezpieczne zarządzanie danymi o transakcjach, asynchroniczne wgrywanie fizycznych i cyfrowych 
skanów dokumentów oraz generowanie raportów. Moduł komunikuje się z relacyjną bazą danych 
za pośrednictwem **SQLAlchemy** i wykorzystuje narzędzie **Celery** do asynchronicznego procesowania 
ciężkich zadań w tle (ETL).

## 🛠 Stack Technologiczny
* **Język:** Python 3 (Flask)
* **API:** GraphQL (Graphene, Graphene-SQLAlchemy, Relay)
* **Baza danych:** Relacyjna (MySQL) + SQLAlchemy (ORM)
* **Zadania asynchroniczne / Background Jobs:** Celery
* **Walidacja i Serializacja:** Marshmallow (SQLAlchemyAutoSchema)
* **Zarządzanie uprawnieniami i bezpieczeństwo:** Dekoratory autoryzacji oparte o JWT 
(np. `@query_header_jwt_required_with_errors`)

---

## 🎯 Kluczowe Funkcjonalności i Mój Wkład

Jako deweloper backendowy byłem odpowiedzialny za implementację logiki biznesowej modułu w ramach 
przyjętej architektury systemu – od mapowania bazy danych, przez optymalizację zapytań, 
aż po ekspozycję API i orkiestrację procesów pobocznych.

### 1. API i Modelowanie Danych (GraphQL)
Odwzorowałem relacyjną strukturę bazy danych obejmującą tabele główne 
(`RealEstateActs`, `RealEstateActsScan`) oraz zbiór tabel słownikowych. Udostępniłem dla nich 
w pełni funkcjonalne API zapytań i mutacji.
* **Zaawansowane Filtrowanie:** Wdrożyłem silnik filtrów oparty na `graphene_sqlalchemy_filter`. 
Pozwala to klientom na dynamiczne budowanie złożonych zapytań (np. filtrowanie aktów po numerze 
repetytorium, dacie zawarcia umowy przyrzeczonej, rodzaju prawa rzeczowego czy cenach zakupu netto).
* **Paginacja Relay:** Zastosowałem specyfikację Relay (kursory, węzły), co zapewnia wydajne 
ładowanie danych na frontendzie bez obciążania pamięci serwera.

### 2. Zaawansowany System Audytowy (Event Logging)
Restrykcyjne wymagania biznesowe narzucały pełną transparentność operacji. Zaimplementowałem 
mechanizm śledzenia zmian w systemie, który dba o spójność danych.
* **Serializacja Stanu:** Stan obiektów przed i po modyfikacji (Update/Delete) jest w locie 
serializowany za pomocą schematów **Marshmallow** i zapisywany w dedykowanej tabeli logów 
(`RealEstateActsEvent`).
* **Granulacja Zdarzeń:** System odkłada dokładne informacje tekstowe (np. *„akt: UTWORZONO”*, 
*„skan: AKTUALIZOWANO; skan: ZMIENIONA WAGA”*, *„akt: ZMIENIONO KOŃCOWY DZE”*), co pozwala audytorom 
na odtworzenie cyklu życia każdego dokumentu.
* **Soft Delete:** Wdrożyłem bezpieczną logikę miękkiego usuwania. Rekordy bazy i powiązane z nimi 
pliki fizyczne nie są trwale kasowane, lecz oznaczane flagą `deleted_at`, a przypisane pliki 
zmieniają status widoczności.

### 3. Asynchroniczne Raportowanie i Eksport Danych (Celery)
W ramach rozbudowy modułu zintegrowałem operacje generowania raportów z istniejącą architekturą 
Celery, aby uniknąć blokowania wątku głównego API przy kosztownych obliczeniowo operacjach.
* Mutacja `RealEstateActsExportAct` przyjmuje parametry filtrujące od użytkownika i na ich podstawie 
buduje zapytanie bazodanowe. 
* Wykorzystałem mechanizm **kompilacji zapytania SQLAlchemy do surowego SQL** 
(`query.statement.compile(compile_kwargs={"literal_binds": True})`). Skompilowany tekst zapytania 
jest przekazywany do workera Celery.
* Klient natychmiastowo otrzymuje identyfikator zadania (`job_id`), a plik `.xlsx` generowany jest 
w tle, całkowicie odciążając API.

### 4. Złożona Logika Biznesowa i Archiwum Cyfrowe
Skonfigurowałem logikę integrującą tekstowe wpisy w bazie z fizycznymi plikami:
* **Upload Plików w GraphQL:** Wykorzystałem typ `Upload` (multipart request), pozwalający na 
asynchroniczne przesyłanie wielu załączników jednocześnie z procesem tworzenia lub edycji aktu.
* **Kaskadowe Aktualizacje Właściwości (Smart Updates):** Zakodowałem dedykowaną logikę reagującą 
na zmiany kluczowych pól. Przykładowo, jeśli podczas edycji aktu zmieni się jego ranga 
(pole `gold`), system przeszukuje i propaguje zmianę do powiązanych duplikatów w oparciu o numery 
repetytoriów.
* **Automatyzacja Statusów Skanów:** System analizuje wgrywane dokumenty pod kątem fizycznych 
wersji ("dze", "bz") i automatycznie aktualizuje flagę głównego aktu (`final_physical_version_dze`), 
gdy skompletowane zostaną wymagane oryginały.

---

## 📂 Szczegółowa Struktura Modułu

Kod został ustrukturyzowany z rygorystycznym zachowaniem zasady separacji odpowiedzialności 
(*Separation of Concerns*). Poniżej znajduje się przegląd komponentów, za których logikę 
odpowiadałem:

### 1. `models.py` — Warstwa Mapowania Obiektowo-Relacyjnego (ORM)
Zawiera definicje struktur bazodanowych SQLAlchemy powiązanych z kontekstem bazodanowym `infodino`.
* **`RealEstateActs`**: Główna klasa mapująca parametry aktów notarialnych (m.in. dane 
geolokalizacyjne, repetytoria, metraż, księgi wieczyste `kw`, ceny zakupu netto oraz relacje). 
Wykorzystałem zachłanne ładowanie złączeń (`lazy="joined"`) dla skanów i logów, aby unikać problemu 
N+1 na poziomie bazy danych.
* **`RealEstateActsScan`**: Encja reprezentująca fizyczne załączniki i skany dokumentów wraz z 
flagami biznesowymi (`annex`, `sales_contract_annex`, `contract_termination`, `easement`) oraz 
statusami weryfikacji papierowej.
* **`file_real_estate_acts_scan`**: Tabela pośrednicząca (Association Table) typu Many-to-Many 
łącząca wewnętrzny system plików z encjami skanów.
* **`RealEstateActsEvent`**: Model dziennika zdarzeń dziedziczący po bazowej klasie `Event`.

### 2. `objects.py` — Warstwa Prezentacji i Serializacji Graphene/Marshmallow
Odpowiada za transformację modeli ORM na typy zrozumiałe dla silnika GraphQL oraz mechanizmy zapisu 
stanu.
* **`SQLAlchemyObjectType`**: Klasy takie jak `RealEstateActsObject` implementują interfejs 
`graphene.relay.Node`, udostępniając standard Relay dla klientów API.
* **`SQLAlchemyAutoSchema`**: Klasy `RealEstateActsSchema` automatyzują głęboką serializację modeli 
do struktur JSON. Wykorzystują mechanizm pól zagnieżdżonych (`Nested`), co umożliwia bezproblemowy 
zrzut pełnego grafu obiektów (np. aktu wraz z kolekcją powiązanych skanów) na potrzeby audytu.

### 3. `filters.py` — Deklaratywny Silnik Filtrowania Zapytań
* Definiuje klasy `FilterSet` (np. `RealEstateActsFilter`) mapujące dozwolone operacje porównania 
(np. równość, zawieranie, zakresy dat) na kolumnach modeli.
* Umożliwia bezpieczne i zautomatyzowane generowanie klauzul `WHERE` przez ORM, zabezpieczając 
system przed podatnościami typu SQL Injection na poziomie API.

### 4. `query.py` — Endpointy Odczytu i Integracja Autoryzacji
Definiuje strukturę wejściową dla zapytań GraphQL (Queries).
* **`ProtectedFilterableConnectionField`**: Własna klasa rozszerzająca standardowe pole kolekcji z 
filtrowaniem. Nadpisuje metodę klasy `get_query` i dekoruje ją za pomocą 
`@query_header_jwt_required_with_errors`. Gwarantuje to, że żadne zapytanie nie zostanie 
przetworzone bez ważnego i zautoryzowanego tokenu Bearer JWT.

### 5. `mutations.py` — Centralna Logika Biznesowa (Zapis, Edycja, ETL)
Najobszerniejszy plik modułu zawierający imperatywną logikę procesowania danych:
* **Tworzenie i Upload (`RealEstateActsCreateAct`)**: Przyjmuje strumienie plików `Upload`, wywołuje 
wewnętrzny moduł zapisu fizycznego, wiąże obiekty w transakcji bazy danych i generuje pierwszy wpis 
w logu audytowym.
* **Propagacja zmian (`RealEstateActsUpdateAct`)**: Modyfikuje pola encji, obsługuje zmianę 
priorytetów (flaga `gold`) i dba o spójność danych.
* **Zarządzanie stanem skanów (`RealEstateActsUpdateScan`)**: Pilnuje kompletności dokumentacji. 
W przypadku zatwierdzenia kompletu wersji "dze" we wszystkich podrzędnych skanach, kaskadowo 
modyfikuje właściwość `final_physical_version_dze` bezpośrednio na głównym akcie.
* **Asynchroniczny eksport (`RealEstateActsExportAct`)**: Pobiera filtry, buduje obiekt Query, 
kompiluje go na surowy ciąg SQL z dowiązanymi parametrami i deleguje zadanie typu *fire-and-forget* 
do Celery.

### 6. `utils.py` & `roles.py` — Moduły Pomocnicze i Konfiguracyjne
* **`utils.py`**: Deklaruje stałe globalne ścieżek dyskowych oraz unifikuje interfejs odpowiedzi 
mutacji (funkcje `return_success_with_object`, `return_error`), zapewniając przewidywalną strukturę 
błędów dla frontendu.
* **`roles.py`**: Definiuje listę ról wymaganą przez dekoratory bezpieczeństwa do weryfikacji 
uprawnień wewnątrz tokenu JWT.

### 7. `__init__.py` — Punkt Rejestracji i Eksportu Modułu
Działa jako centralna fasada podsystemu. Agreguje wszystkie pola zapytań (`queries`) oraz mutacji 
(`mutations`), mapując je na spójne słowniki konfiguracji GraphQL, które są następnie montowane 
w głównym schemacie aplikacji Flask.

---

## 📡 Specyfikacja API

Moduł dostarcza wysoce modularne endpointy zgodne ze standardem GraphQL.

### Zapytania (Queries) - Read
| Pole | Model Bazy (SQLAlchemy) | Opis |
| :--- | :--- | :--- |
| `real_estate_acts` | `RealEstateActs` | Zestawienie aktów wraz ze wsparciem dla filtrów i paginacji. |
| `real_estate_acts_scan` | `RealEstateActsScan` | Ewidencja skanów, załączników, aneksów i dokumentacji. |
| `real_estate_acts_event` | `RealEstateActsEvent` | Dziennik audytowy śledzący cykl życia encji. |
| `real_estate_acts_*_dic` | Modele słownikowe | Pobieranie list słownikowych (typy umów, prawa rzeczowe itp.). |

### Mutacje (Mutations) - Write/Delete/Jobs
* `real_estate_acts_create_act` – Rejestracja nowego wpisu z obsługą multi-uploadu plików.
* `real_estate_acts_update_act` – Aktualizacja danych z wbudowaną walidacją i logiką propagacji 
(np. flaga `gold`).
* `real_estate_acts_delete_act` – Usunięcie typu Soft Delete (oznaczenie czasu usunięcia 
dla głównych encji i relacji).
* `real_estate_acts_create_scan` / `update_scan` / `delete_scan` – Granularne operacje na fizycznych 
załącznikach.
* `real_estate_acts_export_act` – Asynchroniczne wyzwolenie zadania zrzutu danych do Excela. Zwraca 
`job_id` oraz aktualny `status`.