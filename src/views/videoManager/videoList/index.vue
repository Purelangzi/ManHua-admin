<template>
  <div>
    <div class="header">
      <div class="search">
        <el-form ref="Searchform" :model="Searchform" label-width="80px" style="display: flex;" :inline="true">
          <el-form-item label="视频名称" style="min-width: 240px; display: flex;">
            <el-input v-model.trim="Searchform.name" placeholder="请输入搜索的内容" @keyup.enter.native="onSeach" />
          </el-form-item>
          <el-form-item>
            <el-button v-show="isBtnDisabled" type="primary" size="medium" @click="onSeach">搜索</el-button>
            <el-button v-show="!isBtnDisabled" type="warning " size="medium" @click="onReturn">返回</el-button>
          </el-form-item>

        </el-form>
        <div class="abilityBtn">

          <el-upload
            ref="importRef"
            style="margin-right: 5px;"
            action="https://jsonplaceholder.typicode.com/posts/"
            accept=".xls,.xlsx"
            :on-change="importExcChapter"
            :show-file-list="false"
            :auto-upload="false"
          >
            <el-button type="primary" size="medium">一键导入当前视频列表的剧集列表</el-button>
          </el-upload>
          <el-button type="danger" :disabled="newChapterBtn" size="medium" style="margin: 0 5px;" @click="getMoreNewChapter">一键映射剧集</el-button>
          <el-button type="danger" size="medium" style="margin: 0 5px;" @click="moreDelete">批量删除</el-button>
          <el-button type="danger" size="medium" style="margin: 0 5px;" @click="moreModify">批量修改分类</el-button>
        </div>

      </div>

      <el-dialog
        title="批量修改分类"
        :visible.sync="dialogVisible"
        width="30%"
        :before-close="closeMoreUpdateCtDialog"
      >
        <el-select v-model="selectMoreUpdateCtId" placeholder="分类名称">
          <el-option
            v-for="category in categoryData"
            :key="category.id"
            :label="category.name"
            :value="category.id"
          />
        </el-select>
        <span slot="footer">
          <el-button @click="cancelMoreUpdate">取消</el-button>
          <el-button type="primary" :disabled="!selectMoreUpdateCtId" @click="moreUpdateCategory">确定</el-button>
        </span>
      </el-dialog>

      <div class="log">
        <el-form ref="Searchform" :model="logform" label-width="80px" :inline="false">
          <el-form-item label="执行日志">
            <el-input v-model="logform.data" type="textarea" resize="none" disabled :rows="5" />
          </el-form-item>
        </el-form>
      </div>
    </div>
    <div class="content">
      <el-table ref="listTable" v-loading="loading" :data="listData" border height="calc(100vh - 420px )" size="medium" @selection-change="selectChange">
        <el-table-column type="index" label="序号" width="60" align="center">
          <template v-slot="{$index}">
            <span>{{ (page-1)*pageSize+($index+1) }}</span>
          </template>
        </el-table-column>
        <el-table-column type="selection" fixed="left" />
        <template v-for="col in columns">
          <el-table-column v-if="col.isSlot" :label="col.label" align="center" show-overflow-tooltip>
            <template v-slot="{row}">
              <div v-if="col.prop == 'cover_lateral'">
                <el-image :src="row.cover_lateral" fit="fill" style="width: 56px;height: 56px;" />
              </div>
              <div v-if="col.prop == 'is_vip'">
                <el-tag :type="row.is_vip==1?'danger':'success'" effect="dark">{{ row.is_vip==1?'收费':'不收费' }}</el-tag>
              </div>
            </template>
          </el-table-column>
          <el-table-column v-else :prop="col.prop" :label="col.label" align="center" show-overflow-tooltip />

        </template>
        <el-table-column label="操作" align="center" fixed="right" width="250">
          <template v-slot="{row}">
            <el-button type="primary" round icon="el-icon-edit" size="small" @click="editList(row)">编辑</el-button>
            <el-button type="danger" round icon="el-icon-delete" size="small" @click="deleteList(row)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>

      <el-dialog :visible.sync="dialogEditVisible" :before-close="handlerClose" :close-on-press-escape="false" :close-on-click-modal="false" width="65%">

        <editDialog v-show="detailId" ref="editDialog" :detail-id="detailId" :category-data="categoryData" @getList="getList()" />
      </el-dialog>

      <el-pagination
        style="text-align: right;padding: 20px;"
        :current-page.sync="page"
        :page-sizes="[20, 40, 80, 100]"
        :page-size="pageSize"
        layout="sizes, prev, pager, next, jumper, total"
        :total="totalNum"
        background
        @size-change="sizeChange"
        @current-change="currentChange"
      />
    </div>
  </div>
</template>

<script>
import { ImportXlsx } from '@/utils/file'
import editDialog from './components/editDialog'
export default {
  name: 'VideoList',
  components: { editDialog },
  data() {
    return {
      loading: true,
      isBtnDisabled: true,
      dialogVisible: false,
      dialogEditVisible: false,
      // 搜索表单
      Searchform: {
        name: ''
      },
      // 日志表单
      logform: {
        data: ''
      },
      // 列表数据
      listData: [],
      // 分类数据
      categoryData: [],
      // 已选择的多选框数据
      selectListOption: [],
      // 对应列的内容数据
      editListData: undefined,
      // 批量删除的id
      ids: '',
      // 已选择的批量修改的分类id
      selectMoreUpdateCtId: '',
      // 当前选择的漫画id，
      detailId: '',
      columns: [
        { isSlot: false, prop: 'name', label: '视频名称' },
        { isSlot: false, prop: 'intro', label: '简介' },
        { isSlot: true, prop: 'cover_lateral', label: '封面' },
        { isSlot: false, prop: 'read', label: '阅读量' },
        { isSlot: false, prop: 'fabulous', label: '点赞量' },
        { isSlot: false, prop: 'price', label: '价格' },
        { isSlot: true, prop: 'is_vip', label: '是否收费' }
      ],
      newChapterBtn: false,
      // 当前页
      page: 1,
      // 每页显示的条数
      pageSize: 10,
      // 总条数
      totalNum: 0

    }
  },
  mounted() {
    this.getList()
    this.getCategory()
  },
  methods: {
    // 获取列表数据
    async getList() {
      if (!this.loading) this.loading = true
      if (this.dialogEditVisible) this.handlerClose()
      const { page, pageSize } = this
      const { name } = this.Searchform
      const res = await this.$API['video'].getList({ page, pageSize, name })
      if (res.code === 200) {
        this.listData = res.data.data
        this.totalNum = this.listData.length !== 1 ? res.data.total : 1
      }
      this.loading = false
    },
    // 获取分类数据
    async getCategory() {
      const res = await this.$API['video'].getCategory()
      if (res.code === 200) {
        this.categoryData = res.data.data
      }
    },

    // 搜索
    onSeach() {
      if (!this.Searchform.name) return
      this.page = 1
      this.pageSize = 10
      this.getList()
      this.isBtnDisabled = false
      this.Searchform.name = ''
    },
    // 返回列表
    onReturn() {
      this.isBtnDisabled = true
      this.Searchform.name = ''
      this.getList()
    },

    // 一键导入章节列表
    async importExcChapter(file) {
      this.logform.data = ''
      try {
        const res = await ImportXlsx(file)
        const head = {
          小说名称: 'title',
          章节别名: 'title_alias',
          上家章节ID: 'platform_chapter',
          上家漫画ID: 'platform_comic'
        }
        const list = res.map(item => {
          const obj = {}
          for (const k in item) {
            if (head[k]) {
              obj[head[k]] = item[k]
            }
          }
          return obj
        })
        if (this.listData.length === 0) {
          this.logform.data = `视频信息匹配失败，请检查是否存在视频信息\n` + this.logform.data
          return
        }

        const listLength = list.length
        for (let i = 0; i < this.listData.length; i++) {
          const item = this.listData[i]
          const flag = list.some(el => el.platform_comic === item.platform_comic)
          if (!flag) {
            this.$notify({
              title: '警告',
              message: '当前导入的章节列表不属于当前视频列表的任何一个漫画,导入失败',
              type: 'warning'
            })
            return
          }
          this.logform.data = `开始导入章节\n`
          for (let index = 0; index < listLength; index++) {
            const el = list[index]
            // 章节所属的视频id是否等于当前视频列表已有的十个视频的id
            if (el.platform_comic === item.platform_comic) {
              // 浅拷贝,准备请求对象
              const params = { ...el }
              // 请求对象添加章节所属视频的id并赋值
              params.comic_id = item.id
              // 删除比较的请求参数
              delete params.platform_comic
              const res = await this.$API.video.addChapter(params)
              if (res.code === 200) {
                this.logform.data = `${index + 1}/${listLength}视频章节导入成功\n` + this.logform.data
              } else {
                this.logform.data = `${index + 1}/${listLength}视频章节导入失败,${res.msg}\n` + this.logform.data
              }
            }
          }
          this.logform.data = `第${i + 1}视频章节导入完毕\n` + this.logform.data
        }
      } catch (error) {
        this.logform.data = `导入失败\n` + this.logform.data
      }
    },
    // 一键映射剧集
    async getMoreNewChapter() {
      if (this.listData.length === 0) {
        this.logform.data = `视频信息匹配失败，请检查是否存在视频信息\n` + this.logform.data
        return
      }
      this.newChapterBtn = true
      for (let el = 0; el < this.listData.length; el++) {
        const item = this.listData[el]
        const params = {
          comicId: item.platform_comic,
          type: 1
        }
        const res = await this.$API.common.queryVideolDetail(params)
        if (res.code === 200) {
          this.logform.data = `开始上传剧集...`
          const chapterLength = res.data.length
          for (let index = 0; index < res.data.length; index++) {
            const item1 = res.data[index]
            item1.comic_id = item.id
            delete item1.platform_comic
            const res1 = await this.$API.video.addChapter(item1)
            if (res1.code === 200) {
              this.logform.data = `${index + 1}/${chapterLength}导入成功\n` + this.logform.data
            } else {
              this.logform.data = `${index + 1}/${chapterLength}导入失败,${res1.msg}\n` + this.logform.data
            }
          }
        } else {
          this.$message.error('IP限制,需要梯子')
        }
      }
      this.logform.data = `导入完成\n` + this.logform.data
      this.newChapterBtn = false
    },

    // 多选框改变时
    selectChange(selection) {
      this.selectListOption = selection
    },
    // 批量删除
    moreDelete() {
      if (!this.selectListOption.length) {
        this.$message.warning('请在下面选择要删除的列表')
        return
      }
      // ids = '1,2,44'
      this.ids = this.selectListOption.map(el => el.id).join(',')
      this.dropList(this.ids)
    },
    // 批量修改分类
    moreModify() {
      this.dialogVisible = true
    },
    // 确定批量修改分类
    moreUpdateCategory() {
      // 循环修改当前列表数据的分类id
      this.listData.forEach(async(el) => {
        el.category_id = this.selectMoreUpdateCtId
        try {
          const { msg } = await this.$API.video.updateList(el)
          this.logform.data += `${el.name} ${msg}\n`
        } catch (error) {
          this.logform.data += '修改失败\n'
        }
      })
      this.$message.success('当前页视频的批量修改分类成功')
      this.cancelMoreUpdate()
    },
    // 关闭批量修改分类对话框的回调
    closeMoreUpdateCtDialog(done) {
      this.selectMoreUpdateCtId = ''
      done()
    },
    // 取消批量修改分类
    cancelMoreUpdate() {
      this.selectMoreUpdateCtId = ''
      this.dialogVisible = false
    },

    // 编辑列表
    editList(row) {
      this.detailId = row.id
      this.dialogEditVisible = true
    },
    // 删除列表
    async deleteList(row) {
      this.ids = row.id
      this.dropList(this.ids, row.name)
    },
    // 漫画详情对话框关闭的回调
    handlerClose(done) {
      // 清除子组件检验规则
      this.$refs.editDialog.$refs.chapterForm.clearValidate()
      this.detailId = ''
      this.dialogEditVisible = false
      Object.assign(this.$refs.editDialog.$data.chapterForm, this.$refs.editDialog.$options.data().chapterForm)
      this.$refs.editDialog.activeName = 'first'
      this.$refs.editDialog.loading = true
      this.$refs.editDialog.logData = ''
      this.$refs.editDialog.editForm = {}
      this.$refs.editDialog.chapterList = []
      this.$refs.editDialog.editFileRaw = undefined
      if (done)done()
    },

    // 删除方法
    dropList(ids, name) {
      this.$confirm(`确定要${name ? '删除 ' + name : '批量删除'} 吗?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(async() => {
        const data = await this.$API.video.deleteList({ ids })
        if (data.code === 200) this.$message.success('删除成功')
        this.ids = ''
        this.$refs.listTable.clearSelection()
        this.page = 1
        this.getList()
      }).catch(() => {
        this.$message.info('已取消删除')
        this.ids = ''
        this.$refs.listTable.clearSelection()
      })
    },
    // 每页显示的条数改变时
    sizeChange(pageSize) {
      this.pageSize = pageSize
      this.getList()
    },
    // 当前页改变时
    currentChange(page) {
      this.page = page
      this.getList()
    }
  }
}
</script>

<style lang='scss' scoped>
.header{
  display: flex;
  justify-content: space-between;
  margin: 20px;
  padding-top: 20px;
  border-bottom: 1px solid #ebeef5;
  min-width: 850px;
  background-color: #fff;
  .search{
    display: flex;
    .abilityBtn{
      display: flex;
      align-items: flex-start;
      margin-left: 15px;
    }
  }
  .log{
    margin-right: 20px;
    width: 600px;
  }
}
.content{
  margin: 20px;
  background-color: #fff;
}

</style>
