<script>
import lodash from 'lodash'
import DsContainer from '../components/container'
import SetupCa from '../components/setup-ca'

export default {
  name: 'Index',
  components: {
    DsContainer,
    SetupCa,
  },
  data () {
    return {
      status: undefined,
      startup: {
        loading: false,
        type: () => {
          return (this.status.server && this.status.server.enabled) ? 'primary' : 'default'
        },
        doClick: () => {
          if (this.status.server.enabled) {
            this.apiCall(this.startup, this.$api.shutdown)
          } else {
            this.apiCall(this.startup, this.$api.startup)
          }
        },
      },
      info: {},
      newVersionDownloading: false,
      setting: {},
      server: {
        key: '代理服务',
        loading: false,
        doClick: (checked) => {
          this.onServerClick(checked)
        },
      },
      switchBtns: undefined,
      config: undefined,
      setupCa: {
        visible: false,
      },
      update: { checking: false, downloading: false, progress: 0, newVersion: false },
    }
  },
  computed: {
    _rootCaSetuped () {
      if (this.setting.rootCa) {
        return this.setting.rootCa.setuped === true
      }
      return false
    },
  },
  async created () {
    await this.doCheckRootCa()
    await this.reloadConfig()
    this.$set(this, 'status', this.$status)
    this.switchBtns = this.createSwitchBtns()
    this.$set(this, 'update', this.$global.update)
    if (!this.update.autoChecked && this.config.app.autoChecked) {
      this.update.autoChecked = true // 应用启动时，执行一次
      this.doCheckUpdate(false)
    }
    this.$api.info.get().then((ret) => {
      this.info = ret
    })
  },
  mounted () {
  },
  methods: {
    async modeChange (event) {
      const mode = this.config.app.mode
      if (mode === 'safe') {
        this.config.server.intercept.enabled = false
        this.config.server.dns.speedTest.enabled = true
        this.config.plugin.overwall.enabled = false
      } else if (mode === 'default') {
        this.config.server.intercept.enabled = true
        this.config.server.dns.speedTest.enabled = true
        this.config.plugin.overwall.enabled = false
      } else if (mode === 'ow') {
        console.log('event', event)
        if (!this.setting.overwall) {
          this.wantOW()
          return
        }
        this.config.server.intercept.enabled = true
        this.config.server.dns.speedTest.enabled = true
        this.config.plugin.overwall.enabled = true
      }
      this.$api.config.save(this.config).then(() => {
        this.$message.success('设置已保存')
      })
      if (this.status.server.enabled) {
        return this.$api.server.restart()
      }
    },
    wantOW () {
      this.$success({
        title: '彩蛋（增强模式）',
        content: (
          <div>
            我把它藏在了源码里，感兴趣的话可以找一找它（线索提示 // TODO）
          </div>
        ),
      })
    },
    async doCheckRootCa () {
      const setting = await this.$api.setting.load()
      console.log('setting', setting)
      this.setting = setting || {}
      if (this.setting.rootCa && (this.setting.rootCa.setuped || this.setting.rootCa.noTip)) {
        return
      }
      this.$confirm({
        title: '第一次使用，请先安装CA根证书',
        content: '本应用正常使用，必须安装和信任CA根证书',
        cancelText: '下次安装',
        okText: '去安装',
        onOk: () => {
          this.openSetupCa()
        },
        onCancel: () => {
          this.setting.rootCa = this.setting.rootCa || {}
          // const rootCa = this.setting.rootCa
          // rootCa.noTip = true
          // this.$api.setting.save(this.setting)
        },
      })
    },
    openSetupCa () {
      this.setupCa.visible = true
    },
    getDateTimeStr () {
      const date = new Date() // 创建一个表示当前日期和时间的 Date 对象
      const year = date.getFullYear() // 获取年份
      const month = String(date.getMonth() + 1).padStart(2, '0') // 获取月份（注意月份从 0 开始计数）
      const day = String(date.getDate()).padStart(2, '0') // 获取天数
      const hours = String(date.getHours()).padStart(2, '0') // 获取小时
      const minutes = String(date.getMinutes()).padStart(2, '0') // 获取分钟
      const seconds = String(date.getSeconds()).padStart(2, '0') // 获取秒数
      const milliseconds = String(date.getMilliseconds()).padStart(3, '0') // 获取毫秒
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}.${milliseconds}`
    },
    async handleCaSetuped () {
      console.log('this.config.server.setting.rootCaFile.certPath', this.config.server.setting.rootCaFile.certPath)
      await this.$api.shell.setupCa({ certPath: this.config.server.setting.rootCaFile.certPath })
      this.setting.rootCa = this.setting.rootCa || {}
      const rootCa = this.setting.rootCa

      // 根证书已安装
      rootCa.setuped = true
      // 保存安装时间
      rootCa.setupTime = this.getDateTimeStr()
      // 保存安装描述
      rootCa.desc = '根证书已安装'
      // 删除noTip数据
      // delete rootCa.noTip

      this.$set(this, 'setting', this.setting)
      this.$api.setting.save(this.setting)
    },
    reloadConfig () {
      return this.$api.config.reload().then((ret) => {
        this.config = ret
        return ret
      })
    },
    createSwitchBtns () {
      console.log('api,', this.$api)
      const btns = {}
      const status = this.status
      btns.server = this.createSwitchBtn('server', '代理服务', this.$api.server, status)
      btns.proxy = this.createSwitchBtn('proxy', '系统代理', this.$api.proxy, status)
      lodash.forEach(status.plugin, (item, key) => {
        if (this.config.plugin[key].statusOff) {
          return
        }
        btns[key] = this.createSwitchBtn(key, this.config.plugin[key].name, this.$api.plugin[key], status.plugin, this.config.plugin[key].tip)
      })
      return btns
    },
    createSwitchBtn (key, label, apiTarget, statusParent, tip) {
      return {
        loading: false,
        key,
        label,
        tip,
        status: () => {
          return statusParent[key].enabled
        },
        doClick: (checked) => {
          this.onSwitchClick(this.switchBtns[key], apiTarget.start, apiTarget.close, checked)
        },
      }
    },
    async apiCall (btn, api, param) {
      btn.loading = true
      try {
        const ret = await api(param)
        console.log('this status', this.status)
        return ret
      } catch (err) {
        btn.loading = false // 有时候记录日志会卡死，先设置为false
        console.log('api invoke error:', err)
      } finally {
        btn.loading = false
      }
    },

    onSwitchClick (btn, openApi, closeApi, checked) {
      if (checked) {
        return this.apiCall(btn, openApi)
      } else {
        return this.apiCall(btn, closeApi)
      }
    },
    onServerClick (checked) {
      return this.onSwitchClick(this.server, this.$api.server.start, this.$api.server.close, checked)
    },
    start (checked) {
      this.apiCall(this.startup, this.$api.startup)
    },
    openSettings () {
      this.setting.visible = true
    },
    onConfigChanged (newConfig) {
      console.log('config changed', newConfig)
      this.reloadConfig().then(() => {
        if (this.status.server) {
          return this.$api.server.restart()
        }
      })
    },
    goDonate () {
      this.$message.info('感谢支持')
    },
    doCheckUpdate (fromUser) {
      this.$api.update.checkForUpdate(fromUser)
    },
    async openExternal (url) {
      await this.$api.ipc.openExternal(url)
    },
    onShutdownTipClose (e) {
      this.$confirm({
        title: '是否永久关闭该提示',
        okText: '我已知晓，不再提示',
        cancelText: '下次还显示',
        onOk: () => {
          this.$api.config.update({ app: { showShutdownTip: false } })
        },
      })
    },
  },
}
</script>

<template>
  <DsContainer class="page_index">
    <template slot="header">
      给开发者的辅助工具
    </template>
    <template slot="header-right">
      <a-button style="margin-right:10px" @click="openSetupCa">
        <a-badge :count="_rootCaSetuped ? 0 : 1" dot>安装根证书</a-badge>
      </a-button>

      <a-button
        style="margin-right:10px" :loading="update.downloading || update.checking" :title="`当前版本:${info.version}`"
        @click="doCheckUpdate(true)"
      >
        <a-badge :count="update.newVersion ? 1 : 0" dot>
          <span v-if="update.downloading">{{ update.progress }}%</span>{{ update.downloading ? '新版本下载中' : (`检查更新${update.checking ? '中' : ''}`) }}
        </a-badge>
      </a-button>
    </template>

    <div class="box">
      <a-alert v-if="config && config.app.showShutdownTip" message="本应用开启后会修改系统代理，直接重启电脑可能会无法上网，您可以再次启动本应用即可恢复。如您需要卸载，在卸载前请务必完全退出本应用再进行卸载" banner closable @close="onShutdownTipClose" />
      <div v-if="config && config.app" class="mode-bar" style="margin:20px;">
        <a-radio-group v-model="config.app.mode" button-style="solid" @change="modeChange">
          <a-tooltip placement="topLeft" title="启用测速，关闭拦截，关闭增强（不稳定，不需要安装证书，最安全）">
            <a-radio-button value="safe">
              安全模式
            </a-radio-button>
          </a-tooltip>
          <a-tooltip placement="topLeft" title="启用测速，启用拦截，关闭增强（需要安装证书）">
            <a-radio-button value="default">
              默认模式
            </a-radio-button>
          </a-tooltip>
          <a-tooltip v-if="setting.overwall" placement="topLeft" title="一个简单的梯子（敏感原因，默认隐藏，更多信息请点击左侧增强功能菜单）">
            <a-radio-button value="ow">
              增强模式
            </a-radio-button>
          </a-tooltip>
          <a-tooltip v-else placement="topLeft" title="这个页面有个彩蛋">
            <a-radio-button :disabled="true" value="ow">
              彩蛋
            </a-radio-button>
          </a-tooltip>
        </a-radio-group>
      </div>

      <div
        v-if="status"
        style="margin-top:20px;display: flex; align-items:center;justify-content:space-around;flex-direction: row"
      >
        <div style="text-align: center">
          <div class="big_button">
            <a-button shape="circle" :type="startup.type()" :loading="startup.loading" @click="startup.doClick">
              <img v-if="!startup.loading && !status.server.enabled" width="50" src="/logo/logo-simple.svg">
              <img v-if="!startup.loading && status.server.enabled" width="50" src="/logo/logo-fff.svg">
            </a-button>
            <div class="mt10">
              {{ status.server.enabled ? '已开启' : '已关闭' }}
            </div>
          </div>
        </div>
        <div :span="12">
          <a-form style="margin-top:20px" :label-col="{ span: 15 }" :wrapper-col="{ span: 9 }">
            <a-form-item v-for="(item, key) in switchBtns" :key="key" :label="item.label">
              <a-tooltip placement="topLeft">
                <a-switch
                  style="margin-left:10px" :loading="item.loading" :checked="item.status()" default-checked
                  @change="item.doClick"
                >
                  <a-icon slot="checkedChildren" type="check" />
                  <a-icon slot="unCheckedChildren" type="close" />
                </a-switch>
              </a-tooltip>
            </a-form-item>
          </a-form>
        </div>
      </div>
    </div>

    <SetupCa title="安装证书" :visible.sync="setupCa.visible" @setup="handleCaSetuped" />
    <div slot="footer">
      <div v-if="!setting.overwall" class="star">
        <div class="donate">
          <a-tooltip placement="topLeft" title="彩蛋，点我">
            <span style="display: block;width:100px;height:50px;" @click="wantOW()" />
          </a-tooltip>
        </div>
        <div class="right" />
      </div>
      <div v-if="setting.development == null || !setting.development" class="star">
        <div class="donate" />
        <div class="right">
          <div>
            如果它解决了你的问题，请不要吝啬你的star哟！点这里
            <a-icon style="margin-right:10px;" type="arrow-right" theme="outlined" />
          </div>
          <a @click="openExternal('https://github.com/docmirror/dev-sidecar')"><img
            alt="GitHub stars"
            src="https://img.shields.io/github/stars/docmirror/dev-sidecar?logo=github"
          ></a>
        </div>
      </div>
    </div>
  </DsContainer>
</template>

<style lang="scss">
.page_index {
  .mode-bar {
    margin: 30px;
    text-align: center;
  }

  .star {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    padding: 10px;
    .donate {
      cursor: pointer;
    }

    .right {
      display: flex;
      flex-direction: row;
      justify-content: flex-end;
    }

    > * {
      margin-right: 10px;
    }

    a {
      height: 21px;

      img {
        height: 21px;
      }
    }
  }
}

.payQrcode {
  padding: 10px;
  display: flex;
  justify-content: space-evenly;
}

.big_button > button {
  width: 100px;
  height: 100px;
  border-radius: 100px;
}

.big_button > button i {
  size: 40px;
}

div.ant-form-item {
  margin-bottom: 9px;
}
</style>
