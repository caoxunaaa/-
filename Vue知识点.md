#Vue前端知识点记录
##一.使用Vue前准备工作
环境： 
node.js v12.18.3
vue 2.9.6
安装element-ui
npm install element-ui -S

##二.常用知识点（一般以我项目中使用代码为例）
###1.axios异步通信 csrf_token跨域问题处理
```js
// main.js
import axios from 'axios'
Vue.prototype.$axios = axios
axios.interceptors.request.use((config) => {
  config.headers['X-Requested-With'] = 'XMLHttpRequest'
  let regex = /.*csrftoken=([^;.]*).*$/ // 用于从cookie中匹配 csrftoken值
  config.headers['X-CSRFToken'] = document.cookie.match(regex) === null ? null : document.cookie.match(regex)[1]
  return config
})
```
###2.在nginx部署中解决跨域问题
config\index.js
```js
module.exports = {
  dev: {
    // Paths
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',
    proxyTable: {
      '/api': {  //使用"/api"来代替"http://f.apiplus.c"
        target: 'http://47.115.52.186:8001/', //源地址
        changeOrigin: true, //改变源
        pathRewrite: {
          '^/api': '' //路径重写
        }
      }
    }
    // ...
  }
}
```
###3.父组件给子组件传值
```vue（DeviceRepairManage）
<!--父组件给子组件（DetailRepairInfomation）传值-->
    <el-table :data="tableData">
      <el-table-column type="expand">
        <template slot-scope="props">
          <DetailRepairInfomation :msg="props.row.Detail"></DetailRepairInfomation>
        </template>
      </el-table-column>
      <el-table-column
        label="设备名称"
        prop="device__name"
        width="100">
      </el-table-column>
    </el-table>
```
```vue(DetailRepairInfomation)
<template>
    <el-table :data="detail_info"style="width: 100%">
        <el-table-column
          prop="category_display"
          label="维修类型"
          width="100">
        </el-table-column>
    </el-table>
</template>

<script>
export default {
  name: 'DetailRepairInfomation',
  data () {
    return {
      detail_info: []
    }
  },
  props: ['msg'],
  mounted () {
    console.log(this.msg)
    this.detail_info = this.msg
  }
```
父组件和子组件都是el-table表格，子组件是父组件表格展开的信息;
父传子主要通过在父组件v-model绑定数据，在子组件进行用props进行数据的接收。