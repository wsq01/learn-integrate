<template>
  <div>
    <el-form v-show="showSearch" ref="queryRef" :model="queryParams" :inline="true" label-width="68px">
      <el-form-item label="用户账号" prop="userName">
        <el-input v-model="queryParams.userName" placeholder="请输入用户账号" clearable style="width: 200px" @keyup.enter="handleQuery" />
      </el-form-item>
      <el-form-item label="手机号码" prop="phonenumber">
        <el-input v-model="queryParams.phonenumber" placeholder="请输入手机号码" clearable style="width: 200px" @keyup.enter="handleQuery" />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="用户状态" clearable style="width: 200px">
          <el-option v-for="dict in sys_normal_disable" :key="dict.value" :label="dict.label" :value="dict.value" />
        </el-select>
      </el-form-item>
      <el-form-item label="创建时间" style="width: 300px">
        <el-date-picker v-model="dateRange" value-format="YYYY-MM-DD" type="daterange" range-separator="-" start-placeholder="开始日期" end-placeholder="结束日期"></el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <div class="mb-2 flex justify-between">
      <el-button v-hasPermi="['system:user:add']" type="primary" plain icon="Plus" @click="handleAdd">新增</el-button>
      <el-button v-hasPermi="['system:user:edit']" type="success" plain icon="Edit" :disabled="single" @click="handleUpdate">修改</el-button>
      <el-button v-hasPermi="['system:user:remove']" type="danger" plain icon="Delete" :disabled="multiple" @click="handleDelete">删除</el-button>
      <right-toolbar v-model:showSearch="showSearch" :columns="columns" @queryTable="getList"></right-toolbar>
    </div>

    <el-table border v-loading="loading" :data="list" @selectionChange="handleSelectionChange">
      <el-table-column type="selection" width="50" align="center" />
      <el-table-column align="center" show-overflow-tooltip v-if="columns[0].visible" key="userName" label="用户账号" prop="userName" />
      <el-table-column align="center" show-overflow-tooltip v-if="columns[1].visible" key="nickName" label="用户昵称" prop="nickName" />
      <el-table-column align="center" show-overflow-tooltip v-if="columns[2].visible" key="phonenumber" label="手机号码" prop="phonenumber" width="120" />
      <el-table-column align="center" show-overflow-tooltip v-if="columns[3].visible" key="status" label="状态">
        <template #default="{ row }">
          <el-switch v-model="row.status" active-value="0" inactive-value="1" @change="handleStatusChange(row)"></el-switch>
        </template>
      </el-table-column>
      <el-table-column align="center" show-overflow-tooltip v-if="columns[4].visible" label="创建时间" prop="createTime" width="170" />
      <el-table-column align="center" show-overflow-tooltip label="操作" width="300">
        <template #default="{ row }">
          <template v-if="row.userId !== 1">
            <el-button link v-hasPermi="['system:user:edit']" type="primary" icon="Edit" @click="handleUpdate(row)">修改</el-button>
            <el-button link v-hasPermi="['system:user:remove']" type="danger" icon="Delete" @click="handleDelete(row)">删除</el-button>
            <el-button link v-hasPermi="['system:user:resetPwd']" type="primary" icon="Key" @click="handleResetPwd(row)">重置密码</el-button>
            <el-button link v-hasPermi="['system:user:edit']" type="primary" icon="CircleCheck" @click="handleAuthRole(row)">分配角色</el-button>
          </template>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total > 0" v-model:page="queryParams.pageNum" v-model:limit="queryParams.pageSize" :total="total" @pagination="getList" />

    <!-- 添加或修改用户配置对话框 -->
    <el-dialog v-model="open" :title="title" width="600px" append-to-body draggable>
      <el-form ref="formRef" :model="form" :rules="rules" label-width="80px">
        <el-row>
          <el-col :span="12">
            <el-form-item label="用户昵称" prop="nickName">
              <el-input v-model="form.nickName" placeholder="请输入用户昵称" maxlength="30" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="角色">
              <el-select class="w-full" v-model="form.roleIds" multiple placeholder="请选择">
                <el-option v-for="item in roleOptions" :key="item.roleId" :label="item.roleName" :value="item.roleId" :disabled="item.status == 1"></el-option>
              </el-select>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="12">
            <el-form-item v-if="form.userId == undefined" label="用户账号" prop="userName">
              <el-input v-model="form.userName" placeholder="请输入用户账号" maxlength="30" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item v-if="form.userId == undefined" label="用户密码" prop="password">
              <el-input v-model="form.password" placeholder="请输入用户密码" type="password" maxlength="20" show-password autocomplete="new-password" />
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="12">
            <el-form-item label="手机号码" prop="phonenumber">
              <el-input v-model="form.phonenumber" placeholder="请输入手机号码" maxlength="11" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="状态">
              <el-radio-group v-model="form.status">
                <el-radio v-for="dict in sys_normal_disable" :key="dict.value" :label="dict.value">{{ dict.label }}</el-radio>
              </el-radio-group>
            </el-form-item>
          </el-col>
        </el-row>
        <el-row>
          <el-col :span="24">
            <el-form-item label="备注">
              <el-input v-model="form.remark" type="textarea" show-word-limit maxlength="200" placeholder="请输入内容"></el-input>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <template #footer>
        <el-button @click="cancel">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="User" lang="ts">
import { changeUserStatus, listUser, resetUserPwd, delUser, getUser, updateUser, addUser, deptTreeSelect } from '@/api/system/user'
import { Regular } from '@/utils/validate'

const router = useRouter()
const { proxy } = getCurrentInstance() as any
const formRef = ref<any>()
const { sys_normal_disable } = proxy.useDict('sys_normal_disable')
const list = ref<any[]>([])
const open = ref(false)
const loading = ref(true)
const showSearch = ref(true)
const ids = ref<number[]>([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const title = ref('')
const dateRange = ref<any>([])
const roleOptions = ref<any[]>([])

// 列显隐信息
const columns = ref([
  { key: 0, label: `用户账号`, visible: true },
  { key: 1, label: `用户昵称`, visible: true },
  { key: 2, label: `手机号码`, visible: true },
  { key: 3, label: `状态`, visible: true },
  { key: 4, label: `创建时间`, visible: true }
])

const form = ref<any>({})
const queryParams = ref<any>({})
const rules = ref<any>({
  userName: [
    { required: true, message: '用户账号不能为空', trigger: 'change' },
    { min: 4, max: 20, message: '用户账号长度必须介于 4 和 20 之间', trigger: 'change' },
    { pattern: Regular.Account_number(3, 19), message: '用户名账号以字母开头，只能包含字母、数字和下划线' }
  ],
  nickName: [{ required: true, message: '用户昵称不能为空', trigger: 'change' }],
  password: [
    { required: true, message: '用户密码不能为空', trigger: 'change' },
    { min: 5, max: 20, message: '用户密码长度必须介于 5 和 20 之间', trigger: 'change' }
  ],
  phonenumber: [{ pattern: /^1[3|4|5|6|7|8|9][0-9]\d{8}$/, message: '请输入正确的手机号码', trigger: 'change' }]
})

/** 查询用户列表 */
async function getList() {
  loading.value = true
  const res: any = await listUser(proxy.addDateRange(queryParams.value, dateRange.value))
  loading.value = false
  list.value = res.rows
  total.value = res.total
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}
/** 重置按钮操作 */
function resetQuery() {
  dateRange.value = []
  queryParams.value = {}
  proxy.resetForm('queryRef')
  handleQuery()
}
/** 删除按钮操作 */
async function handleDelete(row: any) {
  const userIds = row.userId || ids.value
  await proxy.$modal.confirm('是否确认删除数据项？')
  await delUser(userIds)
  getList()
  proxy.$modal.msgSuccess('删除成功')
}

/** 用户状态修改  */
async function handleStatusChange(row: any) {
  let text = row.status === '0' ? '启用' : '停用'
  try {
    await proxy.$modal.confirm('确认要"' + text + '""' + row.userName + '"用户吗?')
    await changeUserStatus(row.userId, row.status)
    proxy.$modal.msgSuccess(text + '成功')
  } catch (e: any) {
    row.status = row.status === '0' ? '1' : '0'
  }
}

/** 跳转角色分配 */
function handleAuthRole(row: any) {
  const userId = row.userId
  router.push('/system/user-auth/role/' + userId)
}
/** 重置密码按钮操作 */
async function handleResetPwd(row: any) {
  const { value }: any = await proxy.$prompt('请输入"' + row.userName + '"的新密码', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    closeOnClickModal: false,
    inputPattern: /^.{5,20}$/,
    inputErrorMessage: '用户密码长度必须介于 5 和 20 之间'
  })
  await resetUserPwd(row.userId, value)
  proxy.$modal.msgSuccess('修改成功，新密码是：' + value)
}
/** 选择条数  */
function handleSelectionChange(selection: any[]) {
  ids.value = selection.map(item => item.userId)
  single.value = selection.length !== 1
  multiple.value = !selection.length
}

/** 重置操作表单 */
function reset() {
  form.value = {
    status: '0',
    postIds: [],
    roleIds: []
  }
  proxy.resetForm('formRef')
}
/** 取消按钮 */
function cancel() {
  open.value = false
  reset()
}
/** 新增按钮操作 */
async function handleAdd() {
  reset()
  const response: any = await getUser()
  roleOptions.value = response.roles
  open.value = true
  title.value = '新增'
}
/** 修改按钮操作 */
async function handleUpdate(row: any) {
  reset()
  const response: any = await getUser(row.userId || ids.value)
  form.value = response.data
  roleOptions.value = response.roles
  form.value.roleIds = response.roleIds
  open.value = true
  title.value = '修改'
  form.value.password = ''
}
/** 提交按钮 */
async function submitForm() {
  await formRef.value.validate()
  if (form.value.userId) {
    await updateUser(form.value)
    proxy.$modal.msgSuccess('修改成功')
  } else {
    await addUser(form.value)
    proxy.$modal.msgSuccess('新增成功')
  }
  open.value = false
  getList()
}

getList()
</script>
