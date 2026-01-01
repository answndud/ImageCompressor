# 🖼️ Image Compressor - 기술 문서

> 브라우저에서 이미지 압축을 구현하는 방법에 대한 기술적 설명

---

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [HTML5 Canvas API](#html5-canvas-api)
3. [File API와 이미지 로딩](#file-api와-이미지-로딩)
4. [이미지 압축 알고리즘](#이미지-압축-알고리즘)
5. [리사이징 구현](#리사이징-구현)
6. [포맷 변환](#포맷-변환)
7. [Blob API와 다운로드](#blob-api와-다운로드)
8. [비교 슬라이더 구현](#비교-슬라이더-구현)
9. [배치 처리](#배치-처리)
10. [JSZip을 이용한 ZIP 다운로드](#jszip을-이용한-zip-다운로드)
11. [성능 최적화](#성능-최적화)

---

## 프로젝트 개요

### 핵심 목표
브라우저에서 서버 전송 없이 이미지를 압축하는 도구를 만들었습니다. 핵심 기술은 **HTML5 Canvas API**입니다.

### 데이터 흐름
```
File Input → FileReader → Image Object → Canvas → Blob → Download
```

### 주요 API
| API | 역할 |
|-----|------|
| File API | 사용자 파일 접근 |
| FileReader | 파일 → DataURL 변환 |
| Canvas API | 이미지 처리 및 압축 |
| Blob API | 바이너리 데이터 생성 |
| JSZip | ZIP 파일 생성 |

---

## HTML5 Canvas API

### Canvas란?
Canvas는 JavaScript로 2D 그래픽을 그릴 수 있는 HTML 요소입니다. 이미지를 Canvas에 그린 후 다양한 포맷과 품질로 내보낼 수 있습니다.

### 기본 사용법

```javascript
// Canvas 생성
const canvas = document.createElement('canvas');
const ctx = canvas.getContext('2d');

// 크기 설정
canvas.width = 800;
canvas.height = 600;

// 이미지 그리기
ctx.drawImage(image, 0, 0, 800, 600);
```

### drawImage 메서드

`drawImage`는 다양한 인자를 받습니다:

```javascript
// 기본 형태: 원본 크기로 그리기
ctx.drawImage(img, dx, dy);

// 크기 조절
ctx.drawImage(img, dx, dy, dWidth, dHeight);

// 부분 추출 + 크기 조절
ctx.drawImage(img, sx, sy, sWidth, sHeight, dx, dy, dWidth, dHeight);
```

| 인자 | 의미 |
|------|------|
| `sx, sy` | 소스 이미지에서 자를 시작점 |
| `sWidth, sHeight` | 소스에서 자를 크기 |
| `dx, dy` | Canvas에 그릴 위치 |
| `dWidth, dHeight` | Canvas에 그릴 크기 |

---

## File API와 이미지 로딩

### 드래그 앤 드롭

```javascript
dropZone.addEventListener('dragover', (e) => {
    e.preventDefault(); // 기본 동작 방지 필수!
    dropZone.classList.add('dragover');
});

dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    const files = e.dataTransfer.files;
    handleFiles(files);
});
```

### FileReader로 이미지 읽기

```javascript
function loadImage(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = (e) => {
            const img = new Image();
            img.onload = () => resolve(img);
            img.onerror = reject;
            img.src = e.target.result; // Data URL
        };
        
        reader.onerror = reject;
        reader.readAsDataURL(file);
    });
}
```

### FileReader 메서드들

| 메서드 | 결과 | 용도 |
|--------|------|------|
| `readAsDataURL()` | Base64 Data URL | 이미지 미리보기 |
| `readAsArrayBuffer()` | ArrayBuffer | 바이너리 처리 |
| `readAsText()` | 문자열 | 텍스트 파일 |
| `readAsBinaryString()` | 이진 문자열 | 레거시 |

### 파일 유효성 검사

```javascript
function validateFile(file) {
    // MIME 타입 검사
    const validTypes = /^image\/(jpeg|png|webp)$/;
    if (!validTypes.test(file.type)) {
        throw new Error('지원하지 않는 형식');
    }
    
    // 크기 검사 (10MB)
    const maxSize = 10 * 1024 * 1024;
    if (file.size > maxSize) {
        throw new Error('10MB 초과');
    }
    
    return true;
}
```

---

## 이미지 압축 알고리즘

### Canvas toBlob 메서드

핵심 압축은 `canvas.toBlob()` 메서드가 담당합니다:

```javascript
canvas.toBlob(
    (blob) => {
        // blob: 압축된 이미지 데이터
        console.log('원본:', file.size);
        console.log('압축:', blob.size);
    },
    'image/jpeg',  // MIME 타입
    0.8            // 품질 (0.0 ~ 1.0)
);
```

### 품질 vs 파일 크기

```
품질 1.0 (100%) → 최고 품질, 큰 파일
품질 0.8 (80%)  → 좋은 품질, 적당한 크기 (권장)
품질 0.5 (50%)  → 중간 품질, 작은 파일
품질 0.1 (10%)  → 낮은 품질, 최소 파일
```

### 포맷별 압축 특성

| 포맷 | 품질 인자 | 특징 |
|------|----------|------|
| JPEG | 0.0 ~ 1.0 | 손실 압축, 사진에 적합 |
| PNG | 무시됨 | 무손실, 투명 지원 |
| WebP | 0.0 ~ 1.0 | JPEG보다 20-30% 효율 |

### 압축 함수 구현

```javascript
function compressImage(image, quality, format) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    // 원본 크기 유지
    canvas.width = image.naturalWidth;
    canvas.height = image.naturalHeight;
    
    // 이미지 그리기
    ctx.drawImage(image, 0, 0);
    
    // 압축된 Blob 반환
    return new Promise((resolve) => {
        canvas.toBlob(resolve, `image/${format}`, quality);
    });
}
```

### Data URL vs Blob

```javascript
// Data URL: Base64 문자열 (메모리 비효율)
const dataUrl = canvas.toDataURL('image/jpeg', 0.8);
// "data:image/jpeg;base64,/9j/4AAQSkZJRg..."

// Blob: 이진 데이터 (메모리 효율)
canvas.toBlob((blob) => {
    // Blob { size: 12345, type: "image/jpeg" }
}, 'image/jpeg', 0.8);
```

**권장**: 다운로드용은 Blob, 미리보기용은 Data URL

---

## 리사이징 구현

### 기본 리사이징

```javascript
function resizeImage(image, targetWidth, targetHeight) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    canvas.width = targetWidth;
    canvas.height = targetHeight;
    
    // 새 크기로 그리기 = 리사이징
    ctx.drawImage(image, 0, 0, targetWidth, targetHeight);
    
    return canvas;
}
```

### 비율 유지 리사이징

```javascript
function resizeWithAspectRatio(image, maxWidth, maxHeight) {
    let { naturalWidth: w, naturalHeight: h } = image;
    const ratio = w / h;
    
    if (w > maxWidth) {
        w = maxWidth;
        h = w / ratio;
    }
    
    if (h > maxHeight) {
        h = maxHeight;
        w = h * ratio;
    }
    
    return { width: Math.round(w), height: Math.round(h) };
}
```

### 리사이즈 입력 동기화

```javascript
// 너비 변경 시 높이 자동 계산
resizeWidth.addEventListener('input', () => {
    if (maintainRatio.checked) {
        const ratio = originalImage.naturalWidth / originalImage.naturalHeight;
        resizeHeight.value = Math.round(resizeWidth.value / ratio);
    }
});

// 높이 변경 시 너비 자동 계산
resizeHeight.addEventListener('input', () => {
    if (maintainRatio.checked) {
        const ratio = originalImage.naturalWidth / originalImage.naturalHeight;
        resizeWidth.value = Math.round(resizeHeight.value * ratio);
    }
});
```

### 이미지 보간 품질

```javascript
// 보간 품질 설정 (리사이징 시 품질에 영향)
ctx.imageSmoothingEnabled = true;
ctx.imageSmoothingQuality = 'high'; // 'low', 'medium', 'high'
```

---

## 포맷 변환

### MIME 타입 매핑

```javascript
const mimeTypes = {
    jpeg: 'image/jpeg',
    png: 'image/png',
    webp: 'image/webp'
};

function convertFormat(image, format, quality) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    canvas.width = image.naturalWidth;
    canvas.height = image.naturalHeight;
    ctx.drawImage(image, 0, 0);
    
    return new Promise((resolve) => {
        canvas.toBlob(resolve, mimeTypes[format], quality);
    });
}
```

### PNG 투명 배경 처리

```javascript
// PNG → JPEG 변환 시 투명 부분을 흰색으로
function convertPngToJpeg(image) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    
    canvas.width = image.naturalWidth;
    canvas.height = image.naturalHeight;
    
    // 흰색 배경 먼저 그리기
    ctx.fillStyle = '#FFFFFF';
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    // 이미지 그리기
    ctx.drawImage(image, 0, 0);
    
    return canvas;
}
```

### WebP 지원 확인

```javascript
function isWebPSupported() {
    const canvas = document.createElement('canvas');
    canvas.width = 1;
    canvas.height = 1;
    return canvas.toDataURL('image/webp').startsWith('data:image/webp');
}
```

---

## Blob API와 다운로드

### Blob 객체

Blob(Binary Large Object)은 불변의 원시 데이터를 나타냅니다:

```javascript
// Blob 생성
const blob = new Blob([data], { type: 'image/jpeg' });

// 속성
console.log(blob.size);  // 바이트 크기
console.log(blob.type);  // MIME 타입
```

### Object URL 생성

```javascript
// Blob → URL 변환
const url = URL.createObjectURL(blob);
// "blob:http://localhost:8000/550e8400-e29b-41d4-a716-446655440000"

// 사용 후 메모리 해제 필수!
URL.revokeObjectURL(url);
```

### 다운로드 구현

```javascript
function downloadBlob(blob, filename) {
    // Object URL 생성
    const url = URL.createObjectURL(blob);
    
    // 숨겨진 앵커 태그 생성
    const a = document.createElement('a');
    a.href = url;
    a.download = filename; // 다운로드 파일명
    
    // 클릭 시뮬레이션
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    
    // 메모리 정리
    URL.revokeObjectURL(url);
}
```

### 파일명 처리

```javascript
function getOutputFilename(originalName, format) {
    // 확장자 교체
    // "photo.png" → "photo.jpeg"
    return originalName.replace(/\.[^.]+$/, `.${format}`);
}
```

---

## 비교 슬라이더 구현

### HTML 구조

```html
<div class="comparison-view">
    <div class="comparison-original">
        <img src="original.jpg" alt="Original">
        <span class="comparison-label">원본</span>
    </div>
    <div class="comparison-compressed">
        <img src="compressed.jpg" alt="Compressed">
        <span class="comparison-label">압축</span>
    </div>
    <div class="comparison-slider"></div>
</div>
```

### CSS clip-path 활용

```css
.comparison-original {
    position: absolute;
    width: 100%;
    height: 100%;
    /* 오른쪽 50%를 잘라냄 */
    clip-path: inset(0 50% 0 0);
}

.comparison-slider {
    position: absolute;
    left: 50%;
    width: 4px;
    height: 100%;
    background: var(--accent);
    cursor: ew-resize;
}
```

### 슬라이더 드래그

```javascript
const slider = document.getElementById('comparisonSlider');
const original = document.querySelector('.comparison-original');
let isDragging = false;

function updateSlider(clientX) {
    const rect = container.getBoundingClientRect();
    const percent = ((clientX - rect.left) / rect.width) * 100;
    const clampedPercent = Math.max(0, Math.min(100, percent));
    
    // 슬라이더 위치 업데이트
    slider.style.left = `${clampedPercent}%`;
    
    // clip-path 업데이트
    original.style.clipPath = `inset(0 ${100 - clampedPercent}% 0 0)`;
}

slider.addEventListener('mousedown', () => isDragging = true);
document.addEventListener('mouseup', () => isDragging = false);
document.addEventListener('mousemove', (e) => {
    if (isDragging) updateSlider(e.clientX);
});

// 터치 지원
container.addEventListener('touchmove', (e) => {
    updateSlider(e.touches[0].clientX);
});
```

### clip-path inset 설명

```css
/* inset(top right bottom left) */
clip-path: inset(0 30% 0 0);
/*
  top: 0 (위에서 자르지 않음)
  right: 30% (오른쪽 30% 잘라냄)
  bottom: 0 (아래서 자르지 않음)
  left: 0 (왼쪽에서 자르지 않음)
*/
```

---

## 배치 처리

### 여러 파일 처리

```javascript
async function processMultipleFiles(files) {
    const results = [];
    
    for (const file of files) {
        try {
            const dataUrl = await readFileAsDataUrl(file);
            const image = await loadImage(dataUrl);
            const blob = await compressImage(image, quality, format);
            
            results.push({
                name: file.name,
                original: file.size,
                compressed: blob.size,
                blob: blob
            });
        } catch (error) {
            console.error(`Failed to process ${file.name}:`, error);
        }
    }
    
    return results;
}
```

### Promise 기반 FileReader

```javascript
function readFileAsDataUrl(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = () => reject(new Error('File read failed'));
        reader.readAsDataURL(file);
    });
}
```

### 진행 상태 표시

```javascript
async function compressWithProgress(files, onProgress) {
    const total = files.length;
    
    for (let i = 0; i < total; i++) {
        const file = files[i];
        
        // 진행률 콜백
        onProgress({
            current: i + 1,
            total: total,
            percent: ((i + 1) / total) * 100,
            filename: file.name
        });
        
        await processFile(file);
    }
}

// 사용
compressWithProgress(files, (progress) => {
    console.log(`${progress.current}/${progress.total} 처리 중...`);
    progressBar.style.width = `${progress.percent}%`;
});
```

---

## JSZip을 이용한 ZIP 다운로드

여러 이미지를 압축한 후 개별 다운로드 대신 하나의 ZIP 파일로 묶어서 다운로드할 수 있습니다.

### JSZip 라이브러리

[JSZip](https://stuk.github.io/jszip/)은 브라우저에서 ZIP 파일을 생성하고 읽을 수 있는 JavaScript 라이브러리입니다.

```html
<!-- CDN으로 로드 -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
```

### 기본 사용법

```javascript
// JSZip 인스턴스 생성
const zip = new JSZip();

// 파일 추가
zip.file("hello.txt", "Hello World!");
zip.file("image.jpg", imageBlob);

// 폴더 생성
const folder = zip.folder("images");
folder.file("photo1.jpg", blob1);
folder.file("photo2.jpg", blob2);
```

### ZIP 파일 생성

```javascript
async function createZip(files) {
    const zip = new JSZip();
    
    // 모든 파일을 ZIP에 추가
    files.forEach(({name, blob}) => {
        zip.file(name, blob);
    });
    
    // ZIP Blob 생성
    const zipBlob = await zip.generateAsync({
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: { level: 6 }
    });
    
    return zipBlob;
}
```

### generateAsync 옵션

| 옵션 | 설명 |
|------|------|
| `type` | 출력 타입 (`blob`, `base64`, `uint8array` 등) |
| `compression` | 압축 방식 (`STORE`: 무압축, `DEFLATE`: 압축) |
| `compressionOptions.level` | 압축 레벨 (1-9, 높을수록 더 압축) |

### 전체 구현

```javascript
// 압축된 파일들을 저장할 배열
let batchCompressedFiles = [];

// 배치 압축 후 저장
async function compressAllBatch() {
    batchCompressedFiles = [];
    
    for (const file of files) {
        const blob = await compressImage(file);
        batchCompressedFiles.push({
            name: file.name.replace(/\.[^.]+$/, '.jpg'),
            blob: blob
        });
    }
    
    // ZIP 다운로드 버튼 활성화
    document.getElementById('downloadZipBtn').style.display = 'block';
}

// ZIP 다운로드
async function downloadAsZip() {
    const zip = new JSZip();
    
    // 모든 파일 추가
    batchCompressedFiles.forEach(({name, blob}) => {
        zip.file(name, blob);
    });
    
    // ZIP 생성 및 다운로드
    const zipBlob = await zip.generateAsync({
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: { level: 6 }
    });
    
    // 파일명에 날짜 포함
    const timestamp = new Date().toISOString().slice(0, 10);
    downloadBlob(zipBlob, `compressed_images_${timestamp}.zip`);
}
```

### 진행률 표시

```javascript
async function downloadAsZipWithProgress() {
    const zip = new JSZip();
    
    batchCompressedFiles.forEach(({name, blob}) => {
        zip.file(name, blob);
    });
    
    // 진행률 콜백과 함께 생성
    const zipBlob = await zip.generateAsync(
        { type: 'blob', compression: 'DEFLATE' },
        (metadata) => {
            // metadata.percent: 0-100 진행률
            progressBar.style.width = `${metadata.percent}%`;
            console.log(`ZIP 생성 중: ${metadata.percent.toFixed(0)}%`);
        }
    );
    
    return zipBlob;
}
```

### 메모리 효율적인 스트리밍

대용량 파일의 경우 `generateInternalStream`을 사용할 수 있습니다:

```javascript
zip.generateInternalStream({ type: 'blob' })
    .accumulate((metadata) => {
        console.log(`진행률: ${metadata.percent}%`);
    })
    .then((blob) => {
        downloadBlob(blob, 'archive.zip');
    });
```

---

## 성능 최적화

### 메모리 관리

```javascript
// Object URL 정리
function cleanup() {
    if (previewUrl) {
        URL.revokeObjectURL(previewUrl);
        previewUrl = null;
    }
}

// 새 이미지 로드 시 이전 것 정리
function loadNewImage(file) {
    cleanup();
    // ... 새 이미지 처리
}
```

### 큰 이미지 처리

```javascript
// 매우 큰 이미지는 단계적 리사이징
function stepResize(image, targetSize) {
    let { width, height } = image;
    let canvas = document.createElement('canvas');
    let ctx = canvas.getContext('2d');
    
    // 한 번에 50%씩 축소 (품질 유지)
    while (width > targetSize * 2 || height > targetSize * 2) {
        width = Math.round(width / 2);
        height = Math.round(height / 2);
        
        canvas.width = width;
        canvas.height = height;
        ctx.drawImage(image, 0, 0, width, height);
        
        // 다음 반복을 위해 캔버스를 소스로 사용
        image = canvas;
        canvas = document.createElement('canvas');
        ctx = canvas.getContext('2d');
    }
    
    // 최종 크기로 조절
    canvas.width = targetSize;
    canvas.height = targetSize;
    ctx.drawImage(image, 0, 0, targetSize, targetSize);
    
    return canvas;
}
```

### 지연 압축 (Debounce)

```javascript
let compressTimeout;

qualitySlider.addEventListener('input', () => {
    // 슬라이더 움직이는 동안 계속 압축하지 않음
    clearTimeout(compressTimeout);
    
    // 300ms 후 압축 실행
    compressTimeout = setTimeout(() => {
        compressImage();
    }, 300);
});
```

### Web Worker 고려사항

대용량 이미지의 경우 Web Worker 사용을 고려할 수 있지만, Canvas API는 메인 스레드에서만 작동합니다. OffscreenCanvas를 사용하면 Worker에서 Canvas 작업이 가능합니다:

```javascript
// 메인 스레드
const offscreen = canvas.transferControlToOffscreen();
worker.postMessage({ canvas: offscreen }, [offscreen]);

// Worker 내부
self.onmessage = (e) => {
    const canvas = e.data.canvas;
    const ctx = canvas.getContext('2d');
    // ... 이미지 처리
};
```

---

## 파일 크기 계산

### 바이트 포맷팅

```javascript
function formatBytes(bytes, decimals = 2) {
    if (bytes === 0) return '0 B';
    
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    
    return parseFloat((bytes / Math.pow(k, i)).toFixed(decimals)) + ' ' + sizes[i];
}

// 사용
formatBytes(1234);       // "1.21 KB"
formatBytes(1234567);    // "1.18 MB"
formatBytes(1234567890); // "1.15 GB"
```

### 절감률 계산

```javascript
function calculateSavings(originalSize, compressedSize) {
    const savings = (1 - compressedSize / originalSize) * 100;
    return savings.toFixed(1); // "25.3"
}

// UI 표시
const savings = calculateSavings(originalFile.size, compressedBlob.size);
savingsElement.textContent = `${savings}%`;
savingsElement.className = savings > 0 ? 'success' : 'warning';
```

---

## 결론

### 핵심 기술 요약

| 기능 | 사용 API |
|------|----------|
| 파일 읽기 | FileReader.readAsDataURL() |
| 이미지 로딩 | new Image() |
| 리사이징 | Canvas.drawImage() |
| 압축 | Canvas.toBlob(type, quality) |
| 다운로드 | URL.createObjectURL() + anchor.download |
| ZIP 생성 | JSZip.generateAsync() |

### 브라우저 호환성

모든 주요 브라우저에서 지원됩니다:
- Chrome 50+
- Firefox 45+
- Safari 11+
- Edge 79+

### 제한사항

1. **CORS**: 외부 이미지는 동일 출처 정책의 영향을 받음
2. **메모리**: 매우 큰 이미지(50MP+)는 메모리 부족 가능
3. **WebP**: 일부 구형 Safari에서 미지원

---

*마지막 업데이트: 2026년 1월*

