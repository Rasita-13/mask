<!DOCTYPE html>
<html>
<head>
  <title>Teachable Machine + MQTT</title>
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest"></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest"></script>
  <script src="https://unpkg.com/paho-mqtt/mqttws31.min.js"></script>
</head>
<body>
  <h1>Teachable Machine + MQTT</h1>
  <button onclick="init()">Start</button>

  <script>
    const modelURL = "https://teachablemachine.withgoogle.com/models/YOUR_MODEL_ID/"; // แก้ด้วย URL ของคุณ
    let model, webcam, prediction;
    let client;

    async function init() {
      const modelURL_full = modelURL + "model.json";
      const metadataURL = modelURL + "metadata.json";

      model = await tmImage.load(modelURL_full, metadataURL);
      webcam = new tmImage.Webcam(200, 200, true); // width, height, flip
      await webcam.setup();
      await webcam.play();
      window.requestAnimationFrame(loop);

      // แสดง webcam
      document.body.appendChild(webcam.canvas);

      connectMQTT();
    }

    async function loop() {
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }

    async function predict() {
      const predictions = await model.predict(webcam.canvas);
      let highest = predictions[0];

      for (let i = 1; i < predictions.length; i++) {
        if (predictions[i].probability > highest.probability) {
          highest = predictions[i];
        }
      }

      // ส่งเฉพาะ prediction ที่มีความมั่นใจมากกว่า 0.8
      if (highest.probability > 0.8) {
        console.log(`Predicted: ${highest.className}`);
        sendMQTT(highest.className);
      }
    }

    function connectMQTT() {
      // เปลี่ยนตาม MQTT broker ของคุณ
      const broker = "broker.hivemq.com";
      const port = 8000; // WebSocket port
      const clientId = "tmClient_" + Math.random().toString(16).substr(2, 8);

      client = new Paho.MQTT.Client(broker, port, clientId);

      client.onConnectionLost = (responseObject) => {
        console.log("Connection lost: " + responseObject.errorMessage);
      };

      client.connect({
        onSuccess: () => {
          console.log("Connected to MQTT broker");
        },
        useSSL: false,
      });
    }

    function sendMQTT(message) {
      if (client && client.isConnected()) {
        const topic = "tm/predict"; // เปลี่ยน topic ตามต้องการ
        const payload = new Paho.MQTT.Message(message);
        payload.destinationName = topic;
        client.send(payload);
        console.log(`Sent MQTT: ${message}`);
      }
    }
  </script>
</body>
</html>
