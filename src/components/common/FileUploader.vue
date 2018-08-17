<template>
  <!-- <div class="uploadContainer">
    <div class="upload-area">
      <a class="btn btn-default btn-lg" id="pickfiles" href="###">
        <i class="glyphicon glyphicon-plus"></i>
        <span>选择文件</span>
      </a>
      <input id="html5_1aj3rtennnga1n5h927gjn8343" type="file" style="font-size: 999px; opacity: 0; position: absolute; top: 0px; left: 0px; width: 100%; height: 100%;" multiple="" accept="">
    </div>

  </div> -->
  <div id="uploadContainer" class="uploadContainer">
    <slot name="button">
      <el-button id="pickfiles" class="pickfiles" type="primary">选择文件<i class="el-icon-upload el-icon--right"></i></el-button>
    </slot>
    <span>{{config.msg}}</span>
    <!-- <div><b>上传进度：{{ filePercent }}%</b></div> -->
    <ul class="uploader-file-list" v-if="filesList && filesList.length">
      <li v-for="(file,index) in filesList">
        <div class="files-tool">
          <div>{{file.name}}</div>
          <!-- <el-button @click="startUpload">{{isStart ? '暂停上传' : '开始上传'}}</el-button> -->
        </div>
        <el-progress :percentage="file.percent" :status="file.completeStatus ? file.completeStatus : ''"></el-progress>
        <input type="hidden" name="fileKey" :value="file.id" v-if="file.completeStatus == 'success'"/>
      </li>
    </ul>
  </div>
</template>
<script type="text/javascript">
import 'static/js/plupload/plupload.dev.js'
import 'static/js/qiniu.js'
export default {
  components: {
  },
  data () {
    return {
      filePercent: '',
      uploader: null,
      mimeType: [],
      uploaderConf : {
        multi_selection: false,
        max_file_size: '4mb',
        mime_types: 'jpg,gif,png,doc,docx,rar,zip,xls,pdf',
        isShowProgressBar: false,
        isPreview: false,
      },
      filesList: [],
      isStart : false,
      bucket : 'bioscience',
      msg : '仅支持JPG、GIF、PNG、JPEG、BMP格式，文件小于4M'
    }
  },
  props: {
    config: {
      type: Object,
      default () {
        return {
          multi_selection : false,
          max_file_size: '4mb',
          mime_types: [{title : 'specific file type',extensions: 'jpg,gif,png,doc,docx,rar,zip,xls,pdf'}],
          isShowProgressBar: false,
          isPreview: false,
          msg: '仅支持JPG、GIF、PNG、JPEG、BMP格式，文件小于4M',
          isPublic: 'bioscience' //是否上传到七牛公有空间 bioscience为私有空间bucket值 bio-public为公有空间bucket值
        }
      }
    }
  },
  created () {
  },
  watch: {
    config: {
      handler : function(newVal,oldVal){
        console.log(newVal)
        this.uploaderConf.multi_selection = newVal.multi_selection;
        this.uploaderConf.max_file_size = newVal.max_file_size ? newVal.max_file_size : this.uploaderConf.max_file_size;
        this.uploaderConf.mime_types = newVal.mime_types ? newVal.mime_types : this.uploaderConf.mime_types;
        this.uploaderConf.isShowProgressBar = newVal.isShowProgressBar ? newVal.isShowProgressBar : this.uploaderConf.isShowProgressBar;
        this.uploaderConf.isPreview = newVal.isPreview ? newVal.isPreview : this.uploaderConf.isPreview;
        this.msg = this.config.msg;
        if(this.config.mime_types){
          this.mimeType = [{title : 'specific file type',extensions: this.config.mime_types}];
        }
      },
      deep: true
    }
  },
  mounted () {
    this.initQiniu()
  },
  methods: {
    initQiniu () {
      let _this = this;
      let uploadSuccessList = [];
      /* eslint-disable */
      _this.uploader = Qiniu.uploader({
        runtimes: 'html5,flash,html4',
        container: 'uploadContainer',    // 上传区域DOM ID，默认是browser_button的父元素
        browse_button: 'pickfiles',
        flash_swf_url: 'https://cdn.bootcss.com/plupload/2.1.1/Moxie.swf',
        chunk_size: '4mb', // 分块上传时，每块的体积
        // uptoken : '<Your upload token>', // uptoken是上传凭证，由其他程序生成
        uptoken_url: '/upload/getUploadToken'+'/'+(_this.config.isPublic ? _this.config.isPublic : _this.bucket), //获取token的后端接口
        // uptoken_func: function(){    // 在需要获取uptoken时，该方法会被调用
        //   let uptoken = ''
        //   _this.$axios.get(_this.userConfig.urlPrefix + '/upload/getUploadToken').then((res) => {
        //     if(res && res.code == '200'){
        //       uptoken = res.data
        //     } else {
        //       console.log('获取token错误')
        //     }
        //     console.log(uptoken)
        //     return uptoken;
        //   })
        // },
        get_new_uptoken: false, // 设置上传文件的时候是否每次都重新获取新的uptoken
        unique_names: true, //设置文件名唯一，默认为false,js sdk会自动为文件生成唯一名字
        multi_selection: _this.uploaderConf.multi_selection || false, //是否可以在文件浏览对话框中选择多个文件，true为可以，false为不可以
        domain: 'http://pa51bpurt.bkt.clouddn.com/',
        // dragdrop: true, // 开启可拖曳上传
        // drop_element: 'uploadContainer', // 拖曳上传区域元素的ID，拖曳文件或文件夹后可触发上传
        auto_start: true, // 选择文件后自动上传，若关闭需要自己绑定事件触发上传
        max_retries: 3, // 上传失败最大重试次数
        max_file_size: _this.uploaderConf.max_file_size || '4mb', //限制文件大小
        filters: {
          prevent_duplicates: true, //是否允许选取重复的文件，为true时表示不允许，为false时表示允许，默认为false
          mime_types: _this.mimeType // 限定上传格式上传
        },
        init: {
          'FilesAdded': (up, files) => {
              console.log('文件就绪', files);
              plupload.each(files, function (file) {
                // 文件添加进队列后，处理相关的事情
                console.log(file.target_name)
                let matches = file.name.match(/\.([^.]+)$/);
                let key = ''
                if (matches) {
                  key = file.id + '.' + matches[1];
                }
                _this.filesList.push(Object.assign({}, file, { completeStatus: '',key: key }))
              })

          },
          BeforeUpload: (up, file) => {
            console.log('上传之前', file);
          },
          UploadProgress: (up, file) => {
            
            _this.filesList.map((item) => {
              item.percent = item.id == file.id ? file.percent : item.percent
            })
            console.log('上传中 file',_this.filesList[0].percent);
           // _this.filePercent = file.percent;
          },
          FileUploaded: (up, file, info) => {
            console.log('文件上传完毕')
            _this.filesList.map((item,index) => {
              if(file.id == item.id) {
                console.log('success')
                _this.$set(_this.filesList[index],'completeStatus','success')
              }
            })
          },
          Error: (up, err, errTip) => {
            console.log('上传失败', errTip);
            _this.filesList.map((item,index) => {
              if(err.file.id == item.id) {
                _this.$set(_this.filesList[index],'completeStatus','exception')

              }
            })
            _this.$alert('文件['+err.file.name + ']上传失败' +errTip,'上传有误', {
              confirmButtonText: '确定',
              callback: action => {
                _this.$message({
                  type: 'info',
                  message: `${ action }`
                });
              }
            });
          },
          'UploadComplete': function () {
            // 队列文件处理完毕后，处理相关的事情
            let successList = _this.filesList.filter((item) => item.completeStatus == 'success')
            if(successList.length == _this.filesList.length){
              //全部上传成功
              _this.$emit('uploadCallback',successList)
            }else{
              //部分上传成功
              _this.$emit('uploadCallback',successList)
            }
          }
        }
      });
    },
    startUpload () {
      this.isStart = true;
      this.uploader.start();
    },
    destroyUploader () {
      console.log('destroy')
      this.filesList = [];
    }
  },
  destroy () {
    
  }
}
</script>
<style lang="less" scoped>
.uploadContainer{
  text-align: left;
}
</style>
