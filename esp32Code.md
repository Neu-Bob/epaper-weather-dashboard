#include <WiFi.h>           // Wi-Fi library for managing network connections
#include <WebServer.h>       // Web server library to handle HTTP requests
#include <PubSubClient.h>    // MQTT library for publishing and subscribing to topics
#include <time.h>            // Time library for NTP (Network Time Protocol) functionality

// **Wi-Fi credentials**: Set your Wi-Fi SSID and password
const char* ssid = "Mortlach";       
const char* password = "OU812!!!";   

// **MQTT Broker Configuration**
const char* mqttServer = "192.168.86.103";  // IP address of the MQTT broker
const int mqttPort = 1883;                  // MQTT port (default is 1883)
const char* mqttTopic = "weather/airports/KRMG/metar/#";  // MQTT topic with wildcard to receive METAR updates

// **ESP32 Clients**: Objects for Wi-Fi, MQTT, and HTTP services
WiFiClient espClient;                // Wi-Fi client for networking
PubSubClient mqttClient(espClient);  // MQTT client to communicate with broker
WebServer server(80);                // Web server running on port 80 (HTTP)

// **METAR Data Structure**: Holds the weather and airport data from the MQTT broker
struct MetarData {
  String airportStatus = "VFR";        // Airport status (e.g., VFR, IFR)
  String temperature = "N/A";          // Temperature in Celsius
  String dewPoint = "N/A";             // Dew point in Celsius
  String windSpeed = "N/A";            // Wind speed in knots
  String windDirection = "N/A";        // Wind direction in degrees
  String cloudCover = "N/A";           // Cloud coverage description
  String barometricPressure = "N/A";   // Pressure in inches of mercury (Hg)
  String visibilityMiles = "N/A";      // Visibility in miles
  String humidity = "N/A";             // Humidity percentage
  unsigned long lastIssued = millis(); // Timestamp of last data update
} metar;  

// **Get Local Time**: Retrieves and formats local time from NTP
String getFormattedLocalTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {  // Check if local time is available
    return "N/A";  // Return "N/A" if time sync fails
  }
  char timeStr[50];  
  strftime(timeStr, sizeof(timeStr), "%H:%M %a %b %d LOCAL", &timeinfo);  // Format time
  return String(timeStr);  // Return formatted local time as a string
}

// **Get Zulu Time**: Retrieves and formats UTC time from NTP
String getFormattedZuluTime() {
  time_t now = time(nullptr);         // Get the current time
  struct tm* zuluTime = gmtime(&now);  // Convert to UTC
  char timeStr[50];  
  strftime(timeStr, sizeof(timeStr), "%H:%M %a %b %d ZULU", zuluTime);  // Format UTC time
  return String(timeStr);  // Return formatted Zulu time as a string
}

// **Calculate Minutes Since Last Update**: Returns "X minutes ago"
String getTimeAgo() {
  unsigned long elapsedMillis = millis() - metar.lastIssued;  // Time since last update
  int minutesAgo = (elapsedMillis / 1000) / 60;  // Convert milliseconds to minutes
  return String(minutesAgo) + " min ago";  // Return time elapsed as a string
}

// **MQTT Callback Function**: Processes incoming MQTT messages
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message;
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];  // Convert payload to a string
  }

  String topicStr = String(topic);  // Convert topic to a string

  // **Update MetarData Based on MQTT Topic**
  if (topicStr.endsWith("temperature")) {
    metar.temperature = message + "°C";
  } else if (topicStr.endsWith("dewPoint")) {
    metar.dewPoint = message + "°C";
  } else if (topicStr.endsWith("windSpeed")) {
    metar.windSpeed = message + " kt";
  } else if (topicStr.endsWith("windDirection")) {
    metar.windDirection = message + "°";
  } else if (topicStr.endsWith("cloudCover")) {
    metar.cloudCover = message;
  } else if (topicStr.endsWith("barometricPressure")) {
    metar.barometricPressure = message + " Hg";
  } else if (topicStr.endsWith("visibilityMiles")) {
    metar.visibilityMiles = message + " miles";
  } else if (topicStr.endsWith("humidity")) {
    metar.humidity = message + "%";
  } else if (topicStr.endsWith("airportStatus")) {
    metar.airportStatus = message;
  }

  metar.lastIssued = millis();  // Update timestamp for the latest data
}

// **Connect to MQTT Broker**
void connectToMQTT() {
  while (!mqttClient.connected()) {  // Keep trying until connected
    Serial.print("Connecting to MQTT...");
    if (mqttClient.connect("ESP32_Client")) {  // Attempt connection
      Serial.println(" Connected!");
      mqttClient.subscribe(mqttTopic);  // Subscribe to the METAR topic
    } else {
      Serial.print(" Failed, rc=");  
      Serial.println(mqttClient.state());  // Print connection state
      delay(5000);  // Retry after 5 seconds
    }
  }
}

void handleRoot() {
String html = "<html><head><title>KRMG METAR</title>";
html += "<style>";
html += "body { font-family: Arial; background-color: #f0f0f0; margin: 0; padding: 0; } ";
html += ".time-bar { display: flex; justify-content: space-around; align-items: center; padding: 10px 0; } ";
html += ".time { font-size: 48px; font-weight: bold; } ";  // Larger font size for time
html += ".date { font-size: 18px; margin-top: -10px; } ";  // Smaller font for date on a new line
html += ".airport-info { display: flex; justify-content: space-between; padding: 5px 10px; font-size: 24px; } ";
html += ".two-column { display: flex; justify-content: space-between; margin: 10px 20px; } ";
html += ".status-bar { display: flex; justify-content: space-between; padding: 10px; background-color: #e0e0e0; } ";
html += "</style></head><body>";

// Time and Status Bar
html += "<div class='time-bar'>";
html += "<div><div class='time'>" + getFormattedLocalTime().substring(0, 5) + "</div>";
html += "<div class='date'>" + getFormattedLocalTime().substring(6) + "</div></div>";  // Date on new line
html += "<div class='large-text' style='font-size: 60px; font-weight: bold;'>" + metar.airportStatus + "</div>";
html += "<div><div class='time'>" + getFormattedZuluTime().substring(0, 5) + "</div>";
html += "<div class='date'>" + getFormattedZuluTime().substring(6) + "</div></div>";  // Date on new line
html += "</div>";

  // Airport Information
  html += "<div class='airport-info'><div>KRMG</div><div>Richard B Russell Regional</div></div>";
  html += "<hr>";  // Add horizontal line after the KRMG section

  // METAR Data
  html += "<div class='two-column'><div>Observations</div><div>" + metar.barometricPressure + " Hg</div></div>";
  html += "<div class='two-column'><div>" + metar.cloudCover + "</div><div>Humidity: " + metar.humidity + "%</div></div>";
  html += "<div class='two-column'><div>Wind: " + metar.windDirection + "° at " + metar.windSpeed + " kt</div></div>";
  html += "<div class='two-column'><div>Visibility: " + metar.visibilityMiles + "</div></div>";
  html += "<div class='two-column'><div>Temp/Dew: " + metar.temperature + " / " + metar.dewPoint + "</div></div>";

  // Status Bar
  html += "<div class='status-bar'><div>Issued: " + getTimeAgo() + "</div>";
  html += "<div><img src='https://img.icons8.com/ios-filled/50/000000/wifi.png'/></div></div>";

  html += "</body></html>";
  server.send(200, "text/html", html);
}

// **Setup Function**: Initializes Wi-Fi, MQTT, and HTTP services
void setup() {
  Serial.begin(115200);  
  configTime(-4 * 3600, 0, "pool.ntp.org");  

  WiFi.begin(ssid, password);  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");  
  }
  Serial.println("\nConnected to Wi-Fi!");

  mqttClient.setServer(mqttServer, mqttPort);  
  mqttClient.setCallback(mqttCallback);  
  connectToMQTT();  

  server.on("/", handleRoot);  
  server.begin();  
  Serial.println("Web server started!");
}

// **Main Loop**: Maintains MQTT and HTTP connections
void loop() {
  if (!mqttClient.connected()) {  
    connectToMQTT();  
  }
  mqttClient.loop();  
  server.handleClient();  
}