# Moduły Zaawansowane: <br> FreeRTOS, Wi-Fi oraz BLE 
---
Niniejsza sekcja została przygotowana z myślą o osobach, które opanowały już podstawy programowania mikrokontrolerów i chcą poznać techniki stosowane w zaawansowanych systemach wbudowanych (*embedded systems*). 

Podczas realizacji poniższych modułów będziemy korzystać z naszej płytki edukacyjnej z układem **ESP32-C6**. Skupimy się natomiast na oprogramowaniu: prawdziwej wielozadaniowości, budowie interfejsów sieciowych oraz komunikacji w standardzie Bluetooth Low Energy.

---

## MODUŁ 1: Wprowadzenie do FreeRTOS – Prawdziwa wielozadaniowość

W klasycznym podejściu programy pisane w środowisku Arduino działają wewnątrz jednej głównej pętli `loop()`. Jeśli chcemy realizować kilka operacji jednocześnie, musimy tworzyć rozbudowane maszyny stanów i stale kontrolować upływ czasu za pomocą funkcji `millis()`. Takie rozwiązanie staje się trudne w utrzymaniu, szczególnie gdy poszczególne procesy wymagają precyzyjnego rygoru czasowego (np. bezwzględnego wywoływania kodu dokładnie co 10 ms). Istnieje jednak o wiele wygodniejsza alternatywa:

Układ ESP32-C6 natywnie działa pod kontrolą systemu operacyjnego czasu rzeczywistego **FreeRTOS** (*Free Real-Time Operating System*). System ten posiada **planistę (schedulera)**, który potrafi przełączać kontekst wykonania i przydzielać czas procesora do niezależnych bloków kodu, nazywanych **Zadaniami (Tasks)**. Każde zadanie posiada własny stos pamięci oraz przypisany priorytet.

### **Wyzwania programowania współbieżnego**
Dzielenie czasu procesora pomiędzy różne zadania daje ogromne możliwości, ale wprowadza również specyficzne dla systemów wielozadaniowych problemy architektoniczne, które programista musi wziąć pod uwagę:

> [!WARNING] Race Condition (Wyścig)
> Sytuacja, w której dwa lub więcej zadań próbuje niemal jednocześnie uzyskać dostęp do tego samego, współdzielonego zasobu (np. modyfikować tę samą zmienną globalną lub zapisywać dane do jednego portu komunikacyjnego) bez odpowiedniej synchronizacji. 
> 
> Ostateczny wynik operacji staje się nieprzewidywalny i zależy od tego, które zadanie akurat zostało wstrzymane lub wznowione przez planistę. Aby temu zapobiec, stosuje się mechanizmy blokujące dostęp innym zadaniom na czas operacji, np. **Muteksy (Mutex)**.

> [!CAUTION] Deadlock (Zakleszczenie)
> Krytyczny błąd w programie współbieżnym, w którym dwa lub więcej zadań blokuje się nawzajem w nieskończoność. 
> 
> Dochodzi do tego, gdy Zadanie A zablokowało dostęp do Zasobu X i czeka na zwolnienie Zasobu Y, podczas gdy Zadanie B posiada zablokowany Zasób Y i nieskończenie czeka na zwolnienie Zasobu X. W efekcie oba zadania zamrażają swoje działanie na stałe.

### Ćwiczenie 1: Niezależne zadania migania diodami
W tym ćwiczeniu zrezygnujemy z kodu w pętli `loop()`. Zaimplementujemy dwa oddzielne zadania: jedno będzie sterować diodą `LED1`, a drugie diodą `LED2`. Każde z zadań będzie działać z zupełnie innym opóźnieniem.

#### Uzupełnij kod i wgraj na płytkę:
```cpp
const int PIN_LED1 = 2;
const int PIN_LED2 = 3;

// Nagłówki naszych funkcji zadań
void TaskDioda1(void *pvParameters);
void TaskDioda2(void *pvParameters);

void setup() {
  Serial.begin(115200);
  
  // Konfiguracja pinów
  pinMode(PIN_LED1, OUTPUT);
  pinMode(PIN_LED2, OUTPUT);

  // Tworzenie Zadania 1
  xTaskCreate(
    TaskDioda1,     // Funkcja realizująca kod zadania
    "ZadanieLED1",  // Nazwa zadania
    1024,           // Rozmiar stosu w bajtach, zazwyczaj potęga 2
    NULL,           // Parametry wejściowe
    1,              // Priorytet (1 - niski)
    NULL            // Uchwyt
  );

  // UZUPEŁNIJ: Utwórz Zadanie 2 (TaskDioda2) o nazwie "ZadanieLED2", z priorytetem 1
  xTaskCreate(
    
  );

  Serial.println("Zadania FreeRTOS uruchomione!");
}

void loop() {
  // Pętla główna pozostaje pusta - zadania działają w tle
  vTaskDelete(NULL); 
}

// ================= IMPLEMENTACJA ZADAŃ =================

void TaskDioda1(void *pvParameters) {
  // Nieskończona pętla zadania
  for (;;) {
    digitalWrite(PIN_LED1, HIGH);

    // Makro pdMS_TO_TICKS przelicza czas w milisekundach na tzw. ticki (takty) planisty.
    // Ticki to bazowe jednostki czasu, w jakich FreeRTOS odmierza działanie systemu 
    vTaskDelay(pdMS_TO_TICKS(200));
    
    digitalWrite(PIN_LED1, LOW);
    vTaskDelay(pdMS_TO_TICKS(200));
  }
}

void TaskDioda2(void *pvParameters) {
  for (;;) {
    digitalWrite(PIN_LED2, HIGH);
    // UZUPEŁNIJ: Ustaw inne opóźnienie, np. 555 milisekund
    vTaskDelay();
    
    digitalWrite(PIN_LED2, LOW);
    // UZUPEŁNIJ: Ustaw opóźnienie, np. 555 milisekund
    vTaskDelay();
  }
}
```

#### Zadanie do samodzielnego wykonania:
Stwórz w programie **trzecie zadanie** (np. `TaskLicznik`), które posiada wyższy priorytet (`2`) i w nieskończonej pętli co sekundę wypisuje na port szeregowy kolejną liczbę (licznik sekund). Zaobserwuj w Monitorze Szeregowym, w jaki sposób FreeRTOS współdzieli czas procesora dla wszystkich trzech procesów.

> [!NOTE] Ciekawostka
> **Pętla loop() to również zadanie!**
> Warto wiedzieć, że w środowisku programistycznym dla układów ESP32 standardowe funkcje `setup()` oraz `loop()` pod spodem nie działają "magicznie" poza systemem. Środowisko automatycznie tworzy dla nich domyślne zadanie FreeRTOS o nazwie `loopTask` i priorytecie 1. Pisząc zwykły kod w Arduino, od zawsze programowałeś wewnątrz zadania FreeRTOS, nawet o tym nie wiedząc! Właśnie dlatego w naszym kodzie mogliśmy bezpiecznie usunąć to domyślne zadanie za pomocą instrukcji `vTaskDelete(NULL)`, zwalniając przydzieloną mu pamięć.

> [!IMPORTANT] Task Watchdog Timer (TWDT)
> **Dlaczego zadanie bez opóźnienia resetuje płytkę?**
> W systemie FreeRTOS dla układów ESP32 stale czuwa wbudowany mechanizm nadzorcy – **Task Watchdog Timer**. Jeśli stworzysz zadanie o wysokim priorytecie, które zajmie procesor w nieskończonej pętli `for(;;)` lub `while(1)` bez wywołania funkcji oddającej czas planiście (takiej jak `vTaskDelay()` lub `yield()`), planista nie będzie w stanie przełączyć kontekstu na inne, krytyczne zadania systemowe (np. obsługę stosu Wi-Fi). 
> 
> Watchdog uzna, że program uległ zawieszeniu, i po jakimś czasie **zresetuje mikrokontroler**, wypisując w Monitorze Szeregowym charakterystyczny błąd: `Task watchdog got triggered`. Zawsze pamiętaj o dodawaniu opóźnień wewnątrz nieskończonych pętli zadań!

---

### Ćwiczenie 2: Bezpieczna wymiana danych – Kolejki (Queues)
W profesjonalnych aplikacjach wielozadaniowych dążymy do całkowitej eliminacji **zmiennych globalnych** przy przekazywaniu danych między poszczególnymi zadaniami. Służą do tego **Kolejki (Queues)**. Kolejka to bezpieczny bufor FIFO (*First-In, First-Out*), do którego jedno zadanie może wrzucać dane, a inne je stamtąd odbierać. Operacje na kolejce są automatycznie synchronizowane przez system, co całkowicie zabezpiecza program przed problemem *Race Condition*.

W tym ćwiczeniu stworzymy dwa zadania:
1. **`TaskNadajnik`**: Odczytuje napięcie z potencjometru (GPIO4) i bezpiecznie wysyła odczytaną wartość do kolejki za pomocą funkcji `xQueueSend`.
2. **`TaskOdbiornik`**: Czeka na pojawienie się nowej wartości w kolejce za pomocą funkcji `xQueueReceive`. Gdy dane nadejdą, wypisuje je w Monitorze Szeregowym i steruje jasnością diody (GPIO2).

#### Uzupełnij kod i wgraj na płytkę:
```cpp
const int PIN_LED = 2;
const int PIN_POTENCJOMETR = 4;

// Globalny uchwyt do naszej kolejki przechowującej liczby całkowite (int)
QueueHandle_t kolejkaDanych;

void TaskNadajnik(void *pvParameters);
void TaskOdbiornik(void *pvParameters);

void setup() {
  Serial.begin(115200);
  pinMode(PIN_LED, OUTPUT);

  // Tworzenie kolejki mogącej pomieścić maksymalnie 5 elementów typu int
  kolejkaDanych = xQueueCreate(5, sizeof(int));

  if (kolejkaDanych == NULL) {
    Serial.println("Blad: Nie udalo sie utworzyc kolejki!");
    return;
  }

  // Tworzenie zadania nadawczego
  xTaskCreate(TaskNadajnik, "Nadajnik", 2048, NULL, 1, NULL);
  
  // Tworzenie zadania odbiorczego z wyższym priorytetem
  xTaskCreate(TaskOdbiornik, "Odbiornik", 2048, NULL, 2, NULL);

  Serial.println("Zadania z kolejka uruchomione!");
}

void loop() {
  vTaskDelete(NULL);
}

void TaskNadajnik(void *pvParameters) {
  for (;;) {
    // Odczyt wartości z przetwornika ADC (potencjometr)
    int odczyt = analogRead(PIN_POTENCJOMETR);

    // UZUPEŁNIJ: Wyślij adres zmiennej '&odczyt' do kolejki 'kolejkaDanych'.
    // Trzeci argument portMAX_DELAY oznacza nieskończone oczekiwanie, jeśli kolejka jest pełna.
    xQueueSend(
      
    );

    // Pobieraj próbkę co 100 milisekund
    vTaskDelay(pdMS_TO_TICKS(100));
  }
}

void TaskOdbiornik(void *pvParameters) {
  int odebranaWartosc;

  for (;;) {
    // UZUPEŁNIJ: Odbierz dane z kolejki 'kolejkaDanych' i zapisz pod adresem '&odebranaWartosc'.
    // Użyj instrukcji warunkowej if (xQueueReceive(...) == pdPASS), która zablokuje zadanie w oczekiwaniu na dane.
    if (xQueueReceive(
      
    ) == pdPASS) {
      Serial.print("Odebrano z kolejki: ");
      Serial.println(odebranaWartosc);

      // Skalowanie odczytu (0-4095) na jasność PWM diody (0-255)
      int jasnosc = map(odebranaWartosc, 0, 4095, 0, 255);
      analogWrite(PIN_LED, jasnosc);
    }
  }
}
```

#### Zadanie do samodzielnego wykonania:
Spróbuj zmienić rozmiar kolejki przy tworzeniu (`xQueueCreate`) na **`1`** i zaobserwuj w Monitorze Szeregowym, czy wpływa to na płynność przekazywania danych. Następnie zmodyfikuj kod w funkcji `TaskNadajnik` dodając instrukcję warunkową, aby mikrokontroler wysyłał dane do kolejki **tylko wtedy**, gdy odczyt z potencjometru zmienił się o więcej niż 50 jednostek w stosunku do poprzedniego pomiaru. Pozwoli to zaoszczędzić czas procesora i nie zapychać kolejki identycznymi, powtarzającymi się wartościami!

---

## MODUŁ 2: Wi-Fi – Access Point, mDNS i Klient HTTP (REST API)

Mikrokontroler ESP32-C6 posiada wbudowany moduł sieci bezprzewodowej Wi-Fi. 

Układ może pracować w dwóch podstawowych trybach:

1. **Tryb Stacji (`WIFI_STA` - Station):** Mikrokontroler łączy się z zewnętrznym routerem (np. w Twoim domu) i staje się klientem w istniejącej sieci, uzyskując dostęp do Internetu.
2. **Tryb Punktu Dostępowego (`WIFI_AP` - Access Point):** Mikrokontroler sam staje się routerem i tworzy własną, nową sieć Wi-Fi, do której mogą podłączać się inne urządzenia (smartfony, laptopy).

Podczas pierwszej części zajęć wykorzystamy tryb **Access Point (`WIFI_AP`)**. Dzięki temu połączysz się z płytką bezpośrednio ze swojego telefonu, tworząc całkowicie niezależny system sterowania.

### Czym jest usługa DNS i protokół mDNS?
W standardowej komunikacji sieciowej urządzenia identyfikują się za pomocą liczbowych adresów IP (np. `192.168.4.1`). Z punktu widzenia człowieka zapamiętywanie ciągów liczb jest bardzo niewygodne. W Internecie problem ten rozwiązuje globalna usługa **DNS (Domain Name System)**, która działa jak potężna rozproszona książka telefoniczna – tłumaczy przyjazne nazwy domenowe (np. `google.com`) na odpowiadające im liczbowe adresy IP serwerów.

W małych, lokalnych sieciach Wi-Fi (gdzie nie ma centralnego serwera DNS) wykorzystuje się protokół **mDNS (Multicast DNS)**. Pozwala on urządzeniom w sieci lokalnej rozgłaszać swoją przyjazną nazwę hosta. Dzięki temu, zamiast wpisywać w przeglądarce surowy adres IP mikrokontrolera, możemy połączyć się z nim wpisując adres z końcówką `.local` (w naszym przypadku będzie to `http://esp32.local`).

### Serwowanie strony WWW z poziomu kodu C++
Aby wyświetlić interfejs graficzny w przeglądarce internetowej, mikrokontroler musi odesłać klientowi kod strony w języku HTML oraz style CSS. 

Kompletny kod strony zapiszemy bezpośrednio w pliku źródłowym w postaci stałego ciągu znaków. Służy do tego konstrukcja tzw. **surowego literału (Raw String Literal)** o składni `R"rawliteral(...)rawliteral"`, która pozwala wygodnie wklejać wielolinijkowy kod HTML zawierający cudzysłowy bez konieczności ich uciążliwego echowania.

### Ćwiczenie 3: Bezprzewodowy włącznik diody z obsługą mDNS
Stworzymy prosty serwer WWW z estetycznym interfejsem graficznym oraz aktywną usługą mDNS. Strona zawiera przyciski, po kliknięciu których przeglądarka wyśle do mikrokontrolera żądanie pod określony adres URL (np. `/on` lub `/off`), co spowoduje fizyczną zmianę stanu wyjścia GPIO.

#### Uzupełnij kod i wgraj na płytkę:
```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <ESPmDNS.h> // Biblioteka niezbędna do obsługi przyjaznych nazw mDNS

const int PIN_LED = 2;

// Konfiguracja naszej sieci Wi-Fi
// WAŻNE: Zmień nazwę sieci na własną, unikalną (np. dodaj swoje imię lub numer stacji)!
// W przeciwnym razie sieci różnych uczestników w jednej sali będą się zakłócać.
const char* nazwaSieci = "ESP32_Siec_Unikalna";
const char* hasloSieci = "12345678"; // Hasło musi mieć minimum 8 znaków

// Tworzymy serwer nasłuchujący na porcie 80 (standardowy port HTTP)
WebServer server(80);

// Surowy literał zawierający pełny kod naszej strony HTML i CSS
const char STRONA_HTML[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Serwer ESP32</title>
  <style>
    body {
      background-color: #1e1e2f;
      color: #ffffff;
      font-family: sans-serif;
      text-align: center;
      padding-top: 50px;
    }
    .karta {
      background: #2a2a3d;
      margin: 0 auto;
      max-width: 350px;
      padding: 30px;
      border-radius: 16px;
      box-shadow: 0 8px 20px rgba(0,0,0,0.5);
    }
    button {
      background-color: #ff6b6b;
      color: white;
      border: none;
      padding: 15px 30px;
      font-size: 18px;
      border-radius: 8px;
      cursor: pointer;
      margin: 10px;
      width: 80%;
    }
    button:hover { background-color: #ff5252; }
    .btn-off { background-color: #4d4d63; }
    .btn-off:hover { background-color: #3b3b4f; }
  </style>
</head>
<body>
  <div class="karta">
    <h2>Sterowanie ESP32</h2>
    <p>Wybierz akcję poniżej:</p>
    <!-- Kliknięcie przekierowuje przeglądarkę na odpowiednią podstronę -->
    <button onclick="location.href='/on'">Włącz Diodę</button>
    <button class="btn-off" onclick="location.href='/off'">Wyłącz Diodę</button>
  </div>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);
  pinMode(PIN_LED, OUTPUT);
  digitalWrite(PIN_LED, LOW);

  // Uruchom sieć Wi-Fi w trybie Access Point (softAP)
  WiFi.softAP(nazwaSieci, hasloSieci);
  
  Serial.print("Sieć utworzona! Połącz się z Wi-Fi: ");
  Serial.println(nazwaSieci);
  Serial.print("Adres IP strony WWW: ");
  Serial.println(WiFi.softAPIP());

  // Uruchomienie usługi mDNS z nazwą hosta "esp32"
  if (MDNS.begin("esp32")) {
    Serial.println("Usługa mDNS aktywna! Strona dostępna pod adresem: http://esp32.local");
  }

  // --- Konfiguracja tras (routingów) serwera ---

  // 1. Wyświetlenie strony głównej po wejściu na czysty adres IP lub domenę .local
  server.on("/", []() {
    server.send(200, "text/html", STRONA_HTML);
  });

  // 2. Obsługa włączenia diody po wejściu na link /on
  server.on("/on", []() {
    digitalWrite(PIN_LED, HIGH);
    
    // Po wykonaniu akcji odsyłamy klienta z powrotem na stronę główną
    server.sendHeader("Location", "/");
    server.send(303); 
  });

  // 3. Obsługa wyłączenia diody po wejściu na link /off
  // UZUPEŁNIJ: Dopisz obsługę metody server.on dla ścieżki "/off", która gasi diodę
  server.on(
    
  );

  // Uruchomienie serwera HTTP
  server.begin();
}

void loop() {
  // Obsługa nadchodzących połączeń od przeglądarek
  server.handleClient();
}
```

#### Zadanie do samodzielnego wykonania:
Zmodyfikuj kod strony HTML oraz logikę serwera w C++, aby dodać obsługę **drugiej diody** znajdującej się na płytce (GPIO3).
1. Dopisz w kodzie HTML kolejne dwa przyciski (np. *Włącz Diodę 2* i *Wyłącz Diodę 2*), kierujące na trasy `/on2` oraz `/off2`.
2. Zarejestruj w funkcji `setup()` odpowiednie procedury `server.on(...)` obsługujące te nowe ścieżki.

---

### Architektura REST API i praca w trybie Stacji (STA)
Do tej pory nasz mikrokontroler pełnił rolę Serwera, który czekał na żądania od przeglądarki. W świecie Internetu Rzeczy równie często zależy nam na odwrotnej sytuacji: mikrokontroler staje się **klientem**, który aktywnie łączy się z zewnętrznymi serwisami w Internecie, aby pobrać dane (np. aktualną prognozę pogody, dokładny czas z serwera NTP czy kursy walut) lub wysłać pomiary z czujników do chmury.

Wykorzystuje się do tego architekturę **REST API** oraz powszechny protokół HTTP. Dane w nowoczesnych serwisach najczęściej wymieniane są w lekkim, ustrukturyzowanym formacie **JSON** (*JavaScript Object Notation*) lub jako czysty tekst.

Aby mikrokontroler uzyskał dostęp do globalnej sieci Internet, musimy przełączyć go w tryb **Stacji (`WIFI_STA`)** i podać mu dane logowania do istniejącego routera z wyjściem na świat (np. przenośnego punktu dostępowego / hotspotu udostępnionego z Twojego smartfona).

### Ćwiczenie 4: Klient HTTP – Pobieranie danych z publicznego REST API
W tym ćwiczeniu połączymy się z ogólnodostępnym, darmowym API serwującym losowe ciekawostki. Użyjemy wbudowanej biblioteki `HTTPClient`, aby wysłać zapytanie `GET` pod określony adres URL, a następnie odczytamy odpowiedź serwera.

> [!IMPORTANT] Konfiguracja Hotspotu
> Przed wgraniem kodu włącz w swoim smartfonie funkcję **Przenośny punkt dostępowy (Hotspot Wi-Fi)**. Wpisz nazwę swojej udostępnionej sieci (SSID) oraz hasło w odpowiednich zmiennych w poniższym kodzie.

#### Uzupełnij kod i wgraj na płytkę:
```cpp
#include <WiFi.h>
#include <HTTPClient.h>

// UZUPEŁNIJ: Wpisz dane logowania do hotspotu w swoim telefonie
const char* ssid = "Nazwa_Twojego_Hotspotu";
const char* password = "Haslo_Do_Hotspotu";

// Adres publicznego API serwującego losowy fakt w formacie JSON
const char* urlAPI = "https://uselessfacts.jsph.pl/api/v2/facts/random";

void setup() {
  Serial.begin(115200);
  
  // Przełączenie w tryb stacji (klienta)
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  Serial.print("Laczenie z siecia Wi-Fi");
  
  // Oczekiwanie na poprawne połączenie z routerem
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nPolaczono z Wi-Fi!");
  Serial.print("Przypisany adres IP: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  // Sprawdzamy, czy połączenie z siecią jest nadal aktywne
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    Serial.println("\nWysylanie zapytania do REST API...");
    
    // Konfiguracja docelowego adresu URL
    http.begin(urlAPI);
    
    // UZUPEŁNIJ: Wykonaj zapytanie metodą GET za pomocą funkcji http.GET()
    int kodOdpowiedzi = ;
    
    // Kod 200 (HTTP_CODE_OK) oznacza standardowy sukces HTTP
    if (kodOdpowiedzi > 0) {
      Serial.print("Kod odpowiedzi HTTP: ");
      Serial.println(kodOdpowiedzi);
      
      if (kodOdpowiedzi == HTTP_CODE_OK) {
        // Pobranie pełnej treści odpowiedzi z serwera jako ciąg znaków (String)
        String odpowiedz = http.getString();
        Serial.println("Odebrane dane z serwera:");
        Serial.println(odpowiedz);
      }
    } else {
      Serial.print("Blad zapytania HTTP: ");
      Serial.println(http.errorToString(kodOdpowiedzi).c_str());
    }
    
    // Zamknięcie połączenia i zwolnienie zasobów
    http.end();
  } else {
    Serial.println("Rozlaczono z siecia Wi-Fi!");
  }
  
  // Odczekaj 10 sekund przed kolejnym zapytaniem
  delay(10000);
}
```

#### Zadanie do samodzielnego wykonania:
Odebrany z serwera tekst zawiera surowy format JSON (np. `{"id":"...","text":"Treść faktu","source":"..."}`). W profesjonalnych projektach do wyciągania poszczególnych pól z takiego ciągu znaków nie używa się ręcznego wycinania tekstu, lecz zewnętrznej biblioteki **ArduinoJson**. 

Zainstaluj w Menedżerze Bibliotek wtyczkę `ArduinoJson` (autor: Benoit Blanchon). Następnie spróbuj sparsować zmienną `odpowiedz` i zaktualizować program tak, aby wypisywał w Monitorze Szeregowym **wyłącznie samą treść faktu** (wartość kryjącą się pod kluczem `"text"`), pomijając całą resztę znaczników i nawiasów JSON!

---

## MODUŁ 3: Bluetooth Low Energy (BLE) – Podstawy protokołu

W świecie nowoczesnego Internetu Rzeczy (IoT) **klasyczny Bluetooth (Bluetooth Classic)** powoli odchodzi do lamusa. Ze względu na konieczność ciągłego podtrzymywania prądożernego połączenia, standard ten ustąpił miejsca technologii **Bluetooth Low Energy (BLE)**.

Protokół BLE został zoptymalizowany pod kątem minimalnego zużycia energii. Urządzenia przesyłają krótkie pakiety danych i natychmiast wracają do trybu uśpienia, co pozwala na wieloletnią pracę na małych bateriach.

### Struktura profilu GATT
Komunikacja w standardzie BLE opiera się na architekturze **GATT** (*Generic Attribute Profile*). W tym modelu nasza płytka pełni rolę **Serwera**, a łączący się z nią smartfon to **Klient**.

Struktura serwera wygląda następująco:
1. **Usługa (Service):** Główny kontener grupujący powiązane funkcjonalności w postaci charakterystyk.
2. **Charakterystyka (Characteristic):** Konkretny punkt wymiany danych wewnątrz usługi. Każda charakterystyka definiuje **właściwości**, określające dozwolone operacje:
   * **Read:** Zezwala klientowi na odczytanie wartości.
   * **Write:** Zezwala klientowi na przesłanie nowej wartości do serwera.
   * **Notify:** Serwer samoczynnie wysyła nową wartość do klienta w momencie jej zmiany.
3. **UUID (Universally Unique Identifier):** Unikalny identyfikator liczbowy przypisany do każdej usługi i charakterystyki, pozwalający jednoznacznie zidentyfikować jej przeznaczenie.

### Ćwiczenie 5: Odbieranie komend ze smartfona przez BLE
Skonfigurujemy ESP32-C6 jako serwer BLE udostępniający jedną usługę z charakterystyką zapisu (Write). Klientem będzie uniwersalna aplikacja narzędziowa **nRF Connect for Mobile** (dostępna bezpłatnie na systemy Android oraz iOS), z poziomu której prześlemy liczbowe komendy sterujące diodą.

#### Uzupełnij kod i wgraj na płytkę:
```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>

const int PIN_LED = 2;

// Definiujemy unikalne identyfikatory UUID dla naszej usługi i charakterystyki.
// Wskazówka: Aby uniknąć potencjalnych konfliktów w laboratorium, warto zmodyfikować 
// kilka znaków w poniższych identyfikatorach na własne.
#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "e322b14e-5100-4b2e-b611-6677945d8b6c"

// Klasa nasłuchująca zdarzeń zapisu z telefonu do charakterystyki
class ObslugaZapisu: public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *pCharacteristic) {
      // Pobranie wartości przesłanej przez klienta
      String wartosc = pCharacteristic->getValue();

      if (wartosc.length() > 0) {
        Serial.print("Odebrano dane BLE (HEX): ");
        for (int i = 0; i < wartosc.length(); i++) {
          Serial.printf("%02X ", (uint8_t)wartosc[i]);
        }
        Serial.println();

        // Sprawdź, czy pierwszy odebrany bajt (wartosc[0]) to 1
        // Jeśli tak, włącz diodę. Jeśli to 0, wyłącz diodę.
        if (wartosc[0] == '1') {
          digitalWrite(PIN_LED, HIGH);
        } else if (wartosc[0] == '0') {
          digitalWrite(PIN_LED, LOW);
        }
      }
    }
};

void setup() {
  Serial.begin(115200);
  pinMode(PIN_LED, OUTPUT);

  // Inicjalizacja urządzenia BLE o widocznej nazwie.
  // WAŻNE: Zmień nazwę urządzenia, aby odróżnić swoją płytkę od płytek innych osób w sali!
  BLEDevice::init("ESP32_BLE_Unikalna");

  // Tworzenie serwera BLE
  BLEServer *pServer = BLEDevice::createServer();

  // UZUPEŁNIJ: Utwórz usługę w serwerze, przekazując zdefiniowany makrem SERVICE_UUID
  BLEService *pService = pServer->createService(
    
  );

  // Tworzenie charakterystyki z właściwością WRITE (Zapis).
  // Właściwość PROPERTY_WRITE nadaje uprawnienia pozwalające zewnętrznemu klientowi (aplikacji)
  // na aktywne przesyłanie i zapisywanie nowych komend do tej charakterystyki.
  BLECharacteristic *pCharacteristic = pService->createCharacteristic(
                                         CHARACTERISTIC_UUID,
                                         BLECharacteristic::PROPERTY_WRITE
                                       );

  // Przypisanie naszej klasy obsługującej zdarzenie zapisu
  pCharacteristic->setCallbacks(new ObslugaZapisu());

  // Uruchomienie usługi
  pService->start();

  // Konfiguracja i uruchomienie rozgłaszania (Advertising), aby płytka była widoczna
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  BLEDevice::startAdvertising();

  Serial.println("Serwer BLE gotowy! Otwórz aplikację nRF Connect.");
}

void loop() {
  // Komunikacja BLE realizowana jest asynchronicznie w tle
  delay(2000);
}
```

#### Instrukcja testowania:
1. Uruchom aplikację **nRF Connect for Mobile** na swoim smartfonie.
2. Wyszukaj urządzenie po nazwie i kliknij **CONNECT**.
3. Rozwiń listę przy usłudze o identyfikatorze `4fafc201-...`.
4. Przy widocznej charakterystyce kliknij ikonę **strzałki w górę** (Write).
5. Wybierz typ danych **BYTE** lub **UINT8**, wpisz wartość **`01`** i wyślij. Dioda natychmiast się włączy! Przesłanie wartości **`00`** wyłączy ją.

#### Zadanie do samodzielnego wykonania:
Zmień obsługę logiki w metodzie `onWrite`, aby płytka reagowała na inne, wybrane przez Ciebie wartości liczbowe (np. przesłanie bajtu `0x02` włącza drugą diodę podłączoną do GPIO3, a `0x03` gasi obie).

---

### Wysłanie danych w czasie rzeczywistym – Powiadomienia (Notify)
W poprzednim ćwiczeniu to aplikacja na smartfonie aktywnie przesyłała polecenia do mikrokontrolera (zapis). W systemach IoT opartych na pomiarach zależy nam na sytuacji odwrotnej: mikrokontroler samoczynnie informuje smartfon o nowym odczycie natychmiast po jego wykonaniu, bez konieczności ciągłego, ręcznego odpytywania (Read) ze strony użytkownika.

Służy do tego właściwość **Notify (Powiadomienia)**. Aby jednak klient (smartfon) mógł odbierać powiadomienia, musi najpierw wyrazić na to zgodę (tzw. subskrypcja). Technicznie realizowane jest to poprzez wpisanie odpowiedniej flagi do specjalnego rejestru konfiguracyjnego charakterystyki, nazywanego **Deskryptorem CCCD** (*Client Characteristic Configuration Descriptor*) o ustandaryzowanym identyfikatorze UUID **`0x2902`**.

W bibliotece BLE dla środowiska Arduino służy do tego dedykowana klasa `BLE2902`.

### Ćwiczenie 6: Przesyłanie odczytu z potencjometru do smartfona (Notify)
Skonfigurujemy mikrokontroler tak, aby odczytywał napięcie z potencjometru (GPIO4) i cyklicznie przesyłał zaktualizowany wynik do podłączonego smartfona w formie bezprzewodowego powiadomienia BLE.

> [!IMPORTANT] Pamiętaj o unikalnych UUID
> Aby nowa usługa nie weszła w konflikt z pamięcią podręczną (cache) aplikacji nRF Connect z poprzedniego ćwiczeniem, w poniższym kodzie zdefiniowaliśmy zupełnie nowe, odrębne identyfikatory UUID dla usługi i charakterystyki.

#### Uzupełnij kod i wgraj na płytkę:
```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h> // Biblioteka niezbędna do obsługi deskryptora powiadomień

const int PIN_POTENCJOMETR = 4;

// Nowe, unikalne identyfikatory dla usługi Notify
#define SERVICE_UUID_NOTIFY        "18a55060-705d-4ab0-9b4e-86e0c0903330"
#define CHARACTERISTIC_UUID_NOTIFY "29f37c35-1521-419b-abf7-2d4dfa666e10"

BLEServer* pServer = NULL;
BLECharacteristic* pCharacteristicNotify = NULL;
bool urzadzeniePolaczone = false;

// Klasa śledząca stan połączenia (czy klient podłączył się lub rozłączył)
class ObslugaSerwera: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
      urzadzeniePolaczone = true;
      Serial.println("Smartfon polaczony z serwerem BLE!");
    };

    void onDisconnect(BLEServer* pServer) {
      urzadzeniePolaczone = false;
      Serial.println("Smartfon rozlaczony. Automatyczne wznowienie rozglaszania...");
      // Wznowienie widoczności płytki po rozłączeniu klienta
      BLEDevice::startAdvertising();
    }
};

void setup() {
  Serial.begin(115200);

  // Inicjalizacja BLE z unikalną nazwą
  BLEDevice::init("ESP32_Potencjometr_BLE");

  // Tworzenie serwera i przypisanie obsługi zdarzeń połączenia
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ObslugaSerwera());

  // Tworzenie usługi Notify
  BLEService *pService = pServer->createService(SERVICE_UUID_NOTIFY);

  // Tworzenie charakterystyki z właściwościami READ oraz NOTIFY
  pCharacteristicNotify = pService->createCharacteristic(
                            CHARACTERISTIC_UUID_NOTIFY,
                            BLECharacteristic::PROPERTY_READ   |
                            BLECharacteristic::PROPERTY_NOTIFY
                          );

  // UZUPEŁNIJ: Dodaj deskryptor CCCD (new BLE2902()) do charakterystyki, 
  // co pozwoli smartfonowi włączyć subskrypcję powiadomień!
  pCharacteristicNotify->addDescriptor(
    
  );

  // Uruchomienie usługi i aktywacja rozgłaszania
  pService->start();
  
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID_NOTIFY);
  BLEDevice::startAdvertising();

  Serial.println("Serwer BLE Notify gotowy! Otworz aplikacje nRF Connect.");
}

void loop() {
  // Jeśli smartfon jest połączony, przesyłaj mu na bieżąco odczyty z potencjometru
  if (urzadzeniePolaczone) {
    int odczytADC = analogRead(PIN_POTENCJOMETR);

    // Konwersja liczby całkowitej na ciąg znaków (String)
    String wartoscTekstowa = String(odczytADC);

    // Aktualizacja wartości wewnątrz charakterystyce
    pCharacteristicNotify->setValue(wartoscTekstowa.c_str());

    // UZUPEŁNIJ: Wywołaj metodę notify() na obiekcie pCharacteristicNotify, 
    // aby natychmiastowo przesłać zaktualizowaną wartość do smartfona
    pCharacteristicNotify->
    
    Serial.print("Wyslano powiadomienie BLE: ");
    Serial.println(wartoscTekstowa);
  }

  // Odczekaj pół sekundy przed kolejnym pomiarem
  delay(500);
}
```

#### Instrukcja testowania:
1. Połącz się z urządzeniem **ESP32_Potencjometr_BLE** w aplikacji **nRF Connect**.
2. Odszukaj charakterystykę o identyfikatorze `29f37c35-...`.
3. Zauważysz przy niej ikonę **wielokrotnych strzałek w dół** (Notify). Kliknij ją, aby włączyć subskrypcję powiadomień (ikona zmieni kolor lub zniknie jej przekreślenie).
4. Zaczynaj powoli kręcić potencjometrem na płytce. Na ekranie smartfona w czasie rzeczywistym będą pojawiać się odczytywane wartości napięcia bez konieczności klikania przycisku odświeżania!

#### Zadanie do samodzielnego wykonania:
Ciągłe wysyłanie pakietów radiowych co 500 ms, nawet gdy potencjometr leży całkowicie nieruchomo, niepotrzebnie zużywa energię baterii telefonu i mikrokontrolera. 

Zmodyfikuj kod w pętli `loop()` dodając statyczną lub globalną zmienną pomocniczą `ostatniOdczyt`. Zaprogramuj logikę tak, aby mikrokontroler wywoływał powiadomienie `notify()` **wyłącznie w sytuacji**, gdy aktualny odczyt z potencjometru różni się od poprzedniego pomiaru o co najmniej 50 jednostek.

---

> [!NOTE] Dla ciekawskich
> **Standaryzacja usług i identyfikatory Assigned Numbers**
> W naszym kodzie użyliśmy długich, 128-bitowych identyfikatorów UUID wygenerowanych losowo. Warto wiedzieć, że organizacja certyfikująca *Bluetooth SIG* posiada ścisłą listę ustandaryzowanych, krótkich (16-bitowych) identyfikatorów dla powszechnych typów usług i charakterystyk (np. oficjalny profil *Heart Rate Service* ma zawsze identyfikator `0x180D`, a poziom baterii to `0x180F`) oraz oficjalne kody identyfikujące poszczególnych producentów sprzętu.
> 
> Dzięki temu Twój smartfon automatycznie wie, jaki jest poziom naładowania baterii w podłączonych słuchawkach bezprzewodowych – ponieważ format i identyfikator tej informacji są globalnie ustandaryzowane.
> 
> Pełny wykaz tych zdefiniowanych wartości można znaleźć w oficjalnej dokumentacji standardu:  
> 🔗 **[Oficjalny spis Assigned Numbers w BLE (PDF)](https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Assigned_Numbers/out/en/Assigned_Numbers.pdf)**
>
> **Kurs Nordic Semiconductor**  
> Jeśli chcesz dogłębnie zgłębić architekturę i zaawansowane mechanizmy protokołu BLE, to polecamy ukończenie darmowego kursu od wiodącego producenta układów radiowych:  
> 👉 **[Nordic Semiconductor Academy – Bluetooth Low Energy Fundamentals](https://academy.nordicsemi.com/courses/bluetooth-low-energy-fundamentals/)**  
> *Uwaga: Kurs ten jest prowadzony w oparciu o zaawansowany system operacyjny **Zephyr OS** (w ramach nRF Connect SDK), stanowiący potężną, przemysłową alternatywę dla środowiska Arduino.*
