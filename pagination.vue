<template>
  <el-pagination
      v-if="total > 0"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="currentpage"
      :page-sizes="pageSizes"
      :page-size="pagesize"
      layout="slot, total, prev, pager, next, sizes, jumper"
      :total="total">
    <span style="margin-right: 10px;color: rgb(0, 136, 204);font-size: 12px;">已选择{{selectnum}}条数据</span>
    <span class="page-str">{{currentDataStr}}</span>
  </el-pagination>
</template>

<script>
module.exports = {
  name: "pagination",
  props: {
    pagesizes: [],
    sizetype: "small",
    total: 0,
    currentpage: 1,
    pagesize: 20,
    selectnum:0,
  },
  data() {
    return {
      pageSizes: [],
      pageSizesSmall: [1, 5, 10, 15, 20], // 主要用于弹窗中的列表
      // pageSizesMedium: [20, 50, 100, 200],  // 通用分页尺寸
      pageSizesMedium: [20, 30, 50, 100, 200,500,1000],  // 通用分页尺寸
      pageSizesBig: [100, 300, 500, 1000, 2000, 3000, 5000] // 主要用于分发模块
    }
  },
  computed: {
    currentDataStr() {
      let str1 = this.pagesize * (this.currentpage - 1) + 1;
      let str2 = this.pagesize * this.currentpage <= this.total ? this.pagesize * this.currentpage : this.total;
      let str = this.total ? `第 ${str1}-${str2} 条 / ` : "";

      return str;
    }
  },
  mounted() {
    console.log('selectNum',this.selectNum)
    if (this.pagesizes) {
      this.pageSizes = this.pagesizes;
    } else {
      this.pageSizes = this.sizetype === 'big' ? this.pageSizesBig : (this.sizetype === 'medium' ? this.pageSizesMedium : this.pageSizesSmall);
    }
  },
  methods: {
    handleSizeChange(val) {
      this.$emit("size-change", val);
    },
    handleCurrentChange(val) {
      this.$emit("page-change", val);
    }
  }
}
</script>

<style scoped>
.page-pagination {
  margin: 20px 0;
  text-align: right;
}
.el-pager li {
  width: 32px;
  height: 32px;
  margin-right: 8px;
  background: #FFFFFF;
  border-radius: 2px;
  border: 1px solid #D9D9D9;
  padding: 0;
  min-width: 32px;
  line-height: 32px;
  font-size: 14px;
  font-weight: normal;
  color: #666;
}
.el-pagination button:hover {
  color: #1890FF;
}
.el-pagination button:disabled {
  color: #C0C4CC;
}
.el-pager li.active,
.el-pager li:hover {
  border-color: #006FB4;
  color: #1890FF;
}
.el-pager li.active+li {
  border-left: 1px solid #D9D9D9;
}
.el-pager li.active+li:hover,
.el-pager li.active+li.active {
  border-left: 1px solid #006FB4;
}
.el-pagination button, .el-pagination span:not([class*=suffix]) {
  height: 32px;
  line-height: 32px;
  font-size: 14px;
}
.el-pagination__editor.el-input {
  width: 48px;
  height: 32px;
  line-height: 32px;
  padding: 0;
  margin: 0 8px;
}
.el-pagination__editor.el-input .el-input__inner {
  height: 32px;
  line-height: 32px;
  font-size: 14px;
  padding: 0;
  border-radius: 2px;
  border: 1px solid #D9D9D9;
}
.el-pagination .el-select .el-input {
  width: 120px;
  margin: 0;
}
.el-pagination .el-select .el-input .el-input__inner {
  height: 32px;
  line-height: 32px;
  font-size: 14px;
  border-radius: 2px;
  border: 1px solid #D9D9D9;
}
.el-pagination .btn-next,
.el-pagination .btn-prev{
  padding: 0;
  width: 32px;
  min-width: 32px;
  background: #FFFFFF;
  border-radius: 2px;
  border: 1px solid #D9D9D9;
}
.el-pager {
  margin-left: 8px;
}
.el-pagination__jump {
  margin-left: 16px;
}
.el-pagination__sizes {
  margin-left: 16px;
  margin-right: 0;
}
.el-pagination__total {
  margin-right: 16px;
}
.page-str {
  font-weight: normal;
  color: #666;
}
.el-pagination__total {
  margin-left: 5px;
  color: #666;
}
</style>
