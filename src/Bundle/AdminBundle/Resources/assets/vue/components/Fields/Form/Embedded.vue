<template>
  <form-row :domID="domID" :field="field" :alerts="!embeddedView ? [{ level: 'warning', message: $t('field.embedded.missing_view_warning') }] : globalViolations">
    <multi-field v-if="embeddedView" :field="field" :val="val" @addRow="val.push({})" @removeRow="removeByKey" v-slot:default="multiProps">

      <div class="uk-input-group uk-text-center" v-if="embeddedView.category === 'union' && !unionViews[multiProps.rowKey || 0]">
        <a class="union-select-card uk-card uk-card-small uk-card-default uk-card-body uk-text-center" @click.prevent="setFieldValue('__typename', [view.type], multiProps.rowKey || 0)" v-for="view in embeddedView.possibleViews">
          {{ view.name }}
        </a>
      </div>
      <div v-else class="uk-input-group">

        <div v-if="embeddedView.category === 'union'" class="uk-label uk-label-secondary">
          {{ unionViews[multiProps.rowKey || 0].name }}
          <a @click.prevent="clearUnionView(multiProps.rowKey || 0)" class="uk-icon-link"><icon name="x" /></a>
        </div>

        <component :key="field.id" v-for="field in normalizedViews[multiProps.rowKey || 0].formFields()" :is="$unite.getFormFieldType(field)" :field="field" :value="values[multiProps.rowKey || 0] ? values[multiProps.rowKey || 0][field.id] : undefined" @input="setFieldValue(field.id, arguments, multiProps.rowKey || 0)" :violations="violationsForField(field.id, multiProps.rowKey || 0)" />
      </div>
    </multi-field>
  </form-row>
</template>
<script>
  import _abstract from "./_abstract";
  import FormRow from './_formRow';
  import MultiField from './_multiField';
  import Icon from "../../Icon";
  import UIkit from 'uikit';
  import {getAdminViewByType, removeIntroSpecType} from '../../../plugins/unite';

  export default {

      // Static query methods for unite system.
      queryData(field, unite, depth) {
          return `${ field.id } { ${ getAdminViewByType(unite, field.returnType).queryFormData(depth + 1).join("\n") } }`
      },
      normalizeQueryData(queryData, field, unite, depth) {

          if(!queryData) {
              return queryData;
          }

          let view = getAdminViewByType(unite, field.returnType);

          if(Array.isArray(queryData)) {
              return queryData.map((rowData) => {
                  return view.normalizeQueryData(rowData, depth);
              });
          }

          return view.normalizeQueryData(queryData, depth + 1);
      },
      normalizeMutationData(formData, field, unite, depth) {

          if(!formData) {
              return formData;
          }

          let view = getAdminViewByType(unite, field.returnType);

          if(Array.isArray(formData)) {
              return formData.map((rowData) => {
                  return view.normalizeMutationData(rowData, depth);
              });
          }

          return view.normalizeMutationData(formData, depth + 1);
      },

      // Vue properties for this component.
      extends: _abstract,
      components: { FormRow, MultiField, Icon },

      computed: {
          embeddedView() {
              return getAdminViewByType(this.$unite, this.field.returnType);
          },

          unionViews() {
              let rows = this.values;
              rows = rows.length > 0 ? rows : [{}];
              return rows.map((row) => {
                  return this.unionView(row.__typename)
              });
          },

          normalizedViews() {
              return this.unionViews.map((view) => {
                  return view || this.embeddedView;
              });
          },

          globalViolations() {
              return this.violations.filter((violation) => {
                  return violation.path.length === 1;
              });
          }

      },
      methods: {
          setFieldValue(field, args, key) {
              if(this.field.list_of) {
                  this.val[key] = this.val[key] || {};
                  this.$set(this.val[key], field, args[0]);
              } else {
                  this.val = this.val || {};
                  this.$set(this.val, field, args[0]);
              }
          },

          unionView(type) {
              if(!type || this.embeddedView.category !== 'union') {
                  return null;
              }
              let views = this.embeddedView.possibleViews.filter((view) => { return view.type === type; });
              return views.length > 0 ? views[0] : null;
          },

          clearUnionView(delta) {
              UIkit.modal.confirm(this.$t('field.embedded.confirm.clear_union_selection')).then(() => {
                  this.setValue([{}], delta);
              });
          },

          violationsForField(field, key) {
              return this.violations.filter((violation) => {
                  if(this.field.list_of) {
                      return violation.path.length > 2 && violation.path[1] === '' + key && violation.path[2] === field;
                  } else {
                      return violation.path.length > 1 && violation.path[1] === field;
                  }
              }).map((violation) => {
                  let path = this.field.list_of ? violation.path.slice(2) : violation.path.slice(1);
                  return Object.assign({}, violation, { path });
              });
          }
      }
  }
</script>
<style scoped lang="scss">
  .union-select-card {
    width: 200px;
    display: inline-block
  }
</style>