<script lang="ts">
export function defineC(config: CProps): CProps {
  return config;
}

export function defineU(config: UProps): UProps {
  return config;
}

export function defineR(config: RProps): RProps {
  return config;
}

export function defineD(config: DProps): DProps {
  return config;
}
</script>

<script setup lang="ts">
import { cloneDeep } from 'lodash';
import { ref, reactive, watch, computed, onBeforeMount, onMounted } from 'vue';
import { message } from 'ant-design-vue';
import ColumnSort from './Curd/ColumnSort.vue';
import {
  EyeOutlined,
  EditOutlined,
  DeleteOutlined,
  SearchOutlined,
  PlusOutlined,
  RedoOutlined,
} from '@ant-design/icons-vue';
import Add from './Curd/Add.vue';
import Edit from './Curd/Edit.vue';
import NForm from './Curd/NForm.vue';
import type { CProps, DProps, RProps, UProps, KV } from '@/components/Curd/Types';

interface Props {
  primaryKey: string;

  onBeforeMount?: () => Promise<unknown>;

  c?: CProps;

  u?: UProps;

  r: RProps;

  d?: DProps;
}

// 表格需要的数据源
type tableData = {
  list: { [k: string]: unknown }[];
  total: string;
};

const props = defineProps<Props>();
const emit = defineEmits<{
  (type: 'remove-fail', error: unknown): void;
}>();

// 显示前
const isLoading = ref(false);

const columnConfig = ref(cloneDeep(props.r.columns)!);
function changeColumns(columns: any) {
  columnConfig.value = columns;
}

onBeforeMount(async () => {
  if (props.onBeforeMount) {
    isLoading.value = true;
    await props.onBeforeMount();
    isLoading.value = false;
  }
});

// 默认条件
const CONDITION_DEFAULT = {};

// 筛选条件
// 复制一份, 保留默认值
const formDataCondition = ref({ ...CONDITION_DEFAULT });
// 是否显示更多筛选条件
const isShowMoreCondition = ref(true);

// 表格的选择
// 注意table组件上一定要指定rowKey属性才能生效
const selectedRowKeys = ref<string[]>([]);
function onTableSelectChange(keys: string[]) {
  selectedRowKeys.value = keys;
}

// 分页
const pageCurrent = ref(1);
const pageSize = ref(10);
const pageCount = ref(0);
const isTableLoading = ref(true);
const dataSouce = ref<tableData['list']>([]);
const pagination = computed(() => ({
  total: pageCount.value,
  current: pageCurrent.value,
  pageSize: pageSize.value,
  onChange: (page: number) => {
    pageCurrent.value = page;
  },
  onShowSizeChange: onPageSizeChange,
}));

const tableSize = ref('default');
const tableProps = computed(() => {
  const { r, primaryKey } = props;
  return {
    pagination: { ...r.pagination, ...pagination },
    rowKey: (row: KV) => row[primaryKey],
    rowSelection: r.hideRowSelection ? null : { selectedRowKeys, onChange: onTableSelectChange, ...r.rowSelection },
  };
});

function onPageSizeChange(current: number, size: number) {
  pageSize.value = size;
  pageCurrent.value = 1;
}

// 重置条件
// 并加载数据
function reset() {
  formDataCondition.value = { ...CONDITION_DEFAULT };
  getList();
}

// 加载数据
async function getList() {
  isTableLoading.value = true;
  try {
    const { list, total } = await props.r.done({
      pageNum: pageCurrent.value,
      pageSize: pageSize.value,
      ...formDataCondition.value,
    });
    dataSouce.value = list;
    isTableLoading.value = false;
    pageCount.value = ~~total;
  } catch (error) {
    message.error(error as string);
  }
}
watch([pageCurrent, pageSize], getList, { immediate: true });

// ========= 删除 =========
async function remove(keys: string[], row?: KV) {
  try {
    await props.d!.done(keys, row);
    // message.success('删除成功');
    getList();
  } catch (error) {
    emit('remove-fail', error);
    // message.error('删除失败,请重试!');
  }
}

// 主要用来初始化默认值
const FormDataAdd = reactive({});
const FormDataEdit = reactive({});

// 新增
const addRef = ref<typeof Add | undefined>();
const isAddFormLoading = ref(false);
async function showAddForm() {
  try {
    isAddFormLoading.value = true;
    if (props.c?.before) {
      await props.c.before();
    }
    addRef.value?.show();
  } catch (error) {
    console.log(error);
  } finally {
    isAddFormLoading.value = false;
  }
}

// 编辑
const editRef = ref<typeof Edit | undefined>();

// 详情
const isShowOne = ref(false);
const oneData = ref<KV>();
const isOneLoading = ref(false);
async function showOne(row: KV) {
  isShowOne.value = true;
  isOneLoading.value = true;
  oneData.value = await props.r.getOne!(row);
  isOneLoading.value = false;
}

// const = useColumnSetting();
</script>

<template>
  <a-card class="curd" :loading="isLoading">
    <a-drawer v-if="r.getOne" v-model:visible="isShowOne" title="详情" width="50%">
      <a-skeleton :loading="isOneLoading">
        <slot name="one" v-bind="oneData"></slot>
      </a-skeleton>
    </a-drawer>

    <!-- 编辑 -->
    <Edit ref="editRef" v-if="u" v-model="FormDataEdit" v-bind="u" @success="getList" />

    <!-- 新增 -->
    <Add ref="addRef" v-if="c" v-model="FormDataAdd" v-bind="c" @success="getList" />

    <div class="d-flex align-items-center">
      <!-- 批量操作 -->
      <a-button v-if="c" :loading="isAddFormLoading" type="primary" @click="showAddForm"
        ><plus-outlined />新建</a-button
      >
      <a-popconfirm
        v-if="void 0 !== d"
        title="确定要删除吗?"
        ok-text="确定"
        cancel-text="取消"
        @confirm="remove(selectedRowKeys)"
      >
        <a-button class="ml-1" v-show="selectedRowKeys.length > 0" type="primary" ghost danger
          >批量删除({{ selectedRowKeys.length }}条)</a-button
        >
      </a-popconfirm>

      <p class="flex-1" align="right">
        <a-space :size="16">
          <a class="icon-reset" @click="reset"><redo-outlined /></a>

          <!-- 筛选条件 -->
          <column-sort v-if="r.columns" :columns="(r.columns as any)" @change="changeColumns" />

          <!-- 列密度 -->
          <a-tooltip title="表格尺寸">
            <a-radio-group
              v-model:value="tableSize"
              size="small"
              option-type="button"
              :options="[
                { label: '默认', value: 'default' },
                { label: '中等', value: 'middle' },
                { label: '紧凑', value: 'small' },
              ]"
            />
          </a-tooltip>
        </a-space>
      </p>
    </div>

    <n-form
      class="mt-2"
      v-model="formDataCondition"
      v-if="r.conditionItems"
      :items="r.conditionItems"
      layout="inline"
      :label-col="{ span: 5 }"
    >
      <template #after>
        <a-form-item>
          <a-space>
            <a-button :loading="isTableLoading" @click="reset">重置</a-button>
            <a-button type="primary" :loading="isTableLoading" @click="getList"><search-outlined />查询</a-button>
            <!-- <a-button @click="isShowMoreCondition = !isShowMoreCondition" type="link">
                <template v-if="isShowMoreCondition"><up-outlined />收起</template>
                <template v-else><down-outlined />展开</template>
              </a-button> -->
          </a-space>
        </a-form-item>
      </template>
    </n-form>
    <!-- 表格数据 -->
    <a-table
      bordered
      class="mt-2"
      :size="tableSize"
      :loading="isTableLoading"
      :columns="columnConfig"
      :dataSource="dataSouce"
      v-bind="tableProps"
    >
      <template #bodyCell="{ column, record }">
        <template v-if="column.dataIndex === 'operation' || column.key === 'operation'">
          <slot name="row-buttons-before" v-bind="record"></slot>

          <a-button v-if="r.getOne" type="link" @click="showOne(record)"><eye-outlined />查看</a-button>

          <a-button v-if="void 0 !== u" type="link" size="small" @click="editRef?.show(record)">
            <edit-outlined />编辑</a-button
          >
          <a-popconfirm
            v-if="void 0 !== d"
            title="确定要删除吗?"
            ok-text="确定"
            cancel-text="取消"
            @confirm="remove([record[primaryKey]], record)"
          >
            <a-button type="link" size="small"><delete-outlined />删除</a-button>
          </a-popconfirm>
        </template>
      </template>
    </a-table>
  </a-card>
</template>

<style lang="scss">
.curd {
  &__header {
    background-color: #fff;
  }

  .ant-table-cell:empty {
    &:after {
      content: '暂无';
      text-align: center;
      color: #ccc;
      font-size: 12px;
    }
  }
}

.icon-reset {
  transform: rotate3d(30deg);
  &:active {
  }
}
</style>
