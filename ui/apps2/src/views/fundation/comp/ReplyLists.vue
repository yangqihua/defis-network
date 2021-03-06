<template>
  <div>
    <div class="lists" v-loading="loading">
      <div class="list" v-for="(item, index) in lists" :key="`reply${index}`">
        <div class="flexa">
          <div class="headImg">
            <img width="100%" :src="item.headImg" alt="">
          </div>
          <div class="mainDiv">
            <div class="flexb">
              <div class="name">
                <div>{{ item.fromx }}</div>
                <div class="tip funNum dinReg">{{ $t('fundation.transNum') }}: {{ item.quantity }}</div>
              </div>
              <div class="likeDiv tip flexend" @click="handleShowLike(item)">
                <span>{{ item.likeNum }}</span>
                <img v-if="!item.like_status" src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/newlike.png" alt="">
                <img v-else width="20px" src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/newlike1.png" alt="">
              </div>
            </div>
          </div>
        </div>

        <div class="memo" @click="handleShowToFundation(item)">
          <span>
            <span class="reply">{{ $t('fundation.reply') }}</span>
            <span class="green">{{ item.replyto }}</span>
          </span>
          <span>{{ item.memo }}</span>
        </div>
        <div class="time tip flexa" @click="handleShowToFundation(item)">
          <span>{{ handleToLocalTime(item.dealTime) }}</span>
          <span class="reply">{{ $t('fundation.reply') }}</span>
        </div>
      </div>
    </div>
    <div class="showMore flexa" v-if="lists.length < subLen" @click="handleGetMore">
      <span>{{ $t('fundation.more') }}</span>
      <img src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/showMore.png" alt="">
    </div>
    <div class="closeMore flexa" v-else @click="handleCloseMore">
      <span>{{ $t('fundation.close') }}</span>
      <img src="https://cdn.jsdelivr.net/gh/defis-net/material/icon/showMore.png" alt="">
    </div>
    <!-- 去捐款 -->
    <el-dialog
      class="mydialog"
      :show-close="false"
      :visible="showToFundation">
      <ToFundation v-if="showToFundation" :reply="reply" :replyItem="replyItem"
        @listenClose="handleClose"/>
    </el-dialog>
    <!-- 捐爱心 -->
    <el-dialog
      class="mydialog"
      :show-close="false"
      :visible="showToLike">
      <Like v-if="showToLike" :reply="reply" :replyItem="replyItem" @listenClose="handleClose"/>
    </el-dialog>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import moment from 'moment';
import {getDateDiff, toLocalTime, getRandomImg} from '@/utils/public'

import {get_reply_fundation} from '@/utils/api'
import Like from '../dialog/Like';
import ToFundation from '../dialog/ToFundation';

export default {
  name: 'replyLists',
  components: {
    Like,
    ToFundation,
  },
  props: {
    listsLen: { // 总计多少条回复
      type: Number,
      default: 0
    },
    reply: {
      type: Object,
      default: function rp() {
        return {}
      },
    },
  },
  data() {
    return {
      pageSize: 10,
      page: 1,
      lists: [],
      showToLike: false,
      replyItem: {},
      showToFundation: false,
      subLen: 0,
      loading: true,
    }
  },
  mounted() {
    this.handleGetReplyFirst()
  },
  computed: {
    ...mapState({
      scatter: state => state.app.scatter,
    }),
  },
  methods: {
    handleToLocalTime(time) {
      let t = moment(`${time}`).valueOf()
      // t += 3600 * 8 * 1000;
      const oDate = getDateDiff(t)
      return oDate
    },
    handleShowToFundation(item) {
      this.replyItem = item;
      this.showToFundation = true;
    },
    handleShowLike(item) {
      this.replyItem = item;
      this.showToLike = true;
    },
    handleClose() {
      this.showToLike = false;
      this.showToFundation = false;
    },
    async handleGetReplyFirst(size = 3) {
      // 第一次展开 获取前3条 - mounted调用
      const params = {
        page: this.page,
        limit: size,
        id: this.reply.global_action_seq
      }
      if (this.scatter && this.scatter.identity) {
        params.user = this.scatter.identity.accounts[0].name;
      }
      const {status, result} = await get_reply_fundation(params)
      // console.log(status, result)
      this.loading = false;
      if (!status) {
        return
      }
      const lists = result.data || [];
      this.subLen = result.total;
      lists.forEach(v => {
         this.$set(v, 'headImg', getRandomImg())
        const t = toLocalTime(v.create_time).replace(/-/g, '/');
        const times = Date.parse(t) + 3600 * 8 * 1000;
        this.$set(v, 'dealTime', toLocalTime(times))
        const likeNum = v.like_count * 1000;
        this.$set(v, 'likeNum', likeNum.toFixed(0))
      });
      this.lists = lists;
    },
    // 点击展开更多时调用
    async handleGetMore() {
      const params = {
        page: this.page,
        limit: 10,
        id: this.reply.global_action_seq
      }
      const {status, result} = await get_reply_fundation(params)
      if (!status) {
        return
      }
      const lists = result.data || [];
      lists.forEach(v => {
        const t = toLocalTime(v.create_time).replace(/-/g, '/');
        const times = Date.parse(t) + 3600 * 8 * 1000;
        this.$set(v, 'dealTime', toLocalTime(times))
      });
      // 调用API获取前10条
      if (this.page === 1) { // 第一页时 - 直接覆盖之前数据
        this.lists = lists
      } else { // 下一页时，在后面插入
        this.lists.push(...lists)
      }
      this.page = this.page + 1;
    },
    // 收起弹窗
    handleCloseMore() {
      this.$emit('listenCloseSubLists', this.reply)
    }
  }
}
</script>

<style lang="scss" scoped>
.green{
  color: #29D4B0 !important;
}
.lists{
  .list{
    padding: 20px 0;
    .headImg{
      width: 52px;
      height: 52px;
      overflow: hidden;
      border-radius: 30px;
      margin-right: 8px;
    }
    .likeDiv{
      font-size: 22px;
      img{
        width: 32px;
        margin-left: 8px;
      }
    }
    .mainDiv{
      flex: 1;
      .funNum{
        margin-top: 4px;
        font-size: 22px;
      }
    }
    .memo{
      margin-top: 4px;
      .reply{
        margin: 0 8px 0 0;
      }
      .green{
        margin-right: 8px;
      }
    }
    .time{
      margin-top: 4px;
      font-size: 20px;
    }
    .reply{
      color: #999;
      font-size: 26px;
      margin-left: 8px;
      font-weight: 500;
    }
  }
}
.showMore{
  color: #29D4B0;
  margin-top: 8px;
  img{
    width: 20px;
    margin-left: 6px;
  }
}
.closeMore{
  color: #29D4B0;
  margin-top: 8px;
  &>span{
    margin-right: 6px;
  }
  img{
    width: 20px;
    transform: rotate(180deg);
  }
}
</style>
