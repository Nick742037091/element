<template>
  <form class="el-form" :class="[
    labelPosition ? 'el-form--label-' + labelPosition : '',
    { 'el-form--inline': inline }
  ]">
    <slot></slot>
  </form>
</template>
<script>
  import objectAssign from 'element-ui/src/utils/merge';

  export default {
    name: 'ElForm',

    componentName: 'ElForm',

    // 注入依赖，方便子组件引用
    provide() {
      return {
        elForm: this
      };
    },

    props: {
      model: Object,
      rules: Object,
      // 在labelWidth设置为固定宽度时才生效
      labelPosition: String,
      // 一般设置为auto,标题和表单元素默认会添加间隔
      // 在form设置为固定宽度，form-item设置为auto时有可能会导致表单元素换行显示
      // (如行内元素宽度为100%，form-item设置为auto，则margin-left为0，由于标题设置为float而脱离文档，
      // 100%宽度即为一行宽度，所以会导致超长换行)
      labelWidth: String,
      labelSuffix: {
        type: String,
        default: ''
      },
      // 设置表单元素为行内显示，若表单元素超长会换行显示
      // (如行内元素宽度为100%，由于设置为行内元素，则margin-left为0，由于标题设置为float而脱离文档，
      // 100%宽度即为一行宽度，所以会导致超长换行)
      inline: Boolean,
      inlineMessage: Boolean,
      // 对于input输入框才有效
      statusIcon: Boolean,
      showMessage: {
        type: Boolean,
        default: true
      },
      size: String,
      disabled: Boolean,
      validateOnRuleChange: {
        type: Boolean,
        default: true
      },
      hideRequiredAsterisk: {
        type: Boolean,
        default: false
      }
    },
    watch: {
      rules() {
        // remove then add event listeners on form-item after form rules change
        this.fields.forEach(field => {
          field.removeValidateEvents();
          field.addValidateEvents();
        });

        if (this.validateOnRuleChange) {
          this.validate(() => {});
        }
      }
    },
    computed: {
      autoLabelWidth() {
        if (!this.potentialLabelWidthArr.length) return 0;
        const max = Math.max(...this.potentialLabelWidthArr);
        return max ? `${max}px` : '';
      }
    },
    data() {
      return {
        fields: [],
        potentialLabelWidthArr: [] // use this array to calculate auto width
      };
    },
    created() {
      this.$on('el.form.addField', (field) => {
        if (field) {
          this.fields.push(field);
        }
      });
      /* istanbul ignore next */
      this.$on('el.form.removeField', (field) => {
        if (field.prop) {
          this.fields.splice(this.fields.indexOf(field), 1);
        }
      });
    },
    methods: {
      resetFields() {
        if (!this.model) {
          console.warn('[Element Warn][Form]model is required for resetFields to work.');
          return;
        }
        this.fields.forEach(field => {
          field.resetField();
        });
      },
      clearValidate(props = []) {
        const fields = props.length
          ? (typeof props === 'string'
            ? this.fields.filter(field => props === field.prop)
            : this.fields.filter(field => props.indexOf(field.prop) > -1)
          ) : this.fields;
        fields.forEach(field => {
          field.clearValidate();
        });
      },
      validate(callback) {
        if (!this.model) {
          console.warn('[Element Warn][Form]model is required for validate to work!');
          return;
        }

        let promise;
        // if no callback, return promise
        if (typeof callback !== 'function' && window.Promise) {
          promise = new window.Promise((resolve, reject) => {
            callback = function(valid) {
              // promise只传递校验成功或失败
              valid ? resolve(valid) : reject(valid);
            };
          });
        }

        let valid = true;
        let count = 0;
        // 如果需要验证的fields为空，调用验证时立刻返回callback
        if (this.fields.length === 0 && callback) {
          callback(true);
        }
        let invalidFields = {};
        // 分布校验每个字段
        this.fields.forEach(field => {
          field.validate('', (message, field) => {
            if (message) {
              valid = false;
            }
            invalidFields = objectAssign({}, invalidFields, field);
            if (typeof callback === 'function' && ++count === this.fields.length) {
              // 统一进行一次回调
              callback(valid, invalidFields);
            }
          });
        });

        if (promise) {
          return promise;
        }
      },
      validateField(props, cb) {
        props = [].concat(props);
        const fields = this.fields.filter(field => props.indexOf(field.prop) !== -1);
        if (!fields.length) {
          console.warn('[Element Warn]please pass correct props!');
          return;
        }

        fields.forEach(field => {
          field.validate('', cb);
        });
      },
      getLabelWidthIndex(width) {
        const index = this.potentialLabelWidthArr.indexOf(width);
        // it's impossible
        if (index === -1) {
          throw new Error('[ElementForm]unpected width ', width);
        }
        return index;
      },
      registerLabelWidth(val, oldVal) {
        if (val && oldVal) {
          const index = this.getLabelWidthIndex(oldVal);
          this.potentialLabelWidthArr.splice(index, 1, val);
        } else if (val) {
          this.potentialLabelWidthArr.push(val);
        }
      },
      deregisterLabelWidth(val) {
        const index = this.getLabelWidthIndex(val);
        this.potentialLabelWidthArr.splice(index, 1);
      }
    }
  };
</script>
