#include <Arduino.h>
#include <FS.h>
#include <Preferences.h>
#include <lvgl.h>
#include <TFT_eSPI.h>
#include <SPI.h>
#include <XPT2046_Touchscreen.h>
#include <WiFi.h>
#include <WebServer.h>
#include <WiFiManager.h>
#include <time.h>

// ประกาศใช้งานฟอนต์ภาษาไทยที่สร้างมาจาก LVGL Font Converter
LV_FONT_DECLARE(font_thai_14);

// ==========================================
// 1. ฮาร์ดแวร์และการตั้งค่าพิน
// ==========================================
#define TFT_BL 21
#define XPT2046_IRQ 36  
#define XPT2046_MOSI 32
#define XPT2046_MISO 39
#define XPT2046_CLK 25
#define XPT2046_CS 33

#define RS485_RX_PIN 27
#define RS485_TX_PIN 22

#define TOUCH_MIN_X 280
#define TOUCH_MAX_X 3800
#define TOUCH_MIN_Y 310
#define TOUCH_MAX_Y 3700

XPT2046_Touchscreen ts(XPT2046_CS, XPT2046_IRQ); 
SPIClass touchSPI(HSPI);
TFT_eSPI tft = TFT_eSPI();

WebServer server(80);
Preferences preferences;

// ==========================================
// 2. ธีมสีสไตล์ Forecourt POS
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
#define COLOR_GRAY      0x6B7280  

// ==========================================
// 3. โครงสร้างข้อมูลตู้จ่าย
// ==========================================
#define STX 0x02
#define ETX 0x03
#define EOT 0x04
#define ENQ 0x05
#define NAK 0x15
#define DLE 0x10

struct PumpSlotData {
    bool isOnline = false;
    char rawStatus = '0';         
    char lastRawStatus = '0';
    bool autoAuthSent = false;    

    float liveVolume = 0.0;
    float liveAmount = 0.0;
    float livePrice  = 10.00;

    bool hasPendingBill = false;
    float billVolume = 0.0;
    float billAmount = 0.0;
    float billPrice  = 10.00;
    char timeInStr[16] = "00:00";
};

PumpSlotData pumps[4];       
int selected_pump_idx = 0;   

struct SaleRecord {
    int billNo;
    int pumpId;
    float volume;
    float amount;
    float price;
    char timePaidStr[16];
};

#define MAX_HISTORY 50
SaleRecord historyList[MAX_HISTORY];
int historyCount = 0;
int totalBillCounter = 0;

struct PumpSummary {
    int totalBills = 0;
    float totalVolume = 0.0;
    float totalAmount = 0.0;
};
PumpSummary pumpSummaries[4];

bool need_ui_update = false;
int active_screen = 1;          
unsigned long last_touch_time = 0;
const unsigned long AUTO_RETURN_INTERVAL = 30000;

// LVGL Widgets
lv_obj_t * scr_logo;
lv_obj_t * scr_config;
lv_obj_t * scr_ip_display;
lv_obj_t * scr_pos;
lv_obj_t * scr_pending;
lv_obj_t * scr_history;
lv_obj_t * scr_summary;
lv_obj_t * scr_meters;
lv_obj_t * scr_info;

lv_obj_t * lbl_config_countdown;
lv_obj_t * lbl_ip_address;
lv_obj_t * lbl_network_status;
lv_obj_t * lbl_top_clock;
lv_obj_t * card_pump[4];
lv_obj_t * lbl_card_state[4];
lv_obj_t * lbl_card_amt[4];

lv_obj_t * panel_checkout;
lv_obj_t * lbl_sel_title;
lv_obj_t * lbl_sel_amt;
lv_obj_t * lbl_sel_vol;
lv_obj_t * lbl_sel_price;
lv_obj_t * btn_pay;
lv_obj_t * lbl_btn_pay;

lv_obj_t * ta_pending;
lv_obj_t * ta_history;
lv_obj_t * ta_summary;
lv_obj_t * ta_meters;

TaskHandle_t rs485TaskHandle = NULL;

// Forward Declarations
void updateAllUI();
void updatePendingUI();
void updateHistoryUI();
void updateSummaryUI();
void updateMetersUI();
void savePersistentData();
void loadPersistentData();

void getCurrentTimeString(char* buf, size_t maxLen) {
    struct tm timeinfo;
    if(WiFi.status() == WL_CONNECTED && getLocalTime(&timeinfo)){
        strftime(buf, maxLen, "%H:%M", &timeinfo);
    } else {
        snprintf(buf, maxLen, "00:00");
    }
}

// ==========================================
// 4. บันทึกและโหลดข้อมูลกันหาย (Flash Storage)
// ==========================================
void savePersistentData() {
    preferences.begin("pos_store", false);
    preferences.putInt("tot_bills", totalBillCounter);
    preferences.putInt("hist_cnt", historyCount);
    
    for(int i=0; i<4; i++) {
        char key_b[16], key_v[16], key_a[16];
        snprintf(key_b, sizeof(key_b), "p%d_b", i);
        snprintf(key_v, sizeof(key_v), "p%d_v", i);
        snprintf(key_a, sizeof(key_a), "p%d_a", i);
        
        preferences.putInt(key_b, pumpSummaries[i].totalBills);
        preferences.putFloat(key_v, pumpSummaries[i].totalVolume);
        preferences.putFloat(key_a, pumpSummaries[i].totalAmount);
    }
    preferences.end();
}

void loadPersistentData() {
    preferences.begin("pos_store", true);
    totalBillCounter = preferences.getInt("tot_bills", 0);
    historyCount = preferences.getInt("hist_cnt", 0);
    
    for(int i=0; i<4; i++) {
        char key_b[16], key_v[16], key_a[16];
        snprintf(key_b, sizeof(key_b), "p%d_b", i);
        snprintf(key_v, sizeof(key_v), "p%d_v", i);
        snprintf(key_a, sizeof(key_a), "p%d_a", i);
        
        pumpSummaries[i].totalBills = preferences.getInt(key_b, 0);
        pumpSummaries[i].totalVolume = preferences.getFloat(key_v, 0.0);
        pumpSummaries[i].totalAmount = preferences.getFloat(key_a, 0.0);
    }
    preferences.end();
}

// ==========================================
// 5. RS-485 Master Task
// ==========================================
void rs485Write(const uint8_t* buf, size_t len) {
    delayMicroseconds(2000); 
    Serial2.write(buf, len);
    Serial2.flush();
}

uint8_t calcBCC(const uint8_t* data, size_t len) {
    uint8_t bcc = 0;
    for (size_t i = 0; i < len; i++) bcc ^= data[i];
    return bcc;
}

bool waitSerial(uint32_t timeout_ms) {
    uint32_t t = millis();
    while (!Serial2.available()) {
        if (millis() - t > timeout_ms) return false;
        delay(1);
    }
    return true;
}

void clearRX() {
    while (Serial2.available()) Serial2.read();
}

bool sendAutoAuthorize(uint8_t stationAddr, float priceToAuth) {
    clearRX();
    uint8_t sel_req[] = {EOT, stationAddr, 0x41, ENQ};
    rs485Write(sel_req, 4);

    if (waitSerial(200) && Serial2.read() == DLE) {
        if (waitSerial(50) && Serial2.read() == 0x30) {
            char auth_text[45];
            uint32_t p_int = (uint32_t)(priceToAuth * 100.0 + 0.5);
            snprintf(auth_text, sizeof(auth_text), "11019990002%04lu0000000000000000000000000", (unsigned long)p_int);

            uint8_t frame[64];
            int idx = 0;
            frame[idx++] = STX;
            frame[idx++] = stationAddr;
            frame[idx++] = 0x41;
            memcpy(&frame[idx], auth_text, strlen(auth_text));
            idx += strlen(auth_text);
            frame[idx++] = ETX;
            frame[idx] = calcBCC(&frame[1], idx - 1);
            idx++;

            rs485Write(frame, idx);

            if (waitSerial(200) && Serial2.read() == DLE) {
                if (waitSerial(50) && Serial2.read() == 0x31) {
                    uint8_t eot_cmd = EOT;
                    rs485Write(&eot_cmd, 1);
                    return true;
                }
            }
        }
    }
    return false;
}

void parsePumpStatus61(int p_idx, const char* text) {
    if (strlen(text) < 23 || strncmp(text, "61", 2) != 0) return;

    char cond = text[2];

    char vol_str[7] = {0};
    strncpy(vol_str, &text[3], 6);
    float vol = atol(vol_str) / 100.0;

    char up_str[5] = {0};
    strncpy(up_str, &text[10], 4);
    float up = atol(up_str) / 100.0;

    char amt_str[7] = {0};
    strncpy(amt_str, &text[14], 6);
    float amt = atol(amt_str) / 10.0;

    pumps[p_idx].rawStatus = cond;
    if (up > 0) pumps[p_idx].livePrice = up;

    if (cond == '1') {
        pumps[p_idx].liveVolume = 0.0;
        pumps[p_idx].liveAmount = 0.0;
    }
    else if (cond == '3') {
        pumps[p_idx].liveVolume = vol;
        pumps[p_idx].liveAmount = amt;
        pumps[p_idx].autoAuthSent = true;
    }
    else if (cond == '4') {
        pumps[p_idx].liveVolume = vol;
        pumps[p_idx].liveAmount = amt;
        if (vol > 0.0 || amt > 0.0) {
            if (!pumps[p_idx].hasPendingBill) {
                pumps[p_idx].hasPendingBill = true;
                pumps[p_idx].billVolume = vol;
                pumps[p_idx].billAmount = amt;
                pumps[p_idx].billPrice  = pumps[p_idx].livePrice;
                getCurrentTimeString(pumps[p_idx].timeInStr, sizeof(pumps[p_idx].timeInStr));
            }
        }
        pumps[p_idx].autoAuthSent = false;
    }

    pumps[p_idx].lastRawStatus = cond;
    need_ui_update = true;
}

void rs485MasterTask(void * pvParameters) {
    Serial2.begin(19200, SERIAL_8E1, RS485_RX_PIN, RS485_TX_PIN);
    int current_polling_pump = 0;
    int offline_counters[4] = {0, 0, 0, 0};

    while (1) {
        uint8_t target_station = 0x41 + current_polling_pump; 

        clearRX();
        uint8_t poll_req[] = {EOT, target_station, 0x51, ENQ};
        rs485Write(poll_req, 4);

        bool got_reply = false;
        if (waitSerial(80)) {
            uint8_t c = Serial2.read();
            if (c == EOT) {
                got_reply = true;
                offline_counters[current_polling_pump] = 0;
                if (!pumps[current_polling_pump].isOnline || pumps[current_polling_pump].rawStatus != '0') {
                    if ((pumps[current_polling_pump].rawStatus == '3' || pumps[current_polling_pump].rawStatus == '4') && pumps[current_polling_pump].liveAmount > 0.0) {
                        if (!pumps[current_polling_pump].hasPendingBill) {
                            pumps[current_polling_pump].hasPendingBill = true;
                            pumps[current_polling_pump].billVolume = pumps[current_polling_pump].liveVolume;
                            pumps[current_polling_pump].billAmount = pumps[current_polling_pump].liveAmount;
                            pumps[current_polling_pump].billPrice  = pumps[current_polling_pump].livePrice;
                            getCurrentTimeString(pumps[current_polling_pump].timeInStr, sizeof(pumps[current_polling_pump].timeInStr));
                        }
                    }
                    pumps[current_polling_pump].isOnline = true;
                    pumps[current_polling_pump].rawStatus = '0';
                    pumps[current_polling_pump].autoAuthSent = false;
                    need_ui_update = true;
                }
            } 
            else if (c == STX) {
                if (waitSerial(40)) Serial2.read(); 
                if (waitSerial(40)) Serial2.read(); 

                char rxBuf[64];
                int idx = 0;
                while (waitSerial(40)) {
                    uint8_t d = Serial2.read();
                    if (d == ETX) {
                        if (waitSerial(40)) Serial2.read(); 
                        rxBuf[idx] = '\0';
                        break;
                    }
                    if (idx < 63) rxBuf[idx++] = d;
                }

                uint8_t ack1[] = {DLE, 0x31};
                rs485Write(ack1, 2);
                if (waitSerial(60)) Serial2.read(); 

                got_reply = true;
                offline_counters[current_polling_pump] = 0;
                pumps[current_polling_pump].isOnline = true;
                parsePumpStatus61(current_polling_pump, rxBuf);
            }
        }

        if (!got_reply) {
            offline_counters[current_polling_pump]++;
            if (offline_counters[current_polling_pump] > 5 && pumps[current_polling_pump].isOnline) {
                pumps[current_polling_pump].isOnline = false;
                need_ui_update = true;
            }
        }

        if (pumps[current_polling_pump].isOnline && pumps[current_polling_pump].rawStatus == '1' && !pumps[current_polling_pump].autoAuthSent) {
            vTaskDelay(20 / portTICK_PERIOD_MS);
            if (sendAutoAuthorize(target_station, pumps[current_polling_pump].livePrice)) {
                pumps[current_polling_pump].autoAuthSent = true;
                need_ui_update = true;
            }
        }

        current_polling_pump = (current_polling_pump + 1) % 4;
        vTaskDelay(50 / portTICK_PERIOD_MS);
    }
}

// ==========================================
// 6. ระบบ Gesture สำหรับสลับหน้าจอ
// ==========================================
static void screen_gesture_event_cb(lv_event_t * e) {
    lv_dir_t dir = lv_indev_get_gesture_dir(lv_indev_get_act());
    
    if (dir == LV_DIR_LEFT) {
        if (active_screen == 1) {
            updatePendingUI();
            lv_scr_load_anim(scr_pending, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            active_screen = 2;
        } else if (active_screen == 2) {
            updateHistoryUI();
            lv_scr_load_anim(scr_history, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            active_screen = 3;
        } else if (active_screen == 3) {
            updateSummaryUI();
            lv_scr_load_anim(scr_summary, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            active_screen = 4;
        } else if (active_screen == 4) {
            updateMetersUI();
            lv_scr_load_anim(scr_meters, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            active_screen = 5;
        } else if (active_screen == 5) {
            if(WiFi.status() == WL_CONNECTED) {
                lv_label_set_text_fmt(lbl_network_status, "Connected Wi-Fi\nIP: %s", WiFi.localIP().toString().c_str());
            } else {
                lv_label_set_text(lbl_network_status, "Not Connect wifi");
            }
            lv_scr_load_anim(scr_info, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false);
            active_screen = 6;
        }
    } 
    else if (dir == LV_DIR_RIGHT) {
        if (active_screen == 6) {
            lv_scr_load_anim(scr_meters, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            active_screen = 5;
        } else if (active_screen == 5) {
            lv_scr_load_anim(scr_summary, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            active_screen = 4;
        } else if (active_screen == 4) {
            lv_scr_load_anim(scr_history, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            active_screen = 3;
        } else if (active_screen == 3) {
            lv_scr_load_anim(scr_pending, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            active_screen = 2;
        } else if (active_screen == 2) {
            lv_scr_load_anim(scr_pos, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false);
            active_screen = 1;
        }
    }
}

// ==========================================
// 7. การสร้างหน้าจอ UI ทั้งหมด
// ==========================================
static int pump_indices[4] = {0, 1, 2, 3};
bool config_interrupted = false;

static void btn_config_click_cb(lv_event_t * e) {
    config_interrupted = true;
}

void createScreens() {
    scr_logo = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_logo, lv_color_hex(COLOR_BG), 0);
    lv_obj_clear_flag(scr_logo, LV_OBJ_FLAG_SCROLLABLE);

    lv_obj_t * lbl_logo_title = lv_label_create(scr_logo);
    lv_label_set_text(lbl_logo_title, "POS DISPENSER");
    lv_obj_set_style_text_font(lbl_logo_title, &lv_font_montserrat_20, 0);
    lv_obj_set_style_text_color(lbl_logo_title, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_logo_title, LV_ALIGN_CENTER, 0, -15);

    lv_obj_t * lbl_logo_sub = lv_label_create(scr_logo);
    lv_label_set_text(lbl_logo_sub, "Forecourt Automation System");
    lv_obj_set_style_text_font(lbl_logo_sub, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_logo_sub, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(lbl_logo_sub, LV_ALIGN_CENTER, 0, 15);

    scr_config = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_config, lv_color_hex(COLOR_BG), 0);
    lv_obj_clear_flag(scr_config, LV_OBJ_FLAG_SCROLLABLE);

    lv_obj_t * lbl_cfg_title = lv_label_create(scr_config);
    lv_label_set_text(lbl_cfg_title, "SYSTEM CONFIG MODE");
    lv_obj_set_style_text_font(lbl_cfg_title, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_cfg_title, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_cfg_title, LV_ALIGN_TOP_MID, 0, 15);

    lbl_config_countdown = lv_label_create(scr_config);
    lv_label_set_text(lbl_config_countdown, "Starting POS in 3s...");
    lv_obj_set_style_text_font(lbl_config_countdown, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_config_countdown, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_align(lbl_config_countdown, LV_ALIGN_TOP_MID, 0, 45);

    lv_obj_t * btn_cfg = lv_btn_create(scr_config);
    lv_obj_set_size(btn_cfg, 240, 50);
    lv_obj_align(btn_cfg, LV_ALIGN_CENTER, 0, 10);
    lv_obj_set_style_bg_color(btn_cfg, lv_color_hex(COLOR_BLUE), 0);
    lv_obj_add_event_cb(btn_cfg, btn_config_click_cb, LV_EVENT_CLICKED, NULL);

    lv_obj_t * lbl_btn_cfg = lv_label_create(btn_cfg);
    lv_label_set_text(lbl_btn_cfg, "CONFIG / WI-FI SETUP");
    lv_obj_set_style_text_font(lbl_btn_cfg, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_btn_cfg, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_center(lbl_btn_cfg);

    scr_ip_display = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_ip_display, lv_color_hex(COLOR_BG), 0);
    lv_obj_clear_flag(scr_ip_display, LV_OBJ_FLAG_SCROLLABLE);

    lv_obj_t * lbl_ip_title = lv_label_create(scr_ip_display);
    lv_label_set_text(lbl_ip_title, "WI-FI SETUP MODE (OPEN AP)");
    lv_obj_set_style_text_font(lbl_ip_title, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_ip_title, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_align(lbl_ip_title, LV_ALIGN_TOP_MID, 0, 20);

    lbl_ip_address = lv_label_create(scr_ip_display);
    lv_label_set_text(lbl_ip_address, "SSID: POS_Config_AP\nIP: 192.168.4.1\n\nConnect phone to configure Wi-Fi");
    lv_obj_set_style_text_font(lbl_ip_address, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_ip_address, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_align(lbl_ip_address, LV_ALIGN_CENTER, 0, 0);

    scr_info = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_info, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_info, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);

    lv_obj_t * lbl_inf_title = lv_label_create(scr_info);
    lv_label_set_text(lbl_inf_title, "NETWORK STATUS [Swipe Right]");
    lv_obj_set_style_text_font(lbl_inf_title, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_inf_title, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_inf_title, LV_ALIGN_TOP_LEFT, 6, 6);

    lbl_network_status = lv_label_create(scr_info);
    lv_label_set_text(lbl_network_status, "Not Connect wifi");
    lv_obj_set_style_text_font(lbl_network_status, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_network_status, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_network_status, LV_ALIGN_CENTER, 0, 0);

    // หน้าที่ 1: Monitor หลัก
    scr_pos = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_pos, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_pos, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);

    lv_obj_t * hdr1 = lv_label_create(scr_pos);
    lv_label_set_text(hdr1, "1:MONITOR");
    lv_obj_set_style_text_font(hdr1, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(hdr1, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(hdr1, LV_ALIGN_TOP_LEFT, 6, 6);

    lbl_top_clock = lv_label_create(scr_pos);
    lv_label_set_text(lbl_top_clock, "00:00");
    lv_obj_set_style_text_font(lbl_top_clock, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_top_clock, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_top_clock, LV_ALIGN_TOP_RIGHT, -6, 6);

    lv_obj_t * lbl_watermark = lv_label_create(scr_pos);
    lv_label_set_text(lbl_watermark, "Dx Solution HS3PIK");
    lv_obj_set_style_text_font(lbl_watermark, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_watermark, lv_color_hex(0x374151), 0);
    lv_obj_align(lbl_watermark, LV_ALIGN_BOTTOM_RIGHT, -4, -2);

    for (int i = 0; i < 4; i++) {
        card_pump[i] = lv_obj_create(scr_pos);
        lv_obj_set_size(card_pump[i], 74, 66);
        lv_obj_set_pos(card_pump[i], 4 + (i * 79), 25);
        lv_obj_set_style_radius(card_pump[i], 4, 0);
        lv_obj_set_style_pad_all(card_pump[i], 3, 0);
        lv_obj_clear_flag(card_pump[i], LV_OBJ_FLAG_SCROLLABLE);
        lv_obj_add_flag(card_pump[i], LV_OBJ_FLAG_CLICKABLE);

        lv_obj_t * lbl_id = lv_label_create(card_pump[i]);
        lv_label_set_text_fmt(lbl_id, "0%d", i + 1);
        lv_obj_set_style_text_font(lbl_id, &lv_font_montserrat_12, 0);
        lv_obj_align(lbl_id, LV_ALIGN_TOP_LEFT, 2, 0);

        lbl_card_state[i] = lv_label_create(card_pump[i]);
        lv_label_set_text(lbl_card_state[i], "OFFLINE");
        lv_obj_set_style_text_font(lbl_card_state[i], &lv_font_montserrat_12, 0);
        lv_obj_align(lbl_card_state[i], LV_ALIGN_CENTER, 0, -2);

        lbl_card_amt[i] = lv_label_create(card_pump[i]);
        lv_label_set_text(lbl_card_amt[i], "---");
        lv_obj_set_style_text_font(lbl_card_amt[i], &lv_font_montserrat_12, 0);
        lv_obj_align(lbl_card_amt[i], LV_ALIGN_BOTTOM_RIGHT, -2, 0);
    }

    panel_checkout = lv_obj_create(scr_pos);
    lv_obj_set_size(panel_checkout, 312, 142);
    lv_obj_align(panel_checkout, LV_ALIGN_BOTTOM_MID, 0, -4);
    lv_obj_set_style_bg_color(panel_checkout, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_border_color(panel_checkout, lv_color_hex(COLOR_BLUE), 0);
    lv_obj_set_style_border_width(panel_checkout, 2, 0);
    lv_obj_set_style_radius(panel_checkout, 6, 0);
    lv_obj_set_style_pad_all(panel_checkout, 8, 0);
    lv_obj_clear_flag(panel_checkout, LV_OBJ_FLAG_SCROLLABLE);

    lbl_sel_title = lv_label_create(panel_checkout);
    lv_label_set_text(lbl_sel_title, "PUMP [01] - LIVE MONITOR");
    lv_obj_set_style_text_font(lbl_sel_title, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_sel_title, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(lbl_sel_title, LV_ALIGN_TOP_LEFT, 0, 0);

    lbl_sel_amt = lv_label_create(panel_checkout);
    lv_label_set_text(lbl_sel_amt, "AMOUNT: 0.0 THB");
    lv_obj_set_style_text_font(lbl_sel_amt, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_sel_amt, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_align(lbl_sel_amt, LV_ALIGN_TOP_LEFT, 0, 22);

    lbl_sel_vol = lv_label_create(panel_checkout);
    lv_label_set_text(lbl_sel_vol, "VOLUME: 0.00 L");
    lv_obj_set_style_text_font(lbl_sel_vol, &lv_font_montserrat_14, 0);
    lv_obj_set_style_text_color(lbl_sel_vol, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(lbl_sel_vol, LV_ALIGN_TOP_LEFT, 0, 46);

    lbl_sel_price = lv_label_create(panel_checkout);
    lv_label_set_text(lbl_sel_price, "PRICE: 10.00 THB/L");
    lv_obj_set_style_text_font(lbl_sel_price, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(lbl_sel_price, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_align(lbl_sel_price, LV_ALIGN_TOP_LEFT, 0, 70);

    btn_pay = lv_btn_create(panel_checkout);
    lv_obj_set_size(btn_pay, 290, 36);
    lv_obj_align(btn_pay, LV_ALIGN_BOTTOM_MID, 0, 0);
    lv_obj_set_style_bg_color(btn_pay, lv_color_hex(COLOR_GRAY), 0);
    lv_obj_add_state(btn_pay, LV_STATE_DISABLED);

    lbl_btn_pay = lv_label_create(btn_pay);
    lv_label_set_text(lbl_btn_pay, "NO PENDING BILL");
    // ==========================================
    // ** เปลี่ยนฟอนต์ตรงนี้ให้เป็น font_thai_14 **
    // ==========================================
    lv_obj_set_style_text_font(lbl_btn_pay, &font_thai_14, 0);
    lv_obj_set_style_text_color(lbl_btn_pay, lv_color_hex(COLOR_WHITE), 0);
    lv_obj_center(lbl_btn_pay);

    // หน้าที่ 2: ค้างชำระ
    scr_pending = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_pending, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_pending, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
    lv_obj_t * h2 = lv_label_create(scr_pending);
    lv_label_set_text(h2, "2: PENDING BILLS");
    lv_obj_set_style_text_font(h2, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(h2, lv_color_hex(COLOR_YELLOW), 0);
    lv_obj_align(h2, LV_ALIGN_TOP_LEFT, 8, 8);
    ta_pending = lv_textarea_create(scr_pending);
    lv_obj_set_size(ta_pending, 308, 185);
    lv_obj_align(ta_pending, LV_ALIGN_BOTTOM_MID, 0, -5);
    lv_obj_set_style_bg_color(ta_pending, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_text_color(ta_pending, lv_color_hex(COLOR_WHITE), 0);
    lv_textarea_set_text(ta_pending, "No pending bills.\n");

    // หน้าที่ 3: ชำระแล้ว
    scr_history = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_history, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_history, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
    lv_obj_t * h3 = lv_label_create(scr_history);
    lv_label_set_text(h3, "3: COMPLETED SALES");
    lv_obj_set_style_text_font(h3, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(h3, lv_color_hex(COLOR_GREEN), 0);
    lv_obj_align(h3, LV_ALIGN_TOP_LEFT, 8, 8);
    ta_history = lv_textarea_create(scr_history);
    lv_obj_set_size(ta_history, 308, 185);
    lv_obj_align(ta_history, LV_ALIGN_BOTTOM_MID, 0, -5);
    lv_obj_set_style_bg_color(ta_history, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_text_color(ta_history, lv_color_hex(COLOR_WHITE), 0);
    lv_textarea_set_text(ta_history, "No completed transactions.\n");

    // หน้าที่ 4: สรุปยอดขาย
    scr_summary = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_summary, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_summary, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
    lv_obj_t * h4 = lv_label_create(scr_summary);
    lv_label_set_text(h4, "4: SALES SUMMARY");
    lv_obj_set_style_text_font(h4, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(h4, lv_color_hex(COLOR_ORANGE), 0);
    lv_obj_align(h4, LV_ALIGN_TOP_LEFT, 8, 8);
    ta_summary = lv_textarea_create(scr_summary);
    lv_obj_set_size(ta_summary, 308, 185);
    lv_obj_align(ta_summary, LV_ALIGN_BOTTOM_MID, 0, -5);
    lv_obj_set_style_bg_color(ta_summary, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_text_color(ta_summary, lv_color_hex(COLOR_WHITE), 0);

    // หน้าที่ 5: มิเตอร์สะสม
    scr_meters = lv_obj_create(NULL);
    lv_obj_set_style_bg_color(scr_meters, lv_color_hex(COLOR_BG), 0);
    lv_obj_add_event_cb(scr_meters, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
    lv_obj_t * h5 = lv_label_create(scr_meters);
    lv_label_set_text(h5, "5: METER TOTALS");
    lv_obj_set_style_text_font(h5, &lv_font_montserrat_12, 0);
    lv_obj_set_style_text_color(h5, lv_color_hex(COLOR_CYAN), 0);
    lv_obj_align(h5, LV_ALIGN_TOP_LEFT, 8, 8);
    ta_meters = lv_textarea_create(scr_meters);
    lv_obj_set_size(ta_meters, 308, 185);
    lv_obj_align(ta_meters, LV_ALIGN_BOTTOM_MID, 0, -5);
    lv_obj_set_style_bg_color(ta_meters, lv_color_hex(COLOR_PANEL), 0);
    lv_obj_set_style_text_color(ta_meters, lv_color_hex(COLOR_WHITE), 0);
}

static void card_pump_click_cb(lv_event_t * e) {
    int *pump_idx = (int *)lv_event_get_user_data(e);
    if (pump_idx) {
        selected_pump_idx = *pump_idx;
        updateAllUI();
    }
}

static void btn_pay_click_cb(lv_event_t * e) {
    if (!pumps[selected_pump_idx].hasPendingBill) return;

    totalBillCounter++;
    char paidTime[16];
    getCurrentTimeString(paidTime, sizeof(paidTime));

    if (historyCount < MAX_HISTORY) {
        historyList[historyCount] = {
            totalBillCounter, 
            selected_pump_idx + 1, 
            pumps[selected_pump_idx].billVolume, 
            pumps[selected_pump_idx].billAmount, 
            pumps[selected_pump_idx].billPrice,
            ""
        };
        snprintf(historyList[historyCount].timePaidStr, sizeof(historyList[historyCount].timePaidStr), "%s", paidTime);
        historyCount++;
    }

    pumpSummaries[selected_pump_idx].totalBills++;
    pumpSummaries[selected_pump_idx].totalVolume += pumps[selected_pump_idx].billVolume;
    pumpSummaries[selected_pump_idx].totalAmount += pumps[selected_pump_idx].billAmount;

    pumps[selected_pump_idx].hasPendingBill = false;
    pumps[selected_pump_idx].billAmount = 0.0;
    pumps[selected_pump_idx].billVolume = 0.0;
    pumps[selected_pump_idx].liveAmount = 0.0;
    pumps[selected_pump_idx].liveVolume = 0.0;

    savePersistentData();
    updateAllUI();
}

void updateAllUI() {
    char clockBuf[16];
    getCurrentTimeString(clockBuf, sizeof(clockBuf));
    if(lbl_top_clock) {
        lv_label_set_text(lbl_top_clock, clockBuf);
    }

    for (int i = 0; i < 4; i++) {
        if (!pumps[i].isOnline) {
            lv_obj_set_style_bg_color(card_pump[i], lv_color_hex(COLOR_CARD), 0);
            lv_label_set_text(lbl_card_state[i], "OFFLINE");
            lv_obj_set_style_text_color(lbl_card_state[i], lv_color_hex(COLOR_RED), 0);
            lv_label_set_text(lbl_card_amt[i], "---");
        } 
        else if (pumps[i].hasPendingBill) {
            lv_obj_set_style_bg_color(card_pump[i], lv_color_hex(COLOR_GREEN), 0);
            lv_label_set_text(lbl_card_state[i], "PAY NOW");
            lv_obj_set_style_text_color(lbl_card_state[i], lv_color_hex(0x000000), 0);

            char b_amt[16];
            snprintf(b_amt, sizeof(b_amt), "%.1f B", pumps[i].billAmount);
            lv_label_set_text(lbl_card_amt[i], b_amt);
            lv_obj_set_style_text_color(lbl_card_amt[i], lv_color_hex(0x000000), 0);
        } 
        else if (pumps[i].rawStatus == '3') {
            lv_obj_set_style_bg_color(card_pump[i], lv_color_hex(COLOR_BLUE), 0);
            lv_label_set_text(lbl_card_state[i], "FUELING");
            lv_obj_set_style_text_color(lbl_card_state[i], lv_color_hex(COLOR_WHITE), 0);

            char l_amt[16];
            snprintf(l_amt, sizeof(l_amt), "%.1f B", pumps[i].liveAmount);
            lv_label_set_text(lbl_card_amt[i], l_amt);
            lv_obj_set_style_text_color(lbl_card_amt[i], lv_color_hex(COLOR_WHITE), 0);
        } 
        else if (pumps[i].rawStatus == '1') {
            lv_obj_set_style_bg_color(card_pump[i], lv_color_hex(COLOR_ORANGE), 0);
            lv_label_set_text(lbl_card_state[i], "AUTH OK");
            lv_obj_set_style_text_color(lbl_card_state[i], lv_color_hex(COLOR_WHITE), 0);
            lv_label_set_text(lbl_card_amt[i], "0.0 B");
        } 
        else {
            lv_obj_set_style_bg_color(card_pump[i], lv_color_hex(COLOR_CARD), 0);
            lv_label_set_text(lbl_card_state[i], "READY");
            lv_obj_set_style_text_color(lbl_card_state[i], lv_color_hex(COLOR_CYAN), 0);
            lv_label_set_text(lbl_card_amt[i], "0.0 B");
            lv_obj_set_style_text_color(lbl_card_amt[i], lv_color_hex(COLOR_WHITE), 0);
        }

        if (i == selected_pump_idx) {
            lv_obj_set_style_border_color(card_pump[i], lv_color_hex(COLOR_YELLOW), 0);
            lv_obj_set_style_border_width(card_pump[i], 3, 0);
        } else {
            lv_obj_set_style_border_color(card_pump[i], lv_color_hex(COLOR_GRAY), 0);
            lv_obj_set_style_border_width(card_pump[i], 1, 0);
        }
    }

    PumpSlotData &sel_pump = pumps[selected_pump_idx];
    char title_buf[32];
    snprintf(title_buf, sizeof(title_buf), "PUMP [0%d] - %s", selected_pump_idx + 1, sel_pump.hasPendingBill ? "WAITING PAYMENT" : "LIVE MONITOR");
    lv_label_set_text(lbl_sel_title, title_buf);
    lv_obj_set_style_text_color(lbl_sel_title, lv_color_hex(sel_pump.hasPendingBill ? COLOR_YELLOW : COLOR_CYAN), 0);

    float showAmt   = sel_pump.hasPendingBill ? sel_pump.billAmount : sel_pump.liveAmount;
    float showVol   = sel_pump.hasPendingBill ? sel_pump.billVolume : sel_pump.liveVolume;
    float showPrice = sel_pump.hasPendingBill ? sel_pump.billPrice  : sel_pump.livePrice;

    char s_amt[32], s_vol[32], s_prc[32];
    snprintf(s_amt, sizeof(s_amt), "AMOUNT: %.1f THB", showAmt);
    snprintf(s_vol, sizeof(s_vol), "VOLUME: %.2f L", showVol);
    snprintf(s_prc, sizeof(s_prc), "PRICE: %.2f THB/L", showPrice);

    lv_label_set_text(lbl_sel_amt, s_amt);
    lv_label_set_text(lbl_sel_vol, s_vol);
    lv_label_set_text(lbl_sel_price, s_prc);

    if (sel_pump.hasPendingBill) {
        lv_obj_clear_state(btn_pay, LV_STATE_DISABLED);
        lv_obj_set_style_bg_color(btn_pay, lv_color_hex(COLOR_GREEN), 0);
        lv_label_set_text(lbl_btn_pay, "RECEIVE PAYMENT (รับชำระ)");
    } else {
        lv_obj_add_state(btn_pay, LV_STATE_DISABLED);
        lv_obj_set_style_bg_color(btn_pay, lv_color_hex(COLOR_GRAY), 0);
        lv_label_set_text(lbl_btn_pay, sel_pump.isOnline ? "NO PENDING BILL" : "PUMP OFFLINE");
    }

    updatePendingUI();
    updateHistoryUI();
    updateSummaryUI();
    updateMetersUI();
}

void updatePendingUI() {
    String txt = "";
    bool hasAny = false;
    for (int i = 0; i < 4; i++) {
        if (pumps[i].hasPendingBill) {
            hasAny = true;
            char line[64];
            snprintf(line, sizeof(line), "P0%d | Time:%s | %.2fL | %.1fB\n", 
                     i + 1, pumps[i].timeInStr, pumps[i].billVolume, pumps[i].billAmount);
            txt += String(line);
        }
    }
    if (!hasAny) txt = "No pending bills waiting.\n";
    lv_textarea_set_text(ta_pending, txt.c_str());
}

void updateHistoryUI() {
    if (historyCount == 0) {
        lv_textarea_set_text(ta_history, "No completed transactions.\n");
        return;
    }
    String txt = "";
    for (int i = historyCount - 1; i >= 0; i--) {
        char line[80];
        snprintf(line, sizeof(line), "#%02d|P0%d|%.2fL|%.1fB|%s\n",
                 historyList[i].billNo, historyList[i].pumpId,
                 historyList[i].volume, historyList[i].amount, historyList[i].timePaidStr);
        txt += String(line);
    }
    lv_textarea_set_text(ta_history, txt.c_str());
}

void updateSummaryUI() {
    String txt = "Pump | Bills | Vol (L) | Amount (B)\n";
    txt += "---------------------------------\n";
    for (int i = 0; i < 4; i++) {
        char line[64];
        snprintf(line, sizeof(line), "P0%d  |  %2d   | %6.1f  | %7.1f\n",
                 i + 1, pumpSummaries[i].totalBills,
                 pumpSummaries[i].totalVolume, pumpSummaries[i].totalAmount);
        txt += String(line);
    }
    lv_textarea_set_text(ta_summary, txt.c_str());
}

void updateMetersUI() {
    String txt = "Pump | Total Volume | Total Amount\n";
    txt += "------------------------------------\n";
    for (int i = 0; i < 4; i++) {
        char line[64];
        snprintf(line, sizeof(line), "P0%d  |  %8.2f L  |  %9.1f B\n",
                 i + 1, pumpSummaries[i].totalVolume, pumpSummaries[i].totalAmount);
        txt += String(line);
    }
    lv_textarea_set_text(ta_meters, txt.c_str());
}

void setupWebServer() {
    server.on("/", HTTP_GET, []() {
        String html = "<html><head><meta charset='UTF-8'><title>POS Report</title></head><body style='font-family:sans-serif;'>";
        html += "<h2>Forecourt POS - Report Dashboard</h2>";
        html += "<h3>Meter Totals (ยอดมิเตอร์สะสมล่าสุด)</h3><table border='1' cellpadding='8'><tr><th>Pump</th><th>Total Volume (L)</th><th>Total Amount (THB)</th></tr>";
        for(int i=0; i<4; i++) {
            html += "<tr><td>Pump 0" + String(i+1) + "</td><td>" + String(pumpSummaries[i].totalVolume, 2) + "</td><td>" + String(pumpSummaries[i].totalAmount, 1) + "</td></tr>";
        }
        html += "</table></body></html>";
        server.send(200, "text/html", html);
    });
    server.begin();
}

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
            last_touch_time = millis();
        } else {
            data->state = LV_INDEV_STATE_REL;
        }
    } else {
        data->state = LV_INDEV_STATE_REL;
    }
}

void setup() {
    Serial.begin(115200);

    pinMode(TFT_BL, OUTPUT);
    analogWrite(TFT_BL, 230);

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

    createScreens();
    loadPersistentData();

    // 1. แสดงโลโก้ 3 วินาที
    lv_scr_load(scr_logo);
    lv_timer_handler();
    delay(3000);

    // 2. เข้าหน้าจอ Config นับถอยหลัง 3 วินาที
    lv_scr_load(scr_config);
    config_interrupted = false;
    
    for (int i = 3; i > 0; i--) {
        char buf_cd[32];
        snprintf(buf_cd, sizeof(buf_cd), "Starting POS in %ds...", i);
        lv_label_set_text(lbl_config_countdown, buf_cd);
        
        uint32_t t_start = millis();
        while (millis() - t_start < 1000) {
            lv_timer_handler();
            lv_tick_inc(10);
            if (config_interrupted) break;
            delay(10);
        }
        if (config_interrupted) break;
    }

    // 3. ถ้าผู้ใช้กดปุ่ม Config
    if (config_interrupted) {
        lv_scr_load(scr_ip_display);
        lv_timer_handler();

        WiFiManager wm;
        wm.setConfigPortalTimeout(180);
        wm.setAPStaticIPConfig(IPAddress(192,168,4,1), IPAddress(192,168,4,1), IPAddress(255,255,255,0));
        
        if (!wm.startConfigPortal("POS_Config_AP")) {
            delay(2000);
            ESP.restart();
        }
        ESP.restart();
    }

    // 4. เข้าหน้าจอหลัก POS ปกติ (เพิ่ม Timeout 20 วินาที ป้องกันค้างเวลารอบูตไม่มีเน็ต)
    WiFiManager wm;
    wm.setConnectTimeout(20);
    wm.autoConnect("POS_Config_AP");
    
    lv_scr_load(scr_pos);
    for (int i = 0; i < 4; i++) {
        lv_obj_add_event_cb(card_pump[i], card_pump_click_cb, LV_EVENT_CLICKED, &pump_indices[i]);
    }
    lv_obj_add_event_cb(btn_pay, btn_pay_click_cb, LV_EVENT_CLICKED, NULL);

    updateAllUI();

    configTime(25200, 0, "pool.ntp.org");
    setupWebServer();

    xTaskCreatePinnedToCore(rs485MasterTask, "POS_Master", 4096, NULL, 1, &rs485TaskHandle, 0);
    last_touch_time = millis();
}

uint32_t last_tick = 0;
uint32_t last_clock_update = 0;
void loop() {
    uint32_t current_tick = millis();
    lv_tick_inc(current_tick - last_tick);
    last_tick = current_tick;

    if (need_ui_update) {
        updateAllUI();
        need_ui_update = false;
    }

    if (millis() - last_clock_update > 1000) {
        last_clock_update = millis();
        if (active_screen == 1) {
            char clockBuf[16];
            getCurrentTimeString(clockBuf, sizeof(clockBuf));
            lv_label_set_text(lbl_top_clock, clockBuf);
        }
    }

    if (active_screen != 1 && (millis() - last_touch_time > AUTO_RETURN_INTERVAL)) {
        lv_scr_load_anim(scr_pos, LV_SCR_LOAD_ANIM_FADE_IN, 200, 0, false);
        active_screen = 1;
    }

    server.handleClient();
    lv_timer_handler(); 
    delay(30);
}
