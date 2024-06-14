<template>
  <template v-if="selfclick">
    <span class="download-btn" @click.stop="downloadFile">
      <slot name="icon"></slot>
      <slot name="label"></slot>
    </span>
  </template>
  <template v-else>
    <span class="download-btn">
      <slot name="icon"></slot>
      <slot name="label"></slot>
    </span>
  </template>
</template>

<script>
module.exports = {
  name: "download-file",
  props: {
    fileurl: "", // 文件下载路径
    fileid: "", // 文件下载id或者文件下载查询参数，如果不需要这个参数，请传入"noNeed"
    name: "", // 文件下载名称
    selfclick: false, // 是否使用组件自己的click事件
    datamethod: "get" // 请求方式
  },
  data() {
    return {
      fileUrl: "",
      fileId: "",
      fileMethod: ""
    }
  },
  watch: {
    fileid(nVal, oVal) {
      this.fileId = nVal;
      !this.selfclick && this.downloadFile();
    }
  },
  mounted() {
    this.fileId = this.fileid || "";
    this.fileUrl = this.fileurl ? this.fileurl : "/archivec/CmAttachmentController/download/";
    this.fileMethod = this.datamethod ? this.datamethod : "get";
  },
  methods: {
    downloadFile() {
      let self = this;
      if (!this.fileId) {
        return;
      }
      let fileId = this.fileId === "noNeed" ? "" : this.fileId;
      this.fileUrl = this.fileurl;
      let loading = this.$loading({ lock: true });
      let xhr = new XMLHttpRequest();
      if (this.fileMethod === "post") {
        xhr.open("post", sessionStorage.getItem("requestBasicUrl") + this.fileUrl, true);
        xhr.setRequestHeader('Content-Type', "application/json");
      } else {
        xhr.open("get", sessionStorage.getItem("requestBasicUrl") + this.fileUrl + fileId, true);
      }
      xhr.responseType = "blob";
      xhr.withCredentials = true;
      xhr.setRequestHeader('Authorization', sessionStorage.getItem("_platform6sid"));
      xhr.onload = function () {
        if (this.status == 200) {
          loading.close();
          let fileNameHeader = xhr.getResponseHeader("Content-Disposition");
          let fileName = fileNameHeader ? fileNameHeader.split("''")[1] : "";
          let blob = this.response;
          let link = document.createElement('a');
          document.body.appendChild(link);
          link.style.display = "none";
          let url = window.URL.createObjectURL(blob);
          link.href = url;
          if (self.name || fileName) {
            link.download = self.name || decodeURIComponent(fileName);
          }
          link.click();
          document.body.removeChild(link);
          self.$emit("download-end", true);
        } else {
          loading.close();
          self.$emit("download-end", false);
        }
      }
      if (this.fileMethod === "post") {
        xhr.send(JSON.stringify({ids: this.fileId.split(",")}));
      } else {
        xhr.send();
      }
    }
  }
}
</script>

<style scoped>
.download-btn {
  cursor: pointer;
}
</style>
