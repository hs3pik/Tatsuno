#include <FS.h>
using namespace fs;

#include <Arduino.h>
#include <lvgl.h>
#include <TFT_eSPI.h>
#include <SPI.h>
#include <XPT2046_Touchscreen.h>
#include <Preferences.h>
#include <WiFi.h>
#include <WebServer.h>

Preferences preferences;
WebServer server(80);

// ==========================================
// 1. ฮาร์ดแวร์และการตั้งค่าพิน
// ==========================================
#define TFT_BL 21
#define XPT2046_IRQ 36  
#define XPT2046_MOSI 32
#define XPT2046_MISO 39
#define XPT2046_CLK 25
#define XPT2046_CS 33

#define BUZZER_PIN 22  

#define RS485_RX_PIN 16
#define RS485_TX_PIN 17

#define TOUCH_MIN_X 280
#define TOUCH_MAX_X 3800
#define TOUCH_MIN_Y 310
#define TOUCH_MAX_Y 3700

XPT2046_Touchscreen ts(XPT2046_CS, XPT2046_IRQ); 
SPIClass touchSPI(HSPI);
TFT_eSPI tft = TFT_eSPI();

// ==========================================
// 2. ธีมสีสไตล์ตู้จ่ายน้ำมันดิจิทัล
// ==========================================
#define COLOR_BG        0x0A0F1D  
#define COLOR_PANEL     0x111827  
#define COLOR_CARD      0x1F2937  
#define COLOR_CYAN      0x00F5D4  
#define COLOR_GREEN     0x2ECC71  
#define COLOR_BLUE      0x3B82F6  
#define COLOR_ORANGE    0xE67E22  
#define COLOR_YELLOW    0xF1C40F  
#define COLOR_RED       0xEF4444  
#define COLOR_WHITE     0xFFFFFF  
#define COLOR_GRAY      0x9CA3AF  

// ==========================================
// 3. สถานะและตัวแปรตู้จ่าย
// ==========================================
#define STX 0x02
#define ETX 0x03
#define EOT 0x04
#define ENQ 0x05
#define NAK 0x15
#define DLE 0x10

enum AppScreen {
    SCR_LOGO,
    SCR_MENU,
    SCR_CONFIG_PORTAL,
    SCR_MAIN,
    SCR_TOTAL,
    SCR_LOG
};
AppScreen current_screen = SCR_LOGO;

enum PumpState {
    PUMP_IDLE,          
    PUMP_NOZZLE_OUT,    
    PUMP_FUELING,       
    PUMP_STOPPED,       
    PUMP_COMPLETED
};

struct PumpConfig {
    int opMode = 1;               
    int stationAddr = 1;          
    uint32_t unitPrice = 1000;    
    uint32_t totalAmount = 0;     
    uint32_t totalVolume = 0;     
} cfg;

struct PumpData {
    PumpState state = PUMP_IDLE;
    uint32_t volume = 0;          
    uint32_t amount = 0;          
    bool isAuthorized = false;    
} pump;

bool need_ui_update = false;      

// Widgets หน้าจอ LVGL
lv_obj_t * screen_logo;
lv_obj_t * screen_menu;
lv_obj_t * screen_config_portal;
lv_obj_t * screen_main;
lv_obj_t * screen_total;
lv_obj_t * screen_log;

lv_obj_t * lbl_countdown;
lv_obj_t * lbl_status;
lv_obj_t * lbl_amount_val;
lv_obj_t * lbl_volume_val;
lv_obj_t * lbl_price_val;
lv_obj_t * btn_nozzle;
lv_obj_t * btn_action;
lv_obj_t * lbl_portal_info;

lv_obj_t * lbl_tot_amount_val;
lv_obj_t * lbl_tot_volume_val;
lv_obj_t * ta_log; 

unsigned long screen_timer = 0;
unsigned long total_screen_timer = 0; 

// ตัวแปร Thread-Safe สำหรับเก็บ Log ข้าม Core
portMUX_TYPE logMux = portMUX_INITIALIZER_UNLOCKED;
char log_buffer[512] = "";
char last_tx_log[128] = ""; // ใช้กรอง Log ที่ซ้ำกัน

// Forward Declarations
void build_logo_screen();
void build_menu_screen();
void build_config_portal_screen();
void build_main_screen();
void build_total_screen();
void build_log_screen();
void updateUI();
void update_total_UI();
void logSimulatedTx();

// ==========================================
// 4. ระบบ Log และ Config (Preferences)
// ==========================================
void addLog(const char* dir, const char* data) {
    portENTER_CRITICAL(&logMux);
    if(strlen(log_buffer) < 400) {
        strcat(log_buffer, "[");
        strcat(log_buffer, dir);
        strcat(log_buffer, "] ");
        strcat(log_buffer, data);
        strcat(log_buffer, "\n");
    }
    portEXIT_CRITICAL(&logMux);
}

void processLogs() {
    portENTER_CRITICAL(&logMux);
    String new_logs = String(log_buffer);
    log_buffer[0] = '\0';
    portEXIT_CRITICAL(&logMux);

    if(new_logs.length() > 0 && ta_log != NULL) {
        lv_textarea_add_text(ta_log, new_logs.c_str());
        if(strlen(lv_textarea_get_text(ta_log)) > 1500) {
            lv_textarea_set_text(ta_log, "--- LOG CLEARED ---\n");
            lv_textarea_add_text(ta_log, new_logs.c_str());
        }
    }
}

// ฟังก์ชันสร้าง Log ข้อมูลดิบจำลองสำหรับโหมด Standalone
void logSimulatedTx() {
    if (cfg.opMode != 1) return; // ทำงานเฉพาะโหมด Standalone (โหมด 1)

    char payload[64];
    char cond = '0';
    if (pump.state == PUMP_NOZZLE_OUT) cond = '1';
    else if (pump.state == PUMP_FUELING) cond = '3';
    else if (pump.state == PUMP_STOPPED) cond = '4';

    snprintf(payload, sizeof(payload), "61%c%06lu2%04lu%06lu112",
             cond, (unsigned long)pump.volume,
             (unsigned long)cfg.unitPrice,
             (unsigned long)pump.amount);

    // กรองไม่ให้ Log ซ้ำกันสแปมหน้าจอ
    if (strcmp(last_tx_log, payload) != 0) {
        addLog("TX-SIM", payload);
        strcpy(last_tx_log, payload);
    }
}

void loadConfiguration() {
    preferences.begin("pump_cfg", true);
    cfg.opMode       = preferences.getInt("op_mode", 1);
    cfg.stationAddr  = preferences.getInt("st_addr", 1);
    cfg.unitPrice    = preferences.getUInt("u_price", 1000);
    cfg.totalAmount  = preferences.getUInt("tot_amt", 0);
    cfg.totalVolume  = preferences.getUInt("tot_vol", 0);
    preferences.end();
}

void saveConfiguration(int mode, int addr, uint32_t price, uint32_t amt, uint32_t vol) {
    cfg.opMode       = mode;
    cfg.stationAddr  = addr;
    cfg.unitPrice    = price;
    cfg.totalAmount  = amt;
    cfg.totalVolume  = vol;

    preferences.begin("pump_cfg", false);
    preferences.putInt("op_mode", cfg.opMode);
    preferences.putInt("st_addr", cfg.stationAddr);
    preferences.putUInt("u_price", cfg.unitPrice);
    preferences.putUInt("tot_amt", cfg.totalAmount);
    preferences.putUInt("tot_vol", cfg.totalVolume);
    preferences.end();
}

// ==========================================
// 5. RS-485 Communication (Tatsuno Protocol)
// ==========================================
void rs485Write(const uint8_t* buf, size_t len) {
    delayMicroseconds(2000); 
    Serial2.write(buf, len);
    Serial2.flush();
}

void sendEOT()  { uint8_t b = EOT; rs485Write(&b, 1); }
void sendACK0() { uint8_t b[] = {DLE, 0x30}; rs485Write(b, 2); }
void sendACK1() { uint8_t b[] = {DLE, 0x31}; rs485Write(b, 2); }

uint8_t calcBCC(const uint8_t* data, size_t len) {
    uint8_t bcc = 0;
    for (size_t i = 0; i < len; i++) bcc ^= data[i];
    return bcc;
}

void sendTextFrame(uint8_t sa, uint8_t ua, const char* asciiData) {
    uint8_t frame[128];
    size_t idx = 0;
    frame[idx++] = STX;
    frame[idx++] = sa;
    frame[idx++] = ua;
    size_t dataLen = strlen(asciiData);
    memcpy(&frame[idx], asciiData, dataLen);
    idx += dataLen;
    frame[idx++] = ETX;
    frame[idx] = calcBCC(&frame[1], idx - 1);
    idx++;
    rs485Write(frame, idx);
    
    // พิมพ์ Log ขาออก สำหรับโหมดเชื่อม POS (โหมด 3)
    if (strcmp(last_tx_log, asciiData) != 0) {
        addLog("TX", asciiData); 
        strcpy(last_tx_log, asciiData);
    }
}

void sendStatus61() {
    char payload[32];
    char cond = '0';
    if (pump.state == PUMP_NOZZLE_OUT) cond = '1';
    else if (pump.state == PUMP_FUELING) cond = '3';
    else if (pump.state == PUMP_STOPPED) cond = '4';
    
    uint8_t TatsunoSA = 0x40 + cfg.stationAddr;

    snprintf(payload, sizeof(payload), "61%c%06lu2%04lu%06lu112",
             cond, (unsigned long)pump.volume,
             (unsigned long)cfg.unitPrice,
             (unsigned long)pump.amount);
    sendTextFrame(TatsunoSA, 0x51, payload);
}

void readTatsunoTextFrame() {
    unsigned long startTime = millis();
    while (!Serial2.available() && millis() - startTime < 100) { delay(1); }
    
    if (Serial2.available() && Serial2.read() == STX) {
        while (!Serial2.available() && millis() - startTime < 150) { delay(1); }
        uint8_t sa = Serial2.read();
        while (!Serial2.available() && millis() - startTime < 150) { delay(1); }
        uint8_t ua = Serial2.read();

        char rxData[128];
        int idx = 0;
        bool frameComplete = false;
        uint8_t receivedBCC = 0;

        while (millis() - startTime < 500) {
            if (Serial2.available()) {
                uint8_t c = Serial2.read();
                if (c == ETX) {
                    while (!Serial2.available() && millis() - startTime < 500) { delay(1); }
                    receivedBCC = Serial2.read(); 
                    rxData[idx] = '\0';
                    frameComplete = true;
                    break;
                } else {
                    if (idx < 127) rxData[idx++] = c;
                }
            }
        }

        if (frameComplete) {
            uint8_t frameBCC = sa ^ ua;
            for(int i=0; i<idx; i++) frameBCC ^= rxData[i];
            frameBCC ^= ETX;

            if(frameBCC == receivedBCC) {
                sendACK1(); 
                addLog("RX", rxData); 

                if (strncmp(rxData, "10", 2) == 0 || strncmp(rxData, "11", 2) == 0) {
                    pump.isAuthorized = true;
                    need_ui_update = true; 
                }
                else if (strncmp(rxData, "12", 2) == 0) {
                    pump.isAuthorized = false;
                    need_ui_update = true;
                }
            } else {
                addLog("ERR", "BCC Mismatch");
            }
        }
    }
}

void rs485Task(void * pvParameters) {
    Serial2.begin(19200, SERIAL_8E1, RS485_RX_PIN, RS485_TX_PIN);
    while (1) {
        if (cfg.opMode == 3 && Serial2.available()) {
            uint8_t header = Serial2.read();
            if (header == EOT) {
                delay(2);
                if (Serial2.available() >= 3) {
                    uint8_t sa = Serial2.read();
                    uint8_t ua = Serial2.read();
                    uint8_t enq = Serial2.read();
                    uint8_t targetSA = 0x40 + cfg.stationAddr;

                    if (sa == targetSA && enq == ENQ) {
                        if (ua == 0x51) {  
                            if (pump.state != PUMP_IDLE || pump.isAuthorized) sendStatus61();
                            else sendEOT();
                        } 
                        else if (ua == 0x41) { 
                            sendACK0();
                            readTatsunoTextFrame(); 
                        }
                    }
                }
            }
        }
        vTaskDelay(20 / portTICK_PERIOD_MS);
    }
}

// ==========================================
// 6. Web Server สำหรับหน้าจอ Config
// ==========================================
void handleRoot() {
    String html = "<!DOCTYPE html><html><head><meta charset='utf-8'><title>Tatsuno Pump Config</title>";
    html += "<style>body{font-family:Arial;background:#0A0F1D;color:#fff;text-align:center;padding:20px;}";
    html += "input,select{padding:10px;margin:8px;width:80%;max-width:300px;background:#1F2937;color:#fff;border:1px solid #3B82F6;border-radius:4px;}";
    html += "input[type=submit]{background:#2ECC71;color:#fff;font-weight:bold;cursor:pointer;padding:12px;margin-top:20px;}</style></head>";
    html += "<body><h2>Tatsuno Pump Simulator Config</h2><form action='/save' method='POST'>";
    
    html += "Operation Mode:<br><select name='op'>";
    html += "<option value='1' " + String(cfg.opMode == 1 ? "selected" : "") + ">1 - Standalone</option>";
    html += "<option value='3' " + String(cfg.opMode == 3 ? "selected" : "") + ">3 - Link POS</option></select><br>";
    
    html += "Station Address [1-16]:<br><input type='number' name='addr' min='1' max='16' value='" + String(cfg.stationAddr) + "'><br>";
    
    char priceStr[10];
    snprintf(priceStr, sizeof(priceStr), "%d.%02d", cfg.unitPrice / 100, cfg.unitPrice % 100);
    html += "Unit Price [xx.xx]:<br><input type='number' step='0.01' name='price' value='" + String(priceStr) + "'><br>";
    
    html += "Total Amount [6 digits]:<br><input type='number' name='amt' value='" + String(cfg.totalAmount) + "'><br>";
    html += "Total Volume [6 digits]:<br><input type='number' name='vol' value='" + String(cfg.totalVolume) + "'><br>";
    
    html += "<input type='submit' value='SAVE & REBOOT'></form></body></html>";
    server.send(200, "html", html);
}

void handleSave() {
    int m = server.arg("op").toInt();
    int a = server.arg("addr").toInt();
    float p_float = server.arg("price").toFloat();
    uint32_t p = (uint32_t)(p_float * 100.0 + 0.5); 
    uint32_t am = server.arg("amt").toInt();
    uint32_t vo = server.arg("vol").toInt();
    
    saveConfiguration(m, a, p, am, vo);

    server.send(200, "html", "<h3 style='font-family:Arial;color:#2ECC71;text-align:center;margin-top:50px;'>Saved Successfully! Rebooting ESP32...</h3>");
    delay(2000);
    ESP.restart();
}

// ==========================================
// 7. Event Handlers & Gestures (Swipe)
// ==========================================
static void main_screen_event_cb(lv_event_t * e) {
    lv_dir_t dir = lv_indev_get_gesture_dir(lv_indev_get_act());
    if (current_screen == SCR_MAIN) {
        if (dir == LV_DIR_LEFT) { 
            lv_scr_load_anim(screen_total, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            current_screen = SCR_TOTAL;
            total_screen_timer = millis();
            update_total_UI();
        }
    }
}

static void total_screen_event_cb(lv_event_t * e) {
    lv_dir_t dir = lv_indev_get_gesture_dir(lv_indev_get_act());
    if (dir == LV_DIR_RIGHT) { 
        lv_scr_load_anim(screen_main, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
        current_screen = SCR_MAIN;
        updateUI();
    }
    else if (dir == LV_DIR_LEFT) {
        lv_scr_load_anim(screen_log, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
        current_screen = SCR_LOG;
    }
}

static void log_screen_event_cb(lv_event_t * e) {
    lv_dir_t dir = lv_indev_get_gesture_dir(lv_indev_get_act());
    if (dir == LV_DIR_RIGHT) { 
        lv_scr_load_anim(screen_total, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
        current_screen = SCR_TOTAL;
        total_screen_timer = millis();
    }
    else if (dir == LV_DIR_BOTTOM) {
        lv_scr_load_anim(screen_main, LV_SCR_LOAD_ANIM_MOVE_BOTTOM, 200, 0, false);
        current_screen = SCR_MAIN;
        updateUI();
    }
}

static void btn_goto_config_event_cb(lv_event_t * e) {
    lv_scr_load(screen_config_portal);
    current_screen = SCR_CONFIG_PORTAL;

    lv_label_set_text(lbl_portal_info, "Connect Wi-Fi:\nTatsuno-Pump-Sim\nIP: 192.168.4.1");

    WiFi.mode(WIFI_AP);
    WiFi.softAP("Tatsuno-Pump-Sim");
    server.on("/", handleRoot);
    server.on("/save", handleSave);
    server.begin();
}

static void btn_nozzle_event_cb(lv_event_t * e) {
    if (pump.state == PUMP_IDLE) {
        pump.volume = 0;
        pump.amount = 0;
        pump.state = PUMP_NOZZLE_OUT;
        pump.isAuthorized = (cfg.opMode == 1);
    } 
    else if (pump.state == PUMP_NOZZLE_OUT) {
        pump.state = PUMP_IDLE;
        pump.isAuthorized = false; 
    }
    else if (pump.state == PUMP_STOPPED) {
        cfg.totalVolume += (pump.volume / 100);
        cfg.totalAmount += (pump.amount / 10);
        preferences.begin("pump_cfg", false);
        preferences.putUInt("tot_amt", cfg.totalAmount);
        preferences.putUInt("tot_vol", cfg.totalVolume);
        preferences.end();

        pump.state = PUMP_IDLE;
        pump.isAuthorized = false;
    }
    updateUI();
}

static void btn_action_event_cb(lv_event_t * e) {
    if (pump.state == PUMP_NOZZLE_OUT) {
        if (pump.isAuthorized) {
            pump.state = PUMP_FUELING; 
        }
    } 
    else if (pump.state == PUMP_FUELING) {
        pump.state = PUMP_STOPPED; 
    }
    updateUI();
}

// ==========================================
// 8. สร้างหน้าจอ UI ต่างๆ
// ==========================================
void build_logo_screen() {
    screen_logo = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_logo, lv_color_hex(COLOR_BG), 0);

    lv_obj_t * lbl_logo = lv_label_create(screen_logo);
    lv_label_set_text(lbl_logo, "TATSUNO SIMULATOR");
    lv_obj_set_style_text_font(lbl_logo, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_logo, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_logo, LV_ALIGN_CENTER, 0, -10);

    lv_obj_t * lbl_sub = lv_label_create(screen_logo);
    lv_label_set_text(lbl_sub, "Initializing System...");
    lv_obj_set_style_text_font(lbl_sub, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_sub, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(lbl_sub, LV_ALIGN_CENTER, 0, 20);

    lv_obj_t * lbl_sig = lv_label_create(screen_logo);
    lv_label_set_text(lbl_sig, "Dx Solution HS3PIK");
    lv_obj_set_style_text_font(lbl_sig, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_sig, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(lbl_sig, LV_ALIGN_BOTTOM_RIGHT, -5, -5);
}

void build_menu_screen() {
    screen_menu = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_menu, lv_color_hex(COLOR_BG), 0);

    lbl_countdown = lv_label_create(screen_menu);
    lv_label_set_text(lbl_countdown, "Auto start in 5s");
    lv_obj_set_style_text_font(lbl_countdown, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_countdown, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_countdown, LV_ALIGN_TOP_MID, 0, 30);

    lv_obj_t * btn_cfg = lv_btn_create(screen_menu);
    lv_obj_set_size(btn_cfg, 200, 60);
    lv_obj_align(btn_cfg, LV_ALIGN_CENTER, 0, 10);
    lv_obj_set_style_bg_color(btn_cfg, lv_color_hex(COLOR_ORANGE), 0);
    lv_obj_add_event_cb(btn_cfg, btn_goto_config_event_cb, LV_EVENT_CLICKED, NULL);

    lv_obj_t * lbl_btn = lv_label_create(btn_cfg);
    lv_label_set_text(lbl_btn, "CONFIG");
    lv_obj_set_style_text_font(lbl_btn, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_btn, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_center(lbl_btn);
}

void build_config_portal_screen() {
    screen_config_portal = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_config_portal, lv_color_hex(COLOR_BG), 0);

    lv_obj_t * lbl_title = lv_label_create(screen_config_portal);
    lv_label_set_text(lbl_title, "WEB CONFIG PORTAL");
    lv_obj_set_style_text_font(lbl_title, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_title, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_title, LV_ALIGN_TOP_MID, 0, 15);

    lbl_portal_info = lv_label_create(screen_config_portal);
    lv_label_set_text(lbl_portal_info, "Starting AP Mode...");
    lv_obj_set_style_text_font(lbl_portal_info, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_portal_info, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_align(lbl_portal_info, LV_ALIGN_CENTER, 0, 10);
    lv_label_set_long_mode(lbl_portal_info, LV_LABEL_LONG_WRAP);
    lv_obj_set_width(lbl_portal_info, 300);
    lv_obj_set_style_text_align(lbl_portal_info, LV_TEXT_ALIGN_CENTER, 0);
}

void build_main_screen() {
    screen_main = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_main, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(screen_main, main_screen_event_cb, LV_EVENT_GESTURE, NULL);

    lbl_status = lv_label_create(screen_main);
    lv_label_set_text(lbl_status, "STATUS: IDLE");
    lv_obj_set_style_text_font(lbl_status, &lv_font_montserrat_12, 0);
    lv_obj_align(lbl_status, LV_ALIGN_TOP_MID, 0, 4);

    lv_obj_t * panel_amt = lv_obj_create(screen_main);
    lv_obj_set_size(panel_amt, 305, 52);
    lv_obj_align(panel_amt, LV_ALIGN_TOP_MID, 0, 22);
    lv_obj_set_style_bg_color(panel_amt, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(panel_amt, lv_color_hex(COLOR_CYAN), 0);
    
    lv_obj_t * t1 = lv_label_create(panel_amt);
    lv_label_set_text(t1, "AMOUNT");
    lv_obj_set_style_text_font(t1, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(t1, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(t1, LV_ALIGN_LEFT_MID, 4, 0);

    lbl_amount_val = lv_label_create(panel_amt);
    lv_label_set_text(lbl_amount_val, "0.0");
    lv_obj_set_style_text_font(lbl_amount_val, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_amount_val, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_amount_val, LV_ALIGN_RIGHT_MID, -55, 0);

    lv_obj_t * u1 = lv_label_create(panel_amt);
    lv_label_set_text(u1, "THB");
    lv_obj_set_style_text_font(u1, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(u1, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(u1, LV_ALIGN_RIGHT_MID, -10, 0);

    lv_obj_t * panel_vol = lv_obj_create(screen_main);
    lv_obj_set_size(panel_vol, 305, 52);
    lv_obj_align(panel_vol, LV_ALIGN_TOP_MID, 0, 78);
    lv_obj_set_style_bg_color(panel_vol, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(panel_vol, lv_color_hex(COLOR_CYAN), 0);
    
    lv_obj_t * t2 = lv_label_create(panel_vol);
    lv_label_set_text(t2, "LITERS");
    lv_obj_set_style_text_font(t2, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(t2, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(t2, LV_ALIGN_LEFT_MID, 4, 0);

    lbl_volume_val = lv_label_create(panel_vol);
    lv_label_set_text(lbl_volume_val, "0.00");
    lv_obj_set_style_text_font(lbl_volume_val, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_volume_val, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_volume_val, LV_ALIGN_RIGHT_MID, -55, 0);

    lv_obj_t * u2 = lv_label_create(panel_vol);
    lv_label_set_text(u2, "L");
    lv_obj_set_style_text_font(u2, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(u2, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(u2, LV_ALIGN_RIGHT_MID, -15, 0);

    lv_obj_t * panel_price = lv_obj_create(screen_main);
    lv_obj_set_size(panel_price, 305, 36);
    lv_obj_align(panel_price, LV_ALIGN_TOP_MID, 0, 134);
    lv_obj_set_style_bg_color(panel_price, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(panel_price, lv_color_hex(COLOR_GRAY), 0);
    
    lv_obj_t * t3 = lv_label_create(panel_price);
    lv_label_set_text(t3, "PRICE / LITER");
    lv_obj_set_style_text_font(t3, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(t3, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(t3, LV_ALIGN_LEFT_MID, 4, 0);

    lbl_price_val = lv_label_create(panel_price);
    lv_label_set_text(lbl_price_val, "10.00");
    lv_obj_set_style_text_font(lbl_price_val, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_price_val, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_price_val, LV_ALIGN_RIGHT_MID, -55, 0);

    lv_obj_t * u3 = lv_label_create(panel_price);
    lv_label_set_text(u3, "THB");
    lv_obj_set_style_text_font(u3, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(u3, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(u3, LV_ALIGN_RIGHT_MID, -10, 0);

    btn_nozzle = lv_btn_create(screen_main);
    lv_obj_set_size(btn_nozzle, 148, 42);
    lv_obj_align(btn_nozzle, LV_ALIGN_BOTTOM_LEFT, 8, -6);
    lv_obj_set_style_bg_color(btn_nozzle, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_add_event_cb(btn_nozzle, btn_nozzle_event_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t * l1 = lv_label_create(btn_nozzle);
    lv_label_set_text(l1, "Up Nozzle");
    lv_obj_center(l1);

    btn_action = lv_btn_create(screen_main);
    lv_obj_set_size(btn_action, 148, 42);
    lv_obj_align(btn_action, LV_ALIGN_BOTTOM_RIGHT, -8, -6);
    lv_obj_set_style_bg_color(btn_action, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_add_event_cb(btn_action, btn_action_event_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t * l2 = lv_label_create(btn_action);
    lv_label_set_text(l2, "START");
    lv_obj_center(l2);
}

void build_total_screen() {
    screen_total = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_total, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(screen_total, total_screen_event_cb, LV_EVENT_GESTURE, NULL);

    lv_obj_t * lbl_title = lv_label_create(screen_total);
    lv_label_set_text(lbl_title, "TOTAL ACCUMULATED");
    lv_obj_set_style_text_font(lbl_title, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_title, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_title, LV_ALIGN_TOP_MID, 0, 10);

    lv_obj_t * p_tot_amt = lv_obj_create(screen_total);
    lv_obj_set_size(p_tot_amt, 305, 75);
    lv_obj_align(p_tot_amt, LV_ALIGN_TOP_MID, 0, 40);
    lv_obj_set_style_bg_color(p_tot_amt, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(p_tot_amt, lv_color_hex(COLOR_GREEN), 0);

    lv_obj_t * ta_lbl = lv_label_create(p_tot_amt);
    lv_label_set_text(ta_lbl, "TOTAL AMOUNT");
    lv_obj_set_style_text_font(ta_lbl, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(ta_lbl, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(ta_lbl, LV_ALIGN_TOP_LEFT, 4, 2);

    lbl_tot_amount_val = lv_label_create(p_tot_amt);
    lv_label_set_text(lbl_tot_amount_val, "0");
    lv_obj_set_style_text_font(lbl_tot_amount_val, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_tot_amount_val, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_align(lbl_tot_amount_val, LV_ALIGN_BOTTOM_RIGHT, -10, -5);

    lv_obj_t * p_tot_vol = lv_obj_create(screen_total);
    lv_obj_set_size(p_tot_vol, 305, 75);
    lv_obj_align(p_tot_vol, LV_ALIGN_TOP_MID, 0, 125);
    lv_obj_set_style_bg_color(p_tot_vol, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(p_tot_vol, lv_color_hex(COLOR_GREEN), 0);

    lv_obj_t * tv_lbl = lv_label_create(p_tot_vol);
    lv_label_set_text(tv_lbl, "TOTAL VOLUME");
    lv_obj_set_style_text_font(tv_lbl, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(tv_lbl, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(tv_lbl, LV_ALIGN_TOP_LEFT, 4, 2);

    lbl_tot_volume_val = lv_label_create(p_tot_vol);
    lv_label_set_text(lbl_tot_volume_val, "0");
    lv_obj_set_style_text_font(lbl_tot_volume_val, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_tot_volume_val, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_align(lbl_tot_volume_val, LV_ALIGN_BOTTOM_RIGHT, -10, -5);
}

void build_log_screen() {
    screen_log = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(screen_log, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(screen_log, log_screen_event_cb, LV_EVENT_GESTURE, NULL);

    lv_obj_t * lbl_title = lv_label_create(screen_log);
    lv_label_set_text(lbl_title, "TATSUNO LOG");
    lv_obj_set_style_text_font(lbl_title, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_title, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_title, LV_ALIGN_TOP_MID, 0, 8);

    ta_log = lv_textarea_create(screen_log);
    lv_obj_set_size(ta_log, 310, 200);
    lv_obj_align(ta_log, LV_ALIGN_BOTTOM_MID, 0, -5);
    lv_obj_set_style_bg_color(ta_log, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_text_color(ta_log, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_set_style_text_font(ta_log, &lv_font_montserrat_12, 0);
    lv_obj_set_style_border_color(ta_log, lv_color_hex(COLOR_BLUE), 0);
    lv_textarea_set_text(ta_log, "--- SYSTEM BOOT ---\n");
}

void updateUI() {
    char buf_amt[16], buf_vol[16], buf_price[16];
    snprintf(buf_amt, sizeof(buf_amt), "%d.%d", pump.amount / 10, pump.amount % 10);
    snprintf(buf_vol, sizeof(buf_vol), "%d.%02d", pump.volume / 100, pump.volume % 100);
    snprintf(buf_price, sizeof(buf_price), "%d.%02d", cfg.unitPrice / 100, cfg.unitPrice % 100);

    lv_label_set_text(lbl_amount_val, buf_amt);
    lv_label_set_text(lbl_volume_val, buf_vol);
    lv_label_set_text(lbl_price_val, buf_price);

    switch (pump.state) {
        case PUMP_IDLE:
            lv_label_set_text(lbl_status, "STATUS: IDLE");
            lv_obj_set_style_text_color(lbl_status, lv_color_hex(COLOR_GRAY), 0);
            
            lv_label_set_text(lv_obj_get_child(btn_nozzle, 0), "Up Nozzle");
            lv_obj_set_style_bg_color(btn_nozzle, lv_color_hex(COLOR_GREEN), 0);
            
            lv_obj_add_state(btn_action, LV_STATE_DISABLED);
            lv_label_set_text(lv_obj_get_child(btn_action, 0), "START");
            break;

        case PUMP_NOZZLE_OUT:
            if (cfg.opMode == 3 && !pump.isAuthorized) {
                lv_label_set_text(lbl_status, "WAITING POS AUTH...");
                lv_obj_set_style_text_color(lbl_status, lv_color_hex(COLOR_ORANGE), 0);
                
                lv_obj_add_state(btn_action, LV_STATE_DISABLED); 
                lv_obj_set_style_bg_color(btn_action, lv_color_hex(COLOR_GRAY), 0);
            } 
            else { 
                lv_label_set_text(lbl_status, "POS AUTHORIZED! (READY)");
                lv_obj_set_style_text_color(lbl_status, lv_color_hex(COLOR_GREEN), 0);
                
                lv_obj_clear_state(btn_action, LV_STATE_DISABLED); 
                lv_obj_set_style_bg_color(btn_action, lv_color_hex(COLOR_GREEN), 0);
            }

            lv_label_set_text(lv_obj_get_child(btn_nozzle, 0), "Wait Nozzle");
            lv_obj_set_style_bg_color(btn_nozzle, lv_color_hex(COLOR_BLUE), 0);
            lv_label_set_text(lv_obj_get_child(btn_action, 0), "START");
            break;

        case PUMP_FUELING:
            lv_label_set_text(lbl_status, "STATUS: FUELING...");
            lv_obj_set_style_text_color(lbl_status, lv_color_hex(COLOR_CYAN), 0);
            
            lv_obj_add_state(btn_nozzle, LV_STATE_DISABLED);

            lv_obj_set_style_bg_color(btn_action, lv_color_hex(COLOR_RED), 0);
            lv_label_set_text(lv_obj_get_child(btn_action, 0), "STOP");
            break;

        case PUMP_STOPPED:
            lv_label_set_text(lbl_status, "STATUS: STOPPED");
            lv_obj_set_style_text_color(lbl_status, lv_color_hex(COLOR_ORANGE), 0);
            
            lv_obj_clear_state(btn_nozzle, LV_STATE_DISABLED);
            lv_label_set_text(lv_obj_get_child(btn_nozzle, 0), "Down Nozzle");
            lv_obj_set_style_bg_color(btn_nozzle, lv_color_hex(COLOR_RED), 0);

            lv_obj_add_state(btn_action, LV_STATE_DISABLED);
            break;

        case PUMP_COMPLETED:
            break;
    }
    
    // แจ้งให้สร้าง Log จำลองในโหมด Standalone ทุกครั้งที่มีการขยับสถานะ
    logSimulatedTx();
}

void update_total_UI() {
    char buf_t_amt[20], buf_t_vol[20];
    snprintf(buf_t_amt, sizeof(buf_t_amt), "%u THB", cfg.totalAmount);
    snprintf(buf_t_vol, sizeof(buf_t_vol), "%u L", cfg.totalVolume);
    lv_label_set_text(lbl_tot_amount_val, buf_t_amt);
    lv_label_set_text(lbl_tot_volume_val, buf_t_vol);
}

// ==========================================
// 9. Display & Touch Callbacks
// ==========================================
void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
    uint32_t w = (area->x2 - area->x1 + 1);
    uint32_t h = (area->y2 - area->y1 + 1);
    tft.startWrite();
    tft.setAddrWindow(area->x1, area->y1, w, h);
    tft.pushColors((uint16_t *)&color_p->full, w * h, true);
    tft.endWrite();
    lv_disp_flush_ready(disp);
}

void my_touchpad_read(lv_indev_drv_t *indev_driver, lv_indev_data_t *data) {
    if (ts.touched()) {
        TS_Point p = ts.getPoint();
        if (p.z > 300) {
            data->state = LV_INDEV_STATE_PR;
            data->point.x = map(p.x, TOUCH_MIN_X, TOUCH_MAX_X, 0, 320);
            data->point.y = map(p.y, TOUCH_MIN_Y, TOUCH_MAX_Y, 0, 240);
        } else {
            data->state = LV_INDEV_STATE_REL;
        }
    } else {
        data->state = LV_INDEV_STATE_REL;
    }
}

// ==========================================
// 10. Setup & Loop
// ==========================================
void setup() {
    Serial.begin(115200);
    loadConfiguration();

    pinMode(TFT_BL, OUTPUT);
    analogWrite(TFT_BL, 230);
    pinMode(BUZZER_PIN, OUTPUT);
    digitalWrite(BUZZER_PIN, LOW);

    tft.begin();
    tft.invertDisplay(false);
    tft.setRotation(1);
    tft.fillScreen(TFT_BLACK);

    touchSPI.begin(XPT2046_CLK, XPT2046_MISO, XPT2046_MOSI, XPT2046_CS);
    ts.begin(touchSPI);
    ts.setRotation(1);

    lv_init();
    static lv_disp_draw_buf_t draw_buf;
    static lv_color_t buf[320 * 10]; 
    lv_disp_draw_buf_init(&draw_buf, buf, NULL, 320 * 10);

    static lv_disp_drv_t disp_drv;
    lv_disp_drv_init(&disp_drv);
    disp_drv.hor_res = 320;
    disp_drv.ver_res = 240;
    disp_drv.flush_cb = my_disp_flush;
    disp_drv.draw_buf = &draw_buf;
    lv_disp_drv_register(&disp_drv);

    static lv_indev_drv_t indev_drv;
    lv_indev_drv_init(&indev_drv);
    indev_drv.type = LV_INDEV_TYPE_POINTER;
    indev_drv.read_cb = my_touchpad_read;
    lv_indev_drv_register(&indev_drv);

    build_logo_screen();
    build_menu_screen();
    build_config_portal_screen();
    build_main_screen();
    build_total_screen();
    build_log_screen();

    lv_scr_load(screen_logo);
    current_screen = SCR_LOGO;
    screen_timer = millis();

    xTaskCreatePinnedToCore(rs485Task, "RS485_Task", 4096, NULL, 1, NULL, 0);
}

uint32_t last_tick = 0;
void loop() {
    uint32_t current_tick = millis();
    lv_tick_inc(current_tick - last_tick);
    last_tick = current_tick;

    unsigned long current_millis = millis();

    if (need_ui_update) {
        updateUI();
        need_ui_update = false;
    }
    
    processLogs();

    if (current_screen == SCR_LOGO) {
        if (current_millis - screen_timer >= 3000) { 
            lv_scr_load(screen_menu);
            current_screen = SCR_MENU;
            screen_timer = current_millis; 
        }
    } 
    else if (current_screen == SCR_MENU) {
        int elapsed = (current_millis - screen_timer) / 1000;
        int remaining = 5 - elapsed;
        if (remaining < 0) remaining = 0;

        char buf[32];
        snprintf(buf, sizeof(buf), "Auto start in %ds", remaining);
        lv_label_set_text(lbl_countdown, buf);

        if (current_millis - screen_timer >= 5000) { 
            lv_scr_load(screen_main);
            current_screen = SCR_MAIN;
            updateUI();
        }
    }
    else if (current_screen == SCR_CONFIG_PORTAL) {
        server.handleClient(); 
    }
    else if (current_screen == SCR_MAIN) {
        if (pump.state == PUMP_FUELING) {
            pump.volume += 5; 
            pump.amount = (uint64_t)pump.volume * cfg.unitPrice / 1000;
            updateUI();
        }
    }
    else if (current_screen == SCR_TOTAL) {
        if (current_millis - total_screen_timer >= 10000) {
            lv_scr_load_anim(screen_main, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            current_screen = SCR_MAIN;
            updateUI();
        }
    }

    lv_timer_handler(); 
    delay(50);
}
