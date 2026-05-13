#include <TFT_eSPI.h>
#include <WiFi.h>
#include <DNSServer.h>
#include <WebServer.h>
#include <time.h>

// 获取 CPU 温度
extern "C" uint8_t temprature_sens_read();

TFT_eSPI tft = TFT_eSPI();
WebServer server(80);
DNSServer dnsServer;

// --- 硬件引脚定义 ---
#define BTN_UP    27
#define BTN_DOWN  25
#define BTN_OK    26
#define BTN_BACK  33

// 摇杆引脚
#define JOY_X_PIN  34
#define JOY_Y_PIN  35

// 菜单项（5个选项）
const char* menuItems[] = {" NETWORK CLOCK ", " WIFI SCANNER  ", " CONFIG WIFI   ", " DINO RUN GAME ", " JOYSTICK TEST "};
int selected = 0;
int totalItems = 5;

// --- 基础工具函数 ---
void waitForRelease(int pin) {
  while (digitalRead(pin) == LOW) { delay(10); }
}

void satlineClear() {
  tft.resetViewport();
  tft.fillScreen(TFT_BLACK);
  tft.setViewport(0, 0, 160, 128);
}

void transitionAnim() {
  tft.resetViewport();
  for (int i = 0; i < 132; i += 10) { 
    tft.drawFastHLine(0, i, 162, 0x001F); 
    delay(10); 
  }
  satlineClear();
}

// --- 开机跑码动画 ---
void showBootAnimation() {
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_GREEN, TFT_BLACK);
  tft.setTextSize(1);
  
  const char* bootLines[] = {
    "SatLine OS v2.0",
    "Booting...",
    "[OK] CPU init",
    "[OK] Memory check",
    "[OK] SPI Flash",
    "[OK] TFT Driver",
    "[OK] GPIO setup",
    "[OK] Button init",
    "[OK] WiFi module",
    "[OK] Time service",
    "[OK] Joystick init",
    "System ready.",
    "Starting GUI..."
  };
  
  int totalLines = sizeof(bootLines) / sizeof(bootLines[0]);
  
  for (int i = 0; i < totalLines; i++) {
    tft.setCursor(5, 5 + i * 9);
    tft.print(bootLines[i]);
    delay(120);
  }
  
  tft.drawRect(10, 115, 140, 8, TFT_CYAN);
  for (int p = 0; p <= 140; p += 2) {
    tft.fillRect(11, 116, p, 6, TFT_GREEN);
    delay(5);
  }
  
  delay(500);
}

// --- 网页配网：HTML 页面 ---
const char* HTML_CONTENT = "<html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1.0'><style>body{font-family:sans-serif;text-align:center;}input{padding:10px;width:80%;margin:10px;}</style></head><body><h2>SatLine OS 配网</h2><form method='POST' action='/save'>WiFi名称:<br><input name='ssid' type='text'><br>WiFi密码:<br><input name='pass' type='password'><br><input type='submit' value='保存并连接'></form></body></html>";

// --- 功能1：DNS 劫持配网 ---
void startWebPortal() {
  transitionAnim();
  tft.setTextColor(TFT_YELLOW);
  tft.setCursor(5, 30); tft.println("AP MODE: SatLine_OS");
  tft.setCursor(5, 50); tft.setTextColor(TFT_WHITE);
  tft.println("Connect WiFi with phone");
  tft.println("Browser: 192.168.4.1");

  WiFi.softAP("SatLine_OS");
  dnsServer.start(53, "*", WiFi.softAPIP());
  server.on("/", []() { server.send(200, "text/html", HTML_CONTENT); });
  server.on("/save", HTTP_POST, []() {
    String s = server.arg("ssid");
    String p = server.arg("pass");
    server.send(200, "text/html", "Connecting... Please wait.");
    WiFi.begin(s.c_str(), p.c_str());
  });
  server.onNotFound([]() { server.send(200, "text/html", HTML_CONTENT); });
  server.begin();

  while (WiFi.status() != WL_CONNECTED) {
    dnsServer.processNextRequest();
    server.handleClient();
    if (digitalRead(BTN_BACK) == LOW) break; 
    
    tft.setCursor(5, 100); tft.setTextColor(TFT_CYAN, TFT_BLACK);
    tft.printf("STATUS: %s", (WiFi.status() == WL_IDLE_STATUS ? "WAITING..." : "CONNECTING"));
    delay(10);
  }

  if (WiFi.status() == WL_CONNECTED) {
    satlineClear();
    tft.setCursor(10, 50); tft.setTextColor(TFT_GREEN);
    tft.println("WIFI CONNECTED!");
    tft.setTextColor(TFT_WHITE);
    tft.println(WiFi.localIP());
    delay(2000);
  }
  
  server.stop();
  dnsServer.stop();
  WiFi.softAPdisconnect(true);
  waitForRelease(BTN_BACK);
}

// --- 功能2：网络时钟 ---
void showClock() {
  transitionAnim();
  if (WiFi.status() != WL_CONNECTED) {
    tft.setTextColor(TFT_RED); tft.setCursor(10, 50);
    tft.print("NO WIFI! PLEASE CONFIG");
    delay(2000); return;
  }
  configTime(8 * 3600, 0, "ntp1.aliyun.com", "time.windows.com");
  while (digitalRead(BTN_BACK) == HIGH) {
    struct tm timeinfo;
    if (getLocalTime(&timeinfo)) {
      tft.fillScreen(TFT_BLACK);
      tft.drawRoundRect(5, 20, 150, 80, 5, 0x001F);
      tft.setTextColor(TFT_CYAN); tft.setTextSize(3);
      tft.setCursor(15, 45);
      tft.printf("%02d:%02d:%02d", timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec);
      tft.setTextSize(1); tft.setTextColor(TFT_WHITE);
      tft.setCursor(35, 80);
      tft.printf("%d-%02d-%02d", timeinfo.tm_year+1900, timeinfo.tm_mon+1, timeinfo.tm_mday);
    }
    delay(500);
  }
  waitForRelease(BTN_BACK);
}

// --- 功能3：WiFi 扫描器 ---
void showWiFiScanner() {
  transitionAnim();
  tft.setCursor(10, 10); tft.print("SCANNING...");
  WiFi.mode(WIFI_STA); WiFi.disconnect();
  int n = WiFi.scanNetworks();
  satlineClear();
  tft.fillRect(0, 0, 160, 14, 0x000F);
  tft.setCursor(4, 3); tft.print("FOUND APs");
  for (int i = 0; i < min(n, 5); i++) {
    int y = 20 + (i * 20);
    tft.setCursor(5, y); tft.setTextColor(TFT_CYAN);
    tft.print(WiFi.SSID(i).substring(0, 12));
    int rssi = WiFi.RSSI(i);
    int barW = map(constrain(rssi, -100, -30), -100, -30, 0, 40);
    tft.drawRect(110, y, 42, 8, TFT_DARKGREY);
    tft.fillRect(111, y+1, barW, 6, (rssi > -65 ? TFT_GREEN : TFT_RED));
  }
  while (digitalRead(BTN_BACK) == HIGH) { delay(10); }
  waitForRelease(BTN_BACK);
}

// --- 功能4：恐龙跳一跳（谷歌黑白风格）---
void playDino() {
  transitionAnim();
  
  float dinoY = 103;
  float dinoVY = 0;
  int cactusX = 160;
  int score = 0;
  static int highScore = 0;
  int speed = 5;
  int frame = 0;
  bool jumping = false;
  bool dead = false;
  
  while (digitalRead(BTN_BACK) == HIGH && !dead) {
    if ((digitalRead(BTN_UP) == LOW || digitalRead(BTN_OK) == LOW) && !jumping) {
      dinoVY = -7;
      jumping = true;
    }
    
    dinoVY += 0.8;
    dinoY += dinoVY;
    if (dinoY >= 103) {
      dinoY = 103;
      dinoVY = 0;
      jumping = false;
    }
    
    cactusX -= speed;
    if (cactusX < -10) {
      cactusX = random(130, 180);
      score++;
      if (score % 5 == 0 && speed < 12) speed++;
    }
    
    if (cactusX < 30 && cactusX > 10 && dinoY > 95) {
      dead = true;
      if (score > highScore) highScore = score;
      break;
    }
    
    tft.fillScreen(TFT_WHITE);
    tft.drawFastHLine(0, 108, 160, TFT_BLACK);
    
    for (int i = 0; i < 160; i += 20) {
      tft.drawFastHLine((i + frame * speed) % 160, 115, 10, TFT_BLACK);
    }
    
    tft.fillRect(cactusX, 95, 5, 13, TFT_BLACK);
    tft.fillRect(cactusX - 3, 100, 3, 8, TFT_BLACK);
    tft.fillRect(cactusX + 5, 100, 3, 8, TFT_BLACK);
    
    tft.fillRect(20, dinoY, 12, 12, TFT_BLACK);
    tft.fillCircle(28, dinoY + 4, 2, TFT_WHITE);
    tft.drawFastHLine(32, dinoY + 7, 4, TFT_BLACK);
    tft.fillTriangle(21, dinoY - 2, 23, dinoY + 2, 24, dinoY - 2, TFT_BLACK);
    tft.fillTriangle(25, dinoY - 2, 27, dinoY + 2, 28, dinoY - 2, TFT_BLACK);
    
    if (!jumping && (frame / 6) % 2 == 0) {
      tft.drawFastHLine(22, dinoY + 10, 5, TFT_BLACK);
    } else if (!jumping) {
      tft.drawFastHLine(26, dinoY + 10, 5, TFT_BLACK);
    }
    
    tft.setTextColor(TFT_BLACK);
    tft.setCursor(100, 5);
    tft.print("HI ");
    tft.print(highScore);
    tft.setCursor(135, 5);
    tft.print(score);
    
    if (score == 0) {
      tft.setCursor(35, 60);
      tft.print("PRESS  ^  TO JUMP");
    }
    
    frame++;
    delay(25);
  }
  
  if (dead) {
    tft.fillScreen(TFT_WHITE);
    tft.setTextColor(TFT_BLACK);
    tft.setTextSize(2);
    tft.setCursor(30, 40);
    tft.print("GAME OVER");
    tft.setTextSize(1);
    tft.setCursor(50, 65);
    tft.print("SCORE: ");
    tft.print(score);
    tft.setCursor(50, 80);
    tft.print("BEST: ");
    tft.print(highScore);
    tft.setCursor(35, 105);
    tft.print("PRESS BACK");
    
    while (digitalRead(BTN_BACK) == HIGH) delay(10);
  }
  
  waitForRelease(BTN_BACK);
}

// --- 功能5：摇杆测试（精确校准版）---
void showJoystick() {
  transitionAnim();
  
  analogReadResolution(12);
  analogSetAttenuation(ADC_11db);
  pinMode(JOY_X_PIN, INPUT);
  pinMode(JOY_Y_PIN, INPUT);
  
  int centerX = 80;
  int centerY = 60;
  int radius = 30;
  
  // ========== 根据你的数据精确校准 ==========
  int xCenter = 1867;  // 你的X轴中心值
  int yCenter = 2000;  // 你的Y轴中心值
  int xRange = 2048;   // 最大偏移范围 (4095/2 ≈ 2048)
  int yRange = 2048;
  
  while (digitalRead(BTN_BACK) == HIGH) {
    int rawX = analogRead(JOY_X_PIN);
    int rawY = analogRead(JOY_Y_PIN);
    
    // 计算偏移量（中心归零）
    int offsetX = rawX - xCenter;
    int offsetY = rawY - yCenter;
    
    // 映射到圆环范围 (-radius 到 radius)
    int joyX = map(offsetX, -xRange, xRange, -radius, radius);
    int joyY = map(offsetY, -yRange, yRange, -radius, radius);
    
    // 限制范围
    joyX = constrain(joyX, -radius, radius);
    joyY = constrain(joyY, -radius, radius);
    
    // 限制在圆环内（勾股定理）
    int distance = sqrt(joyX * joyX + joyY * joyY);
    if (distance > radius) {
      joyX = joyX * radius / distance;
      joyY = joyY * radius / distance;
    }
    
    int dotX = centerX + joyX;
    int dotY = centerY + joyY;
    
    tft.fillScreen(TFT_BLACK);
    
    // 圆环
    tft.drawCircle(centerX, centerY, radius, TFT_CYAN);
    tft.drawCircle(centerX, centerY, radius + 1, TFT_CYAN);
    
    // 十字线
    tft.drawFastHLine(centerX - radius - 5, centerY, radius * 2 + 10, TFT_DARKGREY);
    tft.drawFastVLine(centerX, centerY - radius - 5, radius * 2 + 10, TFT_DARKGREY);
    
    // 中心点
    tft.fillCircle(centerX, centerY, 2, TFT_RED);
    
    // 小点
    tft.fillCircle(dotX, dotY, 5, TFT_GREEN);
    tft.fillCircle(dotX, dotY, 3, TFT_WHITE);
    
    // 显示数据
    tft.setTextColor(TFT_WHITE, TFT_BLACK);
    tft.setTextSize(1);
    
    tft.setCursor(5, 5);
    tft.print("RAW X:");
    tft.print(rawX);
    tft.setCursor(5, 15);
    tft.print("RAW Y:");
    tft.print(rawY);
    
    tft.setCursor(5, 30);
    tft.print("OFF X:");
    tft.print(offsetX);
    tft.setCursor(5, 40);
    tft.print("OFF Y:");
    tft.print(offsetY);
    
    tft.setCursor(5, 55);
    tft.print("POS X:");
    tft.print(joyX);
    tft.setCursor(5, 65);
    tft.print("POS Y:");
    tft.print(joyY);
    
    // 方向指示
    tft.setCursor(5, 115);
    if(abs(joyX) < 3 && abs(joyY) < 3) {
      tft.print("CENTER");
    } else if(abs(joyX) > abs(joyY)) {
      tft.print(joyX > 0 ? "RIGHT" : "LEFT");
    } else {
      tft.print(joyY > 0 ? "DOWN" : "UP");
    }
    
    delay(30);
  }
  
  waitForRelease(BTN_BACK);
}

// --- 菜单引擎 ---
void moveSelection(int from, int to) {
  int startY = 35 + (from * 18) - 4;
  int endY = 35 + (to * 18) - 4;
  for (int s = 1; s <= 5; s++) {
    int curY = startY + (endY - startY) * (s - 1) / 5;
    tft.fillRect(4, curY, 154, 14, TFT_BLACK);
    tft.setTextColor(TFT_DARKGREY);
    tft.setCursor(15, 35 + (from * 18)); tft.print(menuItems[from]);
    tft.setCursor(15, 35 + (to * 18)); tft.print(menuItems[to]);
    int nextY = startY + (endY - startY) * s / 5;
    tft.fillRoundRect(4, nextY, 152, 14, 2, 0x001F);
    if (s == 5) {
      tft.setTextColor(TFT_WHITE);
      tft.setCursor(15, 35 + (to * 18)); tft.print(menuItems[to]);
    }
    delay(10);
  }
}

void drawMainMenu() {
  satlineClear();
  tft.fillRect(0, 0, 160, 16, 0x000F); 
  tft.setTextColor(TFT_CYAN); tft.setCursor(4, 4); tft.print("SATLINE OS");
  tft.setTextColor(TFT_WHITE); tft.print(" | v2.0");
  for (int i = 0; i < totalItems; i++) {
    if (i == selected) {
      tft.fillRoundRect(4, 35 + (i * 18) - 4, 152, 14, 2, 0x001F);
      tft.setTextColor(TFT_WHITE);
    } else {
      tft.setTextColor(TFT_DARKGREY);
    }
    tft.setCursor(15, 35 + (i * 18)); tft.print(menuItems[i]);
  }
}

void setup() {
  tft.init();
  tft.setRotation(1);
  tft.setViewport(0, 0, 160, 128);
  
  showBootAnimation();
  
  pinMode(BTN_UP, INPUT_PULLUP); 
  pinMode(BTN_DOWN, INPUT_PULLUP);
  pinMode(BTN_OK, INPUT_PULLUP); 
  pinMode(BTN_BACK, INPUT_PULLUP);
  
  WiFi.begin(); 
  satlineClear();
  tft.setTextColor(TFT_GREEN); 
  tft.setCursor(10, 40); 
  tft.print("OS READY.");
  delay(500);
  drawMainMenu();
}

void loop() {
  if (digitalRead(BTN_UP) == LOW) {
    int p = selected; selected = (selected - 1 + totalItems) % totalItems;
    moveSelection(p, selected); waitForRelease(BTN_UP);
  }
  if (digitalRead(BTN_DOWN) == LOW) {
    int p = selected; selected = (selected + 1) % totalItems;
    moveSelection(p, selected); waitForRelease(BTN_DOWN);
  }
  if (digitalRead(BTN_OK) == LOW) {
    waitForRelease(BTN_OK);
    if (selected == 0) showClock();
    else if (selected == 1) showWiFiScanner();
    else if (selected == 2) startWebPortal();
    else if (selected == 3) playDino();
    else if (selected == 4) showJoystick();
    drawMainMenu();
  }
}
