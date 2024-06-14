<template>
  <div class="table-box" :key="tableconfig.key">
    <div class="top-area">
      <slot name="header"></slot>
      <slot name="btn"></slot>
    </div>
    <!-- 页面表格列表 -->
    <el-table
        class="header-color-table"
        v-loading="loading"
        row-key="id"
        :max-height="tableconfig.maxHeight || 2000"
        ref="dataTable"
        :data="tableData"
        :border="tableconfig.hasBorder || false"
        :summary-method="getSummaries"
        :show-summary="tableconfig.summary"
        @selection-change="handleSelectionChange"
        :header-cell-class-name="handleHeaderClass"
        :cell-class-name="handleCellClass"
        :row-class-name="tableRowClassName"
        @sort-change="handleTableSort"
        @select = "pinSelect"
    >
      <slot name="customheadcolumn"></slot>
      <el-table-column
          v-if="tableconfig.multiChecked"
          type="selection"
          :reserve-selection="true"
          width="80"
          fixed="left">
      </el-table-column>
      <el-table-column
          v-if="tableconfig.showIndex"
          type="index"
          label="序号"
          width="80"
          fixed="left">
      </el-table-column>
      <el-table-column
          v-for="(item,index) in tableconfig.fields"
          :key="item.prop+index"
          :prop="item.prop"
          :label="item.label"
          :show-overflow-tooltip="!tableconfig.showOverflowTooltip"
          :width="item.width||''"
          :min-width="item.minWidth || 100"
          :sortable="item.sort"
          :align="item.align || 'left'"
          :class-name="item.customClass || ''"
      >
        <template #header v-if="item.header">
          {{ item.label }}
          <el-popover trigger="hover" :content="item.header.content">
            <i class="iconfont icon-help" style="color: #c0c4cc" slot="reference"></i>
            <slot :name="item.header.slot"></slot>
          </el-popover>
        </template>
        <!-- 数据是否进一步处理 -->
        <template #default="scope">
          <p v-if="item.icon" :class="[item.expIconClass(scope.row[item.prop])]" v-html="item.icon"></p>
<!--          <i v-if="item.statusIcon" :class="[item.statusIcon, getIconClass(item, scope.row)]"></i>-->
          <el-button v-if="item.clickFn" type="text" @click="item.clickFn(scope.row)">{{
              scope.row[item.prop] === null || scope.row[item.prop] === '' || scope.row[item.prop] === undefined
                  ? '--'
                  : item.exp
                      ? item.exp(scope.row[item.prop], scope.row[item.extraProp])
                      : scope.row[item.prop]
            }}</el-button>
          <span v-else>{{
          scope.row[item.prop] === null || scope.row[item.prop] === '' || scope.row[item.prop] === undefined
              ? '--'
              : item.exp
                  ? item.exp(scope.row[item.prop], scope.row[item.extraProp])
                  : scope.row[item.prop]
            }}</span>
        </template>
      </el-table-column>
      <slot name="operation"></slot>
    </el-table>
    <!-- 分页器 -->
    <pagination
        v-if="total > 0"
        class="page-pagination"
        :total="total"
        :sizetype="sizetypes"
        :currentpage="queryData.pageNum"
        :pagesize="queryData.pageSize"
        :selectnum = "selectnum"
        @size-change="sizeChange"
        @page-change="pageChange">
    </pagination>
  </div>
</template>

<script>
/* 通用表格组件
*  tableconfig
*  key: String，表格唯一key
*  sortBySingleProp: Boolean，有多个排序列时，每次排序是否只使用当前点击的列；不传这个值时，多个排序列组合排序
*  summary: Boolean，是否显示合计行，默认不显示
*  summaryColumns：Array，需要合计的列；summary为true时，这个字段必填
*  multiChecked：Boolean，是否显示多选框，默认不显示
*  showIndex：Boolean，是否显示序号，默认不显示
*  fields：Array，列数据
*  queryDataType：Object，查询数据是否在初始化时就参与查询
*/
module.exports = {
  name: "TableBox",
  props: {
    tableconfig: {},
    querydata: {},
    loading: false,
  },
  components: {
    'pagination': httpVueLoader("components/pagination.vue")
  },
  mounted(){
    window.addEventListener('keydown', code => { // 这个是获取键盘按住事件
      // console.log(code); // 这个是你按住键盘打印出键盘信息，在浏览器中自行查看
      if (code.keyCode === 16 && code.shiftKey) { // 判断是否按住shift键，是就把pin赋值为true
        this.pin = true;
      }
    });
    window.addEventListener('keyup', code => { // 这个是获取键盘松开事件
      if(code.keyCode === 16){ // 判断是否松开shift键，是就把pin赋值为false
        this.pin = false;
      }
    });
  },
  data () {
    return {
      total: 0,
      sizetypes:'small',
      queryData: {
        pageNum: 1,
        pageSize: this.tableconfig.pageSize
      },
      tableData: [],
      loading: false,
      multipleSelection: [],
      sortFields: [],
      checkedRows:[],
      origin: -1, // 这里给一个变量作为起点
      pin: false, // 这里给一个变量，默认为false，不按住
      selectRows:[],
      flagindex:true,
      selectnum:0,
    };
  },
  watch: {
    // querydata: {
    //   handler() {
    //     this.getData("form");
    //   },
    //   deep: true,
    //   immediate: true
    // }
  },
  created() {
    this.sizetypes=this.tableconfig.sizetype
    if (this.tableconfig.queryDataType === "init") {
      this.getData("form");
    } else {
      this.getData("init");
    }
    this.initSortTable();
  },
  methods: {
    setLoading(data){
        this.loading = data;
    },
    getData(type) {
      if (!this.tableconfig.queryUrl) {
        return;
      }
      if (this.loading) {
        return;
      }
      let obj = {};
      if (type !== "init") {
        for(let key in this.querydata) {
          if (this.querydata[key]) {
            obj[key] = this.querydata[key];
          }
        }
      }
      if (type === "init" || type === "form") {
        this.queryData.pageNum = 1;
      }
      obj["pageSize"] = this.queryData.pageSize;
      obj["pageNum"] = this.queryData.pageNum;
      this.loading = true;
      $.ajax({
        headers: {
          'Content-Type': 'application/json',
          'Authorization': sessionStorage.getItem("_platform6sid")
        },
        xhrFields: {
          withCredentials: true
        },
        crossDomain: true,
        type: 'post',
        dataType: 'json',
        data: JSON.stringify(obj),
        url: sessionStorage.getItem("requestBasicUrl") + this.tableconfig.queryUrl,
        success: (result) => {
          this.loading = false;
          if(this.tableconfig.dataFields){
            this.total = 0;
            this.tableData = result[this.tableconfig.dataFields] || []
            this.tableData.forEach((item, index) => {// 遍历索引,赋值给data数据
              item.index = index;
            })
            return
          }
          let res = result.data || {};
          this.total = res.total || 0;
          this.tableData = res.list && res.list.length ? res.list : [];
          this.tableData.forEach((item, index) => {// 遍历索引,赋值给data数据
            item.index = index;
          })
        },
        error: (err) => {
          this.loading = false;
          this.total = 0;
          this.tableData = [];
          console.log(err);
        }
      });
    },
    getIconClass(item, row) {
      let key = item.prop;
      let val = row[key] ? (row[key].value || row[key]) : "0";
      let cls = "";
      switch (val) {
        case "0":
        case "1":
          cls = "status-doing";
          break;
        case "2":
          cls = "status-failed";
          break;
        case "3":
          cls = "status-success";
          break;
        default:
          cls = "status-doing";
      }
      return cls;
    },
    getSummaries(param) {
      const { columns, data } = param;
      const sums = [];
      columns.forEach((column, index) => {
        if (index === 0) {
          sums[index] = "合计";
        } else if (this.tableconfig.summaryColumns.indexOf(index) < 0) {
          sums[index] = "";
        } else {
          let values = data
              .filter((item) => !item.parent)
              .map((item) => Number(item[column.property]));
          if (!values.every((value) => isNaN(value))) {
            sums[index] = values.reduce((prev, curr) => {
              const value = Number(curr);
              if (!isNaN(value)) {
                return prev + curr;
              } else {
                return prev;
              }
            }, 0);
          } else {
            sums[index] = "";
          }
        }
      });
      return sums;
    },
    sizeChange(currentSize) {
      this.queryData.pageSize = currentSize;
      this.queryData.pageNum = 1;
      this.getData("page");

    },
    pageChange(currentPage) {
      this.queryData.pageNum = currentPage;
      this.getData("page");

    },
    handleSelectionChange(val) {
      if(this.flagindex){
        this.multipleSelection = val;
        this.selectnum = this.multipleSelection.length || 0
      }
    },
    // 这里是select事件开始
    pinSelect (selection, row) {
      // this.selectRows=[]
      this.flagindex=true
      const data = this.$refs.dataTable.tableData; // 获取所以数据
      const origin = this.origin; // 起点数 从-1开始
      const endIdx = row.index; // 终点数
      if (this.pin && selection.includes(data[origin])) { // 判断按住
        const sum = Math.abs(origin - endIdx);// 这里记录终点
        const min = Math.min(origin, endIdx);// 这里记录起点

        let i = 0;
        while (i < sum) {
          const index = min + i;
          this.$refs.dataTable.toggleRowSelection(data[index], true); // 通过ref打点调用toggleRowSelection方法，第二个必须为true
          i++;
          if(i==sum){
            this.flagindex=false
          }
        }

      } else {
        this.origin = row.index; // 没按住记录起点
      }
      console.log('selection',selection)
      // console.log('this.selectRows',this.selectRows)
    },

    clearSelection() {
      this.$refs["dataTable"].clearSelection();
    },

    handleCellClass({row, column, rowIndex, columnIndex}) {
      let cls = "";
      if (column.property === "borrowType" && row.borrowType === "1") {
        cls = "borrow-type-1";
      }
      return cls;
    },

    initSortTable() {
      this.sortFields = [];
      this.tableconfig.fields.forEach(f => {
        if (f.sort === "custom") {
          this.sortFields.push({multiOrder: null, property: f.prop});
        }
      });
    },
    handleHeaderClass({ column }) {
      if (column.sortable == "custom") {
        if (column.multiOrder) {
          column.order = column.multiOrder;
        }
      }
    },
    handleOrderChange(column, prop, order) {
      this.sortFields.forEach(item => {
        if (prop === item.property) {
          item.multiOrder = order;
        }
      });
      let obj = {};
      obj["currentProp"] = prop;
      this.sortFields.forEach(item => {
        obj[item.property] = item.multiOrder;
      });
      this.$emit("sort-change", obj);
    },
    handleTableSort({ column, prop, order }) {
      if (column.sortable !== "custom") {
        return;
      }
      if (this.tableconfig.sortBySingleProp) {
        //具体排序操作
        this.handleOrderChange(column, prop, order);
      } else {
        this.setSortColumnClass(column);
        //具体排序操作
        this.handleOrderChange(column, prop, order);
      }
    },
    setSortColumnClass(column) {
      if (!column.multiOrder) {
        column.multiOrder = "ascending";
      } else if (column.multiOrder == "descending") {
        column.multiOrder = null;
      } else if(column.multiOrder == "ascending") {
        column.multiOrder = "descending";
      }
    },
    tableRowClassName({row, rowIndex}){
      return this.tableconfig.tableRowClassName?.({row, rowIndex}) || ''
    }
  }
};
</script>
<style scoped>
.table-box {
  padding: 20px;
}
.top-area {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
  font-size: 15px;
}
.top-area .table-title {
  line-height: 34px;
  display: flex;
  align-items: center;
}
.top-area .table-title em {
  display: inline-block;
  width: 4px;
  height: 16px;
  margin-right: 6px;
  background: #006fb4;
}
.top-area .btns {
  text-align: right;
}
.download {
  text-decoration: none;
  color: #409eff;
}
.iconClass {
  width: 6px;
  height: 6px;
  display: inline-block;
  border-radius: 6px;
  vertical-align: middle;
}
.iconClass.status-doing {
  background: #409EFF !important;
}
.iconClass.status-success {
   background: #67c23a !important;
}
.iconClass.status-failed {
  background: #f56c6c !important;
}
.borrow-type-1 {
  color: #F56C6C;
}
.table-box .page-pagination {
  margin-bottom: 0;
}
.cell>p {
  display: inline-block;
  margin: 0;
  vertical-align: middle;
}
p.blue .status-icon {
  display: inline-block;
  width: 6px;
  height: 6px;
  border-radius: 3px;
  background: #006FB4;
}
p.green .status-icon {
  display: inline-block;
  width: 6px;
  height: 6px;
  border-radius: 3px;
  background: #67C23A;
}
</style>
