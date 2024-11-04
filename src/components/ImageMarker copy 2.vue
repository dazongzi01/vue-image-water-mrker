<template>
    <div class="image-watermark">
    <div class="upload-area">
      <input 
        type="file" 
        multiple 
        accept="image/*" 
        @change="handleFileUpload" 
        :disabled="images.length >= 10"
      >
      <p>最多可上传10张图片</p>
    </div>
    
    <div class="image-grid">
      <div v-for="(image, index) in images" :key="index" class="image-item">
        <div class="preview-section">
          <!-- 预览区域使用固定尺寸 -->
          <div class="canvas-wrapper">
            <canvas :ref="'canvas' + index" class="preview-canvas"></canvas>
          </div>
        </div>
        
        <div class="watermark-form">
          <div class="logo-info">
            <input v-model="image.name" placeholder="人名称">
          </div>
          <div class="watermark-grid">
            <input v-model="image.time" placeholder="拍摄时间">
            <textarea 
              v-model="image.note" 
              placeholder="备注（支持换行，最多两行）" 
              rows="2"
              @input="limitLines($event, image, 'note', 2)"
            ></textarea>
            <input v-model="image.location" placeholder="地点">
          </div>
          <button @click="generateWatermark(index)">预览</button>
        </div>
      </div>
    </div>

    <div class="batch-actions">
      <button 
        @click="batchDownload" 
        :disabled="!images.length"
        class="download-btn"
      >
        批量下载
      </button>
    </div>
  </div>
  </template>
  
  <script>
  import JSZip from 'jszip'
  
  export default {
    name: 'ImageWatermark',
    data() {
      return {
        images: [],
        WATERMARK_HEIGHT: 80,
        FONTS: {
          name: 'bold 40px system-ui',
          watermark: 'bold 20px system-ui'
        },
        SPACING: {
          watermarkTextGap: 5,
          nameMargin: 50,
        },
        LOGO_CONFIG: {
          width: 200,
          height: 180,
          rightMargin: 20
        }
      }
    },
  
    methods: {
      handleFileUpload(e) {
        const files = Array.from(e.target.files);
        if (this.images.length + files.length > 10) {
          alert('最多只能上传10张图片');
          return;
        }
  
        files.forEach(file => {
          const reader = new FileReader();
          reader.onload = (e) => {
            const img = new Image();
            img.onload = () => {
              this.images.push({
                original: img,
                name: '赵公子',
                time: '拍 摄 时 间：',
                location: '地           点：',
                note: '备           注：',
                custom: '',
              });
            };
            img.src = e.target.result;
          };
          reader.readAsDataURL(file);
        });
      },
  
      limitLines(event, image, field, maxLines) {
        const lines = event.target.value.split('\n');
        if (lines.length > maxLines) {
          image[field] = lines.slice(0, maxLines).join('\n');
        }
      },
      generateWatermark(index, targetCanvas = null, isDownload = false) {
  return new Promise((resolve) => {
    const image = this.images[index];
    const canvas = targetCanvas || this.$refs['canvas' + index][0];
    const ctx = canvas.getContext('2d');

    // 保存原始尺寸用于下载
    if (!image.originalWidth) {
      image.originalWidth = image.original.width;
      image.originalHeight = image.original.height;
    }

    let finalWidth, finalHeight;
    
    if (isDownload) {
      // 下载时使用原始尺寸
      finalWidth = image.originalWidth;
      finalHeight = image.originalHeight;
    } else {
      // 预览时计算适配尺寸
      const maxPreviewWidth = canvas.parentElement.offsetWidth;
      const maxPreviewHeight = 300; // 固定预览高度
      
      let previewWidth = image.original.width;
      let previewHeight = image.original.height;
      
      // 保持宽高比例缩放
      const aspectRatio = previewWidth / previewHeight;
      
      if (previewHeight > maxPreviewHeight) {
        previewHeight = maxPreviewHeight;
        previewWidth = previewHeight * aspectRatio;
      }
      
      if (previewWidth > maxPreviewWidth) {
        previewWidth = maxPreviewWidth;
        previewHeight = previewWidth / aspectRatio;
      }
      
      finalWidth = previewWidth;
      finalHeight = previewHeight;
    }

    // 设置画布尺寸（包含水印区域）
    canvas.width = finalWidth;
    canvas.height = finalHeight + this.WATERMARK_HEIGHT;

    // 填充白色背景
    ctx.fillStyle = '#FFFFFF';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    // 绘制原图
    ctx.drawImage(
      image.original,
      0,
      0,
      finalWidth,
      finalHeight
    );

    // 绘制底部白色水印区域
    ctx.fillStyle = '#FFFFFF';
    ctx.fillRect(0, finalHeight, finalWidth, this.WATERMARK_HEIGHT);

    // 设置水印区域内的文字样式
    ctx.fillStyle = '#000000';
    ctx.font = this.FONTS.watermark;
    ctx.textAlign = 'left';
    ctx.textBaseline = 'middle';

    // 计算文字位置
    const halfWidth = finalWidth / 2;
    const padding = 20;
    
    // 计算上下行的y坐标
    const watermarkTop = finalHeight; // 从图片底部开始
    const firstLineY = watermarkTop + 20;
    const secondLineY = watermarkTop + this.WATERMARK_HEIGHT - 20;

    // 绘制第一行文字（时间和备注）
    ctx.fillText(image.time, padding, firstLineY);
    
    // 处理备注的换行
    const noteLines = image.note.split('\n').slice(0, 2);
    noteLines.forEach((line, i) => {
      ctx.fillText(
        line,
        halfWidth + padding,
        firstLineY + (i * 20)
      );
    });

    // 绘制第二行文字（地点和自定义文本）
    ctx.fillText(image.location, padding, secondLineY);
    if (image.custom) {
      ctx.fillText(image.custom, halfWidth + padding, secondLineY);
    }

    // 加载并绘制固定的logo图片
    const logo = new Image();
    logo.onload = () => {
      // 计算logo位置
      const rightEdge = finalWidth - this.LOGO_CONFIG.rightMargin;
      const logoX = rightEdge - this.LOGO_CONFIG.width;
      
      // 计算logo的y坐标（在原图上）
      const logoY = finalHeight - this.SPACING.nameMargin - this.LOGO_CONFIG.height;
      
      // 使用固定大小绘制logo
      ctx.drawImage(
        logo, 
        logoX, 
        logoY, 
        this.LOGO_CONFIG.width, 
        this.LOGO_CONFIG.height
      );
      
      // 绘制人名（白色文字）
      ctx.fillStyle = '#FFFFFF';
      ctx.font = this.FONTS.name;
      ctx.textAlign = 'right';
      const nameY = finalHeight - this.SPACING.nameMargin;
      ctx.fillText(image.name, rightEdge, nameY);
      
      resolve(canvas);
    };
    logo.src = require('@/assets/logo.png');
  });
},
        // 修改批量下载方法
async batchDownload() {
  const zip = new JSZip();
  
  try {
    for (let i = 0; i < this.images.length; i++) {
      const image = this.images[i];
      const tempCanvas = document.createElement('canvas');
      
      // 使用原始尺寸
      tempCanvas.width = image.originalWidth;
      tempCanvas.height = image.originalHeight + this.WATERMARK_HEIGHT;
      
      // 使用原始尺寸重新绘制
      await this.generateWatermark(i, tempCanvas, true);
      
      const blob = await new Promise(resolve => {
        tempCanvas.toBlob(resolve, 'image/png', 1.0);
      });
      
      zip.file(`watermarked_image_${i + 1}.png`, blob);
    }
    
    const content = await zip.generateAsync({type: 'blob'});
    const link = document.createElement('a');
    link.href = URL.createObjectURL(content);
    link.download = 'watermarked_images.zip';
    link.click();
    URL.revokeObjectURL(link.href);
  } catch (error) {
    console.error('下载过程中出现错误:', error);
    alert('下载失败，请重试');
  }
  }
    },
  }
</script>

<style scoped>
.image-watermark {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.upload-area {
  margin-bottom: 20px;
  text-align: center;
}

/* 网格布局 */
.image-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr); /* 2列布局 */
  /* 如果需要3列布局，改为: grid-template-columns: repeat(3, 1fr); */
  gap: 20px;
  margin-bottom: 30px;
}

.image-item {
  border: 1px solid #eee;
  padding: 15px;
  border-radius: 8px;
  background: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* 预览区域固定尺寸 */
.preview-section {
  width: 100%;
  margin-bottom: 15px;
}

.canvas-wrapper {
  width: 100%;
  height: 300px; /* 固定预览高度 */
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f5f5;
  overflow: hidden;
}

.preview-canvas {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain; /* 保持比例 */
}

.watermark-form {
  padding: 10px;
}

.logo-info {
  margin-bottom: 10px;
}

.watermark-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 10px;
  margin: 10px 0;
}

input, textarea {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 100%;
  font-size: 14px;
}

textarea {
  resize: none;
  height: 60px;
}

button {
  padding: 8px 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-right: 10px;
  font-size: 14px;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

.batch-actions {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: white;
  padding: 15px;
  box-shadow: 0 -2px 10px rgba(0,0,0,0.1);
  width: 100%;
  max-width: 1200px;
  text-align: center;
  z-index: 1000;
}

.download-btn {
  font-size: 16px;
  padding: 12px 30px;
}

/* 响应式布局 */
@media (max-width: 1200px) {
  .image-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 768px) {
  .image-grid {
    grid-template-columns: 1fr;
  }
}
</style>