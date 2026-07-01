# ✉️ System Ewidencji Korespondencji i Rejestru Zdarzeń

> ⚠️ **Informacja:** Ze względu na umowę o zachowaniu poufności (NDA), niniejsze repozytorium 
nie zawiera pełnego kodu źródłowego (stanowi część zamkniętego monorepo). Poniższa dokumentacja 
prezentuje przyjęte rozwiązania technologiczne oraz logikę biznesową, którą zaimplementowałem 
w ramach tego modułu.

## 📌 Przegląd projektu
Moduł backendowy stworzony do cyfryzacji, zautomatyzowanego zarządzania pismami oraz szczegółowego 
ewidencjonowania incydentów i interwencji. System pozwala na powiązanie korespondencji z fizyczną 
infrastrukturą (kamery, systemy monitoringu, placówki) oraz rejestruje dokładne przedziały czasowe 
zdarzeń.

Moduł udostępnia bezpieczne API w architekturze **GraphQL**, umożliwiając aplikacjom klienckim 
zarządzanie rejestrem. Posiada wbudowany mechanizm bezpiecznego wgrywania plików, zarządzania 
szablonami dokumentów oraz dynamicznego generowania pism z wykorzystaniem biblioteki 
**python-docx**. Komunikacja z relacyjną bazą danych odbywa się za pośrednictwem **SQLAlchemy**.

## 🛠 Stack Technologiczny
* **Język:** Python 3 (Flask)
* **API:** GraphQL (Graphene, Graphene-SQLAlchemy, Relay)
* **Baza danych:** Relacyjna (MySQL / PostgreSQL) + SQLAlchemy (ORM)
* **Przetwarzanie Dokumentów:** `python-docx` (automatyzacja generowania `.docx`)
* **Walidacja i Serializacja:** Marshmallow (SQLAlchemyAutoSchema)
* **Zarządzanie uprawnieniami i bezpieczeństwo:** Dekoratory autoryzacji oparte o JWT 
(np. `@query_header_jwt_required_with_errors`, `@has_role`) oraz algorytmy haszujące (SHA-256) 
dla plików.

---

## 🎯 Kluczowe Funkcjonalności i Mój Wkład

Jako deweloper backendowy odpowiadałem za architekturę, implementację logiki biznesowej, mapowanie 
relacji w bazie danych oraz obsługę złożonych operacji wejścia/wyjścia na plikach.

### 1. Dynamiczne Generowanie Dokumentów (Szablony Word)
Zaimplementowałem zaawansowany mechanizm automatyzujący tworzenie pism na podstawie wgranych 
wcześniej szablonów.
* **Manipulacja plikami `.docx`:** Wykorzystałem bibliotekę `python-docx` do procesowania akapitów 
w dokumencie. Zbudowałem uniwersalną funkcję (m.in. w `utils.py`), która w locie iteruje 
po strukturze pliku i bezpiecznie podmienia zdefiniowane tagi tekstowe na rzeczywiste dane 
wyciągnięte z bazy.
* **Zarządzanie Szablonami:** Stworzyłem dedykowany przepływ obsługujący wgrywanie (Upload), 
aktualizację i deaktywację wzorów dokumentów dla całego systemu.

### 2. Bezpieczny System Zarządzania Plikami
Zaprojektowałem i wdrożyłem mechanizm asynchronicznego przesyłania załączników z naciskiem 
na bezpieczeństwo systemu plików:
* **Haszowanie Nazw:** Aby uniknąć kolizji nazw i ataków typu *path traversal*, każda nazwa 
wgrywanego pliku jest szyfrowana za pomocą **SHA-256** (z uwzględnieniem timestampu). Na serwerze 
przechowywane są bezpieczne, zanonimizowane pliki, podczas gdy w bazie zapisywana jest 
ich oryginalna nazwa i metadane.
* **Relacje Wielo-do-Wielu (M2M):** Utworzyłem skomplikowany graf relacji plików z wpisami głównymi 
(osobne pule dla "załączników wejściowych" i "plików odpowiedzi"), pozwalając na elastyczne 
zarządzanie cyfrowym archiwum.

### 3. Rejestracja Incydentów i Infrastruktury
Dostosowałem system do potrzeb ścisłego ewidencjonowania spraw bezpieczeństwa:
* Zaprojektowałem obsługę przedziałów czasowych incydentów (`incident_periods`), pozwalającą 
klientowi powiązać jedno pismo z wieloma datami i godzinami zdarzeń, co jest kluczowe przy analizie 
nagrań monitoringu.
* Zbudowałem i zintegrowałem słowniki infrastrukturalne (placówki, kamery, systemy monitoringu), 
wiążąc je z głównym bytem bazy za pośrednictwem relacji asocjacyjnych.

### 4. Wbudowany System Audytowy (Event Logging)
Odwzorowałem rygorystyczne procedury śledzenia zmian w systemie ewidencji:
* **Event Sourcing:** Każda akcja – od wgrania pliku, przez zmianę elementu z kamerą, 
aż po modyfikację samego wpisu czy wygenerowanie odpowiedzi – jest logowana (np. flagami 
*„szablon: WGRANO”*, *„zapis: ZAKTUALIZOWANO”*).
* **Serializacja Danych Historycznych:** Stan obiektów jest zapisywany w formie zrzutów JSON przy 
użyciu schematów **Marshmallow**, co chroni dane przed bezpowrotną utratą i pozwala śledzić pełen 
cykl życia korespondencji.

---

## 📂 Szczegółowa Struktura Modułu

Kod ustrukturyzowałem z zachowaniem zasady *Separation of Concerns*, ułatwiając jego utrzymanie 
i testowanie:

### 1. `models.py` — Warstwa Mapowania Obiektowo-Relacyjnego (ORM)
Definicje struktur bazodanowych uwzględniające głębokie relacje:
* **`LetterDatabase`**: Centralna encja przechowująca metadane pism, numery zgłoszeń, opisy zdarzeń 
i statusy. Korzysta z bogatej siatki relacji do jednostek wydających dokumenty, typów zdarzeń, 
czy systemów monitoringu.
* **Tabele Asocjacyjne**: M.in. `letter_database_entry_files_jun`, `camera_letter_database` mapujące 
połączenia w relacji Many-to-Many.
* **Modele Słownikowe i Logi**: Encje wspierające system takie jak szablony 
(`LetterDatabaseTemplateDic`) czy model zdarzeń (`LetterDatabaseEvent`).

### 2. `objects.py` — Warstwa Prezentacji i Serializacji Graphene/Marshmallow
* **`SQLAlchemyObjectType`**: Konfiguracja węzłów (Nodes) w standardzie **Relay**, udostępniająca 
zoptymalizowany pod kątem frontendu model danych.
* **`SQLAlchemyAutoSchema`**: Schematy automatycznej serializacji. Użyłem klas zagnieżdżonych 
(`Nested`), aby umożliwić bezbolesne parsowanie złożonych list podrzędnych (np. historii incydentów 
`incident_periods` i plików `files_answer`) na format JSON.

### 3. `filters.py` — Deklaratywny Silnik Filtrowania Zapytań
* Mechanizm automatycznej konwersji argumentów GraphQL na klauzule w bazie danych. Dzięki 
zastosowaniu biblioteki `graphene_sqlalchemy_filter` użytkownicy mogą budować elastyczne zapytania 
(np. wyszukiwanie pism po powiązanych ID, datach przyjęcia czy lokalizacji placówek).

### 4. `query.py` — Endpointy Odczytu i Integracja Autoryzacji
* **`ProtectedFilterableConnectionField`** i **`ActiveFilterableConnectionField`**: Autorskie 
nadpisania warstwy odczytu. Gwarantują, że każde zapytanie do bazy jest chronione tokenem JWT 
(`@query_header_jwt_required_with_errors`) oraz odpowiednią rolą (`@has_role(_roles)`), z dodatkową 
automatyczną filtracją po statusie aktywności (soft delete).

### 5. `mutations.py` — Centralna Logika Biznesowa (Pliki, Edycja, Wzorce)
Warstwa modyfikująca stan systemu.
* Odpowiada za proces parsowania wieloczęściowych żądań z plikami (`Upload`), ich walidację, zmianę 
nazwy (SHA-256) oraz fizyczny zapis na serwerze z jednoczesnym zarządzaniem transakcjami w bazie.
* Wdraża wyzwalacze audytowe (`log_event`), które asynchronicznie odkładają ślady w systemie po 
każdej prawidłowo zakończonej mutacji.

### 6. `utils.py` & `roles.py` — Moduły Pomocnicze
* **`utils.py`**: Zawiera dedykowaną logikę domenową – m.in. funkcje do podmieniania ciągów znaków 
w dokumentach Word (`replace_text`), generowanie relacji dat incydentów (`create_incident_periods`) 
oraz precyzyjne wyliczenia (typy zdarzeń `EventType`, formaty dat). Ujednolica też zwrotne 
komunikaty mutacji (funkcje typu `return_success_with_object`).
* **`roles.py`**: Wymusza określoną konfigurację zabezpieczeń (np. kontrola dostępu dla grupy 
biznesowej `letter_database`).

### 7. `__init__.py` — Punkt Rejestracji i Eksportu Modułu
Zarządza udostępnianiem endpointów dla Głównego Schematu API, agregując powiązane pola zapytań 
i mutacji w jeden, spójny moduł domenowy.

---

## 📡 Specyfikacja API (GraphQL Interface)

### 🔍 Zapytania (Queries)

#### Główne i Specyficzne Endpointy Odczytu
* `letter_database` – Centralny punkt dostępowy do ewidencji pism. Zaimplementowałem tu zaawansowane 
wyszukiwanie kombinowane (np. po numerze wniosku, dacie wpływu, powiązanych aparatach/kamerach) 
z obsługą paginacji Relay. Chroniony pełną weryfikacją roli i tokenu JWT.
* `letter_database_template_dic` – Dedykowany punkt pobierania wzorców dokumentów `.docx`. 
W odróżnieniu od standardowych zapytań, rozszerza klasę `ActiveFilterableConnectionField`, która 
automatycznie w locie odrzuca rekordy oznaczone jako usunięte (`status == 1`), optymalizując przesył 
danych.
* `letter_database_incident_periods_ref` – Zapytanie dedykowane analityce przedziałów czasowych 
incydentów powiązanych bezpośrednio ze sprawami i zabezpieczonym materiałem dowodowym.

#### Uogólnione Endpointy Słownikowe (Dictionary Queries)
Zbiorcza pula powtarzalnych endpointów referencyjnych, służących do zasilania list wyboru (select) 
na frontendzie:
* `letter_database_*_dic` (gdzie `*` to: `camera`, `establishment`, `event_type`, `issue_status`, 
`issuing_body`, `monitoring_system`) – Ustandaryzowane punkty dostępowe GraphQL mapujące tabele 
relacyjne, wyposażone w automatyczną generację filtrów typów wejściowych (ID, nazwa, status, zakresy 
dat).

---

### ✍️ Mutacje (Mutations)

#### Operacje na Korespondencji i Generowanie Pism
* `letter_database_create_entry` / `update_entry` – Mutacje procesujące logikę biznesową rejestracji 
korespondencji. Odpowiadają za parsowanie złożonych struktur wejściowych, kaskadowe 
wyzwalanie zapisu przedziałów czasowych oraz mapowanie powiązań wielo-do-wielu (np. przypisywanie 
zestawu kamer).
* `letter_database_delete_entry` – Wdrożenie mechanizmu bezpiecznego usuwania wpisów korespondencji 
bez fizycznego czyszczenia rekordu w bazie danych (Soft Delete).
* `letter_database_generate_response` – Kluczowa mutacja operacyjna odpowiedzialna za uruchomienie 
silnika podstawiania danych w szablonie Word za pomocą `python-docx`, fizyczny zapis wyjściowego 
dokumentu na serwerze i rejestrację nowego pliku odpowiedzi w bazie danych.

#### Zarządzanie Cyklem Życia Szablonów (Templates)
* `letter_database_upload_template` – Mutacja obsługująca wieloczęściowy strumień `Upload`. 
Odpowiada za walidację rozszerzeń, haszowanie nazwy pliku algorytmem SHA-256 z unikalnym znacznikiem 
czasu, zapis dyskowy oraz wyemitowanie zdarzenia audytowego `TEMPLATE_UPLOADED`.
* `letter_database_delete_template` – Zmiana flagi widoczności szablonu, automatycznie odcinająca 
go od widoków klienckich przy zachowaniu integralności historycznie wygenerowanych pism.

#### Uogólnione Mutacje Słownikowe (Dictionary Mutations)
* `letter_database_create_*_item` / `update_*_item` / `delete_*_item` (dla słowników infrastruktury 
i typów zdarzeń) – Zautomatyzowany, powtarzalny zestaw mutacji CRUD dla administratorów systemu, 
pozwalający na dynamiczne rozbudowywanie baz referencyjnych (np. dodawanie nowych placówek, systemów 
rejestracji czy statusów spraw) bez konieczności ingerencji w kod backendu.
