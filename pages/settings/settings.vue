<template>
  <view class="page">
    <view class="section">
      <view class="label">DeepSeek API Key</view>
      <input
        v-model="apiKey"
        class="input"
        placeholder="sk-xxxxxxxxxxxxxxxx"
        password
      />
      <view class="hint">前往 platform.deepseek.com 免费注册获取，测试100篇申论约1元</view>
    </view>
    <button class="btn-primary" @click="save">保存</button>
    <button class="btn-danger" @click="clear">清除Key</button>
  </view>
</template>

<script>
export default {
  data() {
    return { apiKey: '' }
  },
  onLoad() {
    this.apiKey = uni.getStorageSync('deepseek_key') || ''
  },
  methods: {
    save() {
      const key = this.apiKey.trim()
      if (!key.startsWith('sk-')) {
        uni.showToast({ title: 'Key格式不对，应以sk-开头', icon: 'none' })
        return
      }
      uni.setStorageSync('deepseek_key', key)
      uni.showToast({ title: '保存成功', icon: 'success' })
      setTimeout(() => uni.navigateBack(), 1000)
    },
    clear() {
      uni.removeStorageSync('deepseek_key')
      this.apiKey = ''
      uni.showToast({ title: '已清除', icon: 'success' })
    }
  }
}
</script>

<style scoped>
.page { padding: 40rpx; background: #f7f8fa; min-height: 100vh; }
.section { background: #fff; border-radius: 16rpx; padding: 30rpx; margin-bottom: 30rpx; }
.label { font-size: 28rpx; font-weight: bold; color: #333; margin-bottom: 16rpx; }
.input { border: 1rpx solid #eee; border-radius: 10rpx; padding: 20rpx; font-size: 28rpx; width: 100%; }
.hint { margin-top: 16rpx; font-size: 24rpx; color: #999; line-height: 1.6; }
.btn-primary { background: #007aff; color: #fff; border-radius: 12rpx; margin-bottom: 20rpx; border: none; }
.btn-danger { background: #fff; color: #ff3b30; border: 1rpx solid #ff3b30; border-radius: 12rpx; }
</style>
