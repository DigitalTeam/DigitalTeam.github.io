<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ระบบแจ้งเรื่องร้องเรียน</title>
    <!-- ใช้ Tailwind CSS เพื่อความสวยงามแบบรวดเร็ว -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- LINE LIFF SDK -->
    <script src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
</head>
<body class="bg-gray-100 min-h-screen p-4">

    <div class="max-w-md mx-auto bg-white rounded-xl shadow-md overflow-hidden p-6 mt-6">
        <h2 class="text-2xl font-bold mb-4 text-gray-800 text-center">📢 แจ้งเรื่องร้องเรียน</h2>
        
        <div id="user-profile" class="flex items-center space-x-4 mb-6 p-3 bg-gray-50 rounded-lg hidden">
            <img id="user-img" class="w-12 h-12 rounded-full" src="" alt="Profile">
            <div>
                <p class="text-sm text-gray-500">ผู้แจ้งเรื่อง</p>
                <p id="user-name" class="font-medium text-gray-800"></p>
            </div>
        </div>

        <form id="report-form">
            <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">หัวข้อเรื่องร้องเรียน</label>
                <input type="text" id="topic" required class="w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-green-500">
            </div>

            <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">รายละเอียด</label>
                <textarea id="detail" rows="4" required class="w-full px-3 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-green-500"></textarea>
            </div>

            <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">ตำแหน่งที่ตั้ง (GPS)</label>
                <button type="button" id="btn-location" class="w-full bg-blue-500 text-white py-2 rounded-lg font-medium hover:bg-blue-600 transition">
                    📍 กดเพื่อดึงพิกัดปัจจุบัน
                </button>
                <p id="location-status" class="text-xs text-gray-500 mt-1 text-center"></p>
            </div>

            <button type="submit" id="btn-submit" class="w-full bg-green-500 text-white py-3 rounded-lg font-bold hover:bg-green-600 transition mt-4">
                ส่งเรื่องร้องเรียน
            </button>
        </form>
    </div>

    <script>
        // ⚠️ นำ URL ของ Web app จาก Google Apps Script ที่คัดลอกไว้มาวางที่นี่
        const BACKEND_URL = "https://script.google.com/macros/s/AKfycbxiAUr4Alcg0du7ZOyOxXWs60FYmqBw_itlWtEpG7uuQffisEvx5NfUuC_N_1YUaGdHeg/exec"; 
        
        let userId = "";
        let userName = "";
        let latitude = "";
        let longitude = "";

        // 1. เริ่มต้นการทำงานของ LIFF
        async function initLIFF() {
            try {
                // ⚠️ เปลี่ยนเป็น LIFF ID ของคุณที่สร้างใน LINE Developers
                await liff.init({ liffId: "2010716930-EKc20oMJ" }); 
                
                if (!liff.isLoggedIn()) {
                    liff.login(); // ถ้าไม่ได้ล็อกอิน ให้เด้งหน้าล็อกอิน LINE
                } else {
                    getUserProfile();
                }
            } catch (error) {
                console.error("LIFF Initialization failed", error);
            }
        }

        // 2. ดึงข้อมูลโปรไฟล์ผู้ใช้งาน LINE
        async function getUserProfile() {
            const profile = await liff.getProfile();
            userId = profile.userId;
            userName = profile.displayName;

            document.getElementById("user-img").src = profile.pictureUrl;
            document.getElementById("user-name").innerText = profile.displayName;
            document.getElementById("user-profile").classList.remove("hidden");
        }

        // 3. ฟังก์ชันดึงพิกัด GPS จากเบราว์เซอร์มือถือ
        document.getElementById("btn-location").addEventListener("click", () => {
            if (navigator.geolocation) {
                document.getElementById("location-status").innerText = "กำลังค้นหาสัญญาณ GPS...";
                navigator.geolocation.getCurrentPosition((position) => {
                    latitude = position.coords.latitude;
                    longitude = position.coords.longitude;
                    document.getElementById("location-status").innerText = `🎯 ได้รับพิกัดเรียบร้อย (${latitude}, ${longitude})`;
                    document.getElementById("btn-location").classList.replace("bg-blue-500", "bg-gray-500");
                }, (error) => {
                    document.getElementById("location-status").innerText = "❌ ไม่สามารถดึงพิกัดได้ โปรดเปิดสิทธิ์เข้าถึงตำแหน่ง";
                });
            } else {
                document.getElementById("location-status").innerText = "❌ อุปกรณ์นี้ไม่รองรับระบบ GPS";
            }
        });

        // 4. ส่งข้อมูลไปยังหลังบ้าน Google Sheets เมื่อกด Submit Form
        document.getElementById("report-form").addEventListener("submit", async (e) => {
            e.preventDefault();
            
            const btnSubmit = document.getElementById("btn-submit");
            btnSubmit.disabled = true;
            btnSubmit.innerText = "กำลังส่งข้อมูล...";

            const formData = {
                userId: userId,
                userName: userName,
                topic: document.getElementById("topic").value,
                detail: document.getElementById("detail").value,
                latitude: latitude,
                longitude: longitude
            };

            try {
                // ยิงคำขอแบบ POST ไปยัง Google Apps Script
                const response = await fetch(BACKEND_URL, {
                    method: "POST",
                    body: JSON.stringify(formData)
                });
                
                const result = await response.json();
                
                if (result.status === "success") {
                    alert("ส่งเรื่องร้องเรียนสำเร็จ!");
                    liff.closeWindow(); // ปิดหน้าต่าง LIFF ย้อนกลับไปหน้าแชท LINE อัตโนมัติ
                } else {
                    alert("เกิดข้อผิดพลาด: " + result.message);
                    btnSubmit.disabled = false;
                    btnSubmit.innerText = "ส่งเรื่องร้องเรียน";
                }
            } catch (error) {
                alert("เกิดข้อผิดพลาดในการเชื่อมต่อหลังบ้าน");
                btnSubmit.disabled = false;
                btnSubmit.innerText = "ส่งเรื่องร้องเรียน";
            }
        });

        // รันสคริปต์ LIFF เมื่อเปิดหน้าเว็บ
        initLIFF();
    </script>
</body>
</html>
