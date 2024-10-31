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
      
      <div class="image-list">
        <div v-for="(image, index) in images" :key="index" class="image-item">
          <canvas :ref="'canvas' + index" class="preview-canvas"></canvas>
          
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
              <!-- <input v-model="image.custom" placeholder="自定义文本"> -->
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
          watermarkTextGap: 5,    // 白色框内文字上下间距
          nameMargin: 50,         // 人名上下间距
        },
        LOGO_CONFIG: {
            width: 200,      // 宽度保持200px
            height: 180,     // 高度改为180px (原80 + 100)
            rightMargin: 20  // 右边距与人名保持一致
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
  
      generateWatermark(index) {
    return new Promise((resolve) => {
      const image = this.images[index];
      const canvas = this.$refs['canvas' + index][0];
      const ctx = canvas.getContext('2d');

      // 设置画布尺寸
      canvas.width = image.original.width;
      canvas.height = image.original.height;

      // 绘制原图
      ctx.drawImage(image.original, 0, 0);

      // 绘制底部白色水印区域
      ctx.fillStyle = '#FFFFFF';
      ctx.fillRect(0, image.original.height - this.WATERMARK_HEIGHT, image.original.width, this.WATERMARK_HEIGHT);

      // 设置水印区域内的文字样式
      ctx.fillStyle = '#000000';
      ctx.font = this.FONTS.watermark;
      ctx.textAlign = 'left';
      ctx.textBaseline = 'middle';

      // 计算文字位置
      const halfWidth = image.original.width / 2;
      const padding = 20;
      
      // 计算上下行的y坐标
      const watermarkTop = image.original.height - this.WATERMARK_HEIGHT;
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
      ctx.fillText(image.custom, halfWidth + padding, secondLineY);

       // 加载并绘制固定的logo图片
       const logo = new Image();
        logo.onload = () => {
          // 计算logo位置，右边与人名对齐
          const rightEdge = image.original.width - this.LOGO_CONFIG.rightMargin;
          const logoX = rightEdge - this.LOGO_CONFIG.width;
          const logoY = image.original.height - this.WATERMARK_HEIGHT - this.SPACING.nameMargin - this.LOGO_CONFIG.height;
          
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
          const nameY = image.original.height - this.WATERMARK_HEIGHT - this.SPACING.nameMargin;
          ctx.fillText(image.name, rightEdge, nameY); // 使用相同的rightEdge确保对齐
          
          resolve();
        };
        logo.src = require('@/assets/logo.png');
      });
    },

// 修改批量下载方法
async batchDownload() {
    const zip = new JSZip();

    try {
      // 等待所有图片处理完成
      for (let i = 0; i < this.images.length; i++) {
        await this.generateWatermark(i);
        const canvas = this.$refs['canvas' + i][0];
        
        // 将canvas转换为blob
        const blob = await new Promise(resolve => {
          canvas.toBlob(resolve, 'image/png', 1.0);
        });

        // 添加到zip
        zip.file(`watermarked_image_${i + 1}.png`, blob);
      }

      // 生成并下载zip
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
  }
}
</script>

<style scoped>
.image-watermark {
  padding: 20px;
}

.upload-area {
  margin-bottom: 20px;
}

.image-list {
  display: flex;
  flex-direction: column;
  gap: 30px;
}

.image-item {
    border: 1px solid #eee;
padding: 15px;
border-radius: 8px;
}
.preview-canvas {
max-width: 100%;
height: auto;
margin-bottom: 15px;
}
.watermark-form {
margin-top: 15px;
}
.logo-info {
display: flex;
gap: 10px;
margin-bottom: 10px;
}
.watermark-grid {
display: grid;
grid-template-columns: 1fr 1fr;
gap: 10px;
margin: 10px 0;
}
input, textarea {
padding: 8px;
border: 1px solid #ddd;
border-radius: 4px;
width: 100%;
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
}
button:disabled {
background-color: #cccccc;
cursor: not-allowed;
}
button:hover:not(:disabled) {
background-color: #45a049;
}
.batch-actions {
margin-top: 30px;
text-align: center;
}
.download-btn {
font-size: 16px;
padding: 10px 24px;
}
</style>