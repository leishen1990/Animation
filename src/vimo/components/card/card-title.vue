<template>
  <div class="ion-card-title" :class="[modeClass,colorClass]">
    <slot></slot>
  </div>
</template>
<script type="text/ecmascript-6">
  export default{
    name: 'CardTitle',
    data(){
      return {
        componentName: 'card-title'
      }
    },
    props: {
      /**
       * 按钮color：primary、secondary、danger、light、dark
       * */
      color: {
        type: String,
        default: '',
      },
      /**
       * mode 按钮风格 ios/window/android/we/alipay
       * */
      mode: {
        type: String,
        default: VM.config.get('mode') || 'ios',
      },
    },
    watch: {},
    computed: {
      // 环境样式
      modeClass () {
        return `${this.componentName} ${this.componentName}-${this.mode}`
      },
      // 颜色
      colorClass () {
        return !!this.color ? (`${this.componentName}-${this.mode}-${this.color}`) : ''
      },
    },
    methods: {},
    created () {},
    mounted () {},
    activated () {},
    components: {}
  }
</script>
