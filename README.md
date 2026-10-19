# Inteligentne Skrzyżowanie Wahadłowe (Arduino + PIR)

Projekt inteligentnej sygnalizacji świetlnej dla skrzyżowania czterostronnego, zrealizowany w środowisku **Tinkercad** i oparty na mikrokontrolerze **Arduino Uno**. 

System steruje ruchem w sposób asynchroniczny – zielone światło otwiera się **wyłącznie na wniosek czujników ruchu PIR**, a nie w sztywnej pętli czasowej. Układ dba o to, aby w danym momencie zielone światło paliło się maksymalnie dla jednego kierunku.

## 🚀 Funkcje projektu
* **Dynamiczne kolejkowanie za pomocą `millis()`**: Zgłoszenia z czterech stron skrzyżowania są zapisywane w kolejce (tablicy) w takiej kolejności, w jakiej podjeżdżały pojazdy. Brak blokującego kodu `delay()`.
* **Pełna niezależność kierunków**: Zielone światło świeci się na danej nitce przez 5 sekund, pod warunkiem, że system wykryje tam ruch.
* **Bufor bezpieczeństwa (OFF)**: Po zakończeniu cyklu zielonego, skrzyżowanie przechodzi w 2-sekundowy stan buforowy, w którym wszystkie kierunki mają światło czerwone (czyszczenie skrzyżowania).
* **Maksymalnie 1 zielone światło**: Architektura kodu fizycznie uniemożliwia jednoczesne włączenie zielonego światła na dwóch różnych wlotach.

## 🛠️ Schemat podłączenia pinów (Arduino Uno)

| Element | Kierunek | Pin Arduino |
| :--- | :--- | :--- |
| **Czujnik PIR (Sygnał)** | Top (Góra) | `13` |
| **Czujnik PIR (Sygnał)** | Left (Lewo) | `12` |
| **Czujnik PIR (Sygnał)** | Bottom (Dół) | `11` |
| **Czujnik PIR (Sygnał)** | Right (Prawo) | `10` |
| **Dioda LED Czerwona** | Top (Góra) | `9` |
| **Dioda LED Zielona** | Top (Góra) | `8` |
| **Dioda LED Czerwona** | Left (Lewo) | `7` |
| **Dioda LED Zielona** | Left (Lewo) | `6` |
| **Dioda LED Czerwona** | Bottom (Dół) | `5` |
| **Dioda LED Zielona** | Bottom (Dół) | `4` |
| **Dioda LED Czerwona** | Right (Prawo) | `3` |
| **Dioda LED Zielona** | Right (Prawo) | `2` |

*Wszystkie diody LED muszą być połączone szeregowo z rezystorami (zalecane 220 Ω lub 330 Ω).*

## 💻 Kod źródłowy projektu

```cpp
#define OFF -1
#define T   0
#define L   1
#define B   2
#define R   3

const int PIR_PIN[]  = {13, 12, 11, 10};
const int RED_PIN[]  = {9,  7,  5,  3};
const int GRN_PIN[]  = {8,  6,  4,  2};

unsigned long czasOstatniejZmiany = 0;
const unsigned long czasZielonego = 5000;
const unsigned long czasCzerwonego = 2000;

int stan = T;
int ostatniStan = T;

int kolejka[4] = {-1, -1, -1, -1};
int rozmiarKolejki = 0;

bool poprPIR[4] = {LOW, LOW, LOW, LOW};

void dodajDoKolejki(int kierunek) {
  if (kierunek == stan || (stan == OFF && kierunek == ostatniStan)) return;
  for (int i = 0; i < rozmiarKolejki; i++) {
    if (kolejka[i] == kierunek) return;
  }
  if (rozmiarKolejki < 4) {
    kolejka[rozmiarKolejki] = kierunek;
    rozmiarKolejki++;
  }
}

int pobierzZKolejki() {
  if (rozmiarKolejki == 0) return -1;
  int nastepny = kolejka[0];
  for (int i = 0; i < rozmiarKolejki - 1; i++) {
    kolejka[i] = kolejka[i + 1];
  }
  kolejka[rozmiarKolejki - 1] = -1;
  rozmiarKolejki--;
  return nastepny;
}

void aktualizujSwiatla() {
  for (int i = 0; i < 4; i++) {
    if (stan == OFF || i != stan) {
      digitalWrite(RED_PIN[i], HIGH);
      digitalWrite(GRN_PIN[i], LOW);
    } else {
      digitalWrite(RED_PIN[i], LOW);
      digitalWrite(GRN_PIN[i], HIGH);
    }
  }
}

void setup() {
  for (int i = 0; i < 4; i++) {
    pinMode(PIR_PIN[i], INPUT);
    pinMode(RED_PIN[i], OUTPUT);
    pinMode(GRN_PIN[i], OUTPUT);
  }
  aktualizujSwiatla();
  czasOstatniejZmiany = millis();
}

void loop() {
  unsigned long t = millis();

  for (int i = 0; i < 4; i++) {
    bool aktualnyStanPIR = digitalRead(PIR_PIN[i]);
    if (aktualnyStanPIR == HIGH && poprPIR[i] == LOW) {
      dodajDoKolejki(i);
    }
    poprPIR[i] = aktualnyStanPIR;
  }

  if (stan != OFF) {
    if (rozmiarKolejki > 0 && (t - czasOstatniejZmiany >= czasZielonego)) {
      ostatniStan = stan;
      stan = OFF;
      aktualizujSwiatla();
      czasOstatniejZmiany = t;
    }
  } else {
    if (t - czasOstatniejZmiany >= czasCzerwonego) {
      int nastepny = pobierzZKolejki();
      if (nastepny != -1) {
        stan = nastepny;
        aktualizujSwiatla();
        czasOstatniejZmiany = t;
      }
    }
  }
}
```

## 🛠️ Jak uruchomić projekt?
1. Skopiuj powyższy kod źródłowy.
2. Wejdź do swojego projektu w **Tinkercad Circuits**.
3. Otwórz panel **Kod (Code)**, przełącz widok na **Tekst (Text)** i wklej kod.
4. Uruchom symulację i testuj działanie poprzez klikanie na poszczególne czujniki PIR w dowolnej, losowej kolejności.
