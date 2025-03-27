#include <HX711_ADC.h>
#include <U8g2lib.h>

// Definice pinů
const int HX711_dout = 6;  // DT pin na HX711
const int HX711_sck = 7;   // SCK pin na HX711
const int buttonCategory = 10;
const int buttonFood = 8;
const int buttonWakeTare = 9;  // Tlačítko pro probuzení a tare

// Inicializace HX711
HX711_ADC LoadCell(HX711_dout, HX711_sck);

// Inicializace OLED displeje SH1107 128x128
U8G2_SH1107_128X128_1_HW_I2C u8g2(U8G2_R0);

// Proměnné pro kategorie a jídla
String categories[] = {"Pecivo", "Ovoce", "Prilohy"};
String foods[3][4] = {
  {"Rohlik", "Chleb", "Bageta", "Makovka"},
  {"Jablko", "Banan", "Hrozny", "Pomeranc"},
  {"Brambory", "Ryze", "Testoviny", "Knedlik"}
};
float carbsPer100g[3][4] = {
  {50, 45, 55, 60},
  {14, 23, 16, 12},
  {17, 28, 25, 42}
};

int currentCategory = 0;
int currentFood = 0;

// Proměnné pro detekci stisku tlačítek
bool lastButtonStateCategory = HIGH;
bool lastButtonStateFood = HIGH;
bool lastButtonStateWakeTare = HIGH;
unsigned long lastDebounceTimeCategory = 0;
unsigned long lastDebounceTimeFood = 0;
unsigned long lastDebounceTimeWakeTare = 0;
const unsigned long debounceDelay = 50;  // Debounce čas v milisekundách

bool isSleeping = true;  // Stav spánku
unsigned long lastActivityTime = 0;  // Čas poslední aktivity
const unsigned long sleepTimeout = 30000;  // 30 sekund pro automatický spánek

// Proměnné pro zrychlení zobrazování
float lastDisplayedWeight = 0;  // Poslední zobrazená hmotnost
const float weightChangeThreshold = 5.0;  // Zobrazit pouze při změně o více než 5 gramů

void setup() {
  Serial.begin(9600);
  LoadCell.begin();
  LoadCell.start(2000);  // Kalibrace váhy
  LoadCell.setCalFactor(-2018.86);  // Nastavení kalibračního faktoru -2027.31

  // Nastavení vyšší vzorkovací frekvence (např. 80 Hz)
  LoadCell.setSamplesInUse(1);  // Použijeme pouze jeden vzorek pro rychlejší odezvu

  u8g2.begin();  // Inicializace displeje
  u8g2.setFont(u8g2_font_ncenB08_tr);  // Nastavení výchozího písma

  pinMode(buttonCategory, INPUT_PULLUP);
  pinMode(buttonFood, INPUT_PULLUP);
  pinMode(buttonWakeTare, INPUT_PULLUP);

  enterSleepMode();  // Uspání zařízení na začátku
}

void loop() {
  // Detekce stisku tlačítka pro probuzení a tare
  bool currentButtonStateWakeTare = digitalRead(buttonWakeTare);
  if (currentButtonStateWakeTare != lastButtonStateWakeTare) {
    if (currentButtonStateWakeTare == LOW && millis() - lastDebounceTimeWakeTare > debounceDelay) {
      if (isSleeping) {
        exitSleepMode();  // Probuzení zařízení
        displayWeightAndCarbs(lastDisplayedWeight);  // Aktualizace displeje po probuzení
      } else {
        LoadCell.tareNoDelay();  // Vynulování váhy (tare)
        Serial.println("Tare provedeno");
      }
      lastActivityTime = millis();  // Reset času poslední aktivity
      lastDebounceTimeWakeTare = millis();  // Uložení času posledního stisku
    }
    lastButtonStateWakeTare = currentButtonStateWakeTare;  // Uložení aktuálního stavu tlačítka
  }

  // Pokud je zařízení v režimu spánku, nic nedělej (kromě detekce tlačítka)
  if (isSleeping) {
    return;
  }

  // Načtení hmotnosti
  if (LoadCell.update()) {
    float weight = LoadCell.getData();

    // Zabránění záporným hodnotám
    if (weight < 0) {
      weight = 0;
    }

    // Detekce aktivity (vážení)
    if (weight > 1.0) {  // Pokud je hmotnost větší než 1 gram
      lastActivityTime = millis();  // Aktualizace času poslední aktivity
    }

    // Zobrazit pouze při významné změně hmotnosti
    if (abs(weight - lastDisplayedWeight) >= weightChangeThreshold) {
      lastDisplayedWeight = weight;  // Uložit novou hmotnost
      displayWeightAndCarbs(weight);  // Zobrazit hmotnost a sacharidy
    }
  }

  // Detekce stisku tlačítka pro kategorii (nezávisle na váze)
  bool currentButtonStateCategory = digitalRead(buttonCategory);
  if (currentButtonStateCategory != lastButtonStateCategory) {
    if (currentButtonStateCategory == LOW && millis() - lastDebounceTimeCategory > debounceDelay) {
      currentCategory = (currentCategory + 1) % 3;
      currentFood = 0;  // Reset výběru jídla při změně kategorie
      lastActivityTime = millis();  // Reset času poslední aktivity
      lastDebounceTimeCategory = millis();  // Uložení času posledního stisku
      displayWeightAndCarbs(lastDisplayedWeight);  // Aktualizace displeje
    }
    lastButtonStateCategory = currentButtonStateCategory;  // Uložení aktuálního stavu tlačítka
  }

  // Detekce stisku tlačítka pro jídlo (nezávisle na váze)
  bool currentButtonStateFood = digitalRead(buttonFood);
  if (currentButtonStateFood != lastButtonStateFood) {
    if (currentButtonStateFood == LOW && millis() - lastDebounceTimeFood > debounceDelay) {
      currentFood = (currentFood + 1) % 4;
      lastActivityTime = millis();  // Reset času poslední aktivity
      lastDebounceTimeFood = millis();  // Uložení času posledního stisku
      displayWeightAndCarbs(lastDisplayedWeight);  // Aktualizace displeje
    }
    lastButtonStateFood = currentButtonStateFood;  // Uložení aktuálního stavu tlačítka
  }

  // Automatické uspání po 30 sekundách nečinnosti
  if (millis() - lastActivityTime > sleepTimeout) {
    enterSleepMode();
    return;  // Uspání zařízení a ukončení dalšího zpracování
  }

  delay(10);  // Krátká pauza pro stabilitu
}

void displayWeightAndCarbs(float weight) {
  // Výpočet sacharidů
  int carbs = (int)((weight * carbsPer100g[currentCategory][currentFood]) / 100.0);

  // Zobrazení na displeji
  u8g2.firstPage();
  do {
    // Horní část: Hmotnost a sacharidy
    u8g2.setFont(u8g2_font_ncenB24_tr);  // Velké písmo pro horní část
    u8g2.setCursor(0, 30);  // Hmotnost vlevo
    u8g2.print((int)weight);  // Zobrazení hmotnosti jako celé číslo
    u8g2.setFont(u8g2_font_ncenB12_tr);  // Menší písmo pro "g"
    u8g2.print("g");  // Přidání jednotky "g" za hmotnost

    u8g2.setFont(u8g2_font_ncenB24_tr);  // Velké písmo pro sacharidy
    u8g2.setCursor(64, 30);  // Sacharidy vpravo
    u8g2.print(carbs);  // Zobrazení sacharidů jako celé číslo
    u8g2.setFont(u8g2_font_ncenB12_tr);  // Menší písmo pro "g"
    u8g2.print("g");  // Přidání jednotky "g" za sacharidy

    // Dolní část: Kategorie a jídlo
    u8g2.setFont(u8g2_font_ncenB14_tr);  // Větší písmo pro dolní část
    u8g2.setCursor(10, 80);  // Posunuto doleva pro kategorii
    u8g2.print(categories[currentCategory]);  // Zobrazení kategorie
    u8g2.setCursor(10, 110);  // Posunuto doleva pro jídlo
    u8g2.print(foods[currentCategory][currentFood]);  // Zobrazení jídla
  } while (u8g2.nextPage());
}

void enterSleepMode() {
  isSleeping = true;
  u8g2.clearDisplay();  // Vymazání displeje
  u8g2.setPowerSave(true);  // Vypnutí displeje
  LoadCell.powerDown();  // Vypnutí váhy
  Serial.println("Rezim spanku aktivovan");
}

void exitSleepMode() {
  isSleeping = false;
  LoadCell.powerUp();  // Zapnutí váhy
  u8g2.setPowerSave(false);  // Zapnutí displeje
  lastActivityTime = millis();  // Reset času poslední aktivity
  Serial.println("Rezim spanku deaktivovan");
}
