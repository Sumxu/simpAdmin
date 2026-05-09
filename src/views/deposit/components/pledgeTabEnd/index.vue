<script setup lang="ts">
import { reactive, ref, onMounted, h, nextTick } from "vue";
import FormSearch from "@/components/opts/form-search.vue";
import TableButtons from "@/components/opts/btns2.vue";
import { PureTable } from "@pureadmin/table";
import * as $userApi from "@/api/member/pledge";
import message from "@/utils/message";
import { formatAddress, formatDate, fromWei } from "@/utils/wallet";
import {
  levelOptions,
  userLevelOptions,
  userTypeMap,
  pledgeOptions,
  pidOptions,
  pledgeTypeOptions,
  pledgeTabOptions
} from "@/constants/constants";
import {
  userlevelConvert,
  pledgeConvert,
  pidScopeConvert,
  userTypeConvert
} from "@/constants/convert";
import { ElMessageBox,ElSwitch, ElSelect, ElOption, ElTable } from "element-plus";
import { saveExcelFile } from "@/utils/file";
import StatusTabs from "@/components/opts/status-tabs.vue";

import { any } from "vue-types";
const pageData: any = reactive({
  searchState: true,
  searchForm: {},
  dateTime: "2025-12-19",
  searchField: [
    {
      type: "input",
      label: "钱包地址",
      prop: "address",
      placeholder: "请输入钱包地址"
    }
  ],
  dataSource: {
    pledgeOptions: pledgeOptions,
    levelOptions: levelOptions,
    pidOptions: pidOptions,
    pledgeTypeOptions: pledgeTypeOptions,
    pledgeTabOptions: pledgeTabOptions
  },
  permission: {
    query: ["defi:user:page"]
  },
  btnOpts: {
    size: "small",
    leftBtns: [
      {
        key: "hideStatus",
        label: "批量隐藏/展示",
        icon: "ep:promotion",
        state: true
      }
    ],
    rightBtns: []
  },
  totalEnd: "",
  totalInfo: {},
  tableParams: {
    selectedRowKeys: [],
    columns: [
      {
        type: "selection",
        width: 50,
        fixed: "left"
      },
      {
        label: "钱包地址",
        prop: "address",
        width: "370px"
      },
      {
        label: "编号",
        prop: "indexNo"
      },
      {
        label: "是否隐藏",
        prop: "hideStatus",
        slot: "hideStatusScope"
      },
      {
        label: "状态",
        prop: "status",
        slot: "statusScope"
      },
      { label: "数量", prop: "amount", slot: "amountSlot" },
      {
        label: "天数",
        prop: "pid",
        slot: "pidScope"
      },
      { label: "赎回时间", prop: "redeemTime" },
      { label: "创建时间", prop: "createTime" },
      { label: "结束时间", prop: "endTime" }
    ],
    list: [],
    loading: false,
    pagination: {
      pageSize: 50,
      defaultPageSize: 50,
      currentPage: 1,
      total: 0,
      background: true,
      pageSizes: [50, 100, 200, 300, 500]
    }
  }
});

// 搜索表单变化
const _updateSearchFormData = (data: any) => (pageData.searchForm = data);
const sumbitHideStatus = () => {
  console.log(pageData.tableParams.selectedRowKeys);
  if (!pageData.tableParams.selectedRowKeys?.length) {
    message.warning("请至少选择一条记录");
    return;
  }
  const list = pageData.tableParams.selectedRowKeys.map((item, index) => {
    return item.id;
  });
  const isHide = ref<boolean>(false); // 是否拉黑
  ElMessageBox({
    title: "批量(显示/隐藏)",
    message: () =>
      h(
        "div",
        {
          style:
            "width: 320px; display: flex; flex-direction: column; gap: 20px;"
        },
        [
          // 1. 是否拉黑 - 使用 ElSwitch
          h(
            "div",
            {
              style:
                "display: flex; align-items: center; justify-content: space-between;"
            },
            [
              h("span", { style: "font-weight: 500;" }, "是否显示隐藏"),
              h(ElSwitch, {
                modelValue: isHide.value,
                "onUpdate:modelValue": (val: boolean) => {
                  isHide.value = val;
                },
                activeText: "显示",
                inactiveText: "隐藏"
              })
            ]
          )
        ]
      ),
    showCancelButton: true,
    beforeClose: async (action, instance, done) => {
      console.log("isBlack==", isHide.value);
      if (action === "confirm") {
        try {
          instance.confirmButtonLoading = true;
          await $userApi.updateHide({
            ids: list,
            status: isHide.value
          });
          message.success("修改成功");
          _loadData();
          done();
        } catch (err: any) {
          console.error("updateLevel error:", err);
          message.error(err?.message || "修改失败");
        } finally {
          instance.confirmButtonLoading = false;
        }
      } else {
        done();
      }
    }
  });
};
// 查询
const _searchForm = (data: any) => {
  pageData.searchForm = data;
  _loadData();
  _loadInfo();
};
function getCurrentDate() {
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, "0");
  const day = String(now.getDate()).padStart(2, "0");

  return `${year}-${month}-${day}`;
}
const timeClick = () => {
  $userApi
    .queryTotal({
      date: pageData.dateTime
    })
    .then((res: any) => {
      if (res.code === 200) {
        pageData.tableParams.totalInfo = res.data;
      } else {
        message.warning(res.msg);
      }
    })
    .finally(() => (pageData.tableParams.loading = false));
};

//查询统计数据
const _loadInfo = () => {
  $userApi
    .queryTotalEnd()
    .then((res: any) => {
      if (res.code === 200) {
        pageData.tableParams.totalEnd = res.data;
      } else {
        message.warning(res.msg);
      }
    })
    .finally(() => (pageData.tableParams.loading = false));
};
const _loadEndInfo = () => {
  $userApi
    .queryInfo()
    .then((res: any) => {
      if (res.code === 200) {
        pageData.tableParams.info = res.data;
      } else {
        message.warning(res.msg);
      }
    })
    .finally(() => (pageData.tableParams.loading = false));
};
// 重置
const _resetSearchForm = (data?) => (pageData.searchForm = data);

// 获取分页参数
const getQueryParams = () => ({
  ...pageData.searchForm,
  current: pageData.tableParams.pagination.currentPage,
  size: pageData.tableParams.pagination.pageSize
});
// 获取表格数据
const _loadData = (page?: number) => {
  pageData.tableParams.list = [];
  pageData.tableParams.loading = true;
  const query = getQueryParams();
  if (page) query.current = page;
  $userApi
    .queryEndPage(query)
    .then((res: any) => {
      if (res.code === 200) {
        pageData.tableParams.list = res.data.records;
        pageData.tableParams.pagination.total = Number(res.data.total);
      } else {
        message.warning(res.msg);
        pageData.tableParams.list = [];
        pageData.tableParams.pagination.total = 0;
      }
    })
    .finally(() => (pageData.tableParams.loading = false));
};
const handleSelectionChange = val => {
  pageData.tableParams.selectedRowKeys = val;
};
// 分页切换
const handleChangePageSize = (val: any) => {
  pageData.tableParams.pagination.pageSize = val;
  _loadData();
  _loadInfo();
};

const handleChangeCurrentPage = (val: any) => {
  pageData.tableParams.pagination.currentPage = val;
  _loadData();
  _loadInfo();
};

// 按钮操作
const btnClickHandle = (key: string) => {
  switch (key) {
    case "search":
      pageData.searchState = !pageData.searchState;
      break;
    case "refresh":
      _loadData();
      _loadInfo();
      break;
    case "hideStatus":
      sumbitHideStatus();
      break;
  }
};
onMounted(() => {
  nextTick(() => {
    _loadData();
    _loadInfo();
  });
});
</script>

<template>
  <el-card :shadow="'never'">
    <el-space wrap>
      <el-card class="box-card" style="width: 320px; margin: 12px 0">
        <template #header>
          <div class="card-header">
            <span>未赎回总数</span>
          </div>
        </template>
        <div class="text item">
          <span>{{ fromWei(pageData.tableParams.totalEnd) }}</span>
        </div>
      </el-card>
    </el-space>
    <form-search
      :show="pageData.searchState"
      :form-field="pageData.searchField"
      :data-source="pageData.dataSource"
      @search-form="_updateSearchFormData"
      @search="_searchForm"
      @reset="_resetSearchForm"
    />
    <table-buttons
      :size="pageData.btnOpts.size"
      :left-btns="pageData.btnOpts.leftBtns"
      :right-btns="pageData.btnOpts.rightBtns"
      @click="btnClickHandle"
    />
    <pure-table
      :data="pageData.tableParams.list"
      :columns="pageData.tableParams.columns"
      row-key="id"
      border
      stripe
      @selection-change="handleSelectionChange"
      :loading="pageData.tableParams.loading"
      :pagination="pageData.tableParams.pagination"
      @page-current-change="handleChangeCurrentPage"
      @page-size-change="handleChangePageSize"
    >
      <template #hideStatusScope="scope">
        <el-switch
          :disabled="true"
          v-model="scope.row[scope.column.property]"
          class="ml-2"
          style="--el-switch-on-color: #13ce66; --el-switch-off-color: #ff4949"
        />
      </template>
      <template #statusScope="scope">
        <span>{{ pledgeConvert(scope.row[scope.column.property]) }}</span>
      </template>

      <template #pidScope="scope">
        <span>{{ pidScopeConvert(scope.row[scope.column.property]) }}</span>
      </template>
      <template #amountSlot="scope">
        <span>{{ fromWei(scope.row[scope.column.property]) }}</span>
      </template>
    </pure-table>
  </el-card>
</template>
