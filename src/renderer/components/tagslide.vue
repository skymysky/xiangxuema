<style scoped>
.bg {
  position: absolute;
  top: 8px;
  bottom: 35px;
  right: 8px;
  left: 36px;
  background: #00000033;
  border-radius: 4px;
  overflow: hidden;
  z-index: 999;
}
.container {
  bottom: 45px;
  width: 380px;
  background: #fff;
  float: right;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.tagContainer {
  padding: 8px;
}
.tagIndex {
  padding-right: 6px;
  background: #fff;
}
.tagIndex:hover {
  background: #e7f3ff;
}
.tagClose {
  width: 18px;
  text-align: center;
}

.all {
  padding-top: 8px;
  flex: 1;
  border-bottom: 1px dashed rgb(204, 204, 204);
  overflow-y: auto;
}
.all .tagIndex {
  padding-right: 19px;
}
.selected {
  padding-top: 8px;
  border-bottom: 1px dashed rgb(204, 204, 204);
}
.selected .tagIndex {
  background: #e7f3ff;
}
.selected .tagIndex:hover {
  background: #fff;
}
.add {
  padding: 12px;
  display: flex;
}
.add div {
  display: inline-block;
}
.input {
  flex: 1;
}
.input input {
  line-height: 28px;
  height: 28px;
  padding: 0px;
  margin: 0px;
  outline: none;
  border: 1px solid #1787fb;
  width: calc(100% - 12px);
  padding-left: 6px;
  padding-right: 6px;
  border-radius: 3px;
}
.btn {
  width: 16px;
  background: #e7f3ff;
  color: #1787fb;
  border: 1px solid #1787fb;
  margin-right: 0px;
  text-align: center;
  font-size: 16px;
}
.btn:hover {
  background: #1787fb;
  color: #fff;
}
.noData{
  line-height:38px;text-align:center;font-size:16px;color:#bbb;
}
</style>
<template>
  <div v-show="show" class="bg" @click="show=false">
    <div @click.stop class="container">
      <div class="selected tagContainer" v-if="tagSelected.length > 0">
        <div
          class="tag tagIndex"
          :key="item.id"
          @click="tagCancelSelect(item,index)"
          v-for="(item,index) in tagSelected"
        >
          <div class="tagText">{{item.title}}</div>
        </div>
      </div>
      <div class="selected tagContainer noData" v-else>
          该{{refer == 'mind'?'脑图':'文章'}}未选择任何标签
      </div>
      <div class="all tagContainer" v-if="tagEles.length > 0">
        <div
          :key="item.id"
          @click="tagSelect(item,index)"
          class="tag tagIndex"
          v-for="(item,index) in tagEles"
        >
          <div class="tagText">{{item.title}}</div>
          <div @click.stop="tagRemove(item,index)" class="tagClose">
            <i class="iconfont icon-guanbi" style="font-size: 16px !important;"></i>
          </div>
        </div>
      </div>
      <div class="all tagContainer noData" v-else>
          标签库中没有可供选择的标签
      </div>
      <div class="add">
        <div class="input">
          <input v-model="tagInputText" @keyup.13="addTag" type="text" placeholder="增加标签，Enter键提交">
        </div>
        <div class="btn" @click="addTag">+</div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  props: ["refer", "id"],
  data() {
    return {
      show: false,
      tagInputText: "",
      tagEles: [],
      tagSelected: []
    };
  },
  mounted() {
    this.getData();
  },
  methods: {
    async tagRemove(item, index) {
      let val = await swal({
        text:
          "你确定要删除这个标签吗？所有与之关联的文章、火花、脑图等，都将取消关联！",
        icon: "warning",
        buttons: ["取消", "删除"],
        dangerMode: true
      });
      if (!val) return;
      await this.db("article_tag")
        .where("tag_id", item.id)
        .del();
      await this.db("flower_tag")
        .where("tag_id", item.id)
        .del();
      await this.db("mind_tag")
        .where("tag_id", item.id)
        .del();
      await this.db("tags")
        .where("id", item.id)
        .del();
      this.tagEles.splice(index, 1);
    },
    async tagCancelSelect(item, index) {
      let obj = {
        tag_id: item.id
      };
      obj[this.refer + "_id"] = this.id;
      await this.db(this.refer + "_tag")
        .where(obj)
        .del();
      this.tagSelected.splice(index, 1);
      this.tagEles.unshift(item);
    },
    async tagSelect(item, index) {
      let obj = {
        tag_id: item.id
      };
      obj[this.refer + "_id"] = this.id;
      await this.db(this.refer + "_tag").insert(obj);
      this.tagEles.splice(index, 1);
      this.tagSelected.unshift(item);
    },
    async getData() {
      //todo 根据refer num 排序
      let allTags = await this.db("tags").orderBy("created_at", "desc");
      let mts = await this.db(this.refer + "_tag").where(this.refer + "_id", this.id);
      allTags.forEach(v => {
        let flag = mts.some(i => i.tag_id == v.id);
        if (flag) this.tagSelected.push(v);
        else this.tagEles.push(v);
      });
    },
    alert(str) {
      swal({
        icon: "error",
        text: str
      });
    },
    async addTag() {
      var text = this.tagInputText.trim();
      if (text.length < 1) {
        this.alert("输入的标签为空");
        return;
      }
      let hasIt = this.$root.tags.some(item => item.title == text);
      if (hasIt) {
        this.alert("已经存在该标签");
        return;
      }
      let tag = {
        title: text
      };
      let row = await this.db("tags").insert(tag);
      tag.id = row[0];
      this.tagEles.unshift(tag);
      this.tagSelect(tag,0);
      this.tagInputText = "";
      this.bus.$emit("tagCount");
    }
  }
};
</script>



