<script>
    window.addEventListener("load", function () {
        const scanner = new jscanify();

        let authToken;
        let authTicket;

        const paperWidth = 1000;
        const paperHeight = 800;

        const originalCanvas = document.getElementById("originalCanvas");
        const highlightedCanvas = document.getElementById("highlightedCanvas");
        const extractedCanvas = document.getElementById("extractedCanvas");

        const originalCanvasCtx = originalCanvas.getContext("2d");
        const highlightedCanvasCtx = highlightedCanvas.getContext("2d");

        const fileInput = document.getElementById("fileInput");
        const downloadButton = document.getElementById("downloadButton");

        fileInput.addEventListener("change", function (event) {
            const file = event.target.files[0];
            if (file) {
                const reader = new FileReader();
                reader.onload = function (e) {
                    const image = new Image();
                    image.src = e.target.result;

                    image.onload = function () {
                        originalCanvas.width = image.width;
                        originalCanvas.height = image.height;

                        // 在第一個Canvas上繪製原始圖像
                        originalCanvasCtx.drawImage(image, 0, 0);

                        // 在第二個Canvas上繪製原始圖像
                        highlightedCanvas.width = image.width;
                        highlightedCanvas.height = image.height;
                        highlightedCanvasCtx.drawImage(image, 0, 0);

                        // Highlight凸顯出文件邊界並將其繪製在第二個Canvas上
                        const resultCanvas = scanner.highlightPaper(originalCanvas);
                        highlightedCanvasCtx.drawImage(resultCanvas, 0, 0);

                        // 將裁剪區域從highlightCanvas繪製到extractedCanvas上
                        const extractedCanvasResult = scanner.extractPaper(image, paperWidth, paperHeight);
                        extractedCanvas.width = extractedCanvasResult.width;
                        extractedCanvas.height = extractedCanvasResult.height;
                        extractedCanvas.getContext("2d").drawImage(extractedCanvasResult, 0, 0);

                        // 顯示下載按鈕
                        downloadButton.style.display = "block";

                        // 要求token 的Rest API
                        requestToken();
                    };
                };
                reader.readAsDataURL(file);
            }
        });

        function requestToken() {
            // 建立 FormData 物件
            const formData = new URLSearchParams();
            formData.append('apiPsw', "ctb");

            // 透過 AJAX 向目的網址要求 token
            $.ajax({
                url: "http://192.168.112.108:8022/api/form/TW_ID-gay4yOfi/requestToken",
                type: "POST",                    
                data: formData.toString(),
                contentType: "application/x-www-form-urlencoded",
                processData: true,
                success: function (response) {
                    //console.log(formData.toString());
                    console.log("Token Request Response:", response);
                    authToken = response.token;
                    //console.log(authToken);

                    // 取得token後將token及圖片上傳
                    uploadAndWait(authToken);
                },
                error: function (error) {
                    console.error("Error requesting token:", error);
                }
            });
        }

        function uploadAndWait(token){
            console.log("authToken: " + token);

            // 取得剪裁過後的圖片
            const extractedCanvas = document.getElementById("extractedCanvas")
            const extractedDataURL = extractedCanvas.toDataURL("image/png");
            const extractedBlob = dataURLtoBlob(extractedDataURL);
            const extractedFile = new File([extractedBlob], 'extracted.png');

            // 建立 FormData 物件
            const formData = new FormData();
            formData.append('token', token);
            formData.append('file', extractedFile);

            // 透過 AJAX 向目的網址要求 uploadAndWait
            $.ajax({
                url: "http://192.168.112.108:8022/api/form/TW_ID-gay4yOfi/uploadAndWait",
                type: "POST",                    
                data: formData,
                contentType: false,
                processData: false,
                success: function (response) {
                    console.log("Upload and Wait Response:", response);
                    authTicket = response.ticket;
                },
                error: function (error) {
                    console.error("Error uploading and waiting:", error);
                }
            });
        }
    });

    // 下載已裁切且梯形修正圖片的方法
    function downloadExtractedCanvas() {
        const extractedCanvas = document.getElementById("extractedCanvas");
        const dataUrl = extractedCanvas.toDataURL("image/png");

        // 將 dataURL 轉換為 Blob
        const blob = dataURLtoBlob(dataUrl);

        // 創建 Blob URL
        const blobUrl = URL.createObjectURL(blob);

        // 創建一個 a 標籤用於下載
        const a = document.createElement("a");

        // 指定 Blob URL 給 a 的 href
        a.href = blobUrl;

        // 使用當下日期時間作為檔案名稱
        const now = new Date();
        const year = now.getFullYear();
        const month = String(now.getMonth() + 1).padStart(2, '0');
        const day = String(now.getDate()).padStart(2, '0');
        const hours = String(now.getHours()).padStart(2, '0');
        const minutes = String(now.getMinutes()).padStart(2, '0');
        const seconds = String(now.getSeconds()).padStart(2, '0');
        const timestamp = `${year}${month}${day}${hours}${minutes}${seconds}`;

        // 將 Blob URL 分配給 a 的 download 屬性
        a.download = `${timestamp}.png`;

        // 模擬點擊下載
        document.body.appendChild(a);
        a.click();

        // 釋放 Blob URL
        URL.revokeObjectURL(blobUrl);

        // 移除 a 元素
        document.body.removeChild(a);
    }

    // 將 dataURL 轉換為 Blob
    function dataURLtoBlob(dataUrl) {
        const arr = dataUrl.split(',');
        const mime = arr[0].match(/:(.*?);/)[1];
        const bstr = atob(arr[1]);
        let n = bstr.length;
        const u8arr = new Uint8Array(n);
        while (n--) {
            u8arr[n] = bstr.charCodeAt(n);
        }
        return new Blob([u8arr], { type: mime });
    }
</script>

<template>
    <div style="padding: 30px">
        <h1 style="margin: 0">20240124new測試剪裁工具</h1>

        <!-- file input for capturing images -->
        <input type="file" accept="image/*" capture="camera" id="fileInput">

        <!-- 第一張圖片，拍照或上傳的原始圖片 -->
        <canvas id="originalCanvas"></canvas>

        <!-- 第二張圖片，拍照或上傳的圖片並標示文件邊界 -->
        <canvas id="highlightedCanvas"></canvas>

        <!-- 第三張圖片，標示的範圍 -->
        <canvas id="extractedCanvas"></canvas>

        <!-- 下載按鈕的容器 -->
        <div id="downloadButtonContainer">
            <!-- 下載按鈕，初始時設置為不顯示 -->
            <button id="downloadButton" style="display: none;" v-on:click="downloadExtractedCanvas()" v-on:touchstart="downloadExtractedCanvas()">下載剪裁過後的圖片</button>
        </div>
    </div>
</template>

<style scoped>
    html, body {
        margin: 0;
    }

    * {
        font-family: system-ui;
    }

    h1{
        font-weight: 400;
    }

    canvas {
        max-height: 400px;
        max-width: 400px;
    }

    #downloadButtonContainer {
        margin-top: 10px;
    }
</style>