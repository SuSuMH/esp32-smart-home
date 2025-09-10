#include <WiFi.h>
#include <ArduinoWebSockets.h>
#include <ArduinoJson.h>
#include "DHT.h"

// --- CẤU HÌNH MẠNG VÀ SERVER ---
const char* ssid = "TEN_WIFI_CUA_BAN";
const char* password = "MAT_KHAU_WIFI";
// URL có thể thay đổi dễ dàng tại đây
const char* websocket_server_host = "your_server_ip_or_domain"; // Ví dụ: "192.168.1.100"
const uint16_t websocket_server_port = 80; // Cổng WebSocket của server
const char* websocket_api_path = "/api/ws"; // Đường dẫn API

// --- CẤU HÌNH THIẾT BỊ ---
const char* AUTH_TOKEN = "AUTH_TOKEN_34567";
#define LED_01_PIN 2 // Chân GPIO kết nối với led_01

// --- CẤU HÌNH CẢM BIẾN ---
// Cảm biến khí Gas MQ2
const int MQ2_ANALOG_PIN = 34;
const int MQ2_DIGITAL_PIN = 18;

// Cảm biến nhiệt độ, độ ẩm DHT22
#define DHTPIN 4
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// Khởi tạo đối tượng WebSocket client
WebSocketsClient webSocket;

// Biến quản lý thời gian gửi dữ liệu
unsigned long lastReportTime = 0;
const long reportInterval = 5000; // Gửi báo cáo mỗi 5 giây

// --- KHAI BÁO CÁC HÀM ---
void webSocketEvent(WStype_t type, uint8_t * payload, size_t length);
void sendAuthentication();
void sendReport();
void handleServerAction(JsonDocument& doc);

// =================================================================
// HÀM SETUP - KHỞI TẠO HỆ THỐNG
// =================================================================
void setup() {
  Serial.begin(115200);
  Serial.println("\n[INFO] Hệ thống giám sát môi trường khởi động!");

  // Khởi tạo các chân I/O
  pinMode(MQ2_DIGITAL_PIN, INPUT);
  pinMode(LED_01_PIN, OUTPUT);
  digitalWrite(LED_01_PIN, LOW);

  // Khởi tạo cảm biến DHT22
  dht.begin();

  // Kết nối WiFi
  Serial.printf("[WIFI] Đang kết nối tới %s ", ssid);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n[WIFI] Đã kết nối!");
  Serial.print("[WIFI] Địa chỉ IP: ");
  Serial.println(WiFi.localIP());

  // Kết nối tới WebSocket server
  webSocket.begin(websocket_server_host, websocket_server_port, websocket_api_path);
  
  // Gán hàm xử lý sự kiện cho WebSocket
  webSocket.onEvent(webSocketEvent);

  // Thiết lập thời gian timeout để thử kết nối lại
  webSocket.setReconnectInterval(5000); // Thử kết nối lại sau mỗi 5 giây nếu mất kết nối
}

// =================================================================
// HÀM LOOP - VÒNG LẶP CHÍNH
// =================================================================
void loop() {
  // Luôn gọi hàm này để client WebSocket hoạt động
  webSocket.loop();

  // Kiểm tra nếu đã đến lúc gửi báo cáo dữ liệu cảm biến
  if (millis() - lastReportTime > reportInterval) {
    if (webSocket.isConnected()) {
      sendReport();
    }
    lastReportTime = millis();
  }
}

// =================================================================
// HÀM XỬ LÝ SỰ KIỆN WEBSOCKET
// =================================================================
void webSocketEvent(WStype_t type, uint8_t * payload, size_t length) {
  switch (type) {
    case WStype_DISCONNECTED:
      Serial.println("[WS] Đã mất kết nối!");
      break;

    case WStype_CONNECTED:
      Serial.printf("[WS] Đã kết nối tới URL: %s\n", (char *)payload);
      // Gửi thông tin xác thực ngay khi kết nối thành công
      sendAuthentication();
      break;

    case WStype_TEXT:
      Serial.printf("[WS] Nhận được tin nhắn: %s\n", (char *)payload);
      
      // Phân tích cú pháp JSON nhận được
      DynamicJsonDocument doc(1024);
      DeserializationError error = deserializeJson(doc, payload, length);
      if (error) {
        Serial.print(F("[JSON] Phân tích lỗi: "));
        Serial.println(error.c_str());
        return;
      }

      // Xử lý sự kiện từ server
      if (doc["event"] == "action") {
        handleServerAction(doc);
      }
      break;

    // Các trường hợp khác không xử lý
    case WStype_BIN:
    case WStype_ERROR:
    case WStype_FRAGMENT_TEXT_START:
    case WStype_FRAGMENT_BIN_START:
    case WStype_FRAGMENT:
    case WStype_FRAGMENT_FIN:
      break;
  }
}

// =================================================================
// HÀM GỬI GÓI TIN XÁC THỰC (AUTH)
// =================================================================
void sendAuthentication() {
  DynamicJsonDocument authDoc(1024);

  authDoc["event"] = "auth";
  authDoc["auth_token"] = AUTH_TOKEN;

  JsonArray devgroups = authDoc.createNestedArray("devgroups");
  JsonObject phongKhach = devgroups.createNestedObject();
  phongKhach["name"] = "Phong_Khach";
  
  JsonObject devices = phongKhach.createNestedObject("devices");
  devices["led_01"] = "led";
  devices["gas_sensor_01"] = "gas_sensor";
  devices["dht22_sensor_01"] = "dht22_sensor"; // Thêm cả cảm biến DHT22

  String output;
  serializeJson(authDoc, output);
  
  webSocket.sendTXT(output);
  Serial.println("[WS] Đã gửi gói tin xác thực.");
}

// =================================================================
// HÀM GỬI BÁO CÁO DỮ LIỆU CẢM BIẾN (REPORT)
// =================================================================
void sendReport() {
  // --- Đọc dữ liệu từ cảm biến MQ2 ---
  int analogValue = analogRead(MQ2_ANALOG_PIN);

  // --- Đọc dữ liệu từ cảm biến DHT22 ---
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  // --- Gửi báo cáo cho cảm biến khí gas ---
  DynamicJsonDocument gasReport(256);
  gasReport["event"] = "report";
  gasReport["device_name"] = "gas_sensor_01";
  JsonObject gasData = gasReport.createNestedObject("data");
  gasData["analog_value"] = analogValue;
  // Bạn có thể chuyển đổi giá trị analog thành PPM nếu cần
  // gasData["ppm"] = your_ppm_calculation; 
  
  String gasOutput;
  serializeJson(gasReport, gasOutput);
  webSocket.sendTXT(gasOutput);
  Serial.println("[WS] Đã gửi báo cáo cảm biến Gas.");

  delay(200); // Đợi một chút trước khi gửi gói tin tiếp theo

  // --- Gửi báo cáo cho cảm biến DHT22 ---
  if (!isnan(humidity) && !isnan(temperature)) {
    DynamicJsonDocument dhtReport(256);
    dhtReport["event"] = "report";
    dhtReport["device_name"] = "dht22_sensor_01";
    JsonObject dhtData = dhtReport.createNestedObject("data");
    dhtData["temperature"] = temperature;
    dhtData["humidity"] = humidity;

    String dhtOutput;
    serializeJson(dhtReport, dhtOutput);
    webSocket.sendTXT(dhtOutput);
    Serial.println("[WS] Đã gửi báo cáo cảm biến DHT22.");
  } else {
    Serial.println("[ERROR] Lỗi đọc dữ liệu từ cảm biến DHT!");
  }
}

// =================================================================
// HÀM XỬ LÝ HÀNH ĐỘNG TỪ SERVER
// =================================================================
void handleServerAction(JsonDocument& doc) {
  JsonArray devices = doc["devices"];
  String action = doc["action"];

  for (JsonVariant device : devices) {
    String deviceName = device.as<String>();
    
    Serial.printf("[ACTION] Nhận lệnh '%s' cho thiết bị '%s'\n", action.c_str(), deviceName.c_str());

    if (deviceName == "led_01") {
      if (action == "ON") {
        digitalWrite(LED_01_PIN, HIGH);
        Serial.println("==> Bật LED 01");
      } else if (action == "OFF") {
        digitalWrite(LED_01_PIN, LOW);
        Serial.println("==> Tắt LED 01");
      }
    }
    // Thêm các thiết bị khác nếu cần
    // if (deviceName == "led_02") { ... }
  }
}