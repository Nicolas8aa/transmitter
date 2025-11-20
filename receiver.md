  #include <SPI.h>
  #include <mcp2515.h>

  struct can_frame canMsg;
  struct MCP2515 mcp2515(5); // CS pin is GPIO 5

  #define CAN_ACK_ID 0x037       // CAN ID for acknowledgment
  #define CAN_DATA_ID 0x036      // CAN ID for incoming data
  #define DAC_PIN 25             // GPIO25 for DAC output
  #define DAC_RESOLUTION 255     // 8-bit DAC (0-255)
  #define DAC_MAX_VOLTAGE 3.3    // ESP32 DAC max voltage

  void setup()
  {
    Serial.begin(115200);
    
    // Configure DAC pin
    pinMode(DAC_PIN, OUTPUT);
    
    // Initialize CAN bus
    SPI.begin();
    mcp2515.reset();
    mcp2515.setBitrate(CAN_500KBPS, MCP_8MHZ);
    mcp2515.setNormalMode();
    
    Serial.println("Receiver initialized");
  }


  void loop()
  {
    // Poll for CAN messages
    if (mcp2515.readMessage(&canMsg) == MCP2515::ERROR_OK)
    {
      if (canMsg.can_id == CAN_DATA_ID)
      {
        // Extract voltage value in millivolts
        int voltageMillivolts = (canMsg.data[0] << 8) | canMsg.data[1];
        
        // Convert millivolts to DAC value (0-255)
        int dacValue = convertMillivoltsToDac(voltageMillivolts);
        
        // Output to DAC
        dacWrite(DAC_PIN, dacValue);
        
        // Display received data
        float voltage = voltageMillivolts / 1000.0;
        Serial.print("Voltage received: ");
        Serial.print(voltage, 3);
        Serial.print(" V -> DAC: ");
        Serial.println(dacValue);
        
        // Send acknowledgment
        sendAcknowledgment();
      }
    }
  }

  /**
  * Convert millivolts to DAC value (0-255)
  * @param millivolts Input voltage in millivolts
  * @return DAC value constrained to 0-255
  */
  int convertMillivoltsToDac(int millivolts) {
    // Convert mV to DAC value: DAC = (mV / 3300) * 255
    int dacValue = (millivolts * DAC_RESOLUTION) / (DAC_MAX_VOLTAGE * 1000);
    
    // Constrain to valid DAC range
    return constrain(dacValue, 0, DAC_RESOLUTION);
  }

  /**
  * Send CAN acknowledgment message
  */
  void sendAcknowledgment() {
    canMsg.can_id = CAN_ACK_ID;
    canMsg.can_dlc = 0; // No data needed for ACK
    
    if (mcp2515.sendMessage(&canMsg) == MCP2515::ERROR_OK) {
      Serial.println("ACK sent");
    }
  }