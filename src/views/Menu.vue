<script setup lang="ts">
import { h } from 'vue';
import Curd, { defineC, defineD, defineR, defineU } from '@/components/Curd.vue';
import arr2tree from '@any86/array-to-tree';
import IconFontSelector from '@/components/IconFontSelector.vue';
import { http } from '@/http';
import { Tag } from 'ant-design-vue';
const primaryKey = 'id';
const r = defineR({
  columns: [
    {
      title: 'id',
      dataIndex: 'id',
    },

    {
      title: '名称',
      dataIndex: 'name',
      // customRender({ record }) {
      //   return h('span',[record.icon, record.name]);
      // },
    },

    {
      title: '路径',
      dataIndex: 'path',
    },

    {
      title: '状态',
      customRender({ record }: any) {
        if (1 === record.state) {
          return h(Tag, { color: 'success' }, () => `开启`);
        } else {
          return h(Tag, { color: 'error' }, () => `关闭`);
        }
      },
    },

    {
      title: '操作',
      key: 'operation',
      fixed: 'right',
      width: 250,
    },
  ],

  async done() {
    const { data } = await http('/menu');
    const tree = arr2tree(data.list);
    return { total: 10, list: tree };
  },
});

const c = defineC({
  formProps: { labelCol: { span: 3 } },
  async done(formData) {
    formData.state = formData.state ? 1 : 0;
    const { data } = await http.post('/menu',formData);
    return data.msg;
  },

  items: () => [
    { is: 'a-input', name: 'name', label: '菜单名称', rules: [{ required: true, message: '必填项' }] },
    { is: 'a-input', name: 'path', label: '路径', rules: [{ required: true, message: '必填项' }] },
    { is: IconFontSelector, name: 'icon', label: '图标名' },
    { is: 'a-switch', name: 'state', label: '是否开启', modelName: 'checked' },
  ],
});

const u = defineU({
  formProps: { labelCol: { span: 3 } },
  async getDefaultValue(formData) {
    try {
      const { data } = await http.get('/menu/' + formData[primaryKey]);
      data.state = Boolean(data.state);
      return data;
    } catch (error) {
      throw error;
    }
  },

  async done(kv) {
    const { [primaryKey]: id, ...formData } = kv;
    const { data } = await http.put('/menu/' + id, formData);
    return data.msg;
  },

  items: c.items,
});

const d = defineD({
  async done(idList) {
    if (0 < idList.length) {
      const { data } = await http.delete('/menu/', {
        params: {
          idList,
        },
      });
      return data.msg;
    } else {
      const { data } = await http.delete('/meny/' + idList[0]);
      return data.msg;
    }
  },
});
</script>

<template>
  <curd v-bind="{ primaryKey, c, u, r, d }"></curd>
</template>
