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
  <div id="webcam-container"></div>
  <div id="result" style="margin-top:10px; font-size:18px; font-weight:bold;"></div>

  <script>
    const modelURL = "https://teachablemachine.withgoogle.com/models/A2LkaBTvm/"; // เปลี่ยนเป็น URL ของคุณ
    let model, webcam, client;

    async function init() {
      try {
        const modelURL_full = modelURL + "model.json";
        const metadataURL = modelURL + "metadata.json";

        model = await tmImage.load(modelURL_full, metadataURL);

        // สร้าง webcam object
        webcam = new tmImage.Webcam(200, 200, true); 
        await webcam.setup();  // ขอสิทธิ์กล้อง
        await webcam.play();   // เริ่มเล่นกล้อง
        window.requestAnimationFrame(loop);

        // แสดง webcam บนหน้าเว็บ
        document.getElementById("webcam-container").appendChild(webcam.canvas);

        connectMQTT();
      } catch (err) {
        console.error("Webcam error: ", err);
        alert("ไม่สามารถเข้าถึงกล้องได้! ตรวจสอบสิทธิ์การใช้งานกล้องและรันผ่าน HTTPS หรือ localhost");
      }
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

      // แสดงผลบนหน้าเว็บ
      document.getElementById("result").innerText = 
        `Predicted: ${highest.className} (${(highest.probability*100).toFixed(2)}%)`;

      // ส่ง MQTT ถ้ามั่นใจ > 0.8
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
        console.log("Connection lost: " + responseObject.errorMessage);
        setTimeout(connectMQTT, 3000); // reconnect อัตโนมัติ
      };

      client.connect({
        onSuccess: () => {
          console.log("Connected to MQTT broker");
        },
        useSSL: location.protocol === "https:",
      });
    }

    function sendMQTT(message) {
      if (client && client.isConnected()) {
        const topic = "tm/predict"; 
        const payload = new Paho.MQTT.Message(message);
        payload.destinationName = topic;
        client.send(payload);
        console.
