<template>
  <el-dialog :title="updateForm.name ? '修改' : '添加'" :visible.sync="isShow" width="30%" :before-close="handleClose">
    <el-form :ref="formName" :model="formName" :rules="SearchFormRules" label-width="80px" :inline="false" size="normal">
      <el-form-item label="分类名称" prop="name">
        <el-input v-show="!isUpdateName" v-model.trim="addForm.name" v-focus placeholder="请输入添加的分类名称" />
        <el-input v-show="isUpdateName" v-model.trim="updateForm.name" v-focus placeholder="请输入修改的分类名称" />
      </el-form-item>
    </el-form>

    <span slot="footer">
      <el-button @click="cancel">取消</el-button>
      <el-button type="primary" @click="addOrUpdate">{{ isUpdateName ? '修改' : '添加' }}</el-button>
    </span>
  </el-dialog>
</template>

<script>

export default {
  name: 'CategoryDialog',
  directives: {
    focus: {
      // 存在页面时只调用一次
      inserted: (el) => {
        el.querySelector('input').focus()
      },
      // 更新后
      update: (el) => {
        el.querySelector('input').focus()
      }

    }
  },
  // 从属分类名,分类表格数据
  props: {
    categoryName: {
      type: String,
      default: ''
    },
    categoryData: {
      type: String,
      default: ''
    }
  },
  data() {
    var validateName = (rule, value, callback) => {
      const { categoryData, formName } = this
      const flag = categoryData.some(el => el.name === formName.name)
      if (flag) {
        callback(new Error('分类名称重复！'))
        return
      }
      callback()
    }
    return {
      isShow: false,
      // 添加表单
      addForm: {
        name: ''
      },
      // 修改表单
      updateForm: {
        name: ''
      },
      SearchFormRules: {
        name: [{ validator: validateName, trigger: 'blur' }, { required: true, message: '不能为空', trigger: 'change' }]
      }
    }
  },
  computed: {
    // 判断是添加还是修改
    formName() {
      return this.updateForm.name ? this.updateForm : this.addForm
    },
    // 如果是更新
    isUpdateName() {
      return !!this.updateForm.name
    }
  },
  methods: {
    // 取消添加或修改
    cancel() {
      this.addForm.name = ''
      this.updateForm.name = ''
      const { formName } = this
      this.isShow = false
      this.$refs[formName].clearValidate()
    },
    // 添加或修改分类
    addOrUpdate() {
      const { addForm, updateForm, categoryName, formName, isUpdateName } = this
      this.$refs[formName].validate(async(valid) => {
        if (valid) {
          if (!isUpdateName) {
            try {
              await this.$API[categoryName].addCategory({ name: addForm.name, parent_id: 0 })
              this.$message.success('添加成功')
              this.cancel()
              // 触发父组件事件，让表格重新刷新
              this.$emit('onLoadTable')
            } catch (error) {
              console.log(error)
            }
          } else {
            const { id, name, parent_id } = updateForm
            try {
              await this.$API[categoryName].updateCategory({ id, name, parent_id })
              this.$message.success('修改成功')
              this.cancel()
              // 触发父组件事件，让表格重新刷新
              this.$emit('onLoadTable')
            } catch (error) {
              console.log(error)
            }
          }
        } else {
          console.log('校验不通过')
        }
      })
    },
    handleClose(done) {
      this.cancel()
      done()
    }
  }
}
</script>

<style lang='scss' scoped></style>
