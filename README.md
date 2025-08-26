<!DOCTYPE html>
<html>
<head>
  <title>Teachable Machine Image Model (Mobile)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0"> <!-- ทำ responsive -->
  <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@latest/dist/tf.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@latest/dist/teachablemachine-image.min.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      text-align: center;
      margin: 0;
      padding: 0;
    }
    #webcam-container {
      max-width: 100%;
      margin: 10px auto;
    }
    video {
      max-width: 100%;
      height: auto;
    }
    #label-container {
      margin: 10px;
      font-size: 1.2em;
    }
    button {
      padding: 10px 20px;
      font-size: 1em;
      margin: 5px;
    }
  </style>
</head>
<body>
  <div>Teachable Machine Image Model (Mobile)</div>
  <button type="button" onclick="init('environment')">Start (Back Camera)</button>
  <button type="button" onclick="init('user')">Start (Front Camera)</button>
  <button type="button" onclick="stop()">Stop</button>
  <div id="webcam-container"></div>
  <div id="label-container"></div>

  <script type="text/javascript">
    const URL = "https://teachablemachine.withgoogle.com/models/A2LkaBTvm/";

    let model, webcam, labelContainer, maxPredictions;

    async function init(facingMode) {
      const modelURL = URL + "model.json";
      const metadataURL = URL + "metadata.json";

      model = await tmImage.load(modelURL, metadataURL);
      maxPredictions = model.getTotalClasses();

      // ตั้งค่า Webcam สำหรับมือถือ โดยระบุ facingMode
      webcam = new tmImage.Webcam(224, 224, false); // ใช้ 224x224 เหมือนเดิม
      try {
        await webcam.setup({ facingMode: facingMode }); // เลือกกล้องหน้า (user) หรือหลัง (environment)
        await webcam.play();
        if (!webcam.canvas) {
          throw new Error("Webcam canvas is not initialized");
        }
        console.log("Webcam initialized, width:", webcam.canvas.width, "facingMode:", facingMode);
        window.requestAnimationFrame(loop);
      } catch (e) {
        console.error("Webcam setup failed:", e);
        document.getElementById("label-container").innerText = "Error: " + e.message;
        return;
      }

      document.getElementById("webcam-container").appendChild(webcam.canvas);
      labelContainer = document.getElementById("label-container");
      labelContainer.innerHTML = ""; // ล้างผลลัพธ์เก่า
      for (let i = 0; i < maxPredictions; i++) {
        labelContainer.appendChild(document.createElement("div"));
      }
    }

    async function loop() {
      webcam.update();
      await predict();
      window.requestAnimationFrame(loop);
    }

    async function predict() {
      const prediction = await model.predict(webcam.canvas);
      for (let i = 0; i < maxPredictions; i++) {
        const classPrediction = prediction[i].className + ": " + prediction[i].probability.toFixed(2);
        labelContainer.childNodes[i].innerHTML = classPrediction;
      }
    }

    function stop() {
      if (webcam) {
        webcam.stop();
        document.getElementById("webcam-container").innerHTML = '';
        document.getElementById("label-container").innerHTML = '';
      }
    }
  </script>
</body>
</html>
