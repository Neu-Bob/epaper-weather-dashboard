Objective: e-ink display that can be framed and placed on a table to display current METAR for a given airport.

Method: 

  Hardware: ESP32 - 
  adafruit Feather V1 ESP32 and a Waveshare 7.5" epaper screen
  
  Software: Node-Red to pull data from unidentified api and then send it to sensors in HomeAssistant.  Within HomeAssistant make a dashboard to push to screen
  

	1.	Data Source:
	•	Identify where to pull the METAR data from. A common source is the NOAA Aviation Weather Center or a public API like AVWX or Open-METAR.
    
    Options for Data
    1.	NOAA (National Oceanic and Atmospheric Administration) – Free and reliable.
    •	Data is provided via the Aviation Weather Center or FTP, but the API requires parsing XML/JSON.
    2.	AVWX API – Offers both free (limited) and paid plans.
    •	Easier JSON-based API with METAR, TAF, and other aviation data.
    •	Requires API key setup if you want more frequent data access.
    3.	Open-METAR – Public API without authentication.
    •	Less reliable but easy to use for small projects.


	2.	Node-RED Setup:
	•	Use Node-RED to query the METAR data from the chosen API at regular intervals.
	•	Format and filter the data as needed (e.g., only specific airports or certain fields).
	3.	Home Assistant Integration:
	•	Send the processed data from Node-RED to sensors in Home Assistant.
	•	Use MQTT or REST API as the communication method between Node-RED and Home Assistant.
	4.	E-Ink Display Setup (Later Phase):
	•	Set up the e-ink display to fetch the relevant sensor data from Home Assistant.



  inspiration - https://community.home-assistant.io/t/use-esphome-with-e-ink-displays-to-blend-in-with-your-home-decor/435428
  https://www.youtube.com/watch?v=XaxYSqeFJ6w


  https://community.home-assistant.io/t/display-materialdesign-icons-on-esphome-attached-to-screen/199790/16