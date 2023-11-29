<!--
 * @Version    : v1.00
 * @Author     : itchaox
 * @Date       : 2023-09-26 15:10
 * @LastAuthor : itchaox
 * @LastTime   : 2023-11-30 00:00
 * @desc       : 
-->
<script setup>
  import { onMounted, watch, ref, watchEffect, nextTick } from 'vue';
  import { bitable } from '@lark-base-open/js-sdk';

  import Chinese from 'chinese-s2t';

  // 目标格式 s 简体; t 繁体
  const target = ref('t');

  // 选择模式 cell 单元格; field 字段; database 数据表
  const selectModel = ref('cell');

  const databaseList = ref();
  const databaseId = ref();
  const viewList = ref();
  const viewId = ref();
  const fieldList = ref();
  const fieldId = ref();

  const changeModelId = ref();
  const modelList = ref([
    {
      id: '1',
      name: '英文字母转小写',
    },
    {
      id: '2',
      name: '英文字母转大写',
    },
    {
      id: '3',
      name: '单词首字母大写',
    },

    {
      id: '4',
      name: '英文字母大小写反转',
    },
  ]);

  const getFn = (id) => {
    switch (id) {
      case '1':
        return convertEnglishToLowerCase;
      case '2':
        return convertEnglishToUpperCase;
      case '3':
        return capitalizeFirstLetterOfWords;
      case '4':
        return toggleCase;
    }
  };

  const isLoading = ref(false);

  const base = bitable.base;

  // 当前点击字段id
  const currentFieldId = ref();
  const recordId = ref();

  const currentValue = ref();

  onMounted(async () => {
    databaseList.value = await base.getTableMetaList();
  });

  // 切换数据表, 默认选择第一个视图
  async function databaseChange() {
    if (selectModel.value === 'field') {
      const table = await base.getTable(databaseId.value);
      viewList.value = await table.getViewMetaList();
      viewId.value = viewList.value[0]?.id;
    }
  }

  // 根据视图列表获取字段列表
  watch(viewId, async (newValue, oldValue) => {
    const table = await base.getTable(databaseId.value);
    const view = await table.getViewById(newValue);
    const _list = await view.getFieldMetaList();

    // 只展示文本相关字段
    fieldList.value = _list.filter((item) => item.type === 1);
  });

  // 切换选择模式时,重置选择
  watch(selectModel, async (newValue, oldValue) => {
    if (newValue === 'cell') return;
    // 单列和数据表模式，默认选中当前数据表和当前视图

    const selection = await base.getSelection();
    databaseId.value = selection.tableId;

    if (newValue === 'field') {
      fieldId.value = '';
      fieldList.value = [];
      viewId.value = '';

      const table = await base.getTable(databaseId.value);
      viewList.value = await table.getViewMetaList();
      viewId.value = selection.viewId;
    }
  });

  // 数据表修改后，自动获取视图列表
  watchEffect(async () => {
    const table = await base.getTable(databaseId.value);
    viewList.value = await table.getViewMetaList();
  });

  base.onSelectionChange(async (event) => {
    // 获取点击的字段id和记录id
    currentFieldId.value = event.data.fieldId;
    recordId.value = event.data.recordId;

    // 获取当前数据表和视图
    databaseId.value = event.data.tableId;
    viewId.value = event.data.viewId;

    // FIXME 数据表切换后，视图自动切换

    const table = await base.getActiveTable();
    if (currentFieldId.value && recordId.value) {
      // 修改当前数据
      let data = await table.getCellValue(currentFieldId.value, recordId.value);
      if (data && data[0].text !== currentValue.value) {
        currentValue.value = data[0].text;
      }
    }
  });

  async function confirm() {
    isLoading.value = true;
    if (selectModel.value === 'cell') {
      await cellModel();
    } else if (selectModel.value === 'field') {
      await fieldModel();
    } else {
      await databaseModel();
    }
    isLoading.value = false;
  }

  async function cellModel() {
    const table = await base.getActiveTable();
    let newValue;

    let fn = getFn(changeModelId.value);

    newValue = fn(currentValue.value);
    if (currentFieldId.value && recordId.value) {
      await table.setCellValue(currentFieldId.value, recordId.value, newValue);
    }
  }

  async function fieldModel() {
    ElMessage({
      message: '开始转换数据~',
      type: 'success',
      duration: 1500,
    });

    const table = await bitable.base.getTable(databaseId.value);
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    let _list = [];

    for (const record of recordList) {
      const id = record.id;
      // 获取索引
      const index = recordList.recordIdList.findIndex((iId) => iId === id);

      // FIXME 用这个api获取 cell，性能很差
      // const cell = await record.getCellByField(fieldId.value);

      const field = await table.getFieldById(fieldId.value);
      const cell = await field.getCell(recordIds[index]);
      const val = await cell.getValue();
      // const val = await cell.val;

      if (!val) continue;

      let newValue;

      let fn = getFn(changeModelId.value);
      newValue = fn(val[0]?.text);

      // FIXME 处理数据
      _list.push({
        recordId: recordIds[index],
        fields: {
          [fieldId.value]: newValue,
        },
      });
    }

    // FIXME 此处一次性全部替换
    await table.setRecords(_list);

    ElMessage({
      message: '数据转换结束!',
      type: 'success',
      duration: 1500,
    });
  }

  async function databaseModel() {
    ElMessage({
      message: '开始转换数据~',
      type: 'success',
      duration: 1500,
    });

    const table = await bitable.base.getTable(databaseId.value);
    const _fieldList = await table.getFieldMetaList();
    const recordList = await table.getRecordList();
    const recordIds = await table.getRecordIdList(); // 获取所有记录 id

    const filterFieldList = _fieldList.filter((item) => item.type === 1);

    for (const item of filterFieldList) {
      let _list = [];
      for (const record of recordList) {
        const id = record.id;
        // 获取索引
        const index = recordList.recordIdList.findIndex((iId) => iId === id);

        // 只遍历文本列
        const field = await table.getFieldById(item.id);
        const cell = await field.getCell(recordIds[index]);
        const val = await cell.getValue();

        if (val) {
          let newValue;
          let fn = getFn(changeModelId.value);
          newValue = fn(val[0]?.text);

          // FIXME 处理数据
          _list.push({
            recordId: recordIds[index],
            fields: {
              [item.id]: newValue,
            },
          });
        }
      }

      // FIXME 此处一次性全部替换
      await table.setRecords(_list);
    }

    ElMessage({
      message: '数据转换结束!',
      type: 'success',
      duration: 1500,
    });
  }

  // 英文全小写
  const convertEnglishToLowerCase = (input) => {
    // 使用正则表达式匹配英文字符，然后转换为小写
    const result = input.replace(/[A-Z]/g, (match) => match.toLowerCase());
    return result;
  };

  // 英文全大写
  const convertEnglishToUpperCase = (input) => {
    // 使用正则表达式匹配英文字符，然后转换为大写
    const result = input.replace(/[a-z]/g, (match) => match.toUpperCase());
    return result;
  };

  // 英文单词首字母大写,其余小写
  const capitalizeFirstLetterOfWords = (input) => {
    // 使用正则表达式匹配单词，然后转换为首字母大写，其他字母小写
    const result = input.replace(/\b[a-zA-Z]+\b/g, (word) => {
      return word.charAt(0).toUpperCase() + word.slice(1).toLowerCase();
    });
    return result;
  };

  // 英文大小写反转
  const toggleCase = (input) => {
    return input.replace(/[a-zA-Z]/g, (char) => {
      return char === char.toUpperCase() ? char.toLowerCase() : char.toUpperCase();
    });
  };
</script>

<template>
  <div class="s2t">
    <div>
      <div class="tip">
        <div class="tip-text title">操作步骤:</div>

        <div class="tip-text">1. 选择目标格式</div>
        <div
          class="tip-text"
          v-if="selectModel === 'cell'"
        >
          2. 选择需要转换的单元格
        </div>
        <div
          class="tip-text"
          v-else-if="selectModel === 'field'"
        >
          2. 选择顺序: 数据表 -> 视图 -> 字段
        </div>
        <div
          class="tip-text"
          v-else-if="selectModel === 'database'"
        >
          2. 选择需要转换的数据表
        </div>
        <div class="tip-text">3. 点击[确认转换]按钮即可</div>
      </div>
    </div>

    <div class="label">
      <div class="text">转化模式:</div>
      <el-select
        v-model="changeModelId"
        placeholder="请选择转换模式"
        popper-class="selectStyle"
      >
        <el-option
          v-for="item in modelList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>

    <div class="label">
      <div class="text">选择模式:</div>
      <el-radio-group v-model="selectModel">
        <el-radio-button label="cell">单元格</el-radio-button>
        <el-radio-button label="field">字段</el-radio-button>
        <el-radio-button label="database">数据表</el-radio-button>
      </el-radio-group>
    </div>

    <div
      class="label"
      v-if="selectModel !== 'cell'"
    >
      <div class="text">数据表:</div>
      <el-select
        v-model="databaseId"
        placeholder="请选择数据表"
        @change="databaseChange"
        popper-class="selectStyle"
      >
        <el-option
          v-for="item in databaseList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>

    <div
      class="label"
      v-if="selectModel === 'field'"
    >
      <div class="text">视图:</div>
      <el-select
        v-model="viewId"
        placeholder="请选择视图"
        popper-class="selectStyle"
      >
        <el-option
          v-for="item in viewList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>
    <div
      class="label"
      v-if="selectModel === 'field'"
    >
      <div class="text">字段:</div>
      <el-select
        v-model="fieldId"
        placeholder="请选择字段"
        popper-class="selectStyle"
      >
        <el-option
          v-for="item in fieldList"
          :key="item.id"
          :label="item.name"
          :value="item.id"
        />
      </el-select>
    </div>
    <el-button
      type="primary"
      class="btn"
      @click="confirm"
      :loading="isLoading"
    >
      确认转换</el-button
    >
  </div>
</template>

<style scoped>
  .s2t {
    font-family: LarkHackSafariFont, LarkEmojiFont, LarkChineseQuote, -apple-system, BlinkMacSystemFont, Helvetica Neue,
      Tahoma, PingFang SC, Microsoft Yahei, Arial, Hiragino Sans GB, sans-serif, Apple Color Emoji, Segoe UI Emoji,
      Segoe UI Symbol, Noto Color Emoji;
    font-weight: 300;
  }

  .tip {
    color: #8f959e;
    font-size: 12px;
    margin-bottom: 24px;
    .tip-text {
      margin-bottom: 4px;
    }

    .title {
      font-size: 14px;
      margin-bottom: 8px;
    }
  }

  .label {
    display: flex;
    align-items: center;
    margin-bottom: 20px;

    .text {
      width: 70px;
      margin-right: 10px;
      white-space: nowrap;
      font-size: 14px;
    }

    :deep(.el-radio-button__original-radio:checked + .el-radio-button__inner) {
      color: #fff;
      background-color: rgb(20, 86, 240);
      border-color: rgb(20, 86, 240);
      box-shadow: 1px 0 0 0 rgb(20, 86, 240);
    }

    :deep(.el-radio-button__inner) {
      font-weight: 300;
    }

    :deep(.el-radio-button__inner:hover) {
      color: rgb(20, 86, 240);
    }

    :deep(.el-input__inner) {
      font-weight: 300;
    }
  }

  .btn {
    margin-top: 14px;
    background-color: rgb(20, 86, 240);
    border-color: rgb(20, 86, 240);
    font-weight: 300;
    &:hover {
      background-color: rgb(51, 109, 244);
      border-color: rgb(20, 86, 240);
    }
  }
</style>

<style>
  .selectStyle {
    .el-select-dropdown__item {
      font-weight: 300 !important;
    }

    .el-select-dropdown__item.selected {
      color: rgb(20, 86, 240);
    }
  }
</style>
