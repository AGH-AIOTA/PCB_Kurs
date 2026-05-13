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

---

## MODUŁ 2: Wi-Fi – Access Point i wbudowany serwer WWW

Mikrokontroler ESP32-C6 posiada wbudowany moduł sieci bezprzewodowej Wi-Fi. 

Układ może pracować w dwóch podstawowych trybach:

1. **Tryb Stacji (`WIFI_STA` - Station):** Mikrokontroler łączy się z zewnętrznym routerem (np. w Twoim domu) i staje się klientem w istniejącej sieci, uzyskując dostęp do Internetu.
2. **Tryb Punktu Dostępowego (`WIFI_AP` - Access Point):** Mikrokontroler sam staje się routerem i tworzy własną, nową sieć Wi-Fi, do której mogą podłączać się inne urządzenia (smartfony, laptopy).

Podczas naszych zajęć wykorzystamy tryb **Access Point (`WIFI_AP`)**. Dzięki temu połączysz się z płytką bezpośrednio ze swojego telefonu, tworząc całkowicie niezależny system sterowania.

### Serwowanie strony WWW z poziomu kodu C++
Aby wyświetlić interfejs graficzny w przeglądarce internetowej, mikrokontroler musi odesłać klientowi kod strony w języku HTML oraz style CSS. 

Kompletny kod strony zapiszemy bezpośrednio w pliku źródłowym w postaci stałego ciągu znaków. Służy do tego konstrukcja tzw. **surowego literału (Raw String Literal)** o składni `R"rawliteral(...)rawliteral"`, która pozwala wygodnie wklejać wielolinijkowy kod HTML zawierający cudzysłowy bez konieczności ich uciążliwego echowania.

### Ćwiczenie 2: Bezprzewodowy włącznik diody
Stworzymy prosty serwer WWW z estetycznym interfejsem graficznym. Strona zawiera przyciski, po kliknięciu których przeglądarka wyśle do mikrokontrolera żądanie pod określony adres URL (np. `/on` lub `/off`), co spowoduje fizyczną zmianę stanu wyjścia GPIO.


#### Uzupełnij kod i wgraj na płytkę:
```cpp
#include <WiFi.h>
#include <WebServer.h>

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
    <!-- Kliknięcie przekierowuje przeglądarkę na odpowiedni podstronę -->
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

  // --- Konfiguracja tras (routingów) serwera ---

  // 1. Wyświetlenie strony głównej po wejściu na czysty adres IP
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

## MODUŁ 3: Bluetooth Low Energy (BLE) – Podstawy protokołu

W świecie nowoczesnego Internetu Rzeczy (IoT) **klasyczny Bluetooth (Bluetooth Classic)** powoli odchodzi do lamusa. Ze względu na konieczność ciągłego podtrzymywania prądożernego połączenia, standard ten ustąpił miejsca technologii **Bluetooth Low Energy (BLE)**.

Protokół BLE został zoptymalizowany pod kątem minimalnego zużycia energii. Urządzenia przesyłają krótkie pakiety danych i natychmiast wracają do trybu uśpienia, co pozwala na wieloletnią pracę na małych bateriach.

### Struktura profilu GATT
Komunikacja w standardzie BLE opiera się na architekturze **GATT** (*Generic Attribute Profile*). W tym modelu nasza płytka pełni rolę **Serwera**, a łączący się z nią smartfon to **Klient**.

Struktura serwera wygląda następująco:
1. **Usługa (Service):** Główny kontener grupujący powiązane funkcjonalności w postacji charakterystyk.
2. **Charakterystyka (Characteristic):** Konkretny punkt wymiany danych wewnątrz usługi. Każda charakterystyka definiuje **właściwości**, określające dozwolone operacje:
   * **Read:** Zezwala klientowi na odczytanie wartości.
   * **Write:** Zezwala klientowi na przesłanie nowej wartości do serwera.
   * **Notify:** Serwer samoczynnie wysyła nową wartość do klienta w momencie jej zmiany.
3. **UUID (Universally Unique Identifier):** Unikalny identyfikator liczbowy przypisany do każdej usługi i charakterystyki, pozwalający jednoznacznie zidentyfikować jej przeznaczenie.

### Ćwiczenie 3: Odbieranie komend ze smartfona przez BLE
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

  // UZUPEŁNIJ: Utwórz usługę w serwerze, przekazując zdefiniowany SERVICE_UUID
  BLEService *pService = pServer->createService();

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
