<template>
    <div class="image-watermark">
    <input 
      type="file" 
      multiple 
      accept="image/*" 
      @change="handleFileUpload" 
      :disabled="images.length >= 10"
    >
    
    <div class="image-list">
      <div v-for="(image, index) in images" :key="index" class="image-item">
        <canvas :ref="'canvas' + index" class="preview-canvas"></canvas>
        
        <div class="watermark-form">
          <!-- Logo和人名称编辑 -->
          <div class="logo-info">
            <input v-model="image.logo" placeholder="Logo文字">
            <input v-model="image.name" placeholder="人名称">
          </div>
          <!-- 四个区域的文字编辑 -->
          <div class="watermark-grid">
            <input v-model="image.time" placeholder="拍摄时间">
            <input v-model="image.note" placeholder="备注">
            <input v-model="image.location" placeholder="地点">
            <input v-model="image.custom" placeholder="自定义文本">
          </div>
          <button @click="generateWatermark(index)">预览</button>
          <button @click="downloadImage(index)">下载</button>
        </div>
      </div>
    </div>
  </div>
  </template>
  
  <script>
  export default {
    name: 'ImageWatermark',
    data() {
      return {
        images: [], // 存储图片信息的数组
        WATERMARK_HEIGHT: 60, // 固定水印高度为60px
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
                logo: '',
                name: '',
                time: '',
                location: '',
                note: '',
                custom: '',
              });
            };
            img.src = e.target.result;
          };
          reader.readAsDataURL(file);
        });
      },
  
      generateWatermark(index) {
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

      // 设置文字样式
      ctx.fillStyle = '#000000';
      ctx.font = '16px Arial';
      ctx.textAlign = 'left'; // 改为左对齐
      ctx.textBaseline = 'middle';

      // 计算区域位置
      const halfWidth = image.original.width / 2;
      const padding = 20; // 文字距离左边的padding
      const top = image.original.height - this.WATERMARK_HEIGHT + 20; // 上半部分文字
      const bottom = image.original.height - 20; // 下半部分文字

      // 绘制四个区域的文字（靠左对齐）
      // 左上
      ctx.fillText(image.time, padding, top);
      // 右上
      ctx.fillText(image.note, halfWidth + padding, top);
      // 左下
      ctx.fillText(image.location, padding, bottom);
      // 右下
      ctx.fillText(image.custom, halfWidth + padding, bottom);

      // 绘制Logo和人名（白色文字）
      ctx.fillStyle = '#FFFFFF';
      ctx.font = '14px Arial';
      ctx.textAlign = 'right';
      
      const logoY = image.original.height - this.WATERMARK_HEIGHT - 35;
      const nameY = image.original.height - this.WATERMARK_HEIGHT - 15;
      
      ctx.fillText(image.logo, image.original.width - 20, logoY);
      ctx.fillText(image.name, image.original.width - 20, nameY);
    },
  
      downloadImage(index) {
        const canvas = this.$refs['canvas' + index][0];
        const link = document.createElement('a');
        link.download = `watermarked_image_${index}.png`;
        link.href = canvas.toDataURL('image/png', 1.0);
        link.click();
      }
    }
  }
  </script>
  
  <style scoped>
  .image-watermark {
    padding: 20px;
  }
  
  .image-list {
    display: flex;
    flex-wrap: wrap;
    gap: 20px;
    margin-top: 20px;
  }
  
  .image-item {
    width: 100%;
    max-width: 800px;
    margin-bottom: 20px;
  }
  
  .preview-canvas {
    max-width: 100%;
    height: auto;
  }
  
  .watermark-form {
    margin-top: 10px;
  }
  
  .watermark-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
    margin: 10px 0;
  }
  
  .logo-info {
    display: flex;
    gap: 10px;
    margin-bottom: 10px;
  }
  
  input {
    padding: 5px;
    width: 100%;
  }
  
  button {
    margin: 5px;
    padding: 5px 15px;
  }
  </style>