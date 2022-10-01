<template>
  <div class="lin-container">
    <div class="lin-title">OTA签名</div>
    <div class="lin-wrap">
      <el-form label-width="420px">

    <input type="radio" checked="true" name="op_type" value="p2litese/" >p2litese &nbsp;  </input>
    <input type="radio" name="op_type" value="p2litese/" >p2litese &nbsp; </input>
    <pre></br></pre>
    <pre></br></pre>

    <input type="radio" checked='true' name="option" value="cmd_apk_sign" >APK签名 &nbsp; </input>
    <input type="radio" name="option" value="cmd_down_upgrade" disabled >ota降级 &nbsp; </input>
    <input type="radio" name="option" value="cmd_ota_sign" >ota签名 &nbsp; </input>
    <input type="radio" name="option" value="cmd_ota_tms" disabled >ota生成TMS包 &nbsp; </input>
    <input type="radio" name="option" value="cmd_logo_gen" >png generate splash.img &nbsp;</input>
    <input type="radio" name="option" value="cmd_v2_sign_per" >mr980 v2 sign &nbsp; </input>
    <input type="radio" name="option" value="cmd_v2_sign_noper" >v2 noper </input>


    <pre></br></pre>
    <pre></br></pre>

    <input id="upload_file" :v-text="选择文件" type="file" :multiple="true" />

    <pre></br></pre>
    <pre></br></pre>

     <div><el-button @click="v_upload_files('upload_file')">上传 <i class="el-icon-upload el-icon--right"></i>  </el-button></div>
    </el-form>
    </div>
  </div>
</template>

<script>
import UploadFiles from '@/component/base/upload-file'

export default {
  name: 'ImgsUploadStage1',
  components: {
    UploadFiles,
  },

 

/*
  data() {
    return {
      fit: 'cover',
      rules: {
        minWidth: 100,
        minHeight: 100,
        maxSize: 5,
      },
    }
  },
  */
  // 计算属性设置
  computed: {},
  // 数据变更监听
  watch: {},
  mounted() {},
  methods: {
    async beforeFuc() {
      this.$message.error('进入自定义校验函数, 并返回false终止上传')
      return false
    },
  async getValue(name) {
      console.log(await this.$refs[name].getValue())
      // eslint-disable-next-line
      alert('已获取数据, 打印在控制台中')
    },

async v_upload_files(name) {

  //获取表单中的file	
    var file_obj = document.getElementById('upload_file').files[0];
    const data = {}
    data[`file_${0}`] = file_obj

    var ops_type = document.getElementsByName('op_type');
    var op_type = ''
    var option = ''
    for (var i=0 ; i<=ops_type.length ;i++){
      if(ops_type[i].checked == 1){
          op_type = ops_type[i].value
          console.log('op_type:' + op_type)
          break;
      }
    }

    var options = document.getElementsByName('option');
    for (var i=0 ; i<=options.length ;i++){
        if(options[i].checked == 1){
            option = options[i].value
            console.log('option:' + option)
            break;
        }
    }


    //var file_obj = this.$refs[name].files[0];
		//alert(file_obj)
    //alert('已获取数据, 打印在控制台中')
		console.log(file_obj);

    data['op_type'] = op_type 
		data['option'] =  option


    this.$axios({
        method: 'post',
        url: '/cms/sign_file',
        timeout:0, //无超时
        data,
      })
        .then(async res => {
          if (!Array.isArray(res) || res.length === 0) {
            throw new Error('文件上传失败')
          }

          for (let i = 0; i < res.length; i += 1) {
            console.log('111 file upload ok' + res[i]['key'])
            console.log('111 file upload ok' + res[i]['path'])
            console.log('222 file upload ok' + res[i]['url'])

             //文件下载
             const a = document.createElement('a')
             a.href = res[i].url 
             console.log('a.href=', a.href)
             a.click()
          }

        /*
          const resObj = res.reduce((acc, item) => {
            acc[item.key] = item
            console.log('文件上传成功：' + item)
            return acc
          }, {})
          */

        })
      },
   },
}
</script>

<style scoped lang="scss"></style>
