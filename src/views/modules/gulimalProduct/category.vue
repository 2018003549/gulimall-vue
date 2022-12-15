<template>
  <div>
    <el-switch
      v-model="isDraggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="isDraggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      :default-expanded-keys="expandKeys"
      :show-checkbox="true"
      node-key="catId"
      :draggable="isDraggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
      ><span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => edit(data)"
          >
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名">
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
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      title: "", //用于显示对话框的标题
      isDraggable: false, //用于控制拖拽功能
      dialogVisible: false, //默认不显示对话框，在点击append方法时改为true
      maxLevel: 0, //存放当前节点的最大深度
      menus: [],
      expandKeys: [],
      updateNodes: [], //存放排序后的节点
      pidArray: [], //存放批量拖拽之后展开的父节点
      category: {
        dialogType: "", //用于标识数据框提交执行的操作（修改或者添加）
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: null,
        productUnit: null,
      },
      defaultProps: {
        children: "children", //指定节点的子树
        label: "name", //节点的值
      },
    };
  },
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/gulimalProduct/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据", data.data);
        this.menus = data.data;
      });
    },
    append(data) {
      console.log("append", data);
      this.dialogVisible = true;
      this.title = "添加分类";
      this.dialogType = "add";
      this.category.parentCid = data.catId; //添加之后展示的父菜单
      //将修改回显的数据清空
      this.category.catLevel = data.catLevel * 1 + 1; //乘1是为了将字符串转成数字，以免拼接
      this.category.name = "";
      this.category.catId = null; //添加节点的catId置空，才可以自增
      this.category.icon = "";
      this.category.productUnit = "";
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/gulimalProduct/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "拖拽完成",
          type: "success",
        });
        //修改成功刷新菜单并展开
        this.getMenus(); //删除成功之后要刷新页面
        //设置需要默认展开的菜单
        this.expandKeys = this.pidArray; //将修改后的节点父id传入
      });
      //拖拽结束之后初始化数据
      this.updateNodes = [];
      this.maxLevel = 0;
    },
    batchDelete() {
      //this.$refs获取vue中所有组件
      let cidArray = [];
      let cnameArray=[];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes(); //获取所有被选中的节点
      for (let i = 0; i < checkedNodes.length; i++) {
        cidArray.push(checkedNodes[i].catId); //存储所有需要被删除的节点的id
        cnameArray.push(checkedNodes[i].name);
      }
      this.$confirm(`是否批量删除【${cnameArray}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl("/gulimalProduct/category/delete"),
          method: "post",
          data: this.$http.adornData(cidArray, false),
        }).then(() => {
          this.$message({
            message: "批量删除成功",
            type: "success",
          });
          this.getMenus();
        });
      });
    },
    addCategory() {
      console.log("三级分类的数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/gulimalProduct/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "添加成功",
          type: "success",
        });
        this.dialogVisible = false; //关闭对话框
        this.getMenus(); //删除成功之后要刷新页面
        //设置需要默认展开的菜单
        this.expandKeys = [this.category.parentCid]; //将新增的节点的父节点id传入
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除${data.name}菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/gulimalProduct/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功",
              type: "success",
            });
            this.getMenus(); //删除成功之后要刷新页面
            //设置需要默认展开的菜单
            this.expandKeys = [node.parent.data.catId]; //将被删除的节点的父节点id传入
          });
        })
        .catch(() => {});
      console.log("remove", node, data);
    },
    edit(data) {
      console.log("要修改的数据", data);
      this.title = "修改分类";
      this.dialogVisible = true;
      this.dialogType = "edit";
      //发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/gulimalProduct/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //请求成功，将数据回显
        console.log("回显的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId; //修改是根据id查询到数据再修改的
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid; //用于修改后展开父菜单
      });
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category; //解构出需要用到的数据，像父菜单和当前id都是不需要修改的
      //封装需要发送的数据，这里的键名必须和数据库封装的变量名一直才可以被接收
      //修改三级分类数据
      this.$http({
        url: this.$http.adornUrl("/gulimalProduct/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功",
          type: "success",
        });
        this.dialogVisible = false; //关闭对话框
        this.getMenus(); //删除成功之后要刷新页面
        //设置需要默认展开的菜单
        this.expandKeys = [this.category.parentCid]; //将新增的节点的父节点id传入
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      //1.当前节点最新的父节点
      let pCid = 0;
      let siblings = null; //保存当前节点的顺序
      if (dropType == "inner") {
        //插入目标内部，那当前节点的父节点就是目标节点
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        //当前节点在目标节点同层,则当前节点父节点也是目标节点的父节点
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId; //如果拖动最大层级，则是没有父节点的
        siblings = dropNode.parent.childNodes;
      }
      this.pidArray.push(pCid); //保存要展开的父节点
      //2.当前节点的最新顺序,通过遍历父节点的所有子节点，找到自己的位置
      for (let i = 0; i < siblings.length; i++) {
        //将当前节点的catId和当前顺序放进数组
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果当前遍历的节点为拖拽节点，就要判断父子层级是否也变了
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level; //当前节点的层级改为最新的层级
            //继续修改当前节点的所有子节点的层级【递归】
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          //兄弟元素只改顺序
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      //3.当前节点的最新层级
      console.log(this.updateNodes);
    },
    updateChildNodeLevel(node) {
      for (let i = 0; i < node.childNodes.length; i++) {
        var cNode = node.childNodes[i].data; //数据库元素节点信息
        this.updateNodes.push({
          catId: cNode.catId,
          catLevel: node.childNodes[i].level,
        });
        this.updateChildNodeLevel(node.childNodes[i]); //递归修改其子节点的层级信息
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode);
      //获取从当前拖动节点为根的最大深度
      let deep = this.maxLevel - draggingNode.level + 1;
      deep = deep < 0 ? 0 : deep; //没有子节点，最大深度就为0
      this.maxLevel = 0;
      if (type == "inner") {
        console.log(deep);
        //拖到内部，将最大深度和目标节点的深度相加，判断是否大于3，大于3无法拖拽
        return deep + dropNode.level <= 3;
      } else {
        //拖拽节点和目标节点是同一层，就只用考虑拖拽节点的深度是否大于3
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //统计当前节点的总层数,需要递归获取
      if (node.childNodes != null && node.childNodes.length > 0) {
        //当前节点有子节点，遍历所有子节点，找到最深的深度
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]); //继续向下挖
        }
      }
    },
  },
  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus(); //创建组件的时候就调用getMenus()方法
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style  scoped>
</style>