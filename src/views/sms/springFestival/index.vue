<template>
  <div class="app-container">
    <el-card class="filter-container" shadow="never">
      <div>
        <i class="el-icon-s-flag"></i>
        <span>2026 年春节活动（MVP 快速配置）</span>
      </div>
      <div style="margin-top: 10px;font-size: 13px;color:#606266;line-height: 1.8;">
        <p>本页面用于在 <b>3 小时内</b> 完成春节活动的基础配置，不新增复杂功能，只复用现有模块：</p>
        <p>1）通过「优惠券列表」创建带有“春节”字样的优惠券；</p>
        <p>2）通过「商品列表」筛选并配置参与春节活动的商品；</p>
        <p>3）前台活动页可根据“春节”券和商品标签/分类进行展示（由后端和前台配合实现）。</p>
      </div>
      <div style="margin-top: 10px;">
        <el-button type="primary" size="small" @click="goCouponList">查看/创建春节优惠券</el-button>
        <el-button size="small" @click="goAddCoupon">新建优惠券</el-button>
        <el-button size="small" @click="goProductList">查看商品列表</el-button>
      </div>
    </el-card>
    <el-card class="filter-container" shadow="never" style="margin-top: 20px;">
      <div>
        <i class="el-icon-search"></i>
        <span>春节优惠券快速查看（名称包含“春节”）</span>
        <el-button
          style="float:right"
          type="primary"
          @click="handleSearchList()"
          size="small">
          刷新
        </el-button>
      </div>
    </el-card>
    <div class="table-container">
      <el-table
        :data="list"
        style="width: 100%;"
        v-loading="listLoading" border>
        <el-table-column label="编号" width="100" align="center">
          <template slot-scope="scope">{{scope.row.id}}</template>
        </el-table-column>
        <el-table-column label="优惠劵名称" align="center">
          <template slot-scope="scope">{{scope.row.name}}</template>
        </el-table-column>
        <el-table-column label="优惠券类型" width="100" align="center">
          <template slot-scope="scope">{{scope.row.type | formatType}}</template>
        </el-table-column>
        <el-table-column label="可使用商品" width="100" align="center">
          <template slot-scope="scope">{{scope.row.useType | formatUseType}}</template>
        </el-table-column>
        <el-table-column label="使用门槛" width="140" align="center">
          <template slot-scope="scope">满{{scope.row.minPoint}}元可用</template>
        </el-table-column>
        <el-table-column label="面值" width="100" align="center">
          <template slot-scope="scope">{{scope.row.amount}}元</template>
        </el-table-column>
        <el-table-column label="适用平台" width="100" align="center">
          <template slot-scope="scope">{{scope.row.platform | formatPlatform}}</template>
        </el-table-column>
        <el-table-column label="有效期" width="180" align="center">
          <template slot-scope="scope">{{scope.row.startTime|formatDate}}至{{scope.row.endTime|formatDate}}</template>
        </el-table-column>
        <el-table-column label="状态" width="100" align="center">
          <template slot-scope="scope">{{scope.row.endTime | formatStatus}}</template>
        </el-table-column>
        <el-table-column label="操作" width="180" align="center">
          <template slot-scope="scope">
            <el-button size="mini"
                       type="text"
                       @click="handleView(scope.$index, scope.row)">查看</el-button>
            <el-button size="mini"
                       type="text"
                       @click="handleUpdate(scope.$index, scope.row)">
              编辑</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <div class="pagination-container">
      <el-pagination
        background
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        layout="total, sizes,prev, pager, next,jumper"
        :current-page.sync="listQuery.pageNum"
        :page-size="listQuery.pageSize"
        :page-sizes="[5,10,15]"
        :total="total">
      </el-pagination>
    </div>
  </div>
</template>

<script>
  import {fetchList} from '@/api/coupon';
  import {formatDate} from '@/utils/date';
  const defaultListQuery = {
    pageNum: 1,
    pageSize: 10,
    name: '春节',
    type: null
  };
  const defaultTypeOptions=[
    {
      label: '全场赠券',
      value: 0
    },
    {
      label: '会员赠券',
      value: 1
    },
    {
      label: '购物赠券',
      value: 2
    },
    {
      label: '注册赠券',
      value: 3
    }
  ];
  export default {
    name:'springFestivalCouponList',
    data() {
      return {
        listQuery:Object.assign({},defaultListQuery),
        list:null,
        total:null,
        listLoading:false,
        typeOptions:Object.assign({},defaultTypeOptions)
      }
    },
    created(){
      this.getList();
    },
    filters:{
      formatType(type){
        for(let i=0;i<defaultTypeOptions.length;i++){
          if(type===defaultTypeOptions[i].value){
            return defaultTypeOptions[i].label;
          }
        }
        return '';
      },
      formatUseType(useType){
        if(useType===0){
          return '全场通用';
        }else if(useType===1){
          return '指定分类';
        }else{
          return '指定商品';
        }
      },
      formatPlatform(platform){
        if(platform===1){
          return '移动平台';
        }else if(platform===2){
          return 'PC平台';
        }else{
          return '全平台';
        }
      },
      formatDate(time){
        if(time==null||time===''){
          return 'N/A';
        }
        let date = new Date(time);
        return formatDate(date, 'yyyy-MM-dd')
      },
      formatStatus(endTime){
        let now = new Date().getTime();
        let endDate = new Date(endTime);
        if(endDate>now){
          return '未过期'
        }else{
          return '已过期';
        }
      }
    },
    methods:{
      getList(){
        this.listLoading=true;
        fetchList(this.listQuery).then(response=>{
          this.listLoading = false;
          this.list = response.data.list;
          this.total = response.data.total;
        });
      },
      handleSearchList() {
        this.listQuery.pageNum = 1;
        this.getList();
      },
      handleSizeChange(val) {
        this.listQuery.pageNum = 1;
        this.listQuery.pageSize = val;
        this.getList();
      },
      handleCurrentChange(val) {
        this.listQuery.pageNum = val;
        this.getList();
      },
      handleView(index, row) {
        this.$router.push({path: '/sms/couponHistory', query: {id: row.id}})
      },
      handleUpdate(index, row) {
        this.$router.push({path: '/sms/updateCoupon', query: {id: row.id}})
      },
      goCouponList(){
        this.$router.push({path:'/sms/coupon'});
      },
      goAddCoupon(){
        this.$router.push({path:'/sms/addCoupon'});
      },
      goProductList(){
        this.$router.push({path:'/pms/product'});
      }
    }
  }
</script>

<style scoped>
  .table-container{
    margin-top: 20px;
  }
</style>
