<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" type="primary" @click="batchSave"
      >批量保存</el-button
    >
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expendKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            @click="() => append(data)"
            v-if="node.level <= 2"
            >添加</el-button
          >
          <el-button type="text" size="mini" @click="() => edit(data)"
            >修改</el-button
          >
          <el-button
            type="text"
            size="mini"
            @click="() => remove(node, data)"
            v-if="node.childNodes.length == 0"
            >删除</el-button
          >
        </span>
      </span>
    </el-tree>
    <el-dialog :title="dialogText" :visible.sync="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      dialogType: "",
      category: {
        name: "",
        parentCid: "",
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogText: "",
      expendKeys: [],
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      dialogVisible: false,
    };
  },
  created() {
    this.getMenus();
  },
  methods: {
    handleNodeClick(data) {
      console.log(data);
    },
    getMenus() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.list;
        //console.log("成功获取到菜单数据", data);
      });
    },
    submitData() {
      if (this.dialogType == "edit") {
        this.editCategory();
      }
      if (this.dialogType == "add") {
        this.addCategory();
      }
    },

    append(data) {
      this.dialogType = "add";
      this.dialogText = "添加分类";
      this.dialogVisible = true;
      this.category.name = "";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
    },

    edit(data) {
      this.dialogType = "edit";
      this.dialogText = "修改分类";
      this.dialogVisible = true;
      //发送请求获取当前借点最新的数据
      this.$http({
        url: this.$http.adornUrl("/product/category/info/" + data.catId),
        method: "get",
      }).then((data) => {
        this.category = data.data.category;
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "菜单删除成功",
                type: "success",
                duration: 1500,
                onClose: () => {
                  this.getMenus();
                  //设置需要默认展开的菜单
                  this.expendKeys = [node.parent.data.catId];
                },
              });
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch({});
    },
    //添加三级分类的方法
    addCategory() {
      //console.log("提交三级分类表单的数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message.success("菜单添加成功");
        this.dialogVisible = false;
        this.category.name = "";
        this.getMenus();
        this.expendKeys = [this.category.parentCid];
      });
    },
    //修改分类的方法
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = { catId, name, icon, productUnit };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: data,
      }).then((res) => {
        if (res.data.msg == "success") {
          this.dialogVisible = false;
          this.$message.success("修改成功");
          this.getMenus();
          this.expendKeys = [this.category.parentCid];
        }
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      //1.判断被拖动的当前节点以及所在的父节点总层数不能大于3
      //1.1判断被拖动的当前节点的总层数
      //console.log("draggingNode", draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+父节点的深度不超过3
      //第三层级深度 3-3+1
      //第二层级深度 3-2+1
      //第一层级深度 3-1+1
      let deep = Math.abs(this.maxLevel - draggingNode.level + 1);
      // console.log("最大深度", this.maxLevel);
      // console.log(this.maxLevel + "-" + draggingNode.data.catLevel + "+1");
      // console.log("深度", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3 ? true : false;
      } else {
        return deep + dropNode.parent.level <= 3 ? true : false;
      }
    },
    countNodeLevel(node) {
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    //拖拽成功事件
    handleDrop(draggingNode, dropNode, dropType) {
      //1.当前节点最新的父节点id
      let pCid = 0;
      let sibiling = null;

      //2.当前拖拽节点的最新顺序
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        sibiling = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        sibiling = dropNode.childNodes;
      }
      this.pCid.push(pCid);
      //遍历所有兄弟 进行排序
      for (let i = 0; i < sibiling.length; i++) {
        //如果遍历的是当前正在拖拽的节点
        if (sibiling[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (sibiling[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = sibiling[i].level;
            //修改子节点的层级
            this.updateChildNodeLevel(sibiling[i]);
          }
          this.updateNodes.push({
            catId: sibiling[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: sibiling[i].data.catId, sort: i });
        }
      }
      //3.当前拖拽节点的最新层级
    },
    updateChildNodeLevel(node) {
      for (let i = 0; i < node.childNodes.length; i++) {
        if (node.childNodes.length > 0) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    //批量保存
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message.success("菜单顺序修改成功");
        this.getMenus();
        this.updateNodes = [];
        this.maxLevel = 0;
        this.expendKeys = [this.pCid];
        //this.pCid=0
      });
    },
    batchDelete() {
      let catIds = [];
      let checkNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的元素", checkNodes);
      for (let i = 0; i < checkNodes.length; i++) {
        catIds.push(checkNodes[i].catId);
      }
      this.$confirm(`是否批量删除【${catIds}】菜单？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(()=>{
        this.$http({
          url:this.$http.adornUrl('/product/category/delete'),
          method:'post',
          data:this.$http.adornData(catIds,false)
        }).then((data)=>{
          this.$message.success("批量删除成功")
          this.getMenus()
        })
      }).catch();
      
    },
  },
};
</script>

<style scoped>
</style>