### 1. Wstęp: Kontekst i Różnice (MCU vs PC)

* **Punkt wyjścia:** Zwykły komputer (PC/Smartfon) vs Mikrokontroler (MCU).
* **Komputer (Procesor aplikacyjny):** Potężny, dużo RAM-u, system operacyjny (Windows/Linux). Robi wiele rzeczy naraz (muzyka, internet, dokumenty). Bywa nieprzewidywalny (ścinki, zawieszenia).
* **Mikrokontroler (Procesor czasu rzeczywistego):** Komputer na jednym układzie scalonym.
* *Jedno zadanie:* Brak systemu operacyjnego, kręci w pętli jeden wgrany program.
* *Peryferia:* Procesor, pamięć (Flash, RAM) i piny wejścia/wyjścia (GPIO) w jednym małym chipie.
* *Czas rzeczywisty:* Reaguje w ułamku milisekundy. Kod wykona się zawsze w przewidywalnym czasie.


* **Po co to jest?** Są małe, tanie i super energooszczędne (miesiące na baterii). Nie odpalisz na tym gry, ale do wyświetlania godziny czy sterowania piecem to idealne rozwiązanie.

---

### 2. Gdzie to siedzi? (Zastosowania)

Otaczają nas wszędzie (systemy wbudowane / *embedded*):

* **AGD:** Pralki, mikrofale, ekspresy.
* **Motoryzacja:** ABS, poduszki, sterowanie silnikiem (dziesiątki MCU w jednym aucie).
* **Smart Home / IoT:** Inteligentne gniazdka, żarówki (współczesne MCU często mają wbudowane **WiFi/Bluetooth**!).
* **Drony:** Używają MCU do obliczania lotu. Aby wiedzieć, jak szybko i w którą stronę lecą, używają akcelerometru/żyroskopu (powiązanie z dzisiejszymi zajęciami).
* **Medycyna/Przemysł:** Rozruszniki serca, roboty fabryczne.

---

### 3. Ekosystem i Debugowanie

* **Co to "Arduino"? (Ważne rozróżnienie):**
* *Płytka:* Fizyczny sprzęt.
* *IDE:* Program na komputerze do pisania kodu.
* *Framework:* Zbiór gotowych funkcji (ułatwień) w kodzie.


* **Debugowanie:** W zaawansowanych projektach używa się profesjonalnych narzędzi (debuggerów), ale na co dzień i na start używa się po prostu wrzucania tekstu na ekran (słynne `print()`).

---

### 4. Techniczne Mięso: Piny i Komunikacja

* **GPIO (General Purpose Input/Output):** Piny ogólnego przeznaczenia. Mogą działać jako Wejście (czytanie) lub Wyjście (sterowanie).
* **Sygnały (Zera i Jedynki):** Fizycznie to stan niski (0V) i wysoki (np. 3.3V).
* **Digital (Cyfrowe):** Widzi tylko skrajności (0 lub 1 / wyłączony lub włączony).
* **Analog (Analogowe):** Czyta małe zakresy napięcia i przypisuje im liczby (rozdzielczość). Np. `0V` to `0`, a `3.3V` to maksymalna wartość `4095`.

**Protokoły komunikacyjne (jak podpinamy zewnętrzne moduły):**

* **UART:** Komunikacja *asynchroniczna*. Brak zegara. Obie strony muszą się z góry umówić na prędkość gadania (baud rate).
* **I2C:** Komunikacja *synchroniczna*. Jest osobna linia dla zegara (synchronizuje tempo) i linia danych. Tylko jedno urządzenie gada na raz, żeby się nie zagłuszały.

---

### 5. Część Praktyczna (Plan działania ze studentami)

1. **Serial print:** Pierwsze uruchomienie (wspomnieć od razu: *to używa właśnie protokołu UART pod spodem!*).
2. **Digital Output:** Podstawowe sterowanie wyjściem cyfrowym.
3. **Analog Input:** Czytanie wartości z otoczenia.
4. **Analog PWM Output:** Udawanie sygnału analogowego, np. do płynnego sterowania jasnością diody.
5. **I2C i czujnik ADXL:** Podpięcie zewnętrznego akcelerometru (jak w dronach).
6. **Komunikacja:** Gadanie między dwoma modułami ESP32.
7. **Zadanie Extra:** Dodatkowe wyzwanie dla najbardziej ambitnych w grupie.
