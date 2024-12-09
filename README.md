#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <LedControl.h>
#include <RTClib.h>

// Configuration de l'écran OLED
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Configuration des broches pour les matrices LED
#define DIN_PIN 11
#define CLK_PIN 8
#define CS_PIN 10
LedControl lc = LedControl(DIN_PIN, CLK_PIN, CS_PIN, 4);

// Initialisation du RTC
RTC_DS1307 rtc;

// Configuration des boutons
#define BUTTON_NEXT 2       // Bouton pour passer au menu suivant
#define BUTTON_PREV 3       // Bouton pour revenir au menu précédent
#define BUTTON_BOTH 4       // Bouton pour valider ou sélectionner une option

// Variables de gestion
int menuSelection = 0;          // Suivi du menu actuellement sélectionné
int melodySelection = 0;        // Suivi du choix de mélodie
int currentHour;                // Heure actuelle
int currentMinute;              // Minute actuelle
int alarmHour = 0;              // Heure de l'alarme
int alarmMinute = 0;            // Minute de l'alarme
int timeFormat = 24;            // Format de l'heure (12 ou 24 heures)
bool colonVisible = true;       // État de clignotement des deux points sur les matrices LED
bool editingTime = false;       // Mode modification de l'heure
bool editingAlarm = false;      // Mode modification de l'alarme
bool choosingMelody = false;    // Mode choix de la mélodie

// Fonction pour afficher un chiffre sur les matrices LED
void displayDigit(int matrixIndex, int digit) {
    static const byte digits[10][8] = {
        {0b00111100, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b00111100}, // 0
        {0b00001000, 0b00011000, 0b00101000, 0b00001000, 0b00001000, 0b00001000, 0b00001000, 0b00011100}, // 1
        {0b00111100, 0b01000010, 0b00000010, 0b00000100, 0b00001000, 0b00010000, 0b00100000, 0b01111110}, // 2
        {0b00111100, 0b01000010, 0b00000010, 0b00011100, 0b00000010, 0b00000010, 0b01000010, 0b00111100}, // 3
        {0b00000100, 0b00001100, 0b00010100, 0b00100100, 0b01111111, 0b00000100, 0b00000100, 0b00000100}, // 4
        {0b01111110, 0b01000000, 0b01000000, 0b01111100, 0b00000010, 0b00000010, 0b01000010, 0b00111100}, // 5
        {0b00111100, 0b01000010, 0b01000000, 0b01111100, 0b01000010, 0b01000010, 0b01000010, 0b00111100}, // 6
        {0b01111110, 0b00000010, 0b00000100, 0b00001000, 0b00010000, 0b00100000, 0b00100000, 0b00100000}, // 7
        {0b00111100, 0b01000010, 0b01000010, 0b00111100, 0b01000010, 0b01000010, 0b01000010, 0b00111100}, // 8
        {0b00111100, 0b01000010, 0b01000010, 0b00111110, 0b00000010, 0b00000010, 0b01000010, 0b00111100}  // 9
    };

    for (int row = 0; row < 8; row++) {
        lc.setRow(matrixIndex, row, digits[digit][row]);
    }
}

// Fonction pour gérer le clignotement des deux points
void toggleColon(bool visible) {
    lc.setLed(1, 3, 0, visible); // Contrôle du point supérieur
    lc.setLed(1, 4, 0, visible); // Contrôle du point inférieur
}

// Fonction pour afficher le menu principal
void displayMenu() {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);

    display.println("Menu:");
    display.println(menuSelection == 0 ? "> Modifier l'heure" : "  Modifier l'heure");
    display.println(menuSelection == 1 ? "> Regler l'alarme" : "  Regler l'alarme");
    display.println(menuSelection == 2 ? "> Changer format" : "  Changer format");
    display.println(menuSelection == 3 ? "> Choix melodie" : "  Choix melodie");

    display.display();
}

// Fonction pour afficher le menu des mélodies
void displayMelodyMenu() {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);

    display.println("Choix melodie:");
    display.println(melodySelection == 0 ? "> Melodie 1" : "  Melodie 1");
    display.println(melodySelection == 1 ? "> Melodie 2" : "  Melodie 2");

    display.display();
}

// Fonction pour gérer le choix de la mélodie
void chooseMelody() {
    if (choosingMelody) {
        displayMelodyMenu();

        if (digitalRead(BUTTON_NEXT) == LOW) {
            melodySelection = (melodySelection + 1) % 2;
            delay(300);
        }

        if (digitalRead(BUTTON_PREV) == LOW) {
            melodySelection = (melodySelection - 1 + 2) % 2;
            delay(300);
        }

        if (digitalRead(BUTTON_BOTH) == LOW) {
            choosingMelody = false;
            displayMenu();
            delay(300);
        }
    }
}

void setup() {
    pinMode(BUTTON_NEXT, INPUT_PULLUP);
    pinMode(BUTTON_PREV, INPUT_PULLUP);
    pinMode(BUTTON_BOTH, INPUT_PULLUP);

    for (int i = 0; i < 4; i++) {
        lc.shutdown(i, false);
        lc.setIntensity(i, 5);
        lc.clearDisplay(i);
    }

    if (!rtc.begin()) {
        while (1);
    }

    if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
        while (1);
    }

    // Initialiser l'heure avec le RTC
    DateTime now = rtc.now();
    currentHour = now.hour();
    currentMinute = now.minute();
}

void loop() {
    if (choosingMelody) {
        chooseMelody();
    } else if (editingTime) {
        modifyTime();
    } else if (editingAlarm) {
        modifyAlarm();
    } else {
        if (digitalRead(BUTTON_NEXT) == LOW) {
            menuSelection = (menuSelection + 1) % 4;
            displayMenu();
            delay(300);
        }

        if (digitalRead(BUTTON_PREV) == LOW) {
            menuSelection = (menuSelection - 1 + 4) % 4;
            displayMenu();
            delay(300);
        }

        if (digitalRead(BUTTON_BOTH) == LOW) {
            switch (menuSelection) {
                case 0:
                    editingTime = true;
                    break;
                case 1:
                    editingAlarm = true;
                    break;
                case 2:
                    toggleTimeFormat();
                    break;
                case 3:
                    choosingMelody = true;
                    break;
            }
            delay(300);
        }

        // Affichage de l'heure actuelle
        DateTime now = rtc.now();
        int hour = now.hour();
        int minute = now.minute();

        if (timeFormat == 12 && hour >= 12) {
            hour -= 12;
        }

        displayDigit(3, hour / 10);
        displayDigit(2, hour % 10);
        displayDigit(1, minute / 10);
        displayDigit(0, minute % 10);

        toggleColon(colonVisible);
        colonVisible = !colonVisible;
        delay(1000);
    }
}

