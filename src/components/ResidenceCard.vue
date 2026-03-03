<template>
  <div class="common-layout">
    <el-container class="full-height">
      <el-aside width="600px" class="sidebar">
        <el-scrollbar>
          <div class="sidebar-content">
            <h2 class="title">居住证信息生成</h2>
            
            <el-form :model="info" label-width="100px" size="large">
              <el-divider content-position="left">基本信息</el-divider>
              
              <el-form-item label="姓名">
                <el-input v-model="info.name" placeholder="请输入姓名" />
              </el-form-item>
              
              <el-form-item label="性别">
                <el-select v-model="info.gender" placeholder="请选择性别">
                  <el-option label="男" value="男" />
                  <el-option label="女" value="女" />
                </el-select>
              </el-form-item>
              
              <el-form-item label="民族">
                <el-input v-model="info.ethnicity" placeholder="如：汉" />
              </el-form-item>
              
              <el-form-item label="出生日期">
                <el-input v-model="info.birthDate" placeholder="格式：YYYY年M月D日" />
              </el-form-item>
              
              <el-form-item label="身份证号">
                <el-input v-model="info.idNumber" placeholder="18位身份证号" />
              </el-form-item>
              
              <el-form-item label="户籍地址">
                <el-input 
                  v-model="info.householdAddress" 
                  type="textarea" 
                  :rows="3" 
                  placeholder="请输入户籍地址" 
                />
              </el-form-item>
              
              <el-form-item label="现居住地址">
                <el-input 
                  v-model="info.currentAddress" 
                  type="textarea" 
                  :rows="3" 
                  placeholder="请输入现居住地址" 
                />
              </el-form-item>
              
              <el-divider content-position="left">签发信息</el-divider>
              
              <el-form-item label="签发机关">
                <el-input v-model="info.issuingAuthority" placeholder="如：XX市公安局XX分局" />
              </el-form-item>
              
              <el-form-item label="有效期限">
                <el-date-picker
                  v-model="validityRange"
                  type="daterange"
                  unlink-panels
                  start-placeholder="开始日期"
                  end-placeholder="结束日期"
                  :default-time="['00:00:00', '00:00:00']"
                  format="YYYY年MM月DD日"
                />
              </el-form-item>
              
              <el-divider content-position="left">水印设置</el-divider>
              <el-form-item label="水印一">
                <el-input v-model="info.watermark1" placeholder="例如：爱山东" />
              </el-form-item>
              <el-form-item label="水印二">
                <el-input v-model="info.watermark2" placeholder="例如：电子认证" />
              </el-form-item>
              <el-form-item label="水印三">
                <el-input v-model="info.watermark3" placeholder="例如：有效期限开始日期" disabled />
              </el-form-item>
              
              <el-divider content-position="left">图片上传</el-divider>
              
              <el-form-item label="证件照片">
                <el-upload
                  class="upload-demo"
                  action="#"
                  :auto-upload="false"
                  :show-file-list="false"
                  :on-change="handlePhotoUpload"
                  accept="image/*"
                >
                  <el-button type="primary" plain>点击上传照片</el-button>
                  <template #tip>
                    <div v-if="photoUrl" class="upload-preview">
                      <img :src="photoUrl" class="preview-img" />
                      <span class="preview-text">已选择</span>
                    </div>
                  </template>
                </el-upload>
              </el-form-item>
              
              <el-form-item label="二维码">
                <el-upload
                  class="upload-demo"
                  action="#"
                  :auto-upload="false"
                  :show-file-list="false"
                  :on-change="handleQrUpload"
                  accept="image/*"
                >
                  <el-button type="primary" plain>点击上传二维码</el-button>
                  <template #tip>
                    <div v-if="qrUrl" class="upload-preview">
                      <img :src="qrUrl" class="preview-img" />
                      <span class="preview-text">已选择</span>
                    </div>
                  </template>
                </el-upload>
              </el-form-item>
              
              <el-collapse v-model="activeNames" class="settings-collapse">
                <el-collapse-item title="位置微调 (高级设置)" name="1">
                  <div v-for="(style, key) in fieldStyles" :key="key" class="pos-group">
                    <span class="pos-label">{{ getLabel(key) }}</span>
                    <div class="pos-inputs">
                      <el-input-number v-model="style.x" size="small" controls-position="right" placeholder="X" @change="drawCanvas" />
                      <el-input-number v-model="style.y" size="small" controls-position="right" placeholder="Y" @change="drawCanvas" />
                      <el-input-number v-if="style.w !== undefined" v-model="style.w" size="small" controls-position="right" placeholder="W" @change="drawCanvas" />
                      <el-input-number v-if="style.h !== undefined" v-model="style.h" size="small" controls-position="right" placeholder="H" @change="drawCanvas" />
                    </div>
                  </div>
                </el-collapse-item>
              </el-collapse>
              
              <div class="actions">
                <el-button type="primary" size="large" @click="exportImage" style="width: 100%">导出图片</el-button>
              </div>
            </el-form>
          </div>
        </el-scrollbar>
      </el-aside>
      
      <el-main class="preview-area">
        <div class="zoom-controls">
          <el-button-group>
            <el-button @click="zoomIn" :icon="Plus" circle />
            <el-button @click="resetZoom" circle>{{ Math.round(scale * 100) }}%</el-button>
            <el-button @click="zoomOut" :icon="Minus" circle />
          </el-button-group>
        </div>
        
        <div class="canvas-wrapper" :style="{ transform: `scale(${scale})` }">
          <canvas ref="canvasRef"></canvas>
        </div>
      </el-main>
    </el-container>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, watch } from 'vue';
import { Plus, Minus } from '@element-plus/icons-vue';
import type { UploadFile } from 'element-plus';
import QRCode from 'qrcode';
import bgImageSrc from '../assets/bgi.png';

// State
const activeNames = ref<string[]>([]);
const scale = ref(0.4);
const canvasRef = ref<HTMLCanvasElement | null>(null);
const bgImage = new Image();
const photoImage = new Image();
const qrImage = new Image();

const photoUrl = ref('');
const qrUrl = ref('');
const validityRange = ref<[Date, Date] | null>(null);

const info = reactive({
  name: '',
  gender: '',
  ethnicity: '',
  birthDate: '',
  idNumber: '',
  householdAddress: '',
  currentAddress: '',
  issuingAuthority: '',
  validityPeriod: '',
  watermark1: '爱山东',
  watermark2: '电子认证',
  watermark3: ''
});

interface FieldStyle {
  x: number;
  y: number;
  w?: number;
  h?: number;
  fontSize?: number;
  lineHeight?: number;
}

interface FieldStyles {
  name: FieldStyle;
  gender: FieldStyle;
  ethnicity: FieldStyle;
  birthDate: FieldStyle;
  idNumber: FieldStyle;
  householdAddress: FieldStyle;
  currentAddress: FieldStyle;
  issuingAuthority: FieldStyle;
  validityPeriod: FieldStyle;
  watermark1: FieldStyle;
  watermark2: FieldStyle;
  watermark3: FieldStyle;
  photo: FieldStyle;
  qrCode: FieldStyle;
  [key: string]: FieldStyle;
}

const fieldStyles = reactive<FieldStyles>({
  name: { x: 420, y: 460 },
  gender: { x: 420, y: 550 },
  ethnicity: { x: 670, y: 550 },
  birthDate: { x: 420, y: 650 },
  idNumber: { x: 420, y: 740 },
  householdAddress: { x: 420, y: 810, w: 420, lineHeight: 50 },
  currentAddress: { x: 420, y: 990, w: 420, fontSize: 34, lineHeight: 45 },
  photo: { x: 929, y: 420, w: 340, h: 397 },
  qrCode: { x: 1064, y: 844, w: 205, h: 215 },
  issuingAuthority: { x: 560, y: 1815,fontSize: 40},
  validityPeriod: { x: 560, y: 1870,fontSize: 40},
  watermark1: { x: 700, y: 1230, fontSize: 22 , w: 100},
  watermark2: { x: 540, y: 1230, fontSize: 22 , w: 100},
  watermark3: { x: 150, y: 1210, fontSize: 22 , w: 160}
});

// Helpers
const getLabel = (key: string) => {
  const map: Record<string, string> = {
    name: '姓名',
    gender: '性别',
    ethnicity: '民族',
    birthDate: '出生日期',
    idNumber: '身份证号',
    householdAddress: '户籍地址',
    currentAddress: '现居住地址',
    photo: '照片',
    qrCode: '二维码',
    issuingAuthority: '签发机关',
    validityPeriod: '有效期限',
    watermark1: '水印一',
    watermark2: '水印二',
    watermark3: '水印三'
  };
  return map[key] || key;
};

// Image Loading
const loadImages = async () => {
  return new Promise<void>((resolve) => {
    bgImage.src = bgImageSrc;
    bgImage.onload = () => {
      resolve();
      drawCanvas();
    };
  });
};

const handlePhotoUpload = (uploadFile: UploadFile) => {
  if (uploadFile.raw) {
    const reader = new FileReader();
    reader.onload = (e) => {
      photoUrl.value = e.target?.result as string;
      photoImage.src = photoUrl.value;
      photoImage.onload = () => drawCanvas();
    };
    reader.readAsDataURL(uploadFile.raw);
  }
};

const handleQrUpload = (uploadFile: UploadFile) => {
  if (uploadFile.raw) {
    const reader = new FileReader();
    reader.onload = (e) => {
      qrUrl.value = e.target?.result as string;
      qrImage.src = qrUrl.value;
      qrImage.onload = () => drawCanvas();
    };
    reader.readAsDataURL(uploadFile.raw);
  }
};

const formatDate = (d: Date) => {
  const y = d.getFullYear();
  const m = `${d.getMonth() + 1}`.padStart(2, '0');
  const day = `${d.getDate()}`.padStart(2, '0');
  return `${y}.${m}.${day}`;
};

const addMonths = (d: Date, m: number) => {
  const nd = new Date(d);
  nd.setHours(0, 0, 0, 0);
  nd.setMonth(nd.getMonth() + m);
  return new Date(nd.getFullYear(), nd.getMonth(), nd.getDate());
};

const initDefaultValidity = () => {
  if (validityRange.value && validityRange.value.length === 2) return;
  const now = new Date();
  now.setHours(0, 0, 0, 0);
  const earliest = addMonths(now, -9);
  const latest = addMonths(now, -3);
  const t = earliest.getTime() + Math.random() * (latest.getTime() - earliest.getTime());
  const start = new Date(t);
  const end = addMonths(start, 12);
  validityRange.value = [start, end];
};

const generateQRCode = async () => {
  if (info.name && info.idNumber) {
    const text = `${info.idNumber}^${info.name}^${info.idNumber}`;
    try {
      const url = await QRCode.toDataURL(text, { errorCorrectionLevel: 'M', margin: 0 });
      qrUrl.value = url;
      qrImage.src = url;
      qrImage.onload = () => drawCanvas();
    } catch (err) {
      console.error(err);
    }
  }
};

// Watch for changes in name and idNumber to auto-generate QR code
watch([() => info.name, () => info.idNumber], () => {
  generateQRCode();
}, { immediate: true });

watch(validityRange, (val) => {
  if (val && val.length === 2) {
    const start = new Date(val[0].getFullYear(), val[0].getMonth(), val[0].getDate());
    const end = new Date(val[1].getFullYear(), val[1].getMonth(), val[1].getDate());
    const startStr = formatDate(start);
    const endStr = formatDate(end);
    info.validityPeriod = `${startStr}-${endStr}`;
    info.watermark3 = startStr + " 00:00:00";
  } else {
    info.validityPeriod = '';
    info.watermark3 = '';
  }
}, { immediate: true });

// Canvas Drawing Logic
const drawCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas || !bgImage.complete) return;

  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // Set canvas size to match image size (high resolution)
  if (canvas.width !== bgImage.width) {
    canvas.width = bgImage.width;
    canvas.height = bgImage.height;
  }

  // Clear canvas
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Draw background
  ctx.drawImage(bgImage, 0, 0);

  // Draw Text Fields
  ctx.fillStyle = '#000000';
  ctx.textBaseline = 'top'; // Align text from top-left

  // Helper for drawing text with wrapping
  const drawText = (text: string, style: FieldStyle) => {
    const fontSize = style.fontSize || 36;
    const lineHeight = style.lineHeight || (fontSize * 1.5);
    ctx.font = `500 ${fontSize}px "Microsoft YaHei"`;
    
    if (style.w) {
      // Wrap text
      const words = text.split('');
      let line = '';
      let y = style.y;
      
      for (let n = 0; n < words.length; n++) {
        const char = words[n];
        if (!char) continue;
        const testLine = line + char;
        const metrics = ctx.measureText(testLine);
        const testWidth = metrics.width;
        
        if (testWidth > style.w && n > 0) {
          ctx.fillText(line, style.x, y);
          line = char;
          y += lineHeight;
        } else {
          line = testLine;
        }
      }
      ctx.fillText(line, style.x, y);
    } else {
      // Single line
      ctx.fillText(text, style.x, style.y);
    }
  };

  // Draw each field
  drawText(info.name, fieldStyles.name);
  drawText(info.gender, fieldStyles.gender);
  // Special handling for Ethnicity which has "民族：" label in the old design but user input is just "汉"
  // Based on the user request, it seems they want to just input "汉".
  // The requirement said: "使用微软雅黑字体... 字体大小36px"
  // Let's assume we just draw the value.
  drawText(info.ethnicity, fieldStyles.ethnicity); 
  
  drawText(info.birthDate, fieldStyles.birthDate);
  drawText(info.idNumber, fieldStyles.idNumber);
  drawText(info.householdAddress, fieldStyles.householdAddress);
  drawText(info.currentAddress, fieldStyles.currentAddress);
  
  drawText(info.issuingAuthority, fieldStyles.issuingAuthority);
  drawText(info.validityPeriod, fieldStyles.validityPeriod);

  // Draw Images
  if (photoUrl.value && photoImage.complete) {
    const { x, y, w, h } = fieldStyles.photo;
    if (w && h) {
        // Draw image with object-fit: cover logic if needed, or just stretch
        // Simple stretch for now as canvas doesn't have object-fit
        // To do 'cover', we'd need to calculate aspect ratios
        ctx.drawImage(photoImage, x, y, w, h);
    }
  }

  if (qrUrl.value && qrImage.complete) {
    const { x, y, w, h } = fieldStyles.qrCode;
    if (w && h) {
      ctx.drawImage(qrImage, x, y, w, h);
    }
  }

  const drawRotated = (text: string, style: FieldStyle) => {
    const fontSize = style.fontSize || 22;
    ctx.save();
    ctx.fillStyle = '#9b9292';
    ctx.font = `500 ${fontSize}px "Microsoft YaHei"`;
    ctx.translate(style.x, style.y);
    ctx.rotate(-Math.PI / 4);
    const lineHeight = style.lineHeight || fontSize * 1.5;
    if (style.w) {
      const chars = text.split('');
      let line = '';
      let y = 0;
      for (let i = 0; i < chars.length; i++) {
        const char = chars[i];
        if (!char) continue;
        const testLine = line + char;
        const width = ctx.measureText(testLine).width;
        if (width > style.w && i > 0) {
          ctx.fillText(line, 0, y);
          line = char;
          y += lineHeight;
        } else {
          line = testLine;
        }
      }
      ctx.fillText(line, 0, y);
    } else {
      ctx.fillText(text, 0, 0);
    }
    ctx.restore();
  };

  if (info.watermark1) drawRotated(info.watermark1, fieldStyles.watermark1);
  if (info.watermark2) drawRotated(info.watermark2, fieldStyles.watermark2);
  if (info.watermark3) drawRotated(info.watermark3, fieldStyles.watermark3);
};

// Export
const exportImage = async () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  
  // Helper to get blob
  const getBlob = (q: number): Promise<Blob | null> => {
    return new Promise(resolve => canvas.toBlob(resolve, 'image/jpeg', q));
  };

  let quality = 0.95;
  let blob = await getBlob(quality);
  
  // Loop to reduce quality if size > 1MB (1024 * 1024 bytes)
  while (blob && blob.size > 1024 * 1024 && quality > 0.1) {
    quality -= 0.1;
    blob = await getBlob(quality);
  }

  if (blob) {
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    // Use name from form as filename
    const filename = info.name ? `${info.name}.jpg` : '居住证.jpg';
    link.download = filename;
    link.href = url;
    link.click();
    URL.revokeObjectURL(url);
  }
};

// Zoom
const zoomIn = () => scale.value += 0.1;
const zoomOut = () => scale.value = Math.max(0.1, scale.value - 0.1);
const resetZoom = () => scale.value = 0.6;

// Watchers
watch(info, drawCanvas, { deep: true });
watch(fieldStyles, drawCanvas, { deep: true });

// Lifecycle
onMounted(() => {
  initDefaultValidity();
  loadImages();
});
</script>

<style scoped>
.common-layout {
  height: 100vh;
  width: 100vw;
  background-color: #f0f2f5;
}

.full-height {
  height: 100%;
}

.sidebar {
  background-color: white;
  border-right: 1px solid #dcdfe6;
}

.sidebar-content {
  padding: 20px;
}

.title {
  text-align: center;
  margin-bottom: 20px;
  color: #303133;
}

.upload-demo {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.upload-preview {
  margin-top: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.preview-img {
  width: 40px;
  height: 40px;
  object-fit: cover;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
}

.preview-text {
  font-size: 12px;
  color: #67c23a;
}

.settings-collapse {
  margin: 20px 0;
}

.pos-group {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
  border-bottom: 1px dashed #ebeef5;
  padding-bottom: 10px;
}

.pos-label {
  font-size: 13px;
  color: #606266;
  width: 70px;
}

.pos-inputs {
  display: flex;
  gap: 5px;
  flex: 1;
}

.actions {
  margin-top: 30px;
  padding-bottom: 20px;
}

.preview-area {
  background-color: #545c64;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  overflow: hidden;
  padding: 0;
}

.zoom-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 100;
  background: white;
  border-radius: 20px;
  padding: 4px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.canvas-wrapper {
  transform-origin: center center;
  transition: transform 0.1s ease-out;
  box-shadow: 0 0 20px rgba(0,0,0,0.5);
}

canvas {
  display: block;
  /* Max width logic is handled by scale transform */
}
</style>
