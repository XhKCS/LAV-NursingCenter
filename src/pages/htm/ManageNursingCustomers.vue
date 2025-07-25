<!-- 健康管家端 健康管家模块 设置护理对象第二部分 -->
<script setup lang="ts">
import { onMounted, reactive, ref } from 'vue';
import { type User, type Customer } from '@/lib/type.d';
import { ElMain, ElMessage, ElNotification, ElTable, ElButton, ElCol, ElDialog, ElMessageBox } from 'element-plus'
import Button from '@/components/ui/button/Button.vue';
import { useRoute, useRouter } from 'vue-router';
import { axiosInstance as axios } from '@/lib/core'
import { useNursingCustomersStore } from '@/lib/store';

const router = useRouter()
const nursingCustomersStore = useNursingCustomersStore()

onMounted(() => {
    currentNurse.value = nursingCustomersStore.getCurrentNurse.value
    loadData()
})

const goBack = () => {
    router.back()
}

let currentNurse = ref({} as User)

// 分页数据
let availableCustomers = ref([] as Customer[]) //当前无管家的客户
let currentCustomers = ref([] as Customer[]) //当前管家已负责服务的客户

// 多选框选中的护理记录（用于批量移除）
let multipleSelection = ref([] as Customer[])

const handleSelectionChange = (val: Customer[]) => {
    multipleSelection.value = val
}

/**
 * 无管家客户的分页
 */
let availableCustomer_total = ref(0)
let availableCustomer_queryEntity = ref({
    customerType: 1, //只看护理老人
    nurseId: 0, //代表当前无管家的客户
    name: "",
    current: 1,
    size: 5
})

const availableCustomer_handleSizeChange = (val: number) => {
    availableCustomer_queryEntity.value.size = val
    loadData()
}

const availableCustomer_handleCurrentChange = (val: number) => {
    availableCustomer_queryEntity.value.current = val
    loadData()
}

/**
 * 当前服务客户的分页
 */
let currentCustomer_total = ref(0)
let currentCustomer_queryEntity = ref({
    customerType: 1, //只看护理老人
    nurseId: currentNurse.value.userId, //代表当前无管家的客户
    name: "",
    current: 1,
    size: 5
})

const currentCustomer_handleSizeChange = (val: number) => {
    currentCustomer_queryEntity.value.size = val
    loadData()
}

const currentCustomer_handleCurrentChange = (val: number) => {
    currentCustomer_queryEntity.value.current = val
    loadData()
}

const assignNurse = (customer: Customer) => {
    axios.post("/customer/assignNurse", {
        customerId: customer.customerId,
        nurseId: currentNurse.value.userId
    }).then(res => {
        if (res.data.status == 200) {
            loadData()
            ElMessage({ message: "分配成功！", type: "success" })
        } else {
            loadData()
            ElNotification({
                title: 'Error',
                message: res.data.msg,
                type: 'error',
            })
        }
    })
}

// 将选中的单个客户转移至无管家列表
const start_resetNurse = (customer: Customer) => {
    ElMessageBox.confirm('确定将该客户移出该管家的服务列表吗？', '移除确认', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
    }).then(() => {
        axios.post("/customer/resetNurse", { customerId: customer.customerId })
            .then(res => {
                if (res.data.status == 200) {
                    loadData()
                    ElMessage({ message: "移除成功！", type: "success" })
                } else {
                    loadData()
                    ElNotification({
                        title: 'Error',
                        message: res.data.msg,
                        type: 'error',
                    })
                }
            })
    }).catch(() => {
        ElMessage({
            type: 'info',
            message: '已取消移除'
        });
    });
}

const start_resetNurseBatch = () => {
    if (multipleSelection.value.length == 0) {
        ElMessage({ message: '请先勾选至少一名客户！', type: 'warning' })
        return;
    }
    ElMessageBox.confirm('确定将所有选中的客户都移出该管家的服务列表吗？', '移除确认', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
    }).then(() => {
        axios.post("/customer/resetNurseBatch", multipleSelection.value)
            .then(res => {
                if (res.data.status == 200) {
                    loadData()
                    ElMessage({ message: res.data.data, type: "success" })
                } else {
                    loadData()
                    ElNotification({
                        title: 'Error',
                        message: res.data.msg,
                        type: 'error',
                    })
                }
            })
    }).catch(() => {
        ElMessage({
            type: 'info',
            message: '已取消移除'
        });
    });
}

const loadAvailableCustomers = () => {
    axios.post("/customer/pageByNurseId", availableCustomer_queryEntity.value)
        .then(res => {
            if (res.data.status == 200) {
                availableCustomers.value = res.data.data
                availableCustomer_total.value = res.data.total
            } else {
                availableCustomers.value = res.data.data
                availableCustomer_total.value = res.data.total

            }
        })
}

const loadCurrentCustomers = () => {
    currentCustomer_queryEntity.value.nurseId = currentNurse.value.userId
    axios.post("/customer/pageByNurseId", currentCustomer_queryEntity.value)
        .then(res => {
            if (res.data.status == 200) {
                currentCustomers.value = res.data.data
                currentCustomer_total.value = res.data.total
            } else {
                currentCustomers.value = res.data.data
                currentCustomer_total.value = res.data.total

            }
        })
}

const loadData = () => {
    loadAvailableCustomers()
    loadCurrentCustomers()
}
</script>

<template>
    <el-container style="padding: 0;">
        <el-card shadow="hover" class="section-card" style="width: 100%;">
            <p>
                <label style="font-size: 18px; font-weight: bold;">管家服务对象配置 - {{ currentNurse.name }}</label>
                <Button @click="goBack()" style="font-size: 14px; margin-left: 3vh;">返回管家列表</Button>
            </p>

            <br>
            <!-- 左右表格区域 -->
            <el-row style="margin-top: 2vh; width: 100%;">
                <div style="width: 48%; height: 70vh; overflow-y: auto;">
                    <!-- 搜索框 -->
                    <p>
                        <el-input v-model="availableCustomer_queryEntity.name" clearable placeholder="客户姓名"
                            style="width: 30vh;"></el-input>
                        <Button @click="loadAvailableCustomers" class="add-button" style="margin-left: 2vh;">查询</Button>
                    </p>
                    <br>

                    <!-- <div
                        style="background-color: #007bff; font-size: 16px; font-weight: bold; width: 100%; height: 3vh;">
                        <label style="text-align: center; color: white; font-size: 15px; "> 无管家的客户信息</label>
                    </div> -->
                    <div class="text-white px-4 py-2 font-semibold rounded-t-md" style="background-color: #007bff;">
                        无管家的客户信息
                    </div>
                    <el-table class="shadow-md rounded-b-md" :data="availableCustomers" :border="true" :stripe="true"
                        style="width: 100%;" :fit="true">
                        <el-table-column type="index" label="序号" align="center">
                        </el-table-column>
                        <el-table-column property="name" label="姓名" align="center">
                        </el-table-column>
                        <el-table-column property="age" label="年龄" align="center">
                        </el-table-column>
                        <el-table-column label="性别" align="center">
                            <template #default="scope">
                                <span v-if="scope.row.gender == 0">女</span>
                                <span v-else-if="scope.row.gender == 1">男</span>
                            </template>
                        </el-table-column>
                        <el-table-column property="bedNumber" label="床位号" align="center">
                        </el-table-column>
                        <el-table-column property="nursingLevelName" label="护理级别" align="center">
                        </el-table-column>
                        <el-table-column label="操作" align="center">
                            <template #default="scope">
                                <el-button @click="assignNurse(scope.row)" style="color: #007bff;">添加</el-button>
                            </template>
                        </el-table-column>
                    </el-table>

                    <el-pagination :current-page="availableCustomer_queryEntity.current" :page-sizes="[1, 5, 10, 50]"
                        :default-page-size="availableCustomer_queryEntity.size"
                        @update:page-size="availableCustomer_handleSizeChange"
                        @update:current-page="availableCustomer_handleCurrentChange"
                        layout="total, sizes, prev, pager, next, jumper" :total="availableCustomer_total"
                        style="margin-top: 10vh;" />
                </div>

                <div style="width: 48%; height: 70vh; margin-left: 4%; overflow-y: auto;">
                    <!-- 搜索框 -->
                    <p>
                        <el-input v-model="currentCustomer_queryEntity.name" clearable placeholder="客户姓名"
                            style="width: 30vh;"></el-input>
                        <Button @click="loadCurrentCustomers" class="add-button" style="margin-left: 2vh;">查询</Button>
                    </p>
                    <br>

                    <!-- <div
                        style="background-color: #007bff; font-size: 16px; font-weight: bold; width: 100%; height: 3vh;">
                        <label style="text-align: center; color: white; font-size: 15px; "> {{ currentNurse.name }} -
                            服务中的客户</label>
                    </div> -->
                    <div class="text-white px-4 py-2 font-semibold rounded-t-md" style="background-color: #007bff;">
                        {{ currentNurse.name }} -
                        服务中的客户
                    </div>
                    <el-table class="shadow-md rounded-b-md" :data="currentCustomers" :border="true" :stripe="true" :fit="true" style="width: 100%;"
                        @selection-change="handleSelectionChange">
                        <el-table-column type="selection" align="center"></el-table-column>
                        <el-table-column type="index" label="序号" align="center">
                        </el-table-column>
                        <el-table-column property="name" label="姓名" align="center">
                        </el-table-column>
                        <el-table-column property="age" label="年龄" align="center">
                        </el-table-column>
                        <el-table-column label="性别" align="center">
                            <template #default="scope">
                                <span v-if="scope.row.gender == 0">女</span>
                                <span v-else-if="scope.row.gender == 1">男</span>
                            </template>
                        </el-table-column>
                        <el-table-column property="bedNumber" label="床位号" align="center">
                        </el-table-column>
                        <el-table-column property="nursingLevelName" label="护理级别" align="center">
                        </el-table-column>
                        <el-table-column label="操作" align="center">
                            <template #default="scope">
                                <el-button @click="start_resetNurse(scope.row)" style="color: red; ">移除</el-button>
                            </template>
                        </el-table-column>
                    </el-table>

                    <Button style="background-color: red; font-size: 15px; margin-top: 2vh; font-weight: bold;"
                        @click="start_resetNurseBatch">批量移除</Button>

                    <el-pagination :current-page="currentCustomer_queryEntity.current" :page-sizes="[1, 5, 10, 50]"
                        :default-page-size="currentCustomer_queryEntity.size"
                        @update:page-size="currentCustomer_handleSizeChange"
                        @update:current-page="currentCustomer_handleCurrentChange"
                        layout="total, sizes, prev, pager, next, jumper" :total="currentCustomer_total"
                        style="margin-top: 10vh;" />
                </div>

            </el-row>
        </el-card>

    </el-container>
</template>
