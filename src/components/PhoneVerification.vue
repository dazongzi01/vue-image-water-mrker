<template>
    <div class="phone-verification">
      <div class="input-group">
        <input 
          type="tel"
          v-model="phoneNumber"
          @input="validatePhone"
          placeholder="请输入手机号码"
          maxlength="11"
          class="phone-input"
        >
        <span class="validation-message" :class="{ 'is-valid': isValid, 'is-invalid': !isValid && phoneNumber }">
          {{ validationMessage }}
        </span>
      </div>
      
      <button 
        @click="handleSubmit"
        :disabled="!isValid"
        class="submit-btn"
      >
        继续
      </button>
    </div>
  </template>
  
  <script>
  export default {
    name: 'PhoneVerification',
    data() {
      return {
        phoneNumber: '',
        isValid: false,
        validationMessage: ''
      }
    },
    methods: {
      validatePhone() {
        // 移除所有非数字字符
        this.phoneNumber = this.phoneNumber.replace(/\D/g, '');
        
        // 手机号码正则验证
        const phoneRegex = /^1[3456789]\d{9}$/;
        
        if (!this.phoneNumber) {
          this.isValid = false;
          this.validationMessage = '';
        } else if (this.phoneNumber.length < 11) {
          this.isValid = false;
          this.validationMessage = '请输入完整的手机号码';
        } else if (!phoneRegex.test(this.phoneNumber)) {
          this.isValid = false;
          this.validationMessage = '请输入正确的手机号码';
        } else {
          this.isValid = true;
          this.validationMessage = '手机号码格式正确';
        }
      },
      
      handleSubmit() {
        if (this.isValid) {
          // 这里添加验证通过后的处理逻辑
          this.$emit('phone-verified', this.phoneNumber);
        }
      }
    }
  }
  </script>
  
  <style scoped>
  .phone-verification {
    max-width: 400px;
    margin: 20px auto;
    padding: 20px;
  }
  
  .input-group {
    margin-bottom: 20px;
  }
  
  .phone-input {
    width: 100%;
    padding: 12px;
    font-size: 16px;
    border: 1px solid #ddd;
    border-radius: 4px;
    outline: none;
    transition: border-color 0.3s;
  }
  
  .phone-input:focus {
    border-color: #4CAF50;
  }
  
  .validation-message {
    display: block;
    margin-top: 5px;
    font-size: 14px;
    min-height: 20px;
  }
  
  .is-valid {
    color: #4CAF50;
  }
  
  .is-invalid {
    color: #f44336;
  }
  
  .submit-btn {
    width: 100%;
    padding: 12px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.3s;
  }
  
  .submit-btn:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
  
  .submit-btn:hover:not(:disabled) {
    background-color: #45a049;
  }
  </style>