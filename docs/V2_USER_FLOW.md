# SudoSync v2 — uzgodniony przepływ użytkownika

Ten dokument zapisuje wymagania Romana dla działania wtyczki. Treść zrzutu ekranu
„Ocena koncepcji wtyczki” traktujemy jako wymagania użytkownika; odpowiedzi
asystenta widoczne na zrzucie nie są źródłem wymagań.

## Zasada używania

- Użytkownik korzysta tylko z jednego Kodi naraz. Kończy pracę na urządzeniu A,
  a dopiero potem uruchamia urządzenie B.
- Usługa SudoSync działa w tle Kodi, dlatego na pozostałych urządzeniach nie
  uruchamia się w tym czasie Kodi z aktywną usługą SudoSync.
- Ochrona przed przypadkowym równoczesnym uruchomieniem jest dodatkowym
  zabezpieczeniem. Nie jest warunkiem zaprojektowania ani ukończenia zwykłego,
  sekwencyjnego przepływu użytkownika.

## Pierwsze uruchomienie

1. Użytkownik instaluje SudoSync i uruchamia Kodi. Usługa SudoSync startuje
   automatycznie.
2. Jeżeli lokalnie nie ma jeszcze zapisanej ścieżki do udziału, wtyczka otwiera
   selektor katalogu Kodi. Użytkownik wybiera folder sieciowy; nie musi ręcznie
   wpisywać adresu SMB/NFS. Jeśli wybrany folder nie jest już `.SudoSync`,
   wtyczka używa w nim podkatalogu `.SudoSync`.
3. Wtyczka sprawdza dostęp do wybranego miejsca i plik
   `<base_path>/SudoSync.cfg`.
4. Jeśli plik istnieje i jest poprawny, wtyczka odczytuje wspólne ustawienia
   sieciowe i stosuje je lokalnie. Uszkodzony lub nieczytelny plik nie może być
   uznany za brak konfiguracji ani nadpisany kreatorem.
5. Jeśli pliku nie ma, wtyczka uruchamia wizualny konfigurator pierwszego
   uruchomienia. Pyta o ustawienia sieciowe, ale nie pyta ponownie o ścieżkę,
   którą wybrano w selektorze. Domyślnie próbuje używać SudoSync ID; jeśli to
   się nie powiedzie, przechodzi na awaryjne identyfikatory IMDb, gdy są
   dostępne.
6. Po ukończeniu konfiguratora wtyczka zapisuje wspólną konfigurację do
   `SudoSync.cfg`, sprawdza zapis przez ponowny odczyt i oznacza konfigurator
   jako ukończony. Dopiero potem rozpoczyna normalną pracę.

## Istniejąca instalacja i aktualizacje

- Brak `SudoSync.cfg` nie oznacza automatycznie nowej instalacji. Wtyczka
  rozpoznaje istniejącą strukturę SudoSync i kieruje taką instalację do jawnej
  migracji, zachowując dotychczasowe identyfikatory oraz pliki i dane klientów.
- Nowe wersje wtyczki są sprawdzane bezpośrednio przez GitHub Releases
  (`BreakTechEu/SudoSync-Kodi-addon`) automatycznie maksymalnie raz na dobę (86400 s) lub
  na żądanie z poziomu ustawień. Automatyczna cicha aktualizacja w tle jest
  wyłączona — wtyczka informuje o dostępności nowej wersji i pyta użytkownika
  o zgodę na aktualizację.
- Użytkownik ma także możliwość ręcznej aktualizacji lokalnej poprzez wskazanie
  pliku `.zip` w selektorze plików z poziomu ustawień wtyczki. Każda paczka ZIP
  (zarówno z GitHuba, jak i wskazana lokalnie) przechodzi pełną walidację
  struktury i wersji przed rozpakowaniem.
- W pliku `SudoSync.cfg` zapisywane jest pole `min_addon_version` określające
  najwyższą wersję wtyczki zainstalowaną w sieci. Jeżeli wtyczka na innym
  urządzeniu Kodi wykryje w `SudoSync.cfg` nowszą wersję niż własna, wymusza
  aktualizację (z GitHub Releases lub wskazanego pliku ZIP) albo wstrzymuje
  działanie usługi SudoSync do czasu wykonania aktualizacji.
- Zawartość `SudoSync.cfg` jest wspólna dla sieci. Nazwa tego pliku nie oznacza,
  że wolno umieszczać w nim ustawienia konkretnego urządzenia Kodi.

## Co jest wspólne, a co lokalne

- Wspólny udział zawiera konfigurację sieci, współdzielone snapshoty i dane
  potrzebne do synchronizacji.
- Każde Kodi zachowuje lokalnie własny stabilny `client_id`, nazwę urządzenia
  i ustawienia profilu. Dane lokalne nie mogą być kopiowane między klientami
  przez `SudoSync.cfg`.
- Współdzielone dane nie mogą zawierać ścieżek ani lokalnych identyfikatorów
  `movieid`/`episodeid` Kodi jako globalnych identyfikatorów.

## Zwykła synchronizacja

- Synchronizowane są stan odtworzenia, data ostatniego odtworzenia, pozycja
  wznowienia i ocena użytkownika dla filmów i odcinków już obecnych w lokalnej
  bibliotece.
- Dopasowanie odbywa się po mocnych identyfikatorach z uwzględnieniem typu
  medium: SudoSync ID, IMDb, TMDb, TVDb lub Trakt. Sam tytuł, rok, ścieżka ani
  lokalny ID Kodi nie uprawniają do automatycznego dopasowania.
- Nie zakładaj, że każdy ma IMDb ID. Zbieraj i łącz wszystkie dostępne,
  obsługiwane identyfikatory z Kodi oraz plików `.nfo`. W istniejącym silniku v1 parser
  rozpoznaje elementy `<uniqueid type="…">`; dla filmów także `<tmdbid>` oraz
  legacy `<id>tt…</id>` jako IMDb. Kodi może też zwrócić `uniqueid` i
  `imdbnumber` przez JSON-RPC. W ścieżce v1 odczyt NFO uruchamia się dopiero, gdy Kodi
  nie zwróci żadnego ID; v2 ma łączyć oba źródła, aby nie tracić dodatkowych
  identyfikatorów. Jeśli nie ma żadnego mocnego ID i nie można zapisać
  SudoSync ID, pozycji nie wolno dopasować po tytule.
- Nie dodajemy zdalnych pozycji do biblioteki. Niejednoznaczne rekordy i
  konflikty są raportowane i pomijane zamiast cichego nadpisania.
- Podgląd poprzedza operację pierwszej inicjalizacji. Zmiana biblioteki wymaga
  jawnego zatwierdzenia, kopii przed zapisem i weryfikacji wyniku przez Kodi
  JSON-RPC. Baza `MyVideos*.db` nigdy nie jest modyfikowana bezpośrednio.
- Podczas odtwarzania zdalne zmiany są odraczane. Weryfikację na urządzeniach
  prowadzi się sekwencyjnie: zmiana na jednym Kodi, zakończenie jego pracy,
  następnie sprawdzenie na kolejnym.

## Wymiana zmian między kolejnymi Kodi

- Zachowaj działającą w silniku v1 zasadę: każde Kodi zapisuje własny snapshot, a przy
  swoim cyklu odczytuje snapshoty pozostałych klientów. Nie wymagaj, by dwa
  Kodi były aktywne jednocześnie ani by zgadzały się co do kolejności startu.
- Zmiany są rozstrzygane oddzielnie dla każdego pola na podstawie wersji
  logicznych, a nie czasu uruchomienia Kodi. Zwycięska wersja jest zachowywana
  przy zastosowaniu na kolejnym kliencie, aby nie powstała pętla zmian.
- Po zastosowaniu zdalnego stanu wtyczka odświeża własny snapshot. Nowe media
  i zmiany po skanowaniu biblioteki nie mogą samą swoją obecnością nadpisać
  zapamiętanego stanu synchronizowanego.
- W sprawdzonym kodzie silnika v1 snapshot peer starszy niż 7 dni jest oznaczany jako
  błąd odczytu, a cykl LIVE przerywa zastosowanie zmian przy błędzie snapshotu.
  To ograniczenie nie spełnia literalnie wymagania catch-up po dowolnie długiej
  przerwie. V2 ma zachować niezależność od odstępu między uruchomieniami;
  nie kopiować tej granicy wieku bez rozwiązania, które nie odrzuca ważnych
  zmian tylko dlatego, że Kodi długo nie było uruchamiane.

## Różny zakres dostępu do zasobów i współdzielona numeracja SudoSync ID

- Różne instancje Kodi w tej samej sieci mogą mieć dostęp do całości lub tylko do części zasobów wideo na NAS (np. jedno Kodi widzi wyłącznie katalog dla dzieci, a inne ma dostęp do pełnej biblioteki).
- Po zaindeksowaniu „całości” na jednym Kodi z mniejszym dostępem, kolejne Kodi z szerszym dostępem musi bezawaryjnie zaindeksować i zsynchronizować swoją bibliotekę:
  - W wersji 2.0.5 identyfikator `SudoSync ID` (`{prefix}-{num:06d}`) jest nadawany wszystkim pozycjom wideo w bibliotece jako niezależny, równoległy identyfikator sieci lokalnej (niezależnie od tego, czy plik posiada już identyfikatory IMDb/TMDb/TVDb/Trakt).
  - Jedynym wyjątkiem są pozycje posiadające już niepusty identyfikator typu `sudosync` (nie nadpisujemy istniejącego SudoSync ID).
  - Podczas wstrzykiwania do pliku `.nfo` zachowywane są wszystkie istniejące znaczniki `<uniqueid>`, a tag `sudosync` otrzymuje `default="false"` (jeżeli w pliku jest już inny tag `default="true"`) lub `default="true"` (gdy w pliku nie ma tagu domyślnego).
  - Wszystkie instancje Kodi w sieci współdzielą informację o najwyższym użytym numerze `SudoSync ID` (`sudosync_id_last_number` w `SudoSync.cfg` i `system/config.json`) i kontynuują numerację bez duplikacji ani nadpisywania identyfikatorów zapisanych już w plikach `.nfo` na NAS.
  - Zasoby zaindeksowane na nowym Kodi jako nieobejrzane (`lc=0, seq=0`) przejmują istniejący stan obejrzenia z innych urządzeń bez zgłaszania fałszywego konfliktu wersji początkowej.
  - Operacja przypisywania SudoSync ID (zarówno faza 1: odczyt i skanowanie biblioteki, jak i faza 2: nadawanie identyfikatorów) korzysta ze wspólnego modalnego okna dialogowego postępu (`DialogProgress`) na pierwszym planie z aktywnym przyciskiem „Anuluj”. Okno wyświetla bieżący etap, licznik pozycji i postęp procentowy oraz zawiera wyraźną informację o automatycznym zamknięciu po zakończeniu. W przypadku kliknięcia „Anuluj” operacja jest bezpiecznie i atomowo przerywana po bieżącej operacji, ze zwolnieniem blokad i bezpiecznym wycofaniem częściowych zmian.

## Stan implementacji i testowania

- Logika domenowa v2, adaptery JSON-RPC/VFS, kreator pierwszego uruchomienia i migracji (`sudosync_bootstrap.py`), dzierżawa pojedynczego aktywnego Kodi (`active_kodi_lock.json`), strażnik minimalnej wersji sieciowej (`min_addon_version` w `SudoSync.cfg`), współdzielony licznik `sudosync_id_last_number`, aktualizator GitHub Releases / lokalnego ZIP oraz pełny cykl `v2_runtime.sync_cycle()` są zaimplementowane i podłączone do `service.py` oraz `default.py`.
- Moduły te są pokryte zautomatyzowanymi testami jednostkowymi (z atrapami API Kodi). Testy funkcjonalne w rzeczywistym środowisku Kodi są prowadzone odrębnie przez Agenta Antigravity i właściciela projektu w ramach testów wersji Beta.


