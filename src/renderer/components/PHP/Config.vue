
<template>
  <div class="php-config">
    <el-input class="block" type="textarea" v-model="config"></el-input>
    <div class="tool">
      <el-button @click="openConfig" >打开</el-button>
      <el-button @click="saveConfig" >保存</el-button>
      <el-button @click="getDefault" >加载默认</el-button>
    </div>
  </div>
</template>

<script>
  import '@/components/Icons/task-start.js'
  import '@/components/Icons/task-stop.js'
  import { join } from 'path'
  import { existsSync } from 'fs'
  import FileUtil from '@shared/FileUtil'
  export default {
    name: 'mo-php-config',
    data () {
      return {
        config: ''
      }
    },
    components: {
    },
    props: {},
    computed: {
    },
    watch: {
    },
    methods: {
      openConfig () {
        this.$electron.remote.shell.showItemInFolder(this.configpath)
      },
      saveConfig () {
        FileUtil.writeFileAsync(this.configpath, this.config).then(conf => {
          this.$message.success('配置文件保存成功')
        })
      },
      getConfig () {
        FileUtil.readFileAsync(this.configpath).then(conf => {
          this.config = conf
        })
      },
      getDefault () {
        let configpath = join(global.Server.PhpDir, 'common/conf/php.ini.default')
        if (!existsSync(configpath)) {
          this.$message.error('未找到默认配置文件')
          return
        }
        FileUtil.readFileAsync(configpath).then(conf => {
          this.config = conf
        })
      }
    },
    created: function () {
      this.configpath = join(global.Server.PhpDir, 'common/conf/php.ini')
      this.getConfig()
    }
  }
</script>

<style lang="scss">
  .php-config{
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 20px 0 0 20px;
    .block{
      display: flex;
      align-items: center;
      flex: 1;
      font-size: 15px;
      textarea{
        height: 100%;
      }
    }
    .tool{
      flex-shrink: 0;
      width: 100%;
      display: flex;
      align-items: center;
      padding: 30px 0;
    }
  }
</style>
