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
#define BUTTON_NEXT 2       
#define BUTTON_PREV 3       
#define BUTTON_BOTH 4       


// Variables de gestion
int menuSelection = 0;  
int currentHour = 0; 
int currentMinute = 0;  
int alarmHour = 0;
int alarmMinute = 0;
int timeFormat = 24;
bool colonVisible = true;
bool editingTime = false;
bool editingAlarm = false;


// Fonction pour afficher un chiffre sur les matrices LED
void displayDigit(int matrixIndex, int digit) {
    static const byte digits[10][8] = {
        {0b00111100, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b01000010, 0b00111100},
        {0b00001000, 0b00011000, 0b00101000, 0b00001000, 0b00001000, 0b00001000, 0b00001000, 0b00011100}, 
        {0b00111100, 0b01000010, 0b00000010, 0b00000100, 0b00001000, 0b00010000, 0b00100000, 0b01111110}, 
        {0b00111100, 0b01000010, 0b00000010, 0b00011100, 0b00000010, 0b00000010, 0b01000010, 0b00111100}, 
        {0b00000100, 0b00001100, 0b00010100, 0b00100100, 0b01111111, 0b00000100, 0b00000100, 0b00000100},
        {0b01111110, 0b01000000, 0b01000000, 0b01111100, 0b00000010, 0b00000010, 0b01000010, 0b00111100}, 
        {0b00111100, 0b01000010, 0b01000000, 0b01111100, 0b01000010, 0b01000010, 0b01000010, 0b00111100}, 
        {0b01111110, 0b00000010, 0b00000100, 0b00001000, 0b00010000, 0b00100000, 0b00100000, 0b00100000}, 
        {0b00111100, 0b01000010, 0b01000010, 0b00111100, 0b01000010, 0b01000010, 0b01000010, 0b00111100}, 
        {0b00111100, 0b01000010, 0b01000010, 0b00111110, 0b00000010, 0b00000010, 0b01000010, 0b00111100} 
    };

    for (int row = 0; row < 8; row++) {
        lc.setRow(matrixIndex, row, digits[digit][row]);
    }
}


// Fonction pour gérer le clignotement des deux points
void toggleColon(bool visible) {
    lc.setLed(1, 3, 0, visible);
    lc.setLed(1, 4, 0, visible);
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


// Fonction pour modifier l'heure
void modifyTime() {
    if (editingTime) {
        display.clearDisplay();
        display.setCursor(0, 0);
        display.println("Modif Heure:");
        display.print(currentHour);
        display.print(":");
        display.println(currentMinute);
        display.display();


        if (digitalRead(BUTTON_NEXT) == LOW) {
            currentHour = (currentHour + 1) % 24;
            delay(10);
        }


        if (digitalRead(BUTTON_PREV) == LOW) {
            currentMinute = (currentMinute + 1) % 60;
            delay(10);
        }


        if (digitalRead(BUTTON_BOTH) == LOW) {
            rtc.adjust(DateTime(2023, 1, 1, currentHour, currentMinute, 0));
            editingTime = false;
            displayMenu();
            delay(100);

        }


        displayDigit(3, currentHour / 10);
        displayDigit(2, currentHour % 10);
        displayDigit(1, currentMinute / 10);
        displayDigit(0, currentMinute % 10);

    }

}


// Fonction pour modifier l'alarme
void modifyAlarm() {
    if (editingAlarm) {
        display.clearDisplay();
        display.setCursor(0, 0);
        display.println("Reglage Alarme:");
        display.print(alarmHour);
        display.print(":");
        display.println(alarmMinute);
        display.display();


        if (digitalRead(BUTTON_NEXT) == LOW) {
            alarmHour = (alarmHour + 1) % 24;
            delay(10);
        }


        if (digitalRead(BUTTON_PREV) == LOW) {
            alarmMinute = (alarmMinute + 1) % 60;
            delay(10);
        }


        if (digitalRead(BUTTON_BOTH) == LOW) {
            editingAlarm = false;
            displayMenu();
            delay(10);
        }


        displayDigit(3, alarmHour / 10);
        displayDigit(2, alarmHour % 10);
        displayDigit(1, alarmMinute / 10);
        displayDigit(0, alarmMinute % 10);
    }

}


// Fonction pour changer le format d'heure
void toggleTimeFormat() {
    timeFormat = (timeFormat == 24);
    displayMenu();
}


// Fonction pour gérer le choix de la mélodie
void chooseMelody() {
    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("Choix melodie:");
    display.println("1. Melodie 1");
    display.println("2. Melodie 2");
    display.display();
    delay(3000);
    displayMenu();
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


    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
    displayMenu();
}


void loop() {
    if (digitalRead(BUTTON_NEXT) == LOW) {
        menuSelection = (menuSelection + 1) % 4; 
        displayMenu();
        delay(500);
    }


    if (digitalRead(BUTTON_PREV) == LOW) {
        menuSelection = (menuSelection - 1 + 4) % 4;
        displayMenu();
        delay(500);
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
                chooseMelody();
                break;
        }
        delay(100);
    }


    if (editingTime) modifyTime();

    if (editingAlarm) modifyAlarm();


    toggleColon(colonVisible);
    colonVisible = !colonVisible;
    delay(1000);

} 
