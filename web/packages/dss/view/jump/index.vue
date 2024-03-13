<template>
  <div class="parent-box">
    <div class="contant-box">
      <div class="contant-title">Loading...</div>
    </div>

  </div>
</template>
<script>
import api from '@dataspherestudio/shared/common/service/api';
import storage from '@dataspherestudio/shared/common/helper/storage';
import mixin from '@dataspherestudio/shared/common/service/mixin';
import { db } from '@dataspherestudio/shared/common/service/db/index.js';
import { config } from '@dataspherestudio/shared/common/config/db.js';
import util from '@dataspherestudio/shared/common/util/';
import tab from '@/scriptis/service/db/tab.js';
// import eventbus from '@dataspherestudio/shared/common/helper/eventbus';
// import plugin from '@dataspherestudio/shared/common/util/plugin'
import {
  development_logout_url,
  production_logout_url,
} from '../../assets/javascripts/config.js';

export default {
  data() {
    return {
      loading: false,
      loginForm: {
        user: 'hadoop',
        password: '',
      },
      ruleInline: {
        user: [
          { required: true, message: this.$t('message.common.login.userName'), trigger: 'blur' },
          // {type: 'string', pattern: /^[0-9a-zA-Z\.\-_]{4,16}$/, message: '无效的用户名！', trigger: 'change'},
        ],
        password: [
          { required: true, message: this.$t('message.common.login.password'), trigger: 'blur' },
          // {type: 'string', pattern: /^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])[0-9a-zA-Z]{8,18}$/, message: '请输入6至12位的密码', trigger: 'change'},
        ],
      },
      rememberUserNameAndPass: false,
      publicKeyData: null,
      Authorization: decodeURIComponent(this.$route.query.token || ''),
      menuId: this.$route.query.menuId || '',
      workspaceId: this.$route.query.workspaceId || '',
    };
  },
  mixins: [mixin],
  created() {
    this.getThirdLogin()
  },
  mounted() {
    storage.set('close_db_table_suggest', false);
    const workspaceId = this.getCurrentWorkspaceId()
    sessionStorage.removeItem(`work_flow_lists_${workspaceId}`)
  },
  methods: {
    logout() {
      api.fetch('/user/logout', {}).then(() => {
        this.$emit('clear-session');
        storage.set('need-refresh-proposals-hql', true);
        storage.set('need-refresh-proposals-python', true);
        this.$router.push({ path: '/login' });
      });
    },
    // 获取登录后的url调转
    getPageHomeUrl() {
      const currentModules = util.currentModules();
      return api.fetch(`${this.$API_PATH.WORKSPACE_PATH}getWorkspaceHomePage`, {
        micro_module: currentModules.microModule || 'dss'
      }, 'get').then((res) => {
        storage.set('noWorkSpace', false, 'local')
        console.log(res)
        return res.workspaceHomePage;
      }).catch((e) => {
        storage.set('noWorkSpace', true, 'local');
        this.logout();
        throw e;
      });
    },

    // 获取登录接口
    async getThirdLogin() {
      // 登录清掉本地缓存
      // 保留Scripts页面打开的tab页面
      // 连续两次退出登录后，会导致数据丢失，所以得判断是否已存切没有使用
      let tabs = await tab.get() || [];
      const tablist = storage.get(this.loginForm.user + 'tabs', 'local');
      if (!tablist || tablist.length <= 0) {
        storage.set(this.loginForm.user + 'tabs', tabs, 'local');
      }
      Object.keys(config.stores).map((key) => {
        db.db[key].clear();
      })

      let option={
        headers: {
          Authorization: this.Authorization
        }
      }
      api.fetch(`/dss-api/subsystem/login?systemType=1`, {},option).then(async (res) => {
        if (res) {
          // // 跳转去旧版
          // if (res.redirectLinkisUrl) {
          //   location.href = res.redirectLinkisUrl;
          //   return
          // }
          this.baseInfo = { username: 'this.loginForm.user' };
          storage.set('baseInfo', this.baseInfo, 'local');
          this.getIsAdmin()
          // 登录之后需要获取当前用户的调转首页的路径
          // const homePageRes = await this.getPageHomeUrl()
          // const all_after_login = await plugin.emitHook('after_login', {
          //   context: this,
          //   homePageRes
          // })
          // eventbus.emit('watermark.refresh');
          // if (all_after_login.length) {
          //   // 有hook返回则hook处理
          // } else {
          //   this.$router.replace({ path: homePageRes.homePageUrl });
          // }
          this.$router.replace({ path: `/${this.menuId}?workspaceId=${this.workspaceId}`});
          this.$Message.success(this.$t('message.common.login.loginSuccess'));
        }
      }).catch(async () =>{
        if (process.env.NODE_ENV === 'development') {
          console.log('当前环境是开发版本');
          // 在开发版本中执行的逻辑
          window.location.replace(`${development_logout_url}`);
        } else {
          console.log('当前环境是生产版本');
          // 在生产版本中执行的逻辑
          window.location.replace(`${production_logout_url}`);
        }

      })
    },
    getIsAdmin() {
      api.fetch(`/jobhistory/governanceStationAdmin`, {}, 'get').then((rst) => {
        this.baseInfo = { username: this.loginForm.user, isAdmin: rst.admin }
        storage.set('baseInfo', this.baseInfo, 'local');
      })
    },

  },
};
</script>
<style lang="less" scoped>
.parent-box {
    height: 100%;
    background: #001C40;
    display: flex;
    align-items: center;
    justify-content: center;

    .contant-box {
        width: 300px;
        height: 30px;
        border-radius: 20px;
        color: #007485;
        border: 2px solid;
        position: relative;
    }
    .contant-box::before {
        content: "";
        position: absolute;
        margin: 2px;
        inset: 0 100% 0 0;
        border-radius: inherit;
        background: #007485;
        animation: cartoon 2s infinite;
    }
    .contant-title{
        position: absolute;
        top: 3px;
        left: 125px;
        color: #ffffff;
    }

    @keyframes cartoon {
        100% {
            inset: 0;
        }
    }
}
</style>

