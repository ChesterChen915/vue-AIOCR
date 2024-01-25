<template>
    <div style="padding: 30px">
        <h1 style="margin: 0">測試剪裁工具</h1>

        <!-- file input for capturing images -->
        <input type="file" accept="image/*" capture="camera" ref="fileInput" @change="handleFileChange">

        <!-- 第一張圖片，拍照或上傳的原始圖片 -->
        <canvas ref="originalCanvas"></canvas>

        <!-- 第二張圖片，拍照或上傳的圖片並標示文件邊界 -->
        <canvas ref="highlightedCanvas"></canvas>

        <!-- 第三張圖片，標示的範圍 -->
        <canvas ref="extractedCanvas"></canvas>

        <!-- 下載按鈕的容器 -->
        <div id="downloadButtonContainer">
            <!-- 下載按鈕，初始時設置為不顯示 -->
            <button ref="downloadButton" style="display: none;" @click="downloadExtractedCanvas" @touchstart="downloadExtractedCanvas">下載剪裁過後的圖片</button>
        </div>
    </div>
</template>

<script>
    import axios from 'axios';

    const dataURLtoBlob = (dataUrl) => {
        const arr = dataUrl.split(',');
        const mime = arr[0].match(/:(.*?);/)[1];
        const bstr = atob(arr[1]);
        let n = bstr.length;
        const u8arr = new Uint8Array(n);
        while (n--) {
            u8arr[n] = bstr.charCodeAt(n);
        }
        return new Blob([u8arr], { type: mime });
    };

    export default{
        data(){
            return{
                authToken: null,
                authTicket: null,
                scanner: null,
            };
        },
    
        methods: {
            handleFileChange(event){
                const file = event.target.files[0];
                if(file){
                    const reader = new FileReader();
                    reader.onload = (e) => {
                        const image = new Image();
                        image.src = e.target.result;

                        image.onload = () => {
                            this.$refs.originalCanvas.width = image.width;
                            this.$refs.originalCanvas.height = image.height;

                            // 在第一個Canvas上繪製原始圖像
                            this.$refs.originalCanvas.getContext("2d").drawImage(image, 0, 0);

                            // 在第二個Canvas上繪製原始圖像
                            this.$refs.highlightedCanvas.width = image.width;
                            this.$refs.highlightedCanvas.height = image.height;
                            this.$refs.highlightedCanvas.getContext("2d").drawImage(image, 0, 0);

                            // Highlight凸顯出文件邊界並將其繪製在第二個Canvas上
                            const resultCanvas = this.scanner.highlightPaper(this.$refs.originalCanvas);
                            this.$refs.highlightedCanvas.getContext("2d").drawImage(resultCanvas, 0, 0);

                            // 將裁剪區域從highlightCanvas繪製到extractedCanvas上
                            const extractedCanvasResult = this.scanner.extractPaper(image, 1000, 800);
                            this.$refs.extractedCanvas.width = extractedCanvasResult.width;
                            this.$refs.extractedCanvas.height = extractedCanvasResult.height;
                            this.$refs.extractedCanvas.getContext("2d").drawImage(extractedCanvasResult, 0, 0);
                            
                            // 顯示下載按鈕
                            this.$refs.downloadButton.style.display = "block";

                            // 要求token 的Rest API
                            this.requestToken();
                        };
                    };
                    reader.readAsDataURL(file);
                }
            },

            requestToken(){
                const formData = new URLSearchParams();
                formData.append('apiPsw', "ctb");

                axios.post("http://192.168.112.108:8022/api/form/TW_ID-gay4yOfi/requestToken", formData)
                    .then((response) => {
                        this.authToken = response.data.token;
                        this.uploadAndWait(this.authToken);
                    })
                    .catch((error) => {
                        console.error("Error requesting token: ", error);
                    });
            },

            uploadAndWait(token) {
                const extractedCanvas = this.$refs.extractedCanvas;
                const extractedDataURL = extractedCanvas.toDataURL("image/png");
                const extractedBlob = dataURLtoBlob(extractedDataURL);
                const extractedFile = new File([extractedBlob], 'extracted.png');

                const formData = new FormData();
                formData.append('token', token);
                formData.append('file', extractedFile);

                const dataUrl = extractedDataURL;

                axios.post("http://192.168.112.108:8022/api/form/TW_ID-gay4yOfi/uploadAndWait", formData)
                    .then((response) => {
                        this.authTicket = response.data.ticket;
                    })
                    .catch((error) => {
                        console.error("Error uploading and waiting: ", error);
                    });
            },

            downloadExtractedCanvas() {
                const extractedCanvas = this.$refs.extractedCanvas;
                const dataURL = extractedCanvas.toDataURL("image/png");

                const blob = this.dataURLtoBlob(dataURL);
                const blobUrl = URL.createObjectURL(blob);

                const a = document.createElement("a");
                a.href = blobUrl;

                const now = new Data();
                const timestamp = `${now.getFullYear()}${String(now.getMonth() + 1).padStart(2, '0')}${String(now.getDate()).padStart(2, '0')}${String(now.getHours()).padStart(2, '0')}${String(now.getMinutes()).padStart(2, '0')}${String(now.getSeconds()).padStart(2, '0')}`;
                a.download = `${timestamp}.png`;

                document.body.appendChild(a);
                a.click();

                URL.revokeObjectURL(blobUrl);
                document.body.removeChild(a);
            },

        },

        mounted() {
            // 在Vue的mounted生命週期中初始化jscanify或其他需要的操作
            this.scanner = new jscanify();
        },       
    };
</script>

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