<script setup lang="ts">
import { reactive, onMounted, ref, h } from "vue";
import FormSearch from "@/components/opts/form-search.vue";
import TableButtons from "@/components/opts/btns2.vue";
import { PureTable } from "@pureadmin/table";
import * as $Api from "@/api/member/user";
import message from "@/utils/message";
import { fromWei } from "@/utils/wallet";
import {
  ElMessageBox,
  ElSelect,
  ElOption,
  ElInput,
  ElMessage,
  ElRadioGroup,
  ElRadio,
  ElSwitch
} from "element-plus";
import {
  pledgeTypeTwoOptions,
  pledgeWorkShopOptions,
  pledgeTypeOptions,
  pledgeLeadOptions
} from "@/constants/constants";
const pageData: any = reactive({
  searchState: true,
  searchForm: {},
  searchField: [
    {
      type: "input",
      label: "钱包地址",
      prop: "address",
      placeholder: "请输入钱包地址",
      width: "370"
    },
    {
      type: "select",
      label: "工作室",
      prop: "isWorkShop",
      placeholder: "请选择状态",
      dataSourceKey: "pledgeWorkShopOptions",
      options: {
        filterable: true,
        keys: {
          prop: "value",
          value: "value",
          label: "label"
        }
      }
    },
    {
      type: "select",
      label: "是否0号线",
      prop: "isLead",
      placeholder: "请选择",
      dataSourceKey: "pledgeLeadOptions",
      options: {
        filterable: true,
        keys: {
          prop: "value",
          value: "value",
          label: "label"
        }
      }
    },
    {
      type: "radio",
      label: "类型",
      prop: "queryType",
      default: 1,
      dataSourceKey: "pledgeTypeTwoOptions",
      options: {
        filterable: true,
        keys: {
          prop: "prop",
          value: "value",
          label: "label"
        }
      }
    },
    {
      type: "input",
      label: "上级地址",
      prop: "inviterAddress",
      placeholder: "请输入钱包地址",
      width: "370"
    }
  ],
  dataSource: {
    pledgeTypeTwoOptions: pledgeTypeTwoOptions,
    pledgeWorkShopOptions: pledgeWorkShopOptions,
    pledgeLeadOptions: pledgeLeadOptions
  },
  permission: {
    query: [""]
  },
  btnOpts: {
    size: "small",
    leftBtns: [],
    rightBtns: [
      { key: "search", label: "查询", icon: "ep:search", state: true },
      { key: "refresh", label: "刷新", icon: "ep:refresh", state: true }
    ]
  },
  tableParams: {
    columns: [
      {
        label: "钱包地址",
        prop: "address",
        minWidth: "370px"
      },
      {
        label: "邀请人地址",
        prop: "inviterAddress",
        minWidth: "370px"
      },
      {
        label: "备注",
        prop: "remark",
        minWidth: "120px"
      },
      {
        label: "是否0号线",
        prop: "isLead",
        minWidth: "120px",
        slot: "leadScope"
      },
      {
        label: "是否为工作室",
        prop: "isWorkshop",
        minWidth: "120px",
        slot: "workShopScope"
      },
      {
        label: "团队业绩",
        prop: "teamPerf",
        minWidth: "120px",
        slot: "usdtScope"
      },
      {
        label: "我的投入",
        prop: "myPerf",
        minWidth: "120px",
        slot: "usdtScope"
      },
      {
        label: "直推人数",
        prop: "directCount",
        minWidth: "120px"
      },
      {
        label: "团队人数",
        prop: "teamCount",
        minWidth: "120px"
      },
      {
        label: "团队奖励（代币)",
        prop: "teamRewardToken",
        minWidth: "160px",
        slot: "usdtScope"
      },
      {
        label: "团队奖励（USDT)",
        prop: "teamRewardUsdt",
        minWidth: "160px",
        slot: "usdtScope"
      },
      {
        label: "团队奖励（Moolah)",
        prop: "teamRewardB",
        minWidth: "160px",
        slot: "usdtScope"
      },
      { label: "创建时间", prop: "createTime", width: "180px" },
      { label: "操作", fixed: "right", slot: "operation", width: "320px" }
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

// 查询
const _searchForm = (data: any) => {
  pageData.searchForm = data;
  _loadData();
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
  $Api
    .queryPage(query)
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

// 分页切换
const handleChangePageSize = (val: any) => {
  pageData.tableParams.pagination.pageSize = val;
  _loadData();
};

const handleChangeCurrentPage = (val: any) => {
  pageData.tableParams.pagination.currentPage = val;
  _loadData();
};

const handleUpdateLead = async (address, isLead = false) => {
  try {
    await ElMessageBox.confirm(
      isLead ? "是否取消0号线？" : "是否设置0号线？",
      "提示",
      {
        confirmButtonText: "确认",
        cancelButtonText: "取消",
        type: "warning"
      }
    );
    await $Api.updateLead({
      address,
      flag: !isLead
    });
    message.success(isLead ? "已取消0号线" : "已设置0号线");
    _loadData();
  } catch (error) {
    console.log(error);
  }
};
const handleUpdateWorkShop = async (address, isWorkshop = false) => {
  try {
    await ElMessageBox.confirm(
      isWorkshop ? "是否取消工作室？" : "是否设置工作室？",
      "提示",
      {
        confirmButtonText: "确认",
        cancelButtonText: "取消",
        type: "warning"
      }
    );
    await $Api.updateWorkshop({
      address,
      flag: !isWorkshop
    });
    message.success(isWorkshop ? "已取消工作室" : "已设置工作室");
    _loadData();
  } catch (error) {
    console.log(error);
  }
};
// 按钮操作
const btnClickHandle = (key: string) => {
  switch (key) {
    case "search":
      pageData.searchState = !pageData.searchState;
      break;
    case "refresh":
      _loadData();
      break;
  }
};
const handleUpdateRemark = (row: any) => {
  const remark = ref<string>(row.remark || "");
  const address = row.address;
  ElMessageBox({
    title: "修改备注",
    message: () =>
      h(
        "div",
        {
          style:
            "width: 320px; display: flex; flex-direction: column; gap: 12px;"
        },
        [
          h(ElInput, {
            modelValue: remark.value,
            "onUpdate:modelValue": (val: string) => {
              remark.value = val;
            },
            placeholder: "请输入备注",
            clearable: true
          })
        ]
      ),

    showCancelButton: true,

    beforeClose: async (action, instance, done) => {
      if (action === "confirm") {
        try {
          instance.confirmButtonLoading = true;
          await $Api.updateRemark({
            address,
            remark: remark.value
          });
          message.success("修改成功");
          _loadData();
          done();
        } catch (err: any) {
          console.error("update remark error:", err);
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
onMounted(() => _loadData());
</script>

<template>
  <el-card :shadow="'never'">
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
      row-key="address"
      border
      stripe
      :loading="pageData.tableParams.loading"
      :pagination="pageData.tableParams.pagination"
      @page-current-change="handleChangeCurrentPage"
      @page-size-change="handleChangePageSize"
    >
      <template #leadScope="scope">
        <el-tag
          :type="scope.row[scope.column.property] ? 'success' : 'danger'"
          :effect="scope.row[scope.column.property] ? 'dark' : 'plain'"
          size="default"
        >
          {{ scope.row[scope.column.property] ? "是" : "否" }}
        </el-tag>
      </template>
      <template #workShopScope="scope">
        <el-tag
          :type="scope.row[scope.column.property] ? 'success' : 'danger'"
          :effect="scope.row[scope.column.property] ? 'dark' : 'plain'"
          size="default"
        >
          {{ scope.row[scope.column.property] ? "是" : "否" }}
        </el-tag>
      </template>
      <template #usdtScope="scope">
        <span>{{ fromWei(scope.row[scope.column.property]) }}</span>
      </template>
      <template #operation="{ row }">
        <el-link type="primary" @click="handleUpdateRemark(row)">
          修改备注
        </el-link>
        <el-link
          type="warning"
          style="margin-left: 14px"
          :disabled="row.isWorkshop ? true : false"
          @click="handleUpdateLead(row.address, row.isLead)"
        >
          {{ row.isLead ? "取消" : "设置" }}0号线
        </el-link>

        <el-link
          type="warning"
          style="margin-left: 14px"
          :disabled="row.isLead ? true : false"
          @click="handleUpdateWorkShop(row.address, row.isWorkshop)"
        >
          {{ row.isWorkshop ? "取消" : "设置" }}工作室
        </el-link>
      </template>
    </pure-table>
  </el-card>
</template>
<style>
.subtree-message-box {
  width: 600px !important;
  max-width: 600px !important;
}
</style>
