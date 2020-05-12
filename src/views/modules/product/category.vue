<template>
  <div>
    <el-button type="danger" @click="batchDelete" size="mini" >批量删除</el-button>
    <el-tree ref="menuTree" :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId"
             v-loading="loading"  :default-expanded-keys="expandedKey" draggable :allow-drop="allowDrop">
     <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            v-if="node.level<= 2"
            @click="() => append(data)">
            Append
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length == 0"
            @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog :title="title" :visible.sync="dialogFormVisible" width="30%" :close-on-click-modal="false" >
      <el-form :model="category" ref="dialogForm">
        <el-form-item label="分类名称" >
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标" >
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位" >
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>
<script>
  export default {
    name: 'component_name',
    created () {
      this.getMenus()
    },
    data () {
      return {
        maxLevel: 0,
        title: '',
        dialogType: '',
        expandedKey: [],
        category: {
          catId: null,
          name: '',
          parentCid: 0,
          catLevel: 0,
          showStatus: 1,
          sort: 0,
          productUnit: '',
          icon: ''
        },
        dialogFormVisible: false,
        menus: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        },
        loading: false
      }
    },
    methods: {
      batchDelete () {
        let catIds = []
        let checkNodes = this.$refs.menuTree.getCheckedNodes()
        for (let i = 0; i < checkNodes.length; i++) {
          catIds[i] = checkNodes[i].catId
        }
        this.$confirm('是否批量删除？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              showClose: true,
              message: '批量删除成功',
              type: 'success'
            })
            this.getMenus()
            this.expandedKey = [this.category.parentCid]
          })
        })
      },
      allowDrop (draggingNode, dropNode, type) {
        this.countNodeLevel(draggingNode.data)
        let currentDeep = (this.maxLevel - draggingNode.data.catLevel) + 1
        if (type === 'inner') {
          return (currentDeep + dropNode.level) <= 3
        } else {
          return (currentDeep + dropNode.parent.level) <= 3
        }
      },
      countNodeLevel (data) {
        if (data.children != null && data.children.length > 0) {
          for (let i = 0; i < data.children.length; i++) {
            if (data.children[i].catLevel > this.maxLevel) {
              this.maxLevel = data.children[i].catLevel
            }
            this.countNodeLevel(data.children[i])
          }
        }
      },
      submitData () {
        if (this.dialogType === 'add') {
          this.addCategory()
        }
        if (this.dialogType === 'edit') {
          this.editCategory()
        }
      },
      addCategory () {
        this.$http({
          url: this.$http.adornUrl('/product/category/save'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(({data}) => {
          this.$message({
            showClose: true,
            message: '添加分类' + this.category.name + '成功',
            type: 'success'
          })
          this.dialogFormVisible = false
          this.getMenus()
          this.expandedKey = [this.category.parentCid]
        })
      },
      editCategory () {
        let {catId, name, icon, productUnit} = this.category
        let data = {catId: catId, name: name, icon: icon, productUnit: productUnit}
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData(data, false)
        }).then(({data}) => {
          this.$message({
            showClose: true,
            message: `分类【${this.category.name}】修改成功`,
            type: 'success'
          })
          this.dialogFormVisible = false
          this.getMenus()
          this.expandedKey = [this.category.parentCid]
        })
      },
      getMenus () {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get'
        }).then(({data}) => {
          this.menus = data.data
        })
      },
      edit (data) {
        this.title = '修改分类'
        this.dialogType = 'edit'
        this.dialogFormVisible = true
        this.$http({
          url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
          method: 'get'
        }).then(({data}) => {
          // this.category.name = data.data.name
          // this.category.catId = data.data.catId
          // this.category.icon = data.data.icon
          // this.category.productUnit = data.data.productUnit
          // this.category.parentCid = data.data.parentCid
          this.category = {...data.data}
        })
      },
      append (data) {
        this.title = '添加分类'
        this.dialogType = 'add'
        this.dialogFormVisible = true
        this.category.parentCid = data.catId
        this.category.catLevel = data.catLevel * 1 + 1
        this.category.name = ''
        this.category.productUnit = ''
      },
      remove (node, data) {
        var ids = [data.catId]
        var name = data.name
        this.$confirm('是否删除【' + name + '】分类？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            this.$message({
              showClose: true,
              message: '分类【' + name + '】删除成功',
              type: 'success'
            })
            this.getMenus()
            this.expandedKey = [this.category.parentCid]
          })
        })
      }
    }
  }
</script>
<style lang="scss" scoped>
</style>
