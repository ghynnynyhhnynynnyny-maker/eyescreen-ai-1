<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EyeScreen - ระบบคัดกรองภาวะซีด</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;600;700&display=swap" rel="stylesheet">
    
    <style>
        * { box-sizing: border-box; font-family: 'Sarabun', sans-serif; }
        body {
            background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
            margin: 0; padding: 20px;
            display: flex; justify-content: center; align-items: center; min-height: 100vh;
        }
        .app-card {
            background: white; width: 100%; max-width: 450px;
            border-radius: 24px; box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            padding: 30px; text-align: center;
        }
        .header h1 { font-size: 22px; color: #2c3e50; margin-bottom: 5px; }
        .header p { font-size: 14px; color: #7f8c8d; margin-top: 0; }
        
        .camera-box {
            width: 100%; aspect-ratio: 1 / 1;
            background-color: #222; border-radius: 20px;
            margin: 25px 0; position: relative; overflow: hidden;
            display: flex; justify-content: center; align-items: center;
        }
        /* วงกลมเล็งเป้าตรวจสีตรงกลางจอ */
        .target-ring {
            position: absolute;
            width: 40px; height: 40px;
            border: 3px solid #00eca6; border-radius: 50%;
            box-shadow: 0 0 10px rgba(0, 236, 166, 0.8);
            z-index: 10; pointer-events: none;
        }
        .target-label {
            position: absolute; top: -25px; left: 50%; transform: translateX(-50%);
            color: #00eca6; font-size: 11px; font-weight: bold; white-space: nowrap;
        }
        video {
            width: 100%; height: 100%; object-fit: cover;
        }
        .btn-action {
            background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
            color: white; border: none; width: 100%; padding: 15px;
            font-size: 18px; font-weight: 600; border-radius: 15px;
            cursor: pointer; transition: all 0.3s ease;
        }
        .result-section {
            margin-top: 25px; padding: 20px; border-radius: 15px;
            background-color: #f8f9fa; border: 1px dashed #e2e8f0; display: none;
        }
        .result-title { font-size: 14px; color: #7f8c8d; }
        .result-value { font-size: 26px; font-weight: 700; margin: 10px 0; }
        .result-tip { font-size: 12px; color: #95a5a6; line-height: 1.5; }
    </style>
</head>
<body>

<div class="app-card">
    <div class="header">
        <h1>EyeScreen Color 👁️</h1>
        <p>ระบบคัดกรองภาวะซีดจากสีเปลือกตาเบื้องต้น</p>
    </div>

    <div class="camera-box">
        <!-- วงกลมเป้าหมาย -->
        <div class="target-ring" id="target" style="display: none;">
            <div class="target-label">เล็งเป้าตรงนี้</div>
        </div>
        <video id="webcam" autoplay playsinline muted></video>
    </div>

    <button class="btn-action" id="main-btn" onclick="startCamera()">เปิดกล้องและตรวจสี</button>

    <div class="result-section" id="result-box">
        <div class="result-title">ผลวิเคราะห์ค่าสีบริเวณเยื่อบุตา</div>
        <div class="result-value" id="result-value">รอผล...</div>
        <div class="result-tip">
            *วิธีใช้: ดึงเปลือกตาล่างลงมา แล้วขยับมือถือให้สีของเยื่อบุตาขาวอยู่ตรงวงกลมสีเขียวกลางจอพอดี
        </div>
    </div>
</div>

<canvas id="hidden-canvas" style="display:none;"></canvas>

<script>
    const video = document.getElementById('webcam');
    const canvas = document.getElementById('hidden-canvas');
    const ctx = canvas.getContext('2d');
    let isAnalyzing = false;

    async function startCamera() {
        try {
            // ขอเปิดกล้องหลัง
            const stream = await navigator.mediaDevices.getUserMedia({
                video: { facingMode: "environment" }
            });
            video.srcObject = stream;
            
            document.getElementById('target').style.display = 'block';
            document.getElementById('result-box').style.display = 'block';
            document.getElementById('main-btn').style.display = 'none';
            
            isAnalyzing = true;
            requestAnimationFrame(analyzeColor);
        } catch (err) {
            alert("ไม่สามารถเปิดกล้องได้: " + err);
        }
    }

    function analyzeColor() {
        if (!isAnalyzing) return;

        if (video.readyState === video.HAVE_ENOUGH_DATA) {
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;
            
            // วาดภาพจากกล้องลง Canvas เพื่อดึงพิกเซลมาคำนวณ
            ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
            
            // ดึงค่าสีจากจุดกึ่งกลางหน้าจอ (บริเวณวงกลมเป้าหมาย)
            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2;
            const pixel = ctx.getImageData(centerX, centerY, 1, 1).data;
            
            const r = pixel[0]; // แดง
            const g = pixel[1]; // เขียว
            const b = pixel[2]; // น้ำเงิน

            const resultValue = document.getElementById('result-value');
            
            // คำนวณความแดง (เทียบสัดส่วนสีแดงกับสีเขียวและน้ำเงิน)
            if (r > 130 && r > (g + 35) && r > (b + 35)) {
                // สีแดงนำโด่ง ชัดเจน = สุขภาพดี สีชมพูแดงปกติ
                resultValue.innerHTML = "✅ สีเยื่อบุตาปกติ";
                resultValue.style.color = "#2ecc71";
            } else if (r > 110 && g > 110 && b > 110) {
                // ถ้าค่าสีอื่นๆ สูงพอกันแสดงว่าเป็นสีขาวหรือสีซีด
                resultValue.innerHTML = "⚠️ มีภาวะซีด/สีซีดจาง";
                resultValue.style.color = "#e74c3c";
            } else {
                resultValue.innerHTML = "🔍 กำลังหาตำแหน่งเยื่อบุตา...";
                resultValue.style.color = "#f39c12";
            }
        }
        
        requestAnimationFrame(analyzeColor);
    }
</script>
</body>
</html>
