<template>
  <div class="ssl-make">
    <div class="nav">
      <div class="left" @click="doClose">
        <mo-icon name="back" width="24" height="24" />
        <span class="ml-15">SSL Make</span>
      </div>
      <el-button class="shrink0" :loading="running" @click="doSave">生成</el-button>
    </div>

    <div class="main-wapper">
      <div class="main">
        <textarea type="text" :class="'input-textarea' + (errs['domains'] ? ' error' : '')" placeholder="domains eg: *.xxx.com, One domain name per line" v-model.trim="item.domains"></textarea>
        <div class="path-choose mt-20 mb-20">
          <input type="text" :class="'input'+(errs['root'] ? ' error' : '')" readonly="readonly" placeholder="Root CA certificate path, if not choose, will create new in SSL certificate save path" :value="item.root"></input>
          <div class="icon-block" @click="chooseRoot('root', true)">
            <mo-icon class="choose" name="folder" width="18" height="18" />
          </div>
        </div>
        <div class="path-choose mt-20 mb-20">
          <input type="text" :class="'input'+(errs['savePath'] ? ' error' : '')" readonly="readonly" placeholder="SSL certificate save path" :value="item.savePath"></input>
          <div class="icon-block" @click="chooseRoot('save')">
            <mo-icon class="choose" name="folder" width="18" height="18" />
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
// import FileUtil from '@shared/FileUtil'
import '@/components/Icons/folder.js'
import '@/components/Icons/back.js'
import { uuid } from '@shared/utils'
import { existsSync, writeFileSync } from 'fs'
import { execSync } from 'child_process'
import { join, basename } from 'path'
import { EOL } from 'os'
import { Message } from 'element-ui'
import { I18n } from '@/components/Locale'

export default {
  name: 'mo-ssl-make',
  data () {
    return {
      running: false,
      item: {
        domains: '',
        root: '',
        savePath: ''
      },
      edit: {},
      errs: {
        domains: false,
        root: false,
        savePath: false
      }
    }
  },
  components: {
  },
  props: {
  },
  computed: {},
  watch: {
    item: {
      handler (n, o) {
        for (let k in this.errs) {
          this.errs[k] = false
        }
      },
      immediate: true,
      deep: true
    }
  },
  methods: {
    doClose () {
      this.$emit('doClose')
    },
    chooseRoot (flag, choosefile = false) {
      const self = this
      let opt = ['openDirectory', 'createDirectory']
      let filters = []
      if (choosefile) {
        opt = ['openFile']
        filters.push({ name: 'ROOT CA Certificate', extensions: ['crt'] })
      }
      this.$electron.remote.dialog.showOpenDialog({
        properties: opt,
        filters: filters
      }).then(({ canceled, filePaths }) => {
        if (canceled || filePaths.length === 0) {
          return
        }
        const [path] = filePaths
        switch (flag) {
          case 'root':
            self.item.root = path
            break
          case 'save':
            self.item.savePath = path
            break
        }
      })
    },
    checkItem () {
      this.errs.domains = this.item.domains.length === 0
      this.errs.savePath = this.item.savePath.length === 0

      for (let k in this.errs) {
        if (this.errs[k]) {
          return false
        }
      }
      return true
    },
    doSave () {
      if (!this.checkItem()) {
        return
      }
      this.running = true
      let domains = this.item.domains.split('\n').map((item) => {
        return item.trim()
      }).filter((item) => {
        return item && item.length > 0
      })
      let saveName = uuid(6) + '.' + domains[0].replace('*.', '')
      let caFile = this.item.root
      let caFileName = basename(caFile)
      if (caFile.length === 0) {
        caFile = join(this.item.savePath, uuid(6) + '.RootCA.crt')
        caFileName = basename(caFile)
      }
      caFile = caFile.replace('.crt', '')
      caFileName = caFileName.replace('.crt', '')
      let opt = { cwd: this.item.savePath }
      if (!existsSync(caFile + '.crt')) {
        let command = `openssl genrsa -out ${caFileName}.key 2048;`
        command += `openssl req -new -key ${caFileName}.key -out ${caFileName}.csr -sha256 -subj "/CN=Dev Root CA ${caFileName}";`
        command += `echo "basicConstraints=CA:true" > ${caFileName}.cnf;`
        command += `openssl x509 -req -in ${caFileName}.csr -signkey ${caFileName}.key -out ${caFileName}.crt -extfile ${caFileName}.cnf -sha256 -days 365;`
        execSync(command, opt)
      }
      let ext = `authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName=@alt_names

[alt_names]${EOL}`
      domains.forEach((item, index) => {
        ext += `DNS.${index + 1} = ${item}${EOL}`
      })
      ext += `IP.1 = 127.0.0.1${EOL}`
      writeFileSync(join(this.item.savePath, `${saveName}.ext`), ext)

      let command = `openssl req -new -newkey rsa:2048 -nodes -keyout ${saveName}.key -out ${saveName}.csr -sha256 -subj "/CN=${saveName}";`
      command += `openssl x509 -req -in ${saveName}.csr -out ${saveName}.crt -extfile ${saveName}.ext -CA ${caFile}.crt -CAkey ${caFile}.key -CAcreateserial -sha256 -days 365;`
      execSync(command, opt)
      if (existsSync(join(this.item.savePath, `${saveName}.crt`))) {
        this.$alert(`SSL自签名证书生成成功,请在打开的钥匙串中,找到Dev Root CA ${caFileName},并修改为"始终信任"该证书`, '提示', {
          confirmButtonText: '确定',
          callback: action => {
            this.doClose()
            this.$electron.remote.shell.openItem(`${caFile}.crt`)
          }
        })
      } else {
        Message.error(I18n.t('task.task-fail'))
        this.running = false
      }
    }
  },
  created: function () {},
  destroyed () {}
}
</script>

<style lang="scss">
  .ssl-make{
    width: 100%;
    height: 100%;
    background: #1d2033;
    display: flex;
    flex-direction: column;
    user-select: none;
    .nav{
      height: 76px;
      flex-shrink: 0;
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 20px;
      background: #282b3d;
      .left{
        cursor: pointer;
        display: flex;
        align-items: center;
        padding: 6px 0;
      }
    }
    .main-wapper{
      flex: 1;
      width: 100%;
      overflow: auto;
      padding: 12px;
      color: rgba(255,255,255,0.7);
      &::-webkit-scrollbar{
        width: 0;
        height: 0;
        display: none;
      }
      .input{
        background: transparent;
        border-top: none;
        border-left: none;
        border-right: none;
        border-bottom: 1px solid rgba(255,255,255,0.7);
        outline: none;
        height: 42px;
        color: #fff;
        overflow: hidden;
        text-overflow:ellipsis;
        white-space: nowrap;
        &::-webkit-input-placeholder{
          color: rgba(255,255,255,0.7);
        }
        &:hover{
          border-bottom: 2px solid rgba(255,255,255,0.7);
        }
        &:focus{
          border-bottom: 2px solid #01cc74;
        }
        &.error{
          border-bottom: 2px solid #cc5441;
        }
      }
      .input-textarea{
        background: transparent;
        border: 1px solid rgba(255,255,255,0.7);
        outline: none;
        height: 120px;
        color: #fff;
        margin-top: 20px;
        border-radius: 8px;
        padding: 10px;
        resize: none;
        line-height: 1.6;
        &::-webkit-input-placeholder{
          color: rgba(255,255,255,0.7);
        }
        &:hover{
          border: 2px solid rgba(255,255,255,0.7);
        }
        &:focus{
          border: 2px solid #01cc74;
        }
        &.nginx-rewrite{
          height: 140px;
          margin-top: 20px;

        }
        &.error{
          border: 2px solid #cc5441;
        }
      }
      .main{
        background: #32364a;
        border-radius: 8px;
        padding: 20px;
        display: flex;
        flex-direction: column;
        .path-choose{
          display: flex;
          align-items: flex-end;
          .input{
            flex: 1;
          }
          .icon-block{
            margin-left: 30px;
            display: flex;
            .choose{
              color: #01cc74;
            }
          }
        }
        .ssl-switch{
          font-size: 15px;
          font-weight: 600;
          display: flex;
          align-items: center;
          justify-content: space-between;
        }
        .port-set{
          display: flex;
          align-items: flex-end;
          .port-type{
            width: 50px;
            margin-right: 30px;
            flex-shrink: 0;
          }
          .input{
            flex: 1;
            &::-webkit-outer-spin-button{
              -webkit-appearance: none !important;
              margin: 0;
            }
            &::-webkit-inner-spin-button{
              -webkit-appearance: none !important;
              margin: 0;
            }
          }
          &.port-ssl{
            .input{
              margin-right: 48px;
            }
          }
        }
      }
      .plant-title{
        padding: 22px 24px;
        font-size: 15px;
        font-weight: 600;
      }
    }
  }
</style>
