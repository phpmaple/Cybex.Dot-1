<template>
  <div class="portrait-wrapper">
    <div v-if="notice" class="notice-circle" />
    <!-- VIP用户显示彩虹犀牛头像 -->
    <canvas
      ref="canvas"
      :class="{ small: size < 64 }"
      :width="size"
      :height="size"
    />
  </div>
</template>

<script>
import jdenticon from 'jdenticon'
import sha256 from 'sha256'

export default {
  props: {
    username: {
      type: String,
      default: ''
    },
    size: {
      type: String,
      default: '64'
    },
    notice: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {}
  },
  watch: {
    username(val) {
      this.draw()
    }
  },
  mounted() {
    this.draw()
  },
  methods: {
    draw() {
      // check whether use rainbow profile image
      const canvas = this.$refs.canvas
      if (canvas && canvas.getContext) {
        const context = canvas.getContext('2d')
        if (context) {
          this.$nextTick(() => {
            jdenticon.drawIcon(context, sha256(this.username), this.size)
          })
        }
      }
    }
  }
}
</script>

<style scoped lang="scss">
.portrait-wrapper {
  position: relative;
  .notice-circle {
    width: 14px;
    height: 14px;
    position: absolute;
    right: -2px;
    top: -2px;
    border-radius: 50%;
    background-color: #d7333d;
    border: solid 2px #1b2230;
  }
}
canvas {
  &.hidden {
    width: 0 !important;
    height: 0 !important;
  }
  width: 32px;
  height: 32px;
  line-height: 64px;
  border-radius: 16px;

  &.small {
    width: 16px;
    height: 16px;
  }
  &.notice:before {
    content: '';
    display: block;
  }
}
</style>
