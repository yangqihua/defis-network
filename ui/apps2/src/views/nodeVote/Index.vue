<template>
  <div class="nodeVote">
    <div class="banner">
      <img class="bgImg" src="https://cdn.jsdelivr.net/gh/defis-net/material/banner/poolsVote.png" alt="">
    </div>
    <div class="mainTitle flexb">
      <span class="act">{{ $t('vote.vote') }}</span>
      <span class="flexa rulesTip tip" @click="showRules = !showRules">
        <span>{{ $t('vote.voteRules') }}</span>
        <img class="tipIcon" src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/tips_icon_btn.svg" alt="">
      </span>
    </div>
    <!-- 代理账户信息 -->
    <ProxyAcc :voteWeight="voteWeight" :getLoading="getLoading"/>
    <!-- 用户票数统计数据 -->
    <AccInfo />
    <!-- tab数据 -->
    <div class="tabDiv">
      <div class="tab flexb">
        <div class="nav flexa">
          <span :class="{'act': act === 1}" @click="handleChangeTab(1)">{{ $t('vote.vote') }}</span>
          <span :class="{'act': act === 2}" @click="handleChangeTab(2)">{{ $t('vote.rank') }}</span>
          <span :class="{'act': act === 3}" @click="handleChangeTab(3)">{{ $t('vote.voted') }}</span>
        </div>
        <div class="search flexc">
          <img class="searchImg" src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/search.png" alt="">
          <el-input @input="handleSearch"
            v-model="search" placeholder="搜索市场"></el-input>
        </div>
      </div>
      <div class="lsContent">
        <NodeList :act="act" :nodeLists="nodeLists"
          :search="search"
          :getLoading="getLoading" :myVote="myVote"/>
      </div>
    </div>

    <!-- 悬浮按钮 -->
    <div class="nullDiv"></div>
    <div class="voteAction flexb">
      <span>{{ $t('vote.checked') }} {{ checkedLeng }}/10</span>
      <span>
        <span v-if="checkedLeng" class="tip" @click="handleCancel">{{ $t('vote.cancelChecked') }}</span>
        <span class="voteBtn" v-loading="voteLoading" @click="handleTovote">{{ $t('vote.toVote') }}</span>
      </span>
    </div>

    <el-dialog
      class="mkListDia pcList"
      :show-close="false"
      :visible.sync="showRules">
      <Rules/>
    </el-dialog>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import { EosModel } from '@/utils/eos';

import ProxyAcc from './comp/ProxyAcc';
import AccInfo from './comp/AccInfo';
import NodeList from './comp/NodeList';
import Rules from './dialog/Rules';

import { get_producers, get_table_rows } from '@/utils/api';
export default {
  name: 'nodeVote',
  components: {
    ProxyAcc,
    AccInfo,
    NodeList,
    Rules,
  },
  data() {
    return {
      showRules: false,
      act: 1,
      search: '',
      voteLoading: false,
      nodeLists: [], // 总节点列表
      voteLists: [], // DFS投票列表
      getLoading: true,
      voteWeight: 0,
      myVote: [], // 我的投票列表
      myVoteLoading: true,
    }
  },
  mounted() {
    this.handleGetNodeLists()
    this.handleGetVoteList();
  },
  computed: {
    ...mapState({
      scatter: state => state.app.scatter,
    }),
    checkedLeng() {
      const list = this.nodeLists.filter(v => v.isChecked)
      return list.length;
    }
  },
  watch: {
    act() {
      this.search = ''
      this.handleCancel()
    },
    scatter: {
      handler: function listen(newVal) {
        if (newVal.identity) {
          this.handleGetAccVoteLists();
        }
      },
      deep: true,
      immediate: true,
    }
  },
  methods: {
    handleSearch() {},
    handleCancel() {
      this.nodeLists.forEach(v => {
        this.$set(v, 'isChecked', false)
      })
    },
    handleTovote() {
      this.voteLoading = true;
      const formName = this.scatter.identity.accounts[0].name;
      const permission = this.scatter.identity.accounts[0].authority;
      const checkedArr = this.nodeLists.filter(v => v.isChecked);
      const producers = []
      checkedArr.forEach(v => {
        producers.push(v.owner);
      })
      const params = {
        actions: [{
          account: 'dfsbpsvoters',
          name: 'vote',
          authorization: [{
            actor: formName, // 转账者
            permission,
          }],
          data: {
            voter: formName,
            producers,
          }
        }]
      }
      EosModel.toTransaction(params, (res) => {
        this.voteLoading = false
        if(res.code && JSON.stringify(res.code) !== '{}') {
          this.$message({
            message: res.message,
            type: 'error'
          });
          return
        }
        setTimeout(() => {
          this.handleGetAccVoteLists();
          this.handleGetVoteList()
        }, 2000);
        this.$message({
          message: this.$t('public.success'),
          type: 'success'
        });
      })
    },
    async handleGetNodeLists() {
      const {status, result} = await get_producers()
      this.getLoading = false;
      if (!status) {
        return
      }
      const rows = result.producers || []
      this.$store.dispatch('setNodeLists', JSON.parse(JSON.stringify(rows)))
      this.nodeLists = rows;
      this.handleDealData()
      // console.log(result)
      // 计算权重值
      this.handleGetWeight()
    },
    // 获取DFS 投票列表
    async handleGetVoteList() {
      const params = {
        "code":"dfsbpsvoters",
        "scope":"dfsbpsvoters",
        "table":"producers",
        "json":true,
        "index_position": 2,
        "key_type": "float64",
        "limit": 1000
      }
      const { status, result } = await get_table_rows(params);
      if (!status) {
        return
      }
      const rows = result.rows || [];
      this.voteLists = rows;
      // console.log(result)
      this.handleDealData()
    },
    handleDealData() {
      if (!this.nodeLists.length || !this.voteLists.length) {
        return;
      }
      this.voteLists.forEach(v => {
        const index = this.nodeLists.findIndex(vv => vv.owner === v.bp)
        if (index === -1) {
          return
        }
        this.$set(this.nodeLists[index], 'dfsRank', v.rank)
        const tVote = parseInt(v.total_votes / 10000)
        this.$set(this.nodeLists[index], 'dfsVote', tVote)
      })
      this.nodeLists.sort((a, b) => {
        return Number(b.dfsVote || 0) - Number(a.dfsVote || 0)
      })
    },
    // 获取账户已投列表
    async handleGetAccVoteLists() {
      if (!this.scatter || !this.scatter.identity) {
        return
      }
      const formName = this.scatter.identity.accounts[0].name;
      const params = {
        "code":"dfsbpsvoters",
        "scope":"dfsbpsvoters",
        "table":"voters",
        "json":true,
        "lower_bound": ` ${formName}`,
        "upper_bound": ` ${formName}`,
      }
      const { status, result } = await get_table_rows(params);
      this.myVoteLoading = false;
      if (!status) {
        return
      }
      const rows = result.rows || [];
      if (!rows.length) {
        this.myVote = []
      } else {
        this.myVote = rows[0].last_vote;
      }
      // console.log(this.myVote)
    },
    // 获取全网权重加成
    async handleGetWeight() {
      let sec_since_lanch = 946684800;
      let weight_1 = parseInt((Date.parse(new Date()) / 1000 - sec_since_lanch) / (86400 * 7)) / 52;
      weight_1 = 1 / Math.pow(2, weight_1) / 10000
      this.voteWeight = weight_1;
    },
    handleChangeTab(num) {
      this.act = num
    },
  }
}
</script>

<style lang="scss" scoped>
.nodeVote{
  .banner{
    font-size: 30px;
    color: #fff;
    position: relative;
    overflow: hidden;
    .bgImg{
      display: block;
      width: 100%;
      height: 100%;
      z-index: 0;
    }
  }
  .mainTitle{
    font-size: 32px;
    text-align: left;
    margin: 30px 0;
    padding: 0 0 0 30px;
    &>span{
      margin-right: 60px;;
    }
    .rulesTip{
      font-size: 26px;
      margin-right: 40px;
      .tipIcon{
        width: 28px;
        margin-left: 8px;
        display: block;
      }
    }
    .act{
      color: $color-black;
      position: relative;
      padding-left: 26px;
      &::before{
        content: '';
        position: absolute;
        width: 8px;
        height: 32px;
        background:#02C698;
        border-radius:4px;
        left: 0;
        top: 50%;
        transform: translateY(-45%);
      }
    }
  }
  .tabDiv{
    font-size: 28px;
    text-align: left;
    margin: 10px 0px;
    .tab{
      padding: 0 26px;
      top: 0;
      background: #FFF;
      position: sticky;
      z-index: 10;
      .nav{
        height: 85px;
        background: #FFF;
        color: #a6a6a6;
        &>span{
          margin-right: 40px;
          padding: 26px 0;
          &:last-child{
            margin-right: 0;
          }
        }
        .act{
          color: #000;
          font-weight: 500;
          position: relative;
          &::before{
            content: '';
            position: absolute;
            width: 32px;
            height: 2px;
            background:#333;
            border-radius:4px;
            left: 50%;
            bottom: 0%;
            transform: translateX(-50%);
          }
        }
      }
      .search{
        background: #F5F6F6;
        border-radius: 30px;
        // padding-left: 20px;
        width: 240px;
        // overflow: hidden;
        position: relative;
        .searchImg{
          width: 26px;
          position: absolute;
          left: 20px;
          top: 50%;
          transform: translateY(-50%);
        }
        /deep/ .el-input{
          // padding-left: 10px;
          .el-input__inner{
            flex: 1;
            text-align: center;
            font-size: 28px;
            height: 60px;
            line-height: 38px;
            background: transparent;
            border: 0px;
            &:focus{
              border-color: #07d79b;
            }
          }
          .el-input__prefix{
            margin-left: 10px;
            font-size: 30px;
            width: 30px;
          }
          .el-icon-search{
            display: flex;
            align-items: center;
            justify-content: center;
          }
        }
      }
    }
  }
}
.nullDiv{
  height: 120px;
  width: 100%;
}
.voteAction{
  position: fixed;
  bottom: 0px;
  width: 100%;
  max-width: 750px;
  height: 120px;
  background: #fff;
  padding: 0 40px;
  font-size: 26px;
  box-sizing: border-box;
  border-top: 1px solid #eee;
  .voteBtn{
    margin-left: 20px;
    background: #29D4B0;
    color: #FFF;
    padding: 12px 40px;
    border-radius: 30px;
  }
}

.mkListDia{
  // animation: none;
  /deep/ .el-dialog{
    width: 600px;
    .el-dialog__body,
    .el-dialog__header{
      padding: 0;
      box-sizing: border-box;
    }
  }
  &.pcList{
    /deep/ .el-dialog{
      position: relative;
      border-radius:30px;
    }
  }
}
</style>