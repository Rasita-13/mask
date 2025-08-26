<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Teachable Machine + MQTT</title>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest"></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest"></script>
  <script src="https://unpkg.com/paho-mqtt@1.1.0/mqttws31.min.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #f4f6f9;
      margin: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      min-height: 100vh;
    }
    header {
      background: #007bff;
      color: #fff;
      width: 100%;
      padding: 20px;
      text-align: center;
      font-size: 24px;
    }
    .container {
      background: #fff;
      margin-top: 20px;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 4px 10px rgba(0,0,0,0.1);
      text-align: center;
      width: 300px;
    }
    button {
      padding: 10px 20px;
      font-size: 16px;
      border: none;
      border-radius: 5px;
      background: #28a745;
      color: white;
      cursor: pointer;
    }
    button:hover {
      background: #218838;
    }
    #webcam-container {
      margin-top: 15px;
    }
    #result {
      margin-top: 10px;
      font-size: 18px;
      font-weight: bold;
      color: #333;
    }
    .status {
      margin-top: 8px;
      font-size: 14px;
      color: #666;
    }
  </style>
</head>
<body>
  <header>🤖 Teachable Machine + MQTT Dashboard</header>
  <div class="container">
    <button onclick="init()">Start Camera & Model</button>
    <div id="webcam-container"></div>
    <div id="result">Result will appear here...</div>
    <div class="status" id="mqtt-status">MQTT: Not connected</div>
  </div>

  <script>
    const modelBaseURL = "https://teachablemachine.withgoogle.com/models/A2LkaBTvm/";
    let model, webcam, client;

    async function init() {
      try {
        const modelURL = modelBaseURL + "model.json";
        const metadataURL = modelBaseURL + "metadata.json";
        model = await tmImage.load(modelURL, metadataURL);

        webcam = new tmImage.Webcam(300, 300, true);
        await webcam.setup();
        await webcam.play();
        document.getElementById("webcam-container").appendChild(webcam.canvas);
        window.requestAnimationFrame(loop);

        connectMQTT();
      } catch (error) {
        console.error("Error setting up webcam or model:", error);
        alert("ไม่สามารถใช้กล้องหรือโหลดโมเดลได้ ตรวจสอบ HTTPS และสิทธิ์การเข้าถึงกล้อง");
      }
    }

    async function loop() {
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }

    async function predict() {
      const predictions = await model.predict(webcam.canvas);
      let highest = predictions.reduce((a, b) => a.probability > b.probability ? a : b);

      document.getElementById("result").innerText =
        `Predicted: ${highest.className} (${(highest.probability * 100).toFixed(2)}%)`;

      if (highest.probability > 0.8) {
        sendMQTT(highest.className);
      }
    }

    function connectMQTT() {
      const broker = "broker.hivemq.com";
      const port = 8000;
      const clientId = "tmClient_" + Math.random().toString(16).substr(2, 8);

      client = new Paho.MQTT.Client(broker, port, clientId);

      client.onConnectionLost = (responseObject) => {
        console.log("Connection lost:", responseObject.errorMessage);
        document.getElementById("mqtt-status").innerText = "MQTT: Disconnected";
        setTimeout(connectMQTT, 3000);
      };

      client.connect({
        onSuccess: () => {
          console.log("Connected to MQTT broker");
          document.getElementById("mqtt-status").innerText = "MQTT: Connected ✅";
        },
        useSSL: location.protocol === "https:",
      });
    }

    function sendMQTT(message) {
      if (client && client.isConnected()) {
        const topic = "tm/predict";
        const msg = new Paho.MQTT.Message(message);
        msg.destinationName = topic;
        client.send(msg);
        console.log("Sent MQTT:", message);
      }
    }
  </script>
</body>
</html>
