<template>
  <div>
    <div class="query-box">
      <el-form
          class="query-form"
          :class="{'form-expanded': queryFormExpanded}"
          size="medium"
          :model="queryForm"
          ref="queryForm"
          @submit.native.prevent>
        <el-form-item v-for="item in fields" :key="item.prop" :label="item.label" :prop="item.prop" :label-width="item.labelWidth || '120px'">
          <!-- 日期范围选择 -->
          <el-date-picker
              v-if="item.type === 'daterange'"
              v-model="rangeDate[item.prop]"
              type="daterange"
              @change="handleRangeDateChange(item.prop, $event)"
              clearable
              :unlink-panels="true"
              value-format="yyyy-MM-dd"
              range-separator="至"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
          ></el-date-picker>
          <!-- 日期选择器 -->
          <el-date-picker
              v-else-if="item.type === 'date'"
              v-model="queryForm[item.prop]"
              clearable
              type="date"
              value-format="yyyy-MM-dd"
              placeholder="选择日期"
          >
          </el-date-picker>
          <!-- 下拉选择 -->
          <!-- value为0值时务必用字符串表示 -->
          <el-select
              v-else-if="item.type === 'select'"
              v-model="queryForm[item.prop]"
              @change="$set(queryForm, item.prop, $event)"
              clearable
              :filterable="item.filterable"
              :placeholder="`请选择${item.label}`"
          >
            <el-option  v-for="v in options[item.prop]" :key="v.label" :label="v.label" :value="v.value"></el-option>
          </el-select>
          <!-- 级联选择器 -->
          <el-cascader
              v-else-if="item.type === 'cascader'"
              v-model="queryForm[item.prop]"
              :options="item.options"
              :props="item.propOptions|| {}"
              :show-all-levels="item.showAllLevel"
              collapse-tags
              clearable>
          </el-cascader>
          <!-- 部门选择-单选 -->
          <el-input
              v-else-if="item.type === 'dept'"
              v-model="deptData[item.prop]"
              readonly
              @click.native="showDeptDialog(item.prop)"
              :placeholder="`请选择${item.label}`"
              class="dept-input"
          >
            <el-button class="dept-input-append" slot="append" icon="el-icon-circle-close" @click.stop="clearDept(item.prop)"></el-button>
          </el-input>
          <!-- 用户选择-多选 -->
          <el-input
              v-else-if="item.type === 'user'"
              v-model="userData[item.prop]"
              readonly
              @click.native="showUserDialog(item.prop)"
              :placeholder="`请选择${item.label}`"
              class="user-input"
          >
            <el-button class="user-input-append" slot="append" icon="el-icon-circle-close" @click.stop="clearUser(item.prop)"></el-button>
          </el-input>
          <!-- 普通输入框 -->
          <el-input
              v-else="item.type === 'text'"
              v-model="queryForm[item.prop]"
              clearable
              :maxlength="item.maxlength || 100"
              :placeholder="`请输入${item.label}`"
          ></el-input>
        </el-form-item>
      </el-form>
      <div class="expand-buttons" v-if="fields.length > 3">
        <i v-if="!queryFormExpanded" class="el-icon-arrow-right" @click="expandQueryForm"></i>
        <i v-if="queryFormExpanded" class="el-icon-arrow-down" @click="colQueryForm"></i>
      </div>
      <div class="buttons">
        <el-button type="primary" size="small" @click="doQuery">查询</el-button>
        <el-button size="small" @click="reset()">重置</el-button>
      </div>
    </div>

    <el-dialog
        v-if="deptDialogVisible"
        title="选择申请部门"
        :visible.sync="deptDialogVisible"
        :close-on-click-modal="false"
        append-to-body
        width="800px"
        @close="cancelDept">
      <choose-dept
          :multiple="false"
          :http="getDept"
          :httpsearch="searchDept"
          :checks="selectedDept"
          :fnok="confirmDept"></choose-dept>
    </el-dialog>

    <el-dialog
        v-if="userDialogVisible"
        title="选择人员"
        :visible.sync="userDialogVisible"
        :close-on-click-modal="false"
        append-to-body
        width="800px"
        @close="cancelUser">
      <choose-user
          :multiple="true"
          :http="getUser"
          :httpsearch="searchUser"
          :checks="selectedUser"
          :fnok="confirmUser"></choose-user>
    </el-dialog>
  </div>
</template>

<script>
module.exports =  {
  name: 'QueryBox',
  props: {
    fields: Array,
    noticefn: ()=>{}
  },
  components: {
    'choose-dept': httpVueLoader('components/choose-dept.vue'),
    'choose-user': httpVueLoader('components/choose-user.vue')
  },
  data() {
    return {
      queryForm: {},
      rangeDate: {},
      options: {},
      cascaderProps: {
        emitPath: false
      },
      queryFormExpanded: false,
      deptDialogVisible: false,
      deptData: {},
      selectedDept: [],
      editedDeptProp: "",
      userDialogVisible: false,
      userData: {},
      selectedUser: [],
      editedUserProp: "",
    }
  },
  watch:{
    queryForm: {
      deep: true,
      handler(val){
        this.noticefn?.(val)
      }
    }
  },
  created() {
    // 初始化下拉列表的数据
    this.fields.forEach(item => {
      if (item.type === 'select') {
        this.getOptions(item.options, item.prop)
      }
    });
  },
  beforeDestroy() {
    this.reset();
  },
  methods: {
    // initQueryForm() {
    //   this.queryForm
    // },
    // 回车搜索
    doQuery() {
      this.$emit('query-data-change', this.queryForm);
    },
    // 日期区间获取转换
    handleRangeDateChange (prop, val) {
      const arr = prop ? prop.split(',') : [];
      if (!arr) return
      this.queryForm[arr[0]] = val && val[0] ? val[0] : "";
      this.queryForm[arr[1]] = val && val[1] ? val[1] : "";
    },
    // 自定义重置表单
    reset() {
      for (let key in this.queryForm) {
        this.queryForm[key] = undefined;
      }
      // this.$refs["queryForm"].resetFields();
      Object.keys(this.rangeDate).forEach(cur => this.handleRangeDateChange(cur));
      this.rangeDate = {};
      this.deptData = {};
      this.userData = {};
      this.doQuery();
    },
    clearFormFields() {
      for (let key in this.queryForm) {
        this.queryForm[key] = undefined;
      }
      Object.keys(this.rangeDate).forEach(cur => this.handleRangeDateChange(cur));
      this.rangeDate = {};
      this.deptData = {};
      this.userData = {};
    },
    // 获取字典选项
    getOptions (arg,prop) {
      if (Array.isArray(arg)){
        this.options[prop] = arg;
      } else {
        arg().then(data => {
          this.$set(this.options, prop, data);
        });
      }
    },
    expandQueryForm() {
      this.queryFormExpanded = true;
    },
    colQueryForm() {
      this.queryFormExpanded = false;
    },

    getDept(data) {
      return new Promise((resolve, reject) => {
        $.ajax({
          headers: {
            'Authorization': sessionStorage.getItem("_platform6sid"),
            'Content-Type': 'application/json'
          },
          xhrFields: {
            withCredentials: true
          },
          crossDomain: true,
          type: 'post',
          dataType: 'json',
          data: JSON.stringify(data),
          url: sessionStorage.getItem("requestBasicUrl") + "/archive/CmCommonController/selectDept",
          success: (res) => {
            if (res.code === 0) {
              resolve(res.data);
            } else {
              resolve([]);
            }
          },
          error: (err) => {
            reject(err);
          }
        });
      });
    },
    searchDept(data) {
      return new Promise((resolve, reject) => {
        $.ajax({
          headers: {
            'Authorization': sessionStorage.getItem("_platform6sid"),
            'Content-Type': 'application/json'
          },
          xhrFields: {
            withCredentials: true
          },
          crossDomain: true,
          type: 'post',
          dataType: 'json',
          data: JSON.stringify(data),
          url: sessionStorage.getItem("requestBasicUrl") + "/archive/CmCommonController/searchDept",
          success: (res) => {
            if (res.code === 0) {
              resolve(res.data);
            } else {
              resolve([]);
            }
          },
          error: (err) => {
            reject(err);
          }
        });
      });
    },
    showDeptDialog(prop) {
      this.editedDeptProp = prop;
      this.deptDialogVisible = true;
    },
    confirmDept(val) {
      if (!Object.keys(val).length) {
        return;
      }
      let arr = this.editedDeptProp ? this.editedDeptProp.split(',') : [];
      if (!arr) return;

      let parentName = val.parentDeptName ? val.parentDeptName + "-" : (val.parent ? val.parent.label + "-" : "");
      this.deptData[this.editedDeptProp] = parentName + (val.label || val.name);
      this.queryForm[arr[0]] = val.id;
      this.cancelDept();
    },
    cancelDept() {
      this.deptDialogVisible = false;
    },
    clearDept(prop) {
      let arr = prop ? prop.split(',') : [];
      if (!arr) return;
      this.$delete(this.deptData, prop);
      this.$set(this.deptData, prop, "");
      this.queryForm[arr[0]] = "";
    },

    getUser(data) {
      return new Promise((resolve, reject) => {
        $.ajax({
          headers: {
            'Authorization': sessionStorage.getItem("_platform6sid"),
            'Content-Type': 'application/json'
          },
          xhrFields: {
            withCredentials: true
          },
          crossDomain: true,
          type: 'post',
          dataType: 'json',
          data: JSON.stringify(data),
          url: sessionStorage.getItem("requestBasicUrl") + "/archive/CmCommonController/selectUser",
          success: (res) => {
            if (res.code === 0) {
              resolve(res.data);
            } else {
              resolve([]);
            }
          },
          error: (err) => {
            reject(err);
          }
        });
      });
    },
    searchUser(data) {
      return new Promise((resolve, reject) => {
        $.ajax({
          headers: {
            'Authorization': sessionStorage.getItem("_platform6sid"),
            'Content-Type': 'application/json'
          },
          xhrFields: {
            withCredentials: true
          },
          crossDomain: true,
          type: 'post',
          dataType: 'json',
          data: JSON.stringify(data),
          url: sessionStorage.getItem("requestBasicUrl") + "/archive/CmCommonController/searchUser",
          success: (res) => {
            if (res.code === 0) {
              resolve(res.data);
            } else {
              resolve([]);
            }
          },
          error: (err) => {
            reject(err);
          }
        });
      });
    },
    showUserDialog(prop) {
      this.editedUserProp = prop;
      this.userDialogVisible = true;
    },
    confirmUser(val) {
      if (!val.length) {
        return;
      }
      let arr = this.editedUserProp ? this.editedUserProp.split(',') : [];
      if (!arr) return;

      let names = [];
      let ids = [];
      val.forEach(user => {
        names.push(user.label || user.loginName);
        ids.push(user.id);
      });
      this.userData[this.editedUserProp] = names.join(", ");
      this.queryForm[arr[0]] = ids;
      this.cancelUser();
    },
    cancelUser() {
      this.userDialogVisible = false;
    },
    clearUser(prop) {
      let arr = prop ? prop.split(',') : [];
      if (!arr) return;
      this.$delete(this.userData, prop);
      this.$set(this.userData, prop, "");
      this.queryForm[arr[0]] = "";
    },
  }
}
</script>
<style scoped>
.query-box {
  box-shadow: 0px 6px 12px 0px rgba(0, 68, 193, 0.08);
  position: relative;
  padding-top: 20px;
  background-color: #fff;
  display: flex;
  justify-content: space-between;
}

.query-box .query-form {
  width: calc(100% - 210px);
  height: 58px;
  overflow: hidden;
}

.query-box .query-form.form-expanded {
  height: auto;
}

.query-box .query-form .el-form-item {
  width: 33%;
  float: left;
}
.query-box .query-form .el-form-item .el-form-item__label {
  font-weight: normal;
}

.query-form .el-form-item .el-date-editor,
.query-form .el-form-item .el-select,
.query-form .el-form-item .el-cascader {
  width: 100%;
}

.query-box .expand-buttons {
  position: absolute;
  right: 195px;
  top: 28px;
  font-size: 18px;
  cursor: pointer;
}

.query-box .buttons {
  width: 190px;
  text-align: right;
}
.dept-input .el-input__inner {
  border-right-width: 0;
}
.dept-input .el-input__inner:focus,
.dept-input .el-input__inner:hover {
  border-color: #DCDFE6;;
}
.dept-input .el-input-group__append {
  padding: 0 12px;
  background: transparent;
}
.dept-input .dept-input-append {
  color: #C0C4CC;
  display: none;
}
.dept-input:hover .dept-input-append {
  display: block;
}
.dept-input .dept-input-append:hover {
  color: #909399;
}
</style>
