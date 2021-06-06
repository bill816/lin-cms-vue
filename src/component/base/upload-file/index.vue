<!--
 Component: UploadFiles
 Describe: 多图片上传组件, 附有预览, 排序, 验证等功能

todo: 使用中间件模式优化信息装载和验证功能
todo: 文件判断使用 serveWorker 优化性能
-->

<template>
    <input
      class="upload-files__input"
      type="file"
      ref="input"
      @change="handleChange"
      :multiple="multiple"
      :accept="accept"
    />
  </div>
</template>

<script>
import { getFileType, isEmptyObj, createId } from './utils'

const ONE_KB = 1024
const ONE_MB = ONE_KB * 1024

  
/**
 * 获取范围类型限制的提示文本
 * @param {String} prx 提示前缀
 * @param {Number} min 范围下限
 * @param {Number} max 范围上限
 * @param {String} unit 单位
 */
function getRangeTip(prx, min, max, unit = '') {
  let str = prx
  if (min && max) {
    // 有范围限制
    str += ` ${min}${unit}~${max}${unit}`
  } else if (min) {
    // 只有最小范围
    str += ` ≥ ${min}${unit}`
  } else if (max) {
    // 只有最大范围
    str += ` ≤ ${max}${unit}`
  } else {
    // 无限制
    str += '无限制'
  }
  return str
}

/** for originUpload: 一次请求最多的文件数量 */
const uploadLimit = 10
/** for originUpload: 文件对象缓存 */
let catchData = []
/** for originUpload: 计时器缓存 */
let time

export default {
  name: 'UploadFiles',
  data() {
    return {
      itemList: [],
      loading: false,
      currentId: '', // 正在操作项的id
      globalImgPriview: '$imagePreview', // 全局图片预览方法名
    }
  },
  props: {
    /** 每一项宽度 */
    width: {
      type: [Number, String],
      default: 160,
    },
    /** 每一项高度 */
    height: {
      type: [Number, String],
      default: 160,
    },
    /** 是否开启自动上传 */
    autoUpload: {
      type: Boolean,
      default: true,
    },
    /** 初始化数据 */
    value: {
      type: Array,
      default: () => [],
    },
    /** 接受的文件类型 */
    accept: {
      type: String,
      default: 'file/*',
    },
    /** 最少图片数量 */
    minNum: {
      type: Number,
      default: 0,
    },
    /** 最多图片数量, 0 表示无限制 */
    maxNum: {
      type: Number,
      default: 0,
    },
    /** 是否可排序 */
    sortable: {
      type: Boolean,
      default: false,
    },
    /** 是否可预览 */
    preview: {
      type: Boolean,
      default: true,
    },
    /** 是否可以一次多选 */
    multiple: {
      type: Boolean,
      default: false,
    },
    /** 图像验证规则 */
    rules: {
      type: [Object, Function],
      default: () => ({
        maxSize: 2,
      }),
    },
    /** 是否禁用, 禁用后只可展示 不可进行编辑操作, 包括: 新增, 修改, 删除, 改变顺序 */
    disabled: {
      type: Boolean,
      default: false,
    },
    /** 上传前插入方法, 属于高级用法 */
    beforeUpload: {
      type: Function,
      default: null,
    },
    /** 重写上传方法, 如果重写则覆盖组件内上传方法 */
    remoteFuc: {
      type: Function,
      default: null,
    },
    /** 图像显示模式 */
    fit: {
      type: String,
      default: 'contain',
    },
    /** 检测是否是动图 */
    animatedCheck: {
      type: Boolean,
      default: false,
    },
  },
  computed: {
    /** 每项容器样式 */
    boxStyle() {
      const { width, height, disabled } = this
      const style = {}
      if (typeof width === 'number') {
        style.width = `${width}px`
      } else if (typeof width === 'string') {
        style.width = width
      }
      if (typeof height === 'number') {
        style.height = `${height}px`
      } else if (typeof height === 'string') {
        style.height = height
      }
      if (disabled) {
        style.cursor = 'not-allowed'
      } else {
        style.cursor = 'pointer'
      }

      /** 提示字体最大尺寸 */
      let fontSize = 12
      /** 每行提示预设 */
      const maxText = 8
      if (typeof width === 'number' && width / maxText < fontSize) {
        fontSize = (width / maxText).toFixed(2)
      }
      style.fontSize = `${fontSize}px`
      style.textAlign = 'center'
      style.position = 'relative'
      style.display = 'flex'
      style.alignItems = 'center'
      style.justifyContent = 'center'
      style.overflow = 'hidden'
      style.lineHeight = '1.3'
      style.flexDirection = 'column'

      return style
    },
    /**
     * 上传图像数量下限
     * @returns {Number}
     */
    min() {
      const { minNum } = this
      return minNum < 0 ? 0 : parseInt(minNum, 10)
    },
    /**
     * 上传图像数量上限
     * @returns {Number}
     */
    max() {
      const { min, maxNum } = this
      // 兼容用最大值小于最小值情况
      return maxNum < min ? min : parseInt(maxNum, 10)
    },
    /**
     * 是否是固定数量(最小等于最大)
     * @returns {Boolean}
     */
    isStable() {
      const { min, max } = this
      return max !== 0 && min === max
    },
    /** 构造图像规范提示 */
    rulesTip() {
      const { rules } = this
      const tips = []

      /** 图像验证规则 */
      let basicRule
      // 针对动态规则模式, 获取输入为空时的规则
      // 动态规则 rule 为函数, 当选择图片后根据选择的图片生成校验规则
      if (typeof rules === 'function') {
        try {
          basicRule = rules()
        } catch (err) {
          basicRule = {}
        }
      } else {
        basicRule = rules || {}
      }

        tips.push(`点击此处+选择文件`)
      return tips
    },
  },

 methods: {
    /**
     * 上传缓存中的图片
     * @param {Array} uploadList 需要上传的缓存集合, 集合中包含回调函数
     */
    uploadCatch(uploadList) {
      const data = {}
      uploadList.forEach((item, index) => {
        data[`file_${index}`] = item.img.file
      })
      //TODO GX 此处为上传的URL
      return this.$axios({
        method: 'post',
        url: '/cms/file',
        data,
      })
        .then(res => {
          if (!Array.isArray(res) || res.length === 0) {
            throw new Error('图像上传失败')
          }

          const resObj = res.reduce((acc, item) => {
            acc[item.key] = item
            return acc
          }, {})

          uploadList.forEach((item, index) => {
            const remoteData = resObj[`file_${index}`]
            item.cb(remoteData)
          })
        })
        .catch(err => {
          uploadList.forEach(item => {
            item.cb(false)
          })
          let msg = '图像上传失败, 请重试'
          if (err.message) {
            // eslint-disable-next-line
            msg = err.message
          }
          console.error(err)
          this.$message.error(msg)
        })
    },
    /**
     * 内置上传文件方法, 使用 debounce 优化提交效率
     * 此处只能使用回调模式, 因为涉及 debounce 处理, promise 不可在外部改变其状态
     * @param {Object} img 需要上传的数据项
     * @param {Function} cb 回调函数
     */
    originUpload(img, cb) {
      // 并且一次最多上传文件数量设为可配置
      // 添加缓存
      //TODO 添加上传缓存 by gx
      catchData.push({
        img,
        cb,
      })

      // 等于上限, 立即上传
      if (catchData.length === uploadLimit) {
        const data = [...catchData]
        catchData = []
        clearTimeout(time)
        time = null
        return this.uploadCatch(data)
      }

      // 清除上次一的定时器
      if (time && catchData.length < uploadLimit) {
        clearTimeout(time)
        // 此时修改上一个 promise 状态为reslove
      }

      // 等待100ms
      time = setTimeout(() => {
        this.uploadCatch([...catchData])
        catchData = []
        time = null
      }, 50)
    },
    /**
     * 上传图像文件
     * @param {Object} 需要上传的项, 包含文化和其它信息
     */
    async uploadImg(item) {
      //TODO GX 开始上传图片

      console.log("item status:" + item.status)

        // 使用内置上传
      return new Promise(resolve => {
        this.originUpload(item, data => {
          reduceResult(item, data)
          if (!data) {
            resolve(false)
          } else {
            resolve(item)
          }
        })
      })
    },

      
          
    /**
     * 选择图像文件后处理, 包括获取图像信息, 验证图像等
     * @param {Event} e input change 事件对象
     */
    async handleChange(e) {
      const { currentId, autoUpload } = this
      const { files } = e.target

      if (!files) return
      /** 中间步骤缓存, 在出错时用于释放 createObjectURL 的内存 */
      let cache = []
      /**
       * 处理单个图片, 返回处理成功的图片数据
       * @param {File} file 图片文件
       */
      const asyncList = []
      for (let i = 0; i < files.length; i += 1) {
        cache.push(files[i])
      }
    },

    /** 清空全部图片 */
    /*
    clear() {
      this.initItemList([])
      this.getValue()
    },*/
    /** 重置图片数据传入属性 */
    reset() {
      this.initItemList(this.value)
    },
  },
}
</script>

<style lang="scss" scoped>
.upload-files-container {
  display: flex;
  flex-wrap: wrap;

  &:focus {
    outline: none;
  }

  .upload-item,
  .thumb-item {
    border: 1px dashed #d9d9d9;
    border-radius: 3px;
    transition: all 0.1s;
    color: #666666;
    margin-right: 1em;
    margin-bottom: 1em;

    &.disabled {
      color: #ababab;
    }

    &:not(.disabled):hover {
      border-color: #3963bc;
      color: #3963bc;
    }
  }

  .thumb-item {
    .info {
      display: flex;
      align-items: center;
      justify-content: center;
      position: absolute;
      width: 100%;
      height: 100%;
      top: 0;
      left: 0;
      transition: all 0.3s;
      transition-delay: 0.1s;

      .wait-upload {
        background: #ffcb71;
        color: white;
        position: absolute;
        display: inline-block;
        width: 1.7em;
        height: 1.5em;
        top: 0;
        right: 0;
        border-radius: 0 0 0 1.4em;
        transition: all 0.1s;

        &::before {
          font-size: 1.4em;
          position: absolute;
          right: -1px;
          transform: scale(0.7);
        }
      }
    }

    .control {
      display: flex;
      align-items: center;
      justify-content: center;
      position: absolute;
      width: 100%;
      height: 100%;
      top: 0;
      left: 0;
      opacity: 0;
      background-color: rgba(0, 0, 0, 0.3);
      transition: all 0.3s;
      transition-delay: 0.1s;

      .del {
        background: #f4516c;
        color: white;
        position: absolute;
        display: inline-block;
        width: 1.7em;
        height: 1.5em;
        top: 0;
        right: 0;
        opacity: 0;
        border-radius: 0 0 0 1.4em;
        transition: all 0.1s;

        &::before {
          font-size: 1.4em;
          position: absolute;
          right: -1px;
          transform: scale(0.7);
        }

        &:hover {
          transform: translate(-0.5em, 0.4em) scale(1.5);
        }
      }

      .preview {
        color: white;
        font-size: 2em;
        transition: all 0.2s;

        &:hover {
          transform: scale(1.2);
        }
      }

      .control-bottom {
        position: absolute;
        bottom: 0;
        left: 0;
        width: 100%;
        color: white;
        background-color: rgba(0, 0, 0, 0.3);
        font-size: 1.5em;
        display: flex;
        justify-content: space-around;
        transform: translate(0, 100%);
        transition: all 0.2s;
        transition-delay: 0.1s;
        padding: 5px 0;

        .control-bottom-btn {
          transform: all 0.2s;

          &.disabled {
            color: #ababab;
            cursor: not-allowed;
          }

          &:not(.disabled):hover {
            transform: scale(1.2);
          }
        }
      }
    }

    &:hover {
      .control {
        opacity: 1;
      }

      .del {
        opacity: 1;
      }

      .control-bottom {
        transform: translate(0, 0);
      }
    }
  }

  .upload-files__input {
    display: none;
  }
}
</style>
