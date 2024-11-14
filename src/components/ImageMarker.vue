<template>
  <div class="image-watermark">
    <div class="control-panel">
      <div class="upload-area">
        <input 
          type="file" 
          multiple 
          accept="image/*" 
          @change="handleFileUpload" 
          :disabled="images.length >= 10"
          ref="fileInput"
        >
        <p>最多可上传10张图片，当前: {{ images.length }}/10</p>
      </div>
      <div class="action-buttons">
        <button 
          @click="clearAllImages" 
          :disabled="!images.length"
          class="clear-btn"
        >
          清空所有图片
        </button>
        <button 
          @click="batchDownload" 
          :disabled="!images.length"
          class="download-btn"
        >
          批量下载
        </button>
      </div>
    </div>
    
    <div class="image-grid">
      <div v-for="(image, index) in images" :key="index" class="image-item">
        <div class="preview-section">
          <div class="canvas-wrapper">
            <canvas :ref="'canvas' + index" class="preview-canvas"></canvas>
          </div>
        </div>
        
        <div class="watermark-form">
          <div class="logo-info">
            <input 
              v-model="image.name" 
              placeholder="人名称"
              @input="updateWatermark(index)"
            >
          </div>
          <div class="watermark-grid">
            <input 
              v-model="image.time" 
              placeholder="拍摄时间"
              @input="updateWatermark(index)"
            >
            <textarea 
              v-model="image.note" 
              placeholder="备注（支持换行，最多两行）" 
              rows="2"
              @input="handleNoteInput($event, index)"
            ></textarea>
            <input 
              v-model="image.location" 
              placeholder="地点"
              @input="updateWatermark(index)"
            >
          </div>
        </div>
      </div>
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
      WATERMARK_CONFIG: {
        HEIGHT_RATIO: 0.12, // 水印高度占图片高度的比例，稍微调小
        MIN_HEIGHT: 50,     // 最小水印高度
        MAX_HEIGHT: 100,    // 最大水印高度
        FONTS: {
          name: {
            style: 'bold',
            sizeRatio: 0.28  // 修改为与其他文字相同的比例
          },
          watermark: {
            style: 'normal',
            sizeRatio: 0.28
          },
          family: '"PingFang SC", "SF Pro SC", sans-serif' // 添加字体设置
        },
        SPACING: {
          nameMargin: 40,    // 调小间距
          padding: 15,       // 调小内边距
          lineSpacing: 1.2   // 行间距倍数
        },
        LOGO: {
          originalWidth: 130,  // logo 原始宽度
          originalHeight: 30,  // logo 原始高度
          aspectRatio: 130/30, // logo 宽高比
          maxWidthRatio: 0.15, // logo 最大宽度占图片宽度的比例
          minWidth: 65,        // logo 最小宽度（原始尺寸的一半）
          rightMargin: 15      // 右边距
        }
    }
    }
  },

  methods: {
    handleFileUpload(e) {
      const files = Array.from(e.target.files);
      
      if (this.images.length + files.length > 10) {
        if (confirm('已超过10张图片限制。是否清空当前图片并重新上传？')) {
          this.clearAllImages();
        } else {
          this.$refs.fileInput.value = ''; // 清空input
          return;
        }
      }

      files.forEach(file => {
        const reader = new FileReader();
        reader.onload = (e) => {
          const img = new Image();
          img.onload = () => {
            const imageData = {
              original: img,
              name: '赵公子',
              time: '拍摄时间：',
              location: '地       点：',
              note: '备        注：',
              width: img.width,
              height: img.height,
              aspectRatio: img.width / img.height,
              isLandscape: img.width > img.height
            };
            this.images.push(imageData);
            // 自动生成水印
            this.$nextTick(() => {
              const index = this.images.length - 1;
              this.generateWatermark(index);
            });
          };
          img.src = e.target.result;
        };
        reader.readAsDataURL(file);
      });
      
      // 清空input，允许重复上传相同文件
      this.$refs.fileInput.value = '';
    },

    clearAllImages() {
      this.images = [];
    },

    handleNoteInput(event, index) {
      const lines = event.target.value.split('\n');
      if (lines.length > 2) {
        this.images[index].note = lines.slice(0, 2).join('\n');
      }
      this.updateWatermark(index);
    },

    updateWatermark(index) {
      // 使用防抖优化性能
      if (this.updateTimeout) {
        clearTimeout(this.updateTimeout);
      }
      this.updateTimeout = setTimeout(() => {
        this.generateWatermark(index);
      }, 300);
    },
    calculateWatermarkDimensions(imageWidth, imageHeight) {
      const config = this.WATERMARK_CONFIG;
      
      // 计算水印高度
      let watermarkHeight = Math.round(imageHeight * config.HEIGHT_RATIO);
      watermarkHeight = Math.max(config.MIN_HEIGHT, Math.min(config.MAX_HEIGHT, watermarkHeight));

      // 计算字体大小
      const nameFontSize = Math.round(watermarkHeight * config.FONTS.name.sizeRatio);
      const watermarkFontSize = Math.round(watermarkHeight * config.FONTS.watermark.sizeRatio);

      // 计算 logo 尺寸（保持原始比例）
      let logoWidth = Math.round(imageWidth * config.LOGO.maxWidthRatio);
      logoWidth = Math.max(config.LOGO.minWidth, logoWidth);
      const logoHeight = Math.round(logoWidth / config.LOGO.aspectRatio);

      return {
        watermarkHeight,
        nameFontSize,
        watermarkFontSize,
        logoWidth,
        logoHeight
      };
    },
    generateWatermark(index, targetCanvas = null, isDownload = false) {
      return new Promise((resolve) => {
        const image = this.images[index];
        const canvas = targetCanvas || this.$refs['canvas' + index][0];
        const ctx = canvas.getContext('2d');

        // 计算预览尺寸
        let finalWidth, finalHeight, scale;
        if (isDownload) {
          finalWidth = image.width;
          finalHeight = image.height;
          scale = 1;
        } else {
          const maxPreviewWidth = canvas.parentElement.offsetWidth;
          const maxPreviewHeight = 220; // 稍微调小预览高度

          // 计算下载时的完整尺寸
          const downloadDimensions = this.calculateWatermarkDimensions(image.width, image.height);
          const totalOriginalHeight = image.height + downloadDimensions.watermarkHeight;
          
          // 计算缩放比例
          const widthScale = maxPreviewWidth / image.width;
          const heightScale = maxPreviewHeight / totalOriginalHeight;
          scale = Math.min(widthScale, heightScale) * 0.85; // 调整为85%
          
          finalWidth = Math.round(image.width * scale);
          finalHeight = Math.round(image.height * scale);
        }

        // 计算水印尺寸
        const dimensions = this.calculateWatermarkDimensions(
          isDownload ? finalWidth : image.width,
          isDownload ? finalHeight : image.height
        );

        // 缩放所有尺寸
        let {
          watermarkHeight,
          nameFontSize,
          watermarkFontSize,
          logoWidth,
          logoHeight
        } = dimensions;

        if (!isDownload) {
          watermarkHeight = Math.round(watermarkHeight * scale);
          nameFontSize = Math.round(nameFontSize * scale);
          watermarkFontSize = Math.round(watermarkFontSize * scale);
          logoWidth = Math.round(logoWidth * scale);
          logoHeight = Math.round(logoHeight * scale);
        }

        // 设置画布尺寸
        canvas.width = finalWidth;
        canvas.height = finalHeight + watermarkHeight;

        // 绘制背景和原图
        ctx.fillStyle = '#FFFFFF';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.drawImage(image.original, 0, 0, finalWidth, finalHeight);

        // 绘制水印背景
        ctx.fillStyle = '#FFFFFF';
        ctx.fillRect(0, finalHeight, finalWidth, watermarkHeight);

        // 设置水印文字样式
        ctx.fillStyle = '#000000';
        ctx.font = `${this.WATERMARK_CONFIG.FONTS.watermark.style} ${watermarkFontSize}px ${this.WATERMARK_CONFIG.FONTS.family}`;
        ctx.textAlign = 'left';
        ctx.textBaseline = 'middle';

        // 计算文字位置
        const padding = Math.round(this.WATERMARK_CONFIG.SPACING.padding * scale);
        const lineHeight = Math.round(watermarkHeight / 2.5); // 保持行高计算
        
        // 在原来的基础上减少10像素（向上移动）
        const firstLineY = finalHeight + lineHeight - 10;
        const secondLineY = firstLineY + (lineHeight * this.WATERMARK_CONFIG.SPACING.lineSpacing);

        // 绘制水印文字
        ctx.fillText(image.time, padding, firstLineY);
        ctx.fillText(image.note, finalWidth / 2 + padding, firstLineY);

        // 绘制地点（最多两行）
        const locationLines = image.location.split('\n');
        locationLines.forEach((line, i) => {
          if (i < 2) {
            ctx.fillText(
              line, 
              padding, 
              secondLineY + (i * watermarkFontSize * this.WATERMARK_CONFIG.SPACING.lineSpacing)
            );
          }
        });

        // 加载并绘制logo
        const logo = new Image();
        logo.onload = () => {
          const logoX = finalWidth - logoWidth - Math.round(this.WATERMARK_CONFIG.LOGO.rightMargin * scale);
          const logoY = finalHeight - logoHeight - Math.round(this.WATERMARK_CONFIG.SPACING.nameMargin * scale) - 30;
          
          ctx.drawImage(logo, logoX, logoY, logoWidth, logoHeight);
          
           // 绘制人名时使用相同的字体设置
          ctx.font = `${this.WATERMARK_CONFIG.FONTS.name.style} ${watermarkFontSize}px ${this.WATERMARK_CONFIG.FONTS.family}`;
          ctx.fillStyle = '#FFFFFF';
          ctx.textAlign = 'right';
          const nameY = finalHeight - Math.round(this.WATERMARK_CONFIG.SPACING.nameMargin * scale);
          ctx.fillText(
            image.name, 
            finalWidth - Math.round(this.WATERMARK_CONFIG.LOGO.rightMargin * scale), 
            nameY
          );
          resolve(canvas);
        };
        logo.src = require('@/assets/logo.png');
      });
  },
    async batchDownload() {
      const zip = new JSZip();
      
      try {
        // 显示加载提示
        this.$message ? this.$message.loading('正在处理图片...') : alert('正在处理图片...');
        
        for (let i = 0; i < this.images.length; i++) {
          const tempCanvas = document.createElement('canvas');
          await this.generateWatermark(i, tempCanvas, true);
          
          const blob = await new Promise(resolve => {
            tempCanvas.toBlob(resolve, 'image/jpeg', 0.92);
          });
          
          zip.file(`watermarked_image_${i + 1}.jpg`, blob);
        }
        
        const content = await zip.generateAsync({
          type: 'blob',
          compression: 'DEFLATE',
          compressionOptions: {
            level: 6
          }
        });
        
        const link = document.createElement('a');
        link.href = URL.createObjectURL(content);
        link.download = `watermarked_images_${new Date().getTime()}.zip`;
        link.click();
        URL.revokeObjectURL(link.href);
        
        // 显示成功提示
        this.$message ? this.$message.success('下载成功！') : alert('下载成功！');
      } catch (error) {
        console.error('下载过程中出现错误:', error);
        this.$message ? this.$message.error('下载失败，请重试') : alert('下载失败，请重试');
      }
    }
  }
}
</script>
<style scoped>
/* @font-face {
  font-family: 'PingFang SC';
  src: url('../assets/fonts/PingFang-SC-Regular.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}

@font-face {
  font-family: 'PingFang SC';
  src: url('../assets/fonts/PingFang-SC-Bold.ttf') format('truetype');
  font-weight: bold;
  font-style: normal;
} */
.image-watermark {
  padding: 20px;
  max-width: 1400px;
  margin: 0 auto;
}

.control-panel {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 8px;
}

.upload-area {
  flex: 1;
}

.action-buttons {
  display: flex;
  gap: 10px;
}

.image-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr); /* 3列布局 */
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

.preview-section {
  width: 100%;
  margin-bottom: 15px;
}

.canvas-wrapper {
  width: 100%;
  min-height: 250px; /* 最小高度 */
  padding: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f5f5f5;
  border-radius: 4px;
}

.preview-canvas {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
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
  font-size: 14px;
}

.clear-btn {
  background-color: #f44336;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  opacity: 0.9;
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
  
  .control-panel {
    flex-direction: column;
    gap: 15px;
  }
  
  .action-buttons {
    width: 100%;
    justify-content: center;
  }
}
</style>