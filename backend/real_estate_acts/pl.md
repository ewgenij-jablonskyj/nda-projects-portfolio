# 🏢 System Ewidencji Aktów Notarialnych i Nieruchomości

> ⚠️ **Informacja:** Ze względu na umowy o poufności (NDA), to repozytorium nie zawiera pełnego 
kodu źródłowego (stanowi część zamkniętego monorepo). Dokumentacja prezentuje zaprojektowaną przeze 
mnie architekturę, wysokopoziomowe rozwiązania technologiczne oraz zaimplementowaną logikę biznesową 
modułu.

## 📌 Przegląd projektu
Moduł backendowy stworzony w celu cyfryzacji, zarządzania i audytowania setek aktów notarialnych, 
umów kupna-sprzedaży nieruchomości oraz ewidencji działek i ksiąg wieczystych (KW). 

Dostarcza on wydajne API w architekturze **GraphQL**, pozwalając aplikacjom klienckim na bezpieczne 
zarządzanie danymi o transakcjach, wgrywanie fizycznych i cyfrowych skanów dokumentów oraz 
generowanie zaawansowanych raportów. Moduł komunikuje się z relacyjną bazą danych za pośrednictwem 
**SQLAlchemy** i wykorzystuje narzędzie **Celery** do asynchronicznego procesowania najcięższych 
zadań w tle.

## 🛠 Stack Technologiczny
* **Język:** Python 3
* **API:** GraphQL (Graphene, Graphene-SQLAlchemy)
* **Baza danych:** Relacyjna (MySQL) + SQLAlchemy (ORM)
* **Zadania asynchroniczne:** Celery
* **Walidacja i Serializacja:** Marshmallow
* **Zarządzanie uprawnieniami:** Dekoratory JWT (np. `@query_header_jwt_required_with_errors`)

---

## 🎯 Zaimplementowane Rozwiązania i Architektura (Mój wkład)

Jako deweloper backendowy przejąłem pełną odpowiedzialność za logikę biznesową tego modułu - od 
modelowania bazy danych, po ekspozycję API i orkiestrację procesów pobocznych.

### 1. Rozwój API i Modelowanie Danych (GraphQL)
Zaprojektowałem relacyjną strukturę bazy danych (tabele główne `RealEstateActs`, 
`RealEstateActsScan` oraz zbiór tabel słownikowych) i udostępniłem dla nich w pełni funkcjonalne 
API.
* **Mutacje:** Stworzyłem złożone endpointy do tworzenia (`RealEstateActsCreateAct`), aktualizacji 
i usuwania aktów oraz skanów.
* **Filtrowanie i Paginacja:** Wdrożyłem silnik filtrów (`graphene_sqlalchemy_filter`), pozwalający 
na precyzyjne przeszukiwanie potężnej bazy (np. po datach podpisania umów przyrzeczonych, cenach 
netto czy specyficznych prawach rzeczowych).

### 2. Zaawansowany System Audytowy (Event Logging)
Restrykcyjne wymagania biznesowe narzucały pełną transparentność operacji. Zaprojektowałem mechanizm
śledzenia zmian w systemie, który chroni spójność danych. 
* Stan obiektów "przed" i "po" operacji jest w locie serializowany (przy użyciu biblioteki 
**Marshmallow**) i zapisywany do dedykowanej tabeli logów (`RealEstateActsEvent`).
* Zbierane są dokładne informacje o każdym zdarzeniu 
(np. *akt: AKTUALIZOWANO; skan: ZMIENIONA WAGA*), co pozwala audytorom na odtworzenie pełnej 
historii życia każdego dokumentu.
* Zaimplementowałem bezpieczną logikę *Soft Delete* - usuwane rekordy i powiązane z nimi pliki nigdy 
nie znikają z bazy fizycznie, lecz są precyzyjnie oznaczane flagą czasową `deleted_at`.

### 3. Asynchroniczne Raportowanie (ETL & Celery)
Rozwiązałem architektoniczny problem blokowania głównego wątku aplikacji przy generowaniu wielkich 
zestawień analitycznych.
* Mutacja `RealEstateActsExportAct` kompiluje dynamiczne zapytania SQLAlchemy do postaci surowego 
SQL (wraz z zaaplikowanymi filtrami nałożonymi przez użytkownika na frontendzie), a następnie zleca 
eksport danych do asynchronicznego zadania w **Celery**.
* Przetwarzanie i wygenerowanie pliku `.xlsx` odbywa się całkowicie w tle. Klient GraphQL 
natychmiast otrzymuje `job_id`, co chroni serwer przed przeciążeniem pamięci.

### 4. Cyfrowe Archiwum i Zarządzanie Plikami
Zbudowałem logikę wiążącą rekordy tekstowe bazy danych z fizycznymi plikami skanów:
* Skonfigurowałem mutacje obsługujące typ `Upload` (multipart) w GraphQL, pozwalając na jednoczesne 
asynchroniczne przesyłanie i linkowanie wielu załączników.
* Zaimplementowałem system kategoryzacji plików (aneksy, służebności, rozwiązania umów) oraz 
mechanizmy automatycznej aktualizacji głównego stanu aktu w momencie dodania końcowej wersji 
fizycznego dokumentu do bazy (tzw. flaga `final_physical_version_dze`).

---

## 📂 Struktura Modułu
Kod został ustrukturyzowany z podziałem na odpowiedzialności (Separation of Concerns):
* `models.py` - Definicje modeli bazodanowych, kluczy obcych i relacji w SQLAlchemy.
* `objects.py` - Typy GraphQL (Node) i schematy serializacji AutoSchema.
* `mutations.py` - Mutacje CRUD i warstwa biznesowa, w tym obsługa wgrywania plików i wyzwalanie 
eventów logowania.
* `query.py` - Konfiguracja węzłów (Connections) z wpiętymi restrykcjami zabezpieczeń JWT.
* `filters.py` - Klasy `FilterSet` generujące skomplikowane i bezpieczne zapytania złączeniowe 
(JOIN) dla zapytań klienckich.
