# Odbiornik alfabetu Morse'a (z wykorzystaniem czujnika światła)

* **Przedmiot:** Technika mikroprocesorowa 2
* **Kierunek:** Elektronika i Telekomunikacja, 3 rok
* **Autor:** Wiktor Baran
* **Data:** 26.01.2026

---

## 1. Cel projektu
Projekt realizuje optyczny dekoder alfabetu Morse'a na płytce rozwojowej FRDM-KL05Z. System analizuje impulsy świetlne z latarki przy użyciu czujnika światła, klasyfikując je w czasie rzeczywistym jako elementy kodu Morse'a (kropki i kreski). Zdekodowane sekwencje są tłumaczone na znaki, a następnie gromadzone i łączone w spójny wyraz. Oprogramowanie wyposażono w autokalibrację progu światła oraz obsługę błędów transmisji, co zapewnia stabilną pracę w różnych warunkach oświetleniowych. Zdekodowane wiadomości są na bieżąco prezentowane na wyświetlaczu LCD 2x16.

## 2. Realizacja sprzętowa
Układ został zrealizowany na platformie rozwojowej FRDM-KL05Z z mikrokontrolerem ARM Cortex-M0+. Interakcja z użytkownikiem odbywa się poprzez wyświetlacz LCD 2x16 sterowany układem HD44780U, który został podłączony do płytki poprzez 8-bitowy ekspander I2C, co zredukowało potrzebną liczbę połączeń sygnałowych. Rolę interfejsu wejściowego pełni moduł analogowego czujnika światła ALS-PT19 (oparty na fototranzystorze), podłączony do wejścia przetwornika ADC mikrokontrolera. Element ten umożliwia ciągły pomiar natężenia oświetlenia, co pozwala na detekcję impulsów świetlnych ze źródła światła oraz realizację procedury autokalibracji względem tła.

Poniższy schemat ideowy ilustruje sposób integracji modułów zewnętrznych z płytką rozwojową:
<img width="1259" height="703" alt="image" src="https://github.com/user-attachments/assets/9c4496d3-db70-44c6-b7cd-c31d79d9266e" />

## 3. Omówienie struktury i działania oprogramowania
Architektura aplikacji opiera się na modelu pętli nieskończonej, w której program ciągle analizuje sygnał analogowy pochodzący z czujnika światła oraz interpretuje go jako sekwencje kodową. Działanie systemu można podzielić na trzy główne etapy:

### 3.1 Inicjalizacja i autokalibracja
Po uruchomieniu program konfiguruje niezbędne moduły sprzętowe, a następnie przechodzi w tryb adaptacji do otoczenia. Program dokonuje pomiaru natężenia światła tła w pomieszczeniu, na podstawie którego wyznaczany jest dynamiczny próg detekcji (`prog_tla`). Dzięki temu urządzenie potrafi odróżnić celowy błysk latarki od zwykłego oświetlenia w pokoju, niezależnie od tego, czy jest dzień, czy noc.

### 3.2 Analiza Czasowa Sygnału
W fazie głównej program nieustannie próbkuje napięcie na czujniku. Gdy wykryty zostanie wzrost jasności powyżej ustalonego progu, system przechodzi do pomiaru czasu trwania impulsu. Pomiar realizowany jest poprzez licznik PIT pracujący jako stopwatch, zliczający impulsy zegara co określony czas.
* **Krótki impuls** jest interpretowany jako **kropka**.
* **Długi impuls** jest interpretowany jako **kreska**.

Zmierzone sygnały są od razu wyświetlane na ekranie oraz gromadzone w buforze pamięci jako ciąg znaków (np. `...---...`). Równocześnie mierzony jest czas przerw między błyskami - jeśli przerwa jest długa, system uznaje, że nadawanie bieżącego znaku dobiegło końca. Na zakończenie cyklu przetwarzania znaku, system wykonuje procedurę rekalibracji, co pozwala na zaadaptowanie urządzenia do ewentualnych zmian oświetlenia. Dodatkowo, system realizuje cykliczną rekalibrację w stanie spoczynku.

### 3.3 Dekodowanie i Prezentacja
Program po skompletowaniu sekwencji dekoduje znak, dopisuje go na wyświetlaczu oraz wpisuje go do tablicy kompletującej wyraz. Dzięki temu na ekranie LCD sukcesywnie budowany jest ciąg znaków. Jeżeli po wypisaniu znaku nastąpi dłuższa przerwa, program zakończy odbieranie wyrazu i umożliwi odbieranie kolejnego.

## 4. Instrukcja obsługi

### 4.1 Uruchomienie
Przed podłączeniem zasilania należy ustawić czujnik światła w stabilnej pozycji, kierując go bezpośrednio w stronę planowanego źródła sygnału (latarki). Po uruchomieniu system zweryfikuje warunki oświetleniowe. W przypadku wykrycia zbyt silnego światła tła, wyświetlony zostanie komunikat **"Za jasno"** - należy wówczas zmienić ustawienie czujnika lub zaciemnić stanowisko. Jeżeli warunki są odpowiednie, na wyświetlaczu pojawi się komunikat **"Gotowy..."**, sygnalizujący pełną gotowość do odbierania sygnałów.

### 4.2 Nadawanie
System pozwala na dostosowanie szybkości dekodowania do preferencji użytkownika (zmiany w kodzie programu). Kluczowe parametry globalne to:
* `jednostkaczasu_ms` (domyślnie: 1000 ms) - określa bazowy czas trwania kropki.
* `tolerancja_ms` (domyślnie: 500 ms) - definiuje dopuszczalny margines błędu.

Czas trwania kreski obliczany jest jako trzykrotność czasu kropki (`3 * jednostkaczasu_ms`). Rozpoznane elementy są wyświetlane w górnym wierszu ekranu. Separacja znaków i wyrazów opiera się na czasie trwania braku oświetlenia:
1. **Koniec znaku:** Przerwa dłuższa niż `3 * jednostkaczasu_ms` zamyka sekwencję i dekoduje znak.
2. **Koniec wyrazu:** Kolejna przerwa o długości `3 * jednostkaczasu_ms` po wyrazie interpretowana jest jako koniec słowa, finalizując je i przygotowując bufor na nową wiadomość.

### 4.3 Obsługa błędów
System został wyposażony w mechanizmy ciągłego monitorowania poprawności transmisji. W przypadku wykrycia anomalii, dekodowanie jest przerywane i wyświetlany jest błąd. Użytkownik może nadawać po pojawieniu się błędu.
* **Błędy czasowe (Err1 / Err2):** Czas trwania impulsu świetlnego nie mieści się w zdefiniowanych oknach tolerancji. Użytkownik ma możliwość powtórzenia źle nadanej kreski lub kropki.
* **Błędy dekodowania (Brak znaku):** Odebrana sekwencja jest poprawna czasowo, ale nie odpowiada żadnemu symbolowi w zaimplementowanym słowniku.
* **Błąd środowiskowy (Za jasno):** Napięcie na czujniku w stanie spoczynku przekracza próg nasycenia (2V). Blokuje program do ustabilizowania oświetlenia.
* **Błędy przepełnienia (Za dlugi...):** Próba nadania więcej niż 5 elementów na znak (anulowanie znaku) lub przekroczenie 16 znaków dla słowa (natychmiastowe usunięcie słowa).
* **Błąd krytyczny ADC (Blad ADC):** Inicjalizacja przetwornika zakończona niepowodzeniem. Błąd sprzętowy wymagający resetu zasilania.

### 4.4 Dodatkowe informacje
* **Autokalibracja w stanie spoczynku:** Wykonywana cyklicznie (co `3 * jednostkaczasu_ms`) w celu bieżącej adaptacji do zmieniających się warunków oświetleniowych.
* **Zabezpieczenie Timeout:** Jeśli impuls świetlny trwa nieprzerwanie powyżej `5 * jednostkaczasu_ms`, system uznaje to za anomalię (np. nasłonecznienie pomieszczenia) i wymusza ponowną kalibrację.

## 5. Opis głównych funkcji

### 5.1 Inicjalizacja licznika (`PIT_Init`)
Funkcja konfiguruje licznik okresowy PIT do pełnienia roli systemowej podstawy czasu, generującej przerwania co 1 ms.
Proces inicjalizacji rozpoczyna się od włączenia sygnału taktującego dla modułu w rejestrze SIM oraz wybudzenia go ze
stanu uśpienia. Następnie do rejestru LDVAL wpisywana jest wartość obliczona na podstawie częstotliwości zegara
magistrali (SystemCoreClock / 2) podzielonej przez 1000, co definiuje pożądany interwał czasowy. W końcowym etapie
funkcja fizycznie uruchamia licznik, zezwala na zgłaszanie żądań przerwań oraz odblokowuje ich obsługę w kontrolerze
NVIC.
    
    void PIT_Init(void) {
        SIM->SCGC6 |= SIM_SCGC6_PIT_MASK;
        PIT->MCR &= ~PIT_MCR_MDIS_MASK;
        PIT->CHANNEL[0].LDVAL = ((SystemCoreClock / 2) / 1000) - 1;
        PIT->CHANNEL[0].TCTRL |= PIT_TCTRL_TIE_MASK | PIT_TCTRL_TEN_MASK;
        NVIC_ClearPendingIRQ(PIT_IRQn);
        NVIC_EnableIRQ(PIT_IRQn);
    }

### 5.2 Przerwanie licznika (`PIT_IRQHandler`)
Procedura obsługi przerwania licznika PIT, wywoływana cyklicznie co 1 ms. Funkcja zatwierdza wystąpienie przerwania
poprzez wyzerowanie odpowiedniej flagi sprzętowej oraz inkrementuje globalną zmienną timer_ms, która służy do
bieżącego odmierzania czasu w systemie.
    
    void PIT_IRQHandler(void) {
        if (PIT->CHANNEL[0].TFLG & PIT_TFLG_TIF_MASK) {
            PIT->CHANNEL[0].TFLG = PIT_TFLG_TIF_MASK;
            timer_ms++;
        }
    }

### 5.3 Przerwanie ADC (`ADC0_IRQHandler`)
Procedura obsługi przerwania przetwornika analogowo cyfrowego, uruchamiana automatycznie po zakończeniu  każdego pomiaru. Funkcja pobiera surowy wynik konwersji z rejestru sprzętowego i przekazuje go do zmiennych programu, sygnalizując pętli głównej gotowość nowych danych o aktualnym natężeniu oświetlenia.

    
    void ADC0_IRQHandler() {
        temp = ADC0->R[0];
        if (!wynik_ok) {
            wynik = temp;
            wynik_ok = 1;
        }
    }

### 5.4 Dekodowanie znaku (`dekoduj_znak`)
Funkcja realizuje translację odebranej sekwencji kropek i kresek na znaki alfanumeryczne. Algorytm porównuje zawartość bufora wejściowego ze  zdefiniowanym słownikiem kodu Morse'a i zwraca odpowiadający mu symbol ASCII lub znak zapytania, jeśli sekwencja nie została rozpoznana.
    
    char dekoduj_znak(char* kod) {
        if (strcmp(kod, ".-") == 0) return 'A';
        if (strcmp(kod, "-...") == 0) return 'B';
        if (strcmp(kod, "-.-.") == 0) return 'C';
        if (strcmp(kod, "-..") == 0) return 'D';
        if (strcmp(kod, ".") == 0) return 'E';
        // ... (reszta alfabetu) ...
        return '?'; // Nieznany znak
    }

### 5.5 Kalibracja tła (`kalibracja`)
Funkcja adaptuje układ do oświetlenia otoczenia. W pierwszej kolejności weryfikuje stan nasycenia czujnika (napięcie > 2V). W przypadku przekroczenia tego progu blokuje dalsze działanie programu i wyświetla komunikat błędu, który nie znika, dopóki warunki nie ulegną poprawie. Jeżeli natomiast pomiar mieści się w normie, funkcja rejestruje aktualnie zmierzone napięcie jako zmienną prog_tla, ustanawiając tym samym nowy punkt odniesienia dla detekcji sygnałów. 
    
    void kalibracja(void) {
        while (!wynik_ok);
        wynik_ok = 0;
        
        while (wynik * adc_volt_coeff >= 2.0) {
            LCD1602_SetCursor(0, 0);
            LCD1602_Print("                ");
            LCD1602_SetCursor(0, 0);
            sprintf(display, "Za jasno: %.2fV", wynik * adc_volt_coeff);
            LCD1602_Print(display);
            DELAY(2000); // zeby nie migalo za czesto
            
            while (!wynik_ok); // Czekaj az ADC zrobi nowy pomiar
            wynik_ok = 0;
        }
        prog_tla = wynik * adc_volt_coeff;
    }

### 5.6 Pomiar impulsu (`measure_high`)
Funkcja realizuje pomiar czasu trwania aktywnego impulsu świetlnego. Po zarejestrowaniu momentu początkowego pętla monitoruje sygnał wejściowy aż do spadku napięcia poniżej progu detekcji (z uwzględnieniem histerezy). Procedura wyposażona jest w zabezpieczenie typu timeout – jeżeli stan wysoki utrzymuje się zbyt długo (powyżej 5 jednostek czasu), system przerywa pomiar i wymusza ponowną kalibrację tła.
    
    void measure_high(void) {
        int start = timer_ms;
        while (1) {
            if (wynik_ok) {
                wynik_ok = 0;
                if ((wynik * adc_volt_coeff) < prog_tla + 0.25) {
                    break;
                }
            }
            if ((timer_ms - start) > 5 * jednostkaczasu_ms) {
                kalibracja();
                break;
            }
        }
        czas_trwania_h = timer_ms - start;
    }

### 5.7 Pomiar przerwy (`measure_low`)
Funkcja realizuje pomiar czasu trwania przerwy między impulsami (stanu niskiego). Po zarejestrowaniu momentu początkowego pętla monitoruje sygnał wejściowy aż do wzrostu napięcia powyżej progu detekcji z uwzględnieniem histerezy (sygnalizującego nadejście kolejnego błysku). Jeżeli stan niski utrzymuje się dłużej niż założony limit (ok. 3 jednostki czasu), system przerywa oczekiwanie i wywołuje kalibrację tła, adaptując się do warunków spoczynkowych.
    
    void measure_low(void) {
        int start = timer_ms;
        while (1) {
            if (wynik_ok) {
                wynik_ok = 0;
                if ((wynik * adc_volt_coeff) >= prog_tla + 0.5) break;
            }
            if ((timer_ms - start) > (3.0 * jednostkaczasu_ms)) {
                kalibracja();
                czas_trwania_l = timer_ms - start;
                break;
            }
        }
        czas_trwania_l = timer_ms - start;
    }
