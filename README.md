# Smart-programmer-
Eeprom Chip programmer 24xx aur 93xx eeprom Read write karta hai 
#include <ESP8266WiFi.h>
#include <ESP8266WebServer.h>
#include <Wire.h>
#include <ArduinoJson.h> 
#include <EEPROM.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Updater.h> 

// ==========================================
// FIRMWARE VERSION
// ==========================================
#define FIRMWARE_VERSION "V4.7 SMART"

// ==========================================
// WIFI & SERVER CONFIG
// ==========================================
const char *ssid = "SmartProgrammer";
const char *password = "12345678";
ESP8266WebServer server(80);

// ==========================================
// HARDWARE PINOUT (SMART PROGRAMMER)
// ==========================================
#define PIN_D1 5   // SDA
#define PIN_D2 4   // SCL
#define PIN_D3 0   // ORG
#define PIN_D5 14  // SK  
#define PIN_D6 12  // CS  
#define PIN_D7 13  // DI  
#define PIN_D8 15  // DO  
#define STATUS_LED 16 
#define BATTERY_PIN A0

// ==========================================
// OLED DISPLAY SETTINGS & BUFFER LOGS
// ==========================================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

char oledLogs[4][22] = {"", "", "", ""}; 
float battEMA = 100.0;                   

// ==========================================
// GLOBAL STATE VARIABLES
// ==========================================
const uint8_t EEPROM_I2C_ADDRESS = 0x50;
uint16_t currentAddress = 0x0000;
bool isActivated = false;

String mainMode = "EEPROM Programmer";
String eepromFamily = "24xx"; 
String eepromSize = "24C04";
String eepromOrg = "x16";     

bool MODE_16BIT = false;
bool isOperating = false;             

String currentAction = "READY";
String errorReason = ""; 
int progressPct = 0;
uint32_t totalBytesActive = 2048;
unsigned long successTimer = 0;
unsigned long lastDisplayUpdate = 0;
bool webLedState = false; 

// ==========================================
// FORWARD DECLARATIONS
// ==========================================
void updateOLED(bool forceUpdate = false);

// ==========================================
// STATUS LED & LOGGING LOGIC
// ==========================================
void pushLog(String msg) {
    strncpy(oledLogs[0], oledLogs[1], 22);
    strncpy(oledLogs[1], oledLogs[2], 22);
    strncpy(oledLogs[2], oledLogs[3], 22);
    snprintf(oledLogs[3], 22, "> %s", msg.c_str());
    updateOLED(true);
}

void clearLogs() {
    for(int i=0; i<4; i++) oledLogs[i][0] = '\0';
}

void triggerError(String reason) {
    currentAction = "ERROR";
    errorReason = reason;
    successTimer = millis(); 
    updateOLED(true);
}

void initStatusLED() { 
    pinMode(STATUS_LED, OUTPUT); 
    digitalWrite(STATUS_LED, HIGH); 
}

void ledFlicker() { 
    static unsigned long lastFlicker = 0;
    if(millis() - lastFlicker > 30) {
        digitalWrite(STATUS_LED, !digitalRead(STATUS_LED)); 
        webLedState = !webLedState; 
        lastFlicker = millis();
    }
}

void updateStatusLED() {
    if (isOperating) return; 
    unsigned long m = millis();
    
    if (currentAction == "ERROR") {
        bool ledOn = (m % 150 < 75);
        digitalWrite(STATUS_LED, ledOn ? LOW : HIGH);
        webLedState = ledOn;
    } else if (currentAction == "SUCCESS") {
        digitalWrite(STATUS_LED, LOW); 
        webLedState = true;
    } else {
        int cycle = m % 1500;
        bool ledOn = (cycle < 80) || (cycle > 180 && cycle < 260);
        digitalWrite(STATUS_LED, ledOn ? LOW : HIGH); 
        webLedState = ledOn; 
    }
}

int getBatteryPercentage() {
    int analogValue = analogRead(BATTERY_PIN);
    battEMA = (battEMA * 0.8) + (analogValue * 0.2); 
    int percentage = map((int)battEMA, 775, 992, 0, 100);
    return constrain(percentage, 0, 100);
}

void safeIdlePins() {
    pinMode(PIN_D3, INPUT); pinMode(PIN_D8, INPUT);
    pinMode(PIN_D5, INPUT); pinMode(PIN_D6, INPUT); pinMode(PIN_D7, INPUT);
}

void setupHardwarePins() {
    if (eepromFamily == "93xx") {
        pinMode(PIN_D6, OUTPUT); pinMode(PIN_D5, OUTPUT); pinMode(PIN_D7, OUTPUT); pinMode(PIN_D8, INPUT);
        digitalWrite(PIN_D6, LOW); digitalWrite(PIN_D5, LOW);
        pinMode(PIN_D3, OUTPUT); 
        if (eepromOrg == "x16" || MODE_16BIT) { MODE_16BIT = true; digitalWrite(PIN_D3, HIGH); } else { MODE_16BIT = false; digitalWrite(PIN_D3, LOW); }
    } else {
        pinMode(PIN_D6, INPUT); pinMode(PIN_D7, INPUT); pinMode(PIN_D5, INPUT); pinMode(PIN_D8, INPUT); 
    }
}

bool is16BitAddress() { return (eepromSize == "24C32" || eepromSize == "24C64" || eepromSize == "24C128" || eepromSize == "24C256" || eepromSize == "24C512"); }

uint8_t getDeviceAddress(uint16_t address) {
    if (eepromSize == "24C16" || eepromSize == "24C08" || eepromSize == "24C04") 
        return EEPROM_I2C_ADDRESS | ((address >> 8) & 0x07);
    return EEPROM_I2C_ADDRESS;
}

bool checkChipConnected() {
    setupHardwarePins();
    delay(30);
    if (eepromFamily == "24xx") {
        Wire.beginTransmission(getDeviceAddress(0));
        if (Wire.endTransmission() == 0) return true;
        errorReason = "CHIP NOT DETECTED";
        return false;
    } else {
        digitalWrite(PIN_D6, HIGH); delayMicroseconds(5); digitalWrite(PIN_D6, LOW); delay(5);
        return true; 
    }
}

bool pollAck(uint8_t deviceAddress) {
    for (int i = 0; i < 100; i++) {
        Wire.beginTransmission(deviceAddress);
        if (Wire.endTransmission() == 0) return true; 
        delayMicroseconds(100); 
    }
    return false;
}

// ==========================================
// 📺 OLED RENDER ENGINE
// ==========================================
void updateOLED(bool forceUpdate) {
    if(!forceUpdate && millis() - lastDisplayUpdate < 200) return; 
    lastDisplayUpdate = millis();

    display.clearDisplay();
    display.fillRect(0, 0, 128, 12, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK); display.setTextSize(1);
    display.setCursor(2, 2); display.print(F("SMART PROGRAMMER"));
    display.setCursor(105, 2); display.printf("%d%%", getBatteryPercentage());
    display.setTextColor(SSD1306_WHITE);

    if(currentAction == "READY") {
        display.setTextSize(1); display.setCursor(0, 18); display.print(F("IC: "));
        display.setTextSize(2); String dispSize = eepromSize; dispSize.replace("_", "C"); display.print(dispSize);
        display.setTextSize(1); display.setCursor(0, 38); display.print(F("BUS: "));
        if(eepromFamily == "93xx") { display.print("93xx ("); display.print(eepromOrg); display.print(")"); } 
        else { display.print("24xx (I2C)"); }
        display.setCursor(0, 52); display.print(F("IP : 192.168.4.1"));
    } 
    else if(currentAction == "READING" || currentAction == "WRITING") {
        display.setTextSize(1); display.setCursor(0, 14);
        display.print(currentAction == "READING" ? F("Reading...") : F("Programming..."));
        display.setCursor(104, 14); display.printf("%d%%", progressPct);
        
        display.drawRect(0, 24, 128, 6, SSD1306_WHITE);
        int fillW = map(progressPct, 0, 100, 0, 124); display.fillRect(2, 26, fillW, 2, SSD1306_WHITE);

        for(int i=0; i<4; i++) { display.setCursor(0, 33 + (i * 8)); display.print(oledLogs[i]); }
    }
    else if(currentAction == "ERROR") {
        display.setTextSize(2); display.setCursor(20, 18); display.print(F("FAILED!"));
        display.setTextSize(1); display.setCursor(0, 42); display.print(errorReason); 
    }
    else if(currentAction == "SUCCESS") {
        display.setTextSize(2); display.setCursor(20, 25); display.print(F("SUCCESS!"));
        display.setTextSize(1); display.setCursor(12, 50); display.print(F("Process Finished."));
    }
    else if(currentAction == "TESTING") {
        display.setTextSize(2); display.setCursor(10, 25); display.print(F("TESTING..."));
    }
    else if(currentAction == "UPDATING") {
        display.setTextSize(2); display.setCursor(0, 25); display.print(F("FLASHING.."));
        display.drawRect(0, 50, 128, 10, SSD1306_WHITE);
        int anim = (millis() / 50) % 124; display.fillRect(2, 52, anim, 6, SSD1306_WHITE);
    }
    display.display();
}

// ==========================================
// HARDWARE DRIVERS FOR EEPROM
// ==========================================
size_t getEEPROMPageSize() {
    if (eepromSize == "24C01" || eepromSize == "24C02") return 8; else if (eepromSize == "24C04" || eepromSize == "24C08" || eepromSize == "24C16") return 16;
    else if (eepromSize == "24C32" || eepromSize == "24C64") return 32; else if (eepromSize == "24C128" || eepromSize == "24C256") return 64; return 16;  
}
size_t getMaxAddress() {
    if (eepromSize == "24C01") return 128; else if (eepromSize == "24C02") return 256; else if (eepromSize == "24C04") return 512;   
    else if (eepromSize == "24C08") return 1024; else if (eepromSize == "24C16") return 2048; else if (eepromSize == "24C32") return 4096;  
    else if (eepromSize == "24C64") return 8192; else if (eepromSize == "24C128") return 16384; else if (eepromSize == "24C256") return 32768; else if (eepromSize == "24C512") return 65536; return 512;  
}
int getTotalSizeGlobal() {
    if (eepromFamily == "93xx") {
        if (eepromSize == "93_46") return 128; if (eepromSize == "93_56") return 256;
        if (eepromSize == "93_66") return 512; if (eepromSize == "93_76") return 1024; return 2048;
    } return getMaxAddress();
}

void MW_Delay() { delayMicroseconds(5); }
void MW_Start() { digitalWrite(PIN_D6, HIGH); MW_Delay(); }
void MW_Stop() { digitalWrite(PIN_D6, LOW); MW_Delay(); }
void MW_SendBit(bool bitData) { digitalWrite(PIN_D7, bitData); digitalWrite(PIN_D5, HIGH); MW_Delay(); digitalWrite(PIN_D5, LOW); MW_Delay(); }
bool MW_ReadBit() { digitalWrite(PIN_D5, HIGH); MW_Delay(); bool val = digitalRead(PIN_D8); digitalWrite(PIN_D5, LOW); MW_Delay(); return val; }
void MW_SendBits(uint16_t data, uint8_t bits) { for(int i = bits - 1; i >= 0; i--) MW_SendBit((data >> i) & 1); }
uint16_t MW_ReadBits(uint8_t bits) { uint16_t value = 0; for(int i = 0; i < bits; i++) { value <<= 1; if(MW_ReadBit()) value |= 1; } return value; }
int getMWAddressBits() { bool is16 = MODE_16BIT; if (eepromSize == "93_46") return is16 ? 6 : 7; if (eepromSize == "93_56" || eepromSize == "93_66") return is16 ? 8 : 9; if (eepromSize == "93_76" || eepromSize == "93_86") return is16 ? 10 : 11; return 8; }
void EEPROM_EWEN() { MW_Start(); MW_SendBit(1); MW_SendBits(0b00, 2); MW_SendBits(0b11, 2); MW_SendBits(0, getMWAddressBits()); MW_Stop(); delay(5); }
void EEPROM_EWDS() { MW_Start(); MW_SendBit(1); MW_SendBits(0b00, 2); MW_SendBits(0b00, 2); MW_SendBits(0, getMWAddressBits()); MW_Stop(); delay(5); }
uint16_t EEPROM_Read_93xx(uint16_t addr) { uint16_t data = 0; MW_Start(); MW_SendBit(1); MW_SendBits(0b10, 2); MW_SendBits(addr, getMWAddressBits()); if(MODE_16BIT) data = MW_ReadBits(16); else data = MW_ReadBits(8); MW_Stop(); return data; }
bool EEPROM_Write_93xx(uint16_t addr, uint16_t data) { EEPROM_EWEN(); MW_Start(); MW_SendBit(1); MW_SendBits(0b01, 2); MW_SendBits(addr, getMWAddressBits()); if(MODE_16BIT) MW_SendBits(data, 16); else MW_SendBits(data, 8); MW_Stop(); delay(10); uint16_t verify = EEPROM_Read_93xx(addr); EEPROM_EWDS(); return (verify == data); }
bool write93XXBlock(uint16_t startAddress, const uint8_t *data, size_t length) { setupHardwarePins(); isOperating = true; int step = MODE_16BIT ? 2 : 1; uint16_t logicalAddr = MODE_16BIT ? (startAddress / 2) : startAddress; for (size_t i = 0; i < length; i += step) { ledFlicker(); yield(); uint16_t wordData = data[i]; if (MODE_16BIT && i + 1 < length) wordData |= (data[i+1] << 8); if(!EEPROM_Write_93xx(logicalAddr, wordData)) { isOperating = false; return false; } logicalAddr++; } isOperating = false; return true; }
bool read93XXBlock(uint16_t startAddress, uint8_t *data, size_t length) { setupHardwarePins(); isOperating = true; int step = MODE_16BIT ? 2 : 1; uint16_t logicalAddr = MODE_16BIT ? (startAddress / 2) : startAddress; for (size_t i = 0; i < length; i += step) { ledFlicker(); yield(); uint16_t wordData = EEPROM_Read_93xx(logicalAddr); data[i] = wordData & 0xFF; if (MODE_16BIT && i + 1 < length) data[i+1] = (wordData >> 8) & 0xFF; logicalAddr++; } isOperating = false; return true; }

bool writeEEPROMPage(uint16_t address, const uint8_t *data, size_t length) {
    setupHardwarePins(); isOperating = true; 
    uint8_t deviceAddress = getDeviceAddress(address);
    Wire.beginTransmission(deviceAddress);
    if (is16BitAddress()) Wire.write((address >> 8) & 0xFF);  
    Wire.write(address & 0xFF);  
    for (size_t i = 0; i < length; i++) Wire.write(data[i]);
    if (Wire.endTransmission() != 0) { isOperating = false; return false; }
    return pollAck(deviceAddress); 
}

bool readEEPROMPage(uint16_t address, uint8_t *data, size_t length) {
    setupHardwarePins(); isOperating = true; 
    uint8_t deviceAddress = getDeviceAddress(address);
    Wire.beginTransmission(deviceAddress);
    if (is16BitAddress()) Wire.write((address >> 8) & 0xFF); 
    Wire.write(address & 0xFF);  
    if (Wire.endTransmission() != 0) { isOperating = false; return false; }
    Wire.requestFrom(deviceAddress, length);
    for (size_t i = 0; i < length; i++) {
        if (Wire.available()) data[i] = Wire.read(); else { isOperating = false; return false; }
    }
    isOperating = false; return true;
}

// ==========================================
// WEB UI DATA (SMART PROGRAMMER THEME)
// ==========================================
const char activation_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>Activation - Smart Programmer</title>
<style>
body { background: #e5d8d0; color: #4a4038; font-family: sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
.glass-box { background: #fff; padding: 40px; border-radius: 20px; text-align: center; width: 300px; box-shadow: 0 10px 30px rgba(0,0,0,0.05); }
input { width: 90%; padding: 15px; margin: 25px 0; background: #fdfaf6; border: 1px solid #e6e2de; color: #4a4038; border-radius: 12px; font-size: 16px; text-align: center; outline:none; }
button { background: #d3695d; color: #fff; border: none; padding: 16px; width: 100%; border-radius: 12px; cursor: pointer; font-size: 16px; font-weight: bold; }
.device-id { font-size: 28px; color: #a28571; font-weight: bold; margin-bottom: 10px; letter-spacing: 2px;}
</style></head>
<body><div class="glass-box"><h2 style="margin-top:0;">SYSTEM LOCKED</h2><p style="font-size: 13px;">Provide Hardware ID to unlock.</p>
<div class="device-id">{DEVICE_ID}</div><input type="number" id="key" placeholder="Enter License Key"><button onclick="activate()">Unlock Programmer</button></div>
<script>
function activate() { let key = document.getElementById('key').value;
    fetch('/activate', { method: 'POST', headers: {'Content-Type': 'application/x-www-form-urlencoded'}, body: 'key='+key })
    .then(res => res.text()).then(data => { if(data === "OK") location.reload(); else alert("Invalid Key!"); }); }
</script></body></html>
)rawliteral";

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>Smart Programmer</title>
    <style>
        body { background: #ebdcd5; color: #5a4b41; font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; padding: 15px; }
        .container { max-width: 950px; margin: 0 auto; display: grid; grid-template-columns: 1fr; gap: 20px; }
        @media (min-width: 850px) { .container { grid-template-columns: 320px 1fr; } }
        
        .card { background: #ebdcd5; border-radius: 25px; padding: 25px; box-shadow: 6px 6px 12px #ccc0b8, -6px -6px 12px #ffffff; border: 1px solid rgba(255,255,255,0.4); margin-bottom: 10px; }
        .header-title { font-size: 22px; font-weight: 800; color: #826f63; margin: 0 0 5px 0; text-align: center; text-shadow: 1px 1px 0px #fff; }
        h3 { font-size: 14px; text-transform: uppercase; letter-spacing: 1px; color: #826f63; margin-top: 0; margin-bottom: 15px; border-bottom: 2px solid rgba(0,0,0,0.05); padding-bottom: 8px;}
        
        select, input[type="text"], input[type="file"] { width: 100%; padding: 14px; margin-bottom: 15px; background: #ebdcd5; border: none; color: #5a4b41; font-size: 14px; font-weight: 600; border-radius: 15px; outline: none; box-sizing: border-box; box-shadow: inset 4px 4px 8px #ccc0b8, inset -4px -4px 8px #ffffff; }
        
        .btn { background: linear-gradient(145deg, #ffffff, #d4c6be); color: #5a4b41; border: 1px solid rgba(255,255,255,0.3); padding: 14px; border-radius: 16px; font-size: 14px; font-weight: 700; cursor: pointer; width: 100%; transition: all 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275); box-sizing: border-box; box-shadow: 5px 5px 10px #ccc0b8, -5px -5px 10px #ffffff; }
        .btn:active { box-shadow: inset 4px 4px 8px #ccc0b8, inset -4px -4px 8px #ffffff; transform: scale(0.98); }
        .btn-primary { background: linear-gradient(145deg, #f8877b, #be5f54); color: #fff; border: none; box-shadow: 4px 4px 10px #be5f54, -4px -4px 10px #ffffff; }
        .btn-primary:active { box-shadow: inset 3px 3px 6px #8c4239, inset -3px -3px 6px #ffffff; }
        .btn-success { background: linear-gradient(145deg, #b69680, #927765); color: #fff; border: none; box-shadow: 4px 4px 10px #927765, -4px -4px 10px #ffffff; }
        .btn-success:active { box-shadow: inset 3px 3px 6px #6d584b, inset -3px -3px 6px #ffffff; }
        .btn-warning { background: linear-gradient(145deg, #fbb069, #d08c48); color: #fff; border: none; box-shadow: 4px 4px 10px #d08c48, -4px -4px 10px #ffffff; }
        .btn-warning:active { box-shadow: inset 3px 3px 6px #9e632b, inset -3px -3px 6px #ffffff; }
        
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 10px; }
        .progress-bg { width: 100%; background: #ebdcd5; border-radius: 10px; height: 12px; margin-bottom: 20px; overflow: hidden; box-shadow: inset 3px 3px 6px #ccc0b8, inset -3px -3px 6px #ffffff; }
        .progress-bar { height: 100%; background: linear-gradient(90deg, #d3695d, #a28571); width: 0%; transition: width 0.2s; }
        
        .hex-viewer { font-family: 'Courier New', monospace; font-size: 13px; background: #1a191d; color: #ebdcd5; padding: 18px; border-radius: 20px; height: 360px; overflow-y: auto; white-space: pre-wrap; line-height: 1.6; box-shadow: inset 5px 5px 12px #000; }
        .hex-addr { color: #6ba2f7; font-weight: bold; } .hex-data { color: #a19ba8; } .hex-ascii { color: #8bbfa3; }
        .diff { background-color: #ef4444; color: #fff; border-radius: 4px; padding: 0 3px; font-weight: bold; box-shadow: 0 0 6px #ef4444; }
        
        .status-box { display: flex; justify-content: space-between; align-items: center; background: #ebdcd5; padding: 14px; border-radius: 15px; font-size: 13px; font-weight: bold; color: #826f63; box-shadow: inset 3px 3px 6px #ccc0b8, inset -3px -3px 6px #ffffff; }
        .switch { position: relative; display: inline-block; width: 36px; height: 20px; } .switch input { opacity: 0; width: 0; height: 0; }
        .slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #ccc; transition: .3s; border-radius: 20px; }
        .slider:before { position: absolute; content: ""; height: 14px; width: 14px; left: 3px; bottom: 3px; background-color: white; transition: .3s; border-radius: 50%; }
        input:checked + .slider { background-color: #d3695d; } input:checked + .slider:before { transform: translateX(16px); }
    </style>
</head>
<body>
    <div class="container">
        <aside>
            <div class="card" style="text-align: center; padding-top:35px;">
                <h1 class="header-title">SMART PROGRAMMER</h1>
                <p style="font-size:12px; color:#826f63; margin-top:5px; font-weight: 600;">Handheld Service Tool</p>
                <div class="status-box" style="margin-top:20px;">
                    <span id="chip-info">Checking Hardware...</span>
                    <div id="v-led" style="width:12px; height:12px; border-radius:50%; background:#ccc; box-shadow: 1px 1px 2px #fff;"></div>
                </div>
                <button onclick="testClip()" class="btn" style="margin-top:15px; padding:10px; font-size:12px;">Verify Physical Contact</button>
            </div>
            <div class="card">
                <h3>Target Selection</h3>
                <select id="eepromFamily" onchange="updateUI()">
                    <option value="24xx" selected>24xx (I2C Standard)</option>
                    <option value="93xx">93xx (Microwire SPI)</option>
                </select>
                <select id="eepromSize" onchange="pushConfig()"></select>
                <select id="eepromOrg" style="display:none;" onchange="pushConfig()">
                    <option value="x16">Word Structure (x16)</option>
                    <option value="x8">Byte Structure (x8)</option>
                </select>
                <button onclick="detectChip()" id="btnDetect" class="btn" style="margin-top:5px;">Execute Auto-Scan</button>
            </div>
            <div style="text-align:center; margin-bottom: 15px;">
                <button onclick="toggleOTA()" style="background:transparent; border:none; color:#826f63; font-size:12px; cursor:pointer; text-decoration:underline; font-weight:700;">⚙️ System Recovery / OTA Update</button>
            </div>
            <div class="card" id="otaPanel" style="display: none;">
                <h3 style="color: #ef4444;">Emergency Flash Guard</h3>
                <form method='POST' action='/update' enctype='multipart/form-data' id='upload_form'>
                    <input type='file' name='update' id='file' accept='.bin'>
                    <button type='submit' class='btn btn-primary'>Write Firmware File</button>
                </form>
            </div>
        </aside>
        <main>
            <div class="card">
                <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:15px;">
                    <h3 style="margin:0; border:none;">Execution Panel</h3>
                    <div style="display:flex; align-items:center; gap:8px; font-size:12px; color:#826f63; font-weight:bold;">
                        <span>Smart Safety Backup</span>
                        <label class="switch"><input type="checkbox" id="autoBackupToggle" checked><span class="slider"></span></label>
                    </div>
                </div>
                <div class="progress-bg"><div id="progressBar" class="progress-bar"></div></div>
                <div class="grid-2">
                    <button id="btnRead" onclick="doRead()" class="btn btn-primary">Read Target</button>
                    <button id="btnWrite" onclick="writeWithBackup()" class="btn btn-success">Program Data</button>
                    <button onclick="document.getElementById('fileInput').click()" class="btn">Import Local File</button>
                    <button onclick="downloadDump()" class="btn">Export Bin to Phone</button>
                    <button id="btnErase" onclick="eraseEEPROM()" class="btn" style="color:#ef4444; border-color:rgba(239,68,68,0.3); grid-column: span 2;">Format Chip (0xFF Wiping)</button>
                </div>
                <input type="file" id="fileInput" style="display:none" accept=".bin,.hex" onchange="handleFile(event)">
            </div>
            <div class="card">
                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px;">
                    <h3 style="margin:0; border:none;">Advanced Hex & Comparator</h3>
                    <div style="display:flex; gap:10px;">
                        <button onclick="toggleAscii()" id="btnAscii" class="btn" style="padding:6px 15px; width:auto; font-size:12px; background:linear-gradient(145deg, #e6e6e6, #c2c2c2);">ASCII: ON</button>
                        <button onclick="startDualCompare()" class="btn btn-warning" style="padding:6px 15px; width:auto; font-size:12px;">Compare 2 Files</button>
                        <button onclick="editHexByte()" class="btn" style="padding:6px 15px; width:auto; font-size:12px;">Patch Byte</button>
                    </div>
                </div>
                <input type="file" id="compFile1" style="display:none" accept=".bin,.hex" onchange="handleComp1(event)">
                <input type="file" id="compFile2" style="display:none" accept=".bin,.hex" onchange="handleComp2(event)">
                <div id="hexViewer" class="hex-viewer">System Idle. Initialize target connection parameters...</div>
            </div>
        </main>
    </div>
    <script>
        let hexBuffer = new Uint8Array(0); 
        let compareBuffer = null;
        let isChipReady = false; 
        let showAscii = true;

        let dualComp1 = null;
        function startDualCompare() {
            alert("Step 1: Please select the FIRST file (Original/Master).");
            document.getElementById('compFile1').click();
        }
        function handleComp1(e) {
            let f = e.target.files[0]; if(!f) return;
            let r = new FileReader(); r.onload = (ev) => { 
                dualComp1 = new Uint8Array(ev.target.result); 
                alert("Step 2: Now select the SECOND file (Modified/Target).");
                document.getElementById('compFile2').click(); 
            }; r.readAsArrayBuffer(f);
        }
        function handleComp2(e) {
            let f = e.target.files[0]; if(!f) return;
            let r = new FileReader(); r.onload = (ev) => { 
                let dualComp2 = new Uint8Array(ev.target.result); 
                hexBuffer = dualComp1; compareBuffer = dualComp2; renderHex();
                alert("Comparator Mode Active!\nRed Highlights show differences between File 1 and File 2."); 
            }; r.readAsArrayBuffer(f);
        }

        function toggleAscii() {
            showAscii = !showAscii;
            document.getElementById('btnAscii').innerText = showAscii ? "ASCII: ON" : "ASCII: OFF";
            renderHex();
        }

        function saveState() {
            sessionStorage.setItem('eepromFamily', document.getElementById('eepromFamily').value);
            sessionStorage.setItem('eepromSize', document.getElementById('eepromSize').value);
            if(hexBuffer.length > 0) { sessionStorage.setItem('hexBuf', JSON.stringify(Array.from(hexBuffer))); }
        }
        function loadState() {
            if(sessionStorage.getItem('eepromFamily')) document.getElementById('eepromFamily').value = sessionStorage.getItem('eepromFamily');
            updateUI();
            if(sessionStorage.getItem('eepromSize')) { document.getElementById('eepromSize').value = sessionStorage.getItem('eepromSize'); pushConfig(); }
            if(sessionStorage.getItem('hexBuf')) { hexBuffer = new Uint8Array(JSON.parse(sessionStorage.getItem('hexBuf'))); renderHex(); }
        }
        window.addEventListener('DOMContentLoaded', loadState);

        const chipOpts = {
            "24xx": ["24C01","24C02","24C04","24C08","24C16","24C32","24C64","24C128","24C256","24C512"],
            "93xx": ["93_46","93_56","93_66","93_76","93_86"]
        };
        const szMap = { '24C01':128, '24C02':256, '24C04':512, '24C08':1024, '24C16':2048, '24C32':4096, '24C64':8192, '24C128':16384, '24C256':32768, '24C512':65536, '93_46':128, '93_56':256, '93_66':512, '93_76':1024, '93_86':2048 };

        function updateUI() {
            let fam = document.getElementById('eepromFamily').value; let sel = document.getElementById('eepromSize'); sel.innerHTML = '';
            chipOpts[fam].forEach(v => { let o = document.createElement('option'); o.value = v; o.innerText = v.replace('_','C'); sel.appendChild(o); });
            if(fam === '24xx') {
                document.getElementById('btnDetect').setAttribute('onclick', 'detectChip()'); document.getElementById('eepromOrg').style.display = 'none';
            } else {
                document.getElementById('btnDetect').setAttribute('onclick', 'detect93xx()'); document.getElementById('eepromOrg').style.display = 'block';
            }
            pushConfig();
        }
        async function pushConfig() {
            let p = { main_mode: "EEPROM", eeprom_mode: "Read", eeprom_family: document.getElementById('eepromFamily').value, eeprom_size: document.getElementById('eepromSize').value, eeprom_org: document.getElementById('eepromOrg').value };
            await fetch('/detail_data', { method: 'POST', body: JSON.stringify(p) }); saveState();
        }
        
        setInterval(() => {
            fetch('/api/status').then(r => r.json()).then(d => {
                let led = document.getElementById('v-led'); let info = document.getElementById('chip-info');
                if(d.act === 'ERROR') { 
                    led.style.background = '#ef4444'; info.innerText = "Blocked: " + d.err; info.style.color = '#ef4444'; isChipReady = false; 
                }
                else if(d.act === 'READING' || d.act === 'WRITING') { 
                    led.style.background = '#f59e0b'; info.innerText = d.act + " " + d.pct + "%"; info.style.color = '#d3695d'; 
                }
                else if(d.act === 'SUCCESS') { 
                    led.style.background = '#4ade80'; info.innerText = "Process Success"; info.style.color = '#4ade80'; isChipReady = true; 
                }
                else {
                    if(d.err === "CHIP NOT FOUND" || d.err === "CHIP NOT DETECTED") {
                        led.style.background = '#ccc'; info.innerText = "CHIP NOT DETECTED"; info.style.color = '#5a4b41'; isChipReady = false;
                    } else {
                        led.style.background = '#4ade80'; info.innerText = "Hardware Connected"; info.style.color = '#4ade80'; isChipReady = true;
                    }
                }
            }).catch(e=>{});
        }, 400);

        function toggleOTA() { let p = document.getElementById('otaPanel'); p.style.display = (p.style.display === 'none') ? 'block' : 'none'; }
        
        async function testClip() {
            await pushConfig(); document.getElementById('chip-info').innerText = "Analyzing...";
            let r = await fetch('/api/test_clip'); let d = await r.json();
            if(!r.ok) { alert("Execution Aborted: Hardware Missing."); isChipReady = false; } 
            else { alert(d.status ? "✅ HARDWARE COUPLING VERIFIED!" : "❌ CONTACT EXCEPTION: Loose alignment detected."); isChipReady = d.status; }
        }
        async function detectChip() {
            let r = await fetch('/detect_chip'); let d = await r.json();
            if(d.found) { document.getElementById('eepromSize').value = d.chip; await pushConfig(); alert("Identified target node: " + d.chip); isChipReady = true; } 
            else { alert("Scan execution failed: Target absent."); isChipReady = false; }
        }
        async function detect93xx() {
            let r = await fetch('/detect_93xx'); let d = await r.json();
            if(d.found) { document.getElementById('eepromSize').value = d.chip; await pushConfig(); alert("Identified microwire structure: " + d.chip); isChipReady = true; } 
            else { alert("Microwire response failure."); isChipReady = false; }
        }

        async function readCore(silent = false) {
            if(!isChipReady && !silent) { alert("CRITICAL ACCESS DENIED: Ensure hardware auto-scan has executed successfully before interaction."); return null; }
            await pushConfig(); let total = szMap[document.getElementById('eepromSize').value]; 
            let tmp = new Uint8Array(total); let chunk = 64; if(!silent) document.getElementById('progressBar').style.width = '0%';
            for(let a = 0; a < total; a += chunk) {
                let len = Math.min(chunk, total - a); let r = await fetch(`/api_web_read?addr=${a}&len=${len}`);
                if(!r.ok) { let e = await r.json(); if(!silent) { alert("Bus Closed Mid-Stream: " + e.error); isChipReady = false; } return null; } 
                let d = await r.json(); for(let i=0; i<d.data.length; i++) tmp[a+i] = d.data[i];
                if(!silent) { document.getElementById('progressBar').style.width = (((a+len)/total)*100).toFixed(0)+'%'; }
            }
            if(!silent) fetch('/api/set_status?s=SUCCESS'); return tmp;
        }
        async function doRead() { compareBuffer = null; let b = await readCore(false); if(b){ hexBuffer = b; renderHex(); saveState(); } }
        
        async function writeWithBackup() {
            if(!isChipReady) { alert("WRITE PROTECTION ACTIVE: Fix physical chip connection to interface controller."); return; }
            if(hexBuffer.length===0) return alert('No code payload loaded in workspace cache.');
            if(document.getElementById('autoBackupToggle').checked) {
                let bk = await readCore(true);
                if(bk) {
                    let nm = `Rollback_${document.getElementById('eepromSize').value}.bin`;
                    let url = URL.createObjectURL(new Blob([bk])); let a = document.createElement('a'); a.href = url; a.download = nm; a.click();
                } else { alert("Safety rule triggered: Backup initialization failed due to loose chip. Operation blocked."); return; }
            }
            await writeCore();
        }
        async function writeCore() {
            if(!isChipReady) return;
            await pushConfig(); let total = szMap[document.getElementById('eepromSize').value];
            document.getElementById('progressBar').style.width = '0%'; let chunk = 64; 
            for(let a = 0; a < total; a += chunk) {
                let chk = hexBuffer.slice(a, a+chunk); if(chk.length===0) break;
                let r = await fetch('/api_web_write', { method: 'POST', body: JSON.stringify({ addr: a, data: Array.from(chk) }) });
                if(!r.ok) { let e = await r.json(); alert("Controller abort event: " + e.error); isChipReady = false; return; }
                document.getElementById('progressBar').style.width = (((a+chk.length)/total)*100).toFixed(0)+'%';
            }
            fetch('/api/set_status?s=SUCCESS'); alert('Workspace successfully mapped into EEPROM registers!');
        }
        async function eraseEEPROM() { 
            if(!isChipReady) { alert("ERASE BLOCKED: Connect chip."); return; }
            if(!confirm('Execute permanent registers wipe to 0xFF? This action cannot be undone.')) return; 
            hexBuffer = new Uint8Array(szMap[document.getElementById('eepromSize').value]).fill(0xFF); renderHex(); await writeWithBackup(); 
        }

        function renderHex() {
            let html = [];
            for (let i = 0; i < hexBuffer.length; i += 16) {
                let addr = i.toString(16).padStart(4, '0').toUpperCase(); let hArr = []; let asc = "";
                for (let j = 0; j < 16; j++) {
                    if (i+j < hexBuffer.length) { 
                        let v = hexBuffer[i+j]; let hStr = v.toString(16).padStart(2, '0').toUpperCase();
                        if(compareBuffer && i+j < compareBuffer.length && compareBuffer[i+j] !== v) hArr.push(`<span class="diff">${hStr}</span>`);
                        else hArr.push(`<span class="hex-data">${hStr}</span>`);
                        asc += (v>=32 && v<=126) ? String.fromCharCode(v).replace(/</g,"&lt;") : '.'; 
                    } else { hArr.push("  "); asc += ' '; }
                }
                if(showAscii) {
                    let processedAscii = asc.replace(/([A-Z0-9-]{4,})/g, '<span style="color:#ef4444; font-weight:bold; background:rgba(239,68,68,0.12); padding:0 3px; border-radius:4px; border:1px solid rgba(239,68,68,0.25); text-shadow:0 0 1px rgba(239,68,68,0.3);">$1</span>');
                    html.push(`<span class="hex-addr">${addr}</span>   ${hArr.join(' ')}  <span class="hex-ascii">${processedAscii}</span>`); 
                } else {
                    html.push(`<span class="hex-addr">${addr}</span>   ${hArr.join(' ')}`); 
                }
            }
            document.getElementById('hexViewer').innerHTML = html.join('\n');
        }

        function editHexByte() {
            if(hexBuffer.length===0) return alert("Buffer space unit has no active memory addresses mapped.");
            let aStr = prompt("Specify Hex Address line location:"); if(!aStr) return; let a = parseInt(aStr, 16); if(isNaN(a) || a>=hexBuffer.length) return alert("Invalid memory alignment index execution.");
            let curr = hexBuffer[a].toString(16).padStart(2,'0').toUpperCase(); let nw = prompt(`Current alignment index byte: ${curr}\nAssign replacement Hex token (00-FF):`, curr);
            if(nw && /^[0-9A-Fa-f]{1,2}$/.test(nw)) { hexBuffer[a] = parseInt(nw, 16); renderHex(); saveState(); }
        }
        function handleFile(e) {
            let f = e.target.files[0]; if(!f) return; compareBuffer = null;
            let r = new FileReader(); r.onload = (ev) => { hexBuffer = new Uint8Array(ev.target.result); renderHex(); saveState(); alert('Hex matrix context loaded from file.'); }; r.readAsArrayBuffer(f);
        }

        function downloadDump() {
            if(hexBuffer.length===0) return alert("Nothing to save! Read a chip or load a file first.");
            let chipName = document.getElementById('eepromSize').value.replace('_', 'C');
            let userInput = prompt("Enter a custom name for this file (Optional):\ne.g. 'Main_Board' or 'Compressor_Fix'", "");
            if (userInput === null) return; 
            let customName = chipName;
            if (userInput.trim() !== "") { customName += "_" + userInput.trim().replace(/ /g, "_"); } else { customName += "_Dump"; }
            customName += ".bin";
            let url = URL.createObjectURL(new Blob([hexBuffer])); let a = document.createElement('a'); a.href = url; a.download = customName; a.click();
        }
    </script>
</body>
</html>
)rawliteral";

// ==========================================
// API & WEB HANDLERS
// ==========================================
void handleDetailDataRequest() {
    if (!isActivated) return server.send(403, "text/plain", "Locked");
    String payload = server.arg("plain"); 
    JsonDocument jsonDoc; 
    deserializeJson(jsonDoc, payload);
    mainMode = jsonDoc["main_mode"] | "";
    if(jsonDoc.containsKey("eeprom_family")) eepromFamily = jsonDoc["eeprom_family"].as<String>();
    if(jsonDoc.containsKey("eeprom_org")) eepromOrg = jsonDoc["eeprom_org"].as<String>();
    eepromSize = jsonDoc["eeprom_size"] | "";
    setupHardwarePins(); updateOLED(true); server.send(200, "text/plain", "OK");
}

void handleStatusSync() {
    String json = "{";
    json += "\"act\":\"" + currentAction + "\",\"err\":\"" + errorReason + "\",\"chip\":\"" + eepromSize + "\",\"fam\":\"" + eepromFamily + "\",";
    json += "\"bat\":" + String(getBatteryPercentage()) + ",\"lic\":" + String(isActivated ? "true" : "false") + ",";
    json += "\"addr\":" + String(currentAddress) + ",\"total\":" + String(totalBytesActive) + ",\"pct\":" + String(progressPct) + ",";
    json += "\"led\":" + String(webLedState ? "true" : "false"); 
    json += "}";
    server.send(200, "application/json", json);
}

void handleTestClip() {
    if(!checkChipConnected()) {
        server.send(400, "application/json", "{\"error\": \"CHIP_ABSENT_OR_ALIGNMENT_FAULT\"}");
        return;
    }
    currentAction = "TESTING"; updateOLED(true);
    delay(500); 
    currentAction = "READY"; updateOLED(true);
    server.send(200, "application/json", "{\"status\": true}");
}

void handleWebApiRead() {
    if (!isActivated) return server.send(403, "application/json", "{\"error\": \"Locked\"}");
    
    if(!checkChipConnected()) {
        triggerError("CHIP NOT DETECTED");
        return server.send(400, "application/json", "{\"error\": \"CHIP_ABSENT_OR_PINS_FLOATING\"}");
    }

    uint16_t startAddr = server.arg("addr").toInt(); 
    uint16_t len = server.arg("len").toInt();
    if(len > 256) len = 256; 
    
    totalBytesActive = getTotalSizeGlobal(); 
    currentAction = "READING"; 
    currentAddress = startAddr;
    
    if(startAddr == 0) clearLogs();
    if(startAddr == 0) pushLog("Syncing " + eepromSize + "...");
    
    uint8_t buffer[256]; bool success = false;
    
    if (eepromFamily == "93xx") {
        success = read93XXBlock(startAddr, buffer, len);
        progressPct = ((startAddr + len) * 100) / totalBytesActive;
        pushLog("Read Addr 0x" + String(startAddr, HEX));
        updateOLED(); 
    } else {
        size_t bytesRead = 0; success = true;
        size_t lastLogMark = 0;
        
        while(bytesRead < len) {
            ledFlicker(); yield(); 
            size_t pageSz = getEEPROMPageSize(); 
            uint16_t currentWriteAddr = startAddr + bytesRead;
            size_t offset = currentWriteAddr % pageSz; 
            size_t toRead = pageSz - offset; 
            if(toRead > (len - bytesRead)) toRead = len - bytesRead;
            
            if(!readEEPROMPage(currentWriteAddr, &buffer[bytesRead], toRead)) { 
                success = false; break; 
            }
            bytesRead += toRead;
            progressPct = ((currentWriteAddr + toRead) * 100) / totalBytesActive;
            
            if (bytesRead - lastLogMark >= 128 || bytesRead == len) {
                pushLog("Read Addr 0x" + String(currentWriteAddr, HEX));
                lastLogMark = bytesRead;
            }
            updateOLED();
        }
    }
    
    if(success) {
        if(progressPct >= 100) pushLog("Done.");
        JsonDocument doc; 
        JsonArray arr = doc["data"].to<JsonArray>();
        for (size_t i = 0; i < len; i++) arr.add(buffer[i]);
        String res; serializeJson(doc, res); 
        server.send(200, "application/json", res);
    } else { 
        triggerError("READ FAIL MIDWAY");
        server.send(500, "application/json", "{\"error\": \"Read processing execution dropped mid-way\"}"); 
    }
}

void handleWebApiWrite() {
    if (!isActivated) return server.send(403, "text/plain", "Locked");

    if(!checkChipConnected()) {
        triggerError("CHIP NOT DETECTED");
        return server.send(400, "application/json", "{\"error\": \"CHIP_ABSENT_OR_PINS_FLOATING\"}");
    }

    JsonDocument doc; 
    deserializeJson(doc, server.arg("plain"));
    uint16_t startAddr = doc["addr"].as<uint16_t>(); 
    JsonArray dataArr = doc["data"].as<JsonArray>(); 
    size_t len = dataArr.size();
    
    totalBytesActive = getTotalSizeGlobal(); 
    currentAction = "WRITING"; 
    currentAddress = startAddr;
    
    if(startAddr == 0) clearLogs();
    if(startAddr == 0) pushLog("Erasing/Writing...");
    
    uint8_t bytes[256]; for (size_t i = 0; i < len; i++) bytes[i] = dataArr[i].as<uint8_t>();
    bool success = false;
    
    if (eepromFamily == "93xx") {
        success = write93XXBlock(startAddr, bytes, len);
        progressPct = ((startAddr + len) * 100) / totalBytesActive;
        pushLog("Write Addr 0x" + String(startAddr, HEX));
        updateOLED();
    } else {
        size_t bytesWritten = 0; success = true;
        size_t lastLogMark = 0;
        
        while(bytesWritten < len) {
            ledFlicker(); yield(); 
            size_t pageSz = getEEPROMPageSize(); 
            uint16_t currentWriteAddr = startAddr + bytesWritten;
            
            size_t offset = currentWriteAddr % pageSz;
            size_t toWrite = pageSz - offset;
            if(toWrite > (len - bytesWritten)) toWrite = len - bytesWritten;
            
            if(!writeEEPROMPage(currentWriteAddr, &bytes[bytesWritten], toWrite)) { 
                success = false; triggerError("WRITE FAILED"); break; 
            }
            
            uint8_t verifyBuf[64];
            if(!readEEPROMPage(currentWriteAddr, verifyBuf, toWrite)) {
                success = false; triggerError("VERIFY FAIL"); break;
            }
            for(size_t v = 0; v < toWrite; v++) {
                if(verifyBuf[v] != bytes[bytesWritten + v]) {
                    success = false; triggerError("VERIFY MISMATCH"); break;
                }
            }
            if(!success) break; 
            
            bytesWritten += toWrite;
            progressPct = ((currentWriteAddr + toWrite) * 100) / totalBytesActive;
            
            if (bytesWritten - lastLogMark >= 128 || bytesWritten == len) {
                pushLog("Write Addr 0x" + String(currentWriteAddr, HEX));
                lastLogMark = bytesWritten;
            }
            updateOLED();
        }
    }
    
    if(success) { 
        if(progressPct >= 100) pushLog("Verified OK!");
        server.send(200, "text/plain", "OK"); 
    } 
    else { 
        server.send(500, "application/json", "{\"error\": \"Write sequence operation register fault\"}"); 
    }
}

void handleDetectChip() {
    String oldFam = eepromFamily; eepromFamily = "24xx"; 
    pinMode(14, INPUT); pinMode(12, INPUT); pinMode(13, INPUT); pinMode(15, INPUT); delay(100); 
    
    JsonDocument doc; 
    bool respond[8]; int count = 0;
    for(int i=0; i<8; i++) { Wire.beginTransmission(0x50 + i); respond[i] = (Wire.endTransmission() == 0); if(respond[i]) count++; }
    if (count > 0) {
        doc["found"] = true; String chipName = "24C02"; 
        if(count == 8) chipName = "24C16"; else if(count == 4 && respond[0] && respond[1] && respond[2] && respond[3]) chipName = "24C08";
        else if(count == 2 && respond[0] && respond[1]) chipName = "24C04"; else if(count == 1 && respond[0]) chipName = "24C02"; 
        doc["chip"] = chipName;
        currentAction = "READY"; errorReason = "";
    } else { 
        doc["found"] = false; 
        currentAction = "ERROR"; errorReason = "CHIP NOT FOUND";
    }
    eepromFamily = oldFam; setupHardwarePins(); updateOLED(true);
    String res; serializeJson(doc, res); server.send(200, "application/json", res);
}

void handleDetect93xx() {
    String originalSize = eepromSize; String originalOrg = eepromOrg; String originalFam = eepromFamily;
    eepromFamily = "93xx"; eepromOrg = "x16"; MODE_16BIT = true; setupHardwarePins(); delay(50); 
    
    JsonDocument doc; eepromSize = "93_86"; 
    EEPROM_Read_93xx(0x00); delay(5);
    
    uint16_t val0 = EEPROM_Read_93xx(0x00); uint16_t val46 = EEPROM_Read_93xx(0x40);  
    uint16_t val56 = EEPROM_Read_93xx(0x80); uint16_t val66 = EEPROM_Read_93xx(0x100); 
    
    if((val0 == 0xFFFF && val46 == 0xFFFF) || (val0 == 0x0000 && val46 == 0x0000)) {
        doc["found"] = false; triggerError("93XX NOT FOUND");
    } else {
        if(val0 == val46) doc["chip"] = "93_46";
        else if(val0 == val56) doc["chip"] = "93_56";
        else if(val0 == val66) doc["chip"] = "93_66";
        else doc["chip"] = "93_86";
        
        doc["found"] = true; currentAction = "READY"; errorReason = "";
    }
    
    eepromSize = doc["found"] ? doc["chip"].as<String>() : originalSize; 
    eepromOrg = doc["found"] ? "x16" : originalOrg; 
    eepromFamily = doc["found"] ? "93xx" : originalFam; 
    setupHardwarePins(); updateOLED(true);
    String res; serializeJson(doc, res); server.send(200, "application/json", res);
}

void handleSetStatus() {
    String st = server.arg("s");
    if(st == "SUCCESS") { currentAction = "SUCCESS"; updateOLED(true); successTimer = millis(); }
    server.send(200, "text/plain", "OK");
}

void handleActivationPOST() {
    uint32_t enteredKey = strtoul(server.arg("key").c_str(), NULL, 10);
    uint32_t correctKey = ((ESP.getChipId() ^ 0x8F3B9A2C) << 3) ^ 0x7860;
    if (enteredKey == correctKey) { EEPROM.write(0, 11); EEPROM.write(1, 22); EEPROM.commit(); isActivated = true; server.send(200, "text/plain", "OK"); } 
    else { server.send(403, "text/plain", "FAIL"); }
}

void handleRoot() {
    if (isActivated) { 
        server.send_P(200, "text/html", index_html); 
    } 
    else { 
        String page = FPSTR(activation_html); 
        page.replace("{DEVICE_ID}", String(ESP.getChipId())); 
        server.send(200, "text/html", page); 
    }
}

// ==========================================
// SETUP & MAIN LOOP
// ==========================================
void setup() {
    Serial.begin(115200);
    initStatusLED(); 
    
    safeIdlePins();
    delay(1000); 
    
    Wire.begin(PIN_D1, PIN_D2);  
    Wire.setClock(100000); 
    setupHardwarePins(); 

    if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) Serial.println("OLED Init Failed");
    
    EEPROM.begin(512);
    if(EEPROM.read(0) == 11 && EEPROM.read(1) == 22) isActivated = true;

    WiFi.softAP(ssid, password);
    
    server.on("/", HTTP_GET, handleRoot);
    server.on("/activate", HTTP_POST, handleActivationPOST);
    server.on("/detail_data", HTTP_POST, handleDetailDataRequest);
    server.on("/api_web_read", HTTP_GET, handleWebApiRead);
    server.on("/api_web_write", HTTP_POST, handleWebApiWrite);
    server.on("/api/test_clip", HTTP_GET, handleTestClip);
    server.on("/detect_chip", HTTP_GET, handleDetectChip);
    server.on("/detect_93xx", HTTP_GET, handleDetect93xx);
    server.on("/api/status", HTTP_GET, handleStatusSync);
    server.on("/api/set_status", HTTP_GET, handleSetStatus);
    
    server.on("/update", HTTP_POST, []() {
        server.sendHeader("Connection", "close");
        server.send(200, "text/plain", (Update.hasError()) ? "UPDATE FAILED" : "UPDATE SUCCESS! REBOOTING...");
        ESP.restart();
    }, []() {
        HTTPUpload& upload = server.upload();
        if (upload.status == UPLOAD_FILE_START) {
            currentAction = "UPDATING"; updateOLED(true);
            uint32_t maxSketchSpace = (ESP.getFreeSketchSpace() - 0x1000) & 0xFFFFF000;
            Update.begin(maxSketchSpace);
        } else if (upload.status == UPLOAD_FILE_WRITE) {
            Update.write(upload.buf, upload.currentSize);
        } else if (upload.status == UPLOAD_FILE_END) {
            Update.end(true);
        }
        yield();
    });

    server.begin();
    updateOLED(true); 
}

void loop() {
    server.handleClient();
    updateStatusLED(); 
    if(currentAction == "SUCCESS" && millis() - successTimer > 2500) { currentAction = "READY"; clearLogs(); updateOLED(true); }
    if(currentAction == "ERROR" && millis() - successTimer > 3500) { currentAction = "READY"; clearLogs(); updateOLED(true); } 
}
