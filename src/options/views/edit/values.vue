<template>
  <div class="edit-values" ref="container">
    <div class="mb-1">
      <button @click="onNew">+</button>
      <div class="inline-block ml-2" v-if="totalPages > 1">
        <button :disabled="!hasPrevious" @click="page = currentPage.page - 1">&larr;</button>
        <span class="mx-1" v-text="page"></span>
        <button :disabled="!hasNext" @click="page = currentPage.page + 1">&rarr;</button>
      </div>
    </div>
    <div class="edit-values-table" v-if="keys">
      <div class="edit-values-empty" v-if="!keys.length">
        <div v-text="i18n('noValues')"></div>
      </div>
      <a
        v-for="key in currentPage.data"
        :key="key"
        class="edit-values-row flex"
        href="#"
        @keydown.delete.ctrl.exact="onRemove(key)"
        @click.prevent="onEdit(key)">
        <div class="ellipsis">
          <span v-text="key"></span>
          <div class="edit-values-btn" @click.stop="onRemove(key)">
            <tooltip :content="`Ctrl-Del: ${i18n('buttonRemove')}`">
              <icon name="trash"/>
            </tooltip>
          </div>
        </div>
        <div class="ellipsis flex-auto" v-text="getValue(key, true)"></div>
      </a>
    </div>
    <div class="edit-values-panel flex flex-col" v-if="current">
      <div class="flex mb-1">
        <h4 class="flex-auto" v-text="i18n('labelEditValue')"></h4>
        <div>
          <button v-text="i18n('editValueSave')" @click="onSave"
                  :class="{'has-error': current.error}"
                  :title="current.error"
                  :disabled="current.error"/>
          <button v-text="i18n('editValueCancel')" @click="onCancel"></button>
        </div>
      </div>
      <label class="mb-1" v-text="i18n('valueLabelKey')"></label>
      <input type="text" v-model="current.key" :readOnly="!current.isNew"
             ref="key"
             spellcheck="false"
             @keydown.esc.exact.stop="onCancel">
      <label class="mt-1 mb-1" v-text="i18n('valueLabelValue')"></label>
      <textarea class="flex-auto" v-model="current.value"
                ref="value"
                spellcheck="false"
                @input="onChange"
                @keydown.esc.exact.stop="onCancel"/>
    </div>
  </div>
</template>

<script>
import Tooltip from 'vueleton/lib/tooltip/bundle';
import { dumpScriptValue, sendCmd } from '#/common';
import Icon from '#/common/ui/icon';
import storage from '#/common/storage';
import { showMessage } from '../../utils';

const PAGE_SIZE = 25;
const MAX_LENGTH = 1024;
const MAX_JSON_DURATION = 10; // ms
let scriptStorageKey;
let focusedElement;

const reparseJson = (str) => {
  try {
    return JSON.stringify(JSON.parse(str), null, '  ');
  } catch (e) {
    // This shouldn't happen but the storage may get corrupted or modified directly
    return str;
  }
};

export default {
  props: ['active', 'script'],
  components: {
    Icon,
    Tooltip,
  },
  data() {
    return {
      current: null,
      keys: null,
      values: null,
    };
  },
  computed: {
    totalPages() {
      if (!this.keys) return 0;
      return Math.floor(this.keys.length / PAGE_SIZE) + 1;
    },
    currentPage() {
      const page = Math.max(1, Math.min(this.page, this.totalPages));
      const offset = PAGE_SIZE * (page - 1);
      return {
        page,
        data: this.keys ? this.keys.slice(offset, offset + PAGE_SIZE) : null,
      };
    },
    hasPrevious() {
      return this.currentPage.page > 1;
    },
    hasNext() {
      return this.currentPage.page < this.totalPages;
    },
  },
  watch: {
    active(val) {
      if (val) {
        storage.value.getOne(this.script.props.id).then(this.setData);
        scriptStorageKey = storage.value.prefix + this.script?.props.id;
        (focusedElement || this.$refs.container.querySelector('button')).focus();
      }
      browser.storage.onChanged[`${val ? 'add' : 'remove'}Listener`](this.onStorageChanged);
    },
    current(val, oldVal) {
      if (val) {
        focusedElement = document.activeElement;
        this.$nextTick(() => {
          const el = this.$refs[val.isNew ? 'key' : 'value'];
          el.setSelectionRange(0, 0);
          el.focus();
        });
      } else if (oldVal) {
        focusedElement?.focus();
      }
    },
  },
  methods: {
    getValue(key, sliced) {
      let value = this.values[key];
      const type = value[0];
      value = value.slice(1);
      if (type === 's') value = JSON.stringify(value);
      else if (!sliced) value = reparseJson(value);
      if (sliced && value.length > MAX_LENGTH) {
        value = value.slice(0, MAX_LENGTH);
      }
      return value;
    },
    setData(values = {}) {
      this.values = values;
      this.keys = Object.keys(values).sort();
      this.page = Math.min(this.page, Math.ceil(this.keys.length / PAGE_SIZE)) || 1;
    },
    updateValue({ key, isNew }) {
      const rawValue = dumpScriptValue(this.current?.jsonValue) || '';
      const { id } = this.script.props;
      return sendCmd('UpdateValue', { id, key, value: rawValue })
      .then(() => {
        if (rawValue) {
          this.$set(this.values, key, rawValue);
          if (isNew) this.keys.push(key);
        } else {
          const i = this.keys.indexOf(key);
          if (i >= 0) this.keys.splice(i, 1);
          this.$delete(this.values, key);
        }
      });
    },
    onNew() {
      this.current = {
        isNew: true,
        key: '',
        value: '',
      };
    },
    onRemove(key) {
      this.updateValue({ key })
      .then(() => {
        if (this.current && this.current.key === key) {
          this.current = null;
        }
      });
    },
    onEdit(key) {
      this.current = {
        isNew: false,
        key,
        value: this.getValue(key),
      };
    },
    async onSave() {
      const { current } = this;
      if (current.jsonPaused) {
        current.jsonPaused = false;
        this.onChange();
      }
      if (current.error) {
        const pos = +current.error.match(/position\s+(\d+)|$/)[1] || 0;
        this.$refs.value.setSelectionRange(pos, pos + 1);
        this.$refs.value.focus();
        showMessage({ text: current.error });
        return;
      }
      await this.updateValue(current);
      this.current = null;
    },
    onCancel() {
      this.current = null;
    },
    onChange() {
      const { current } = this;
      current.error = null;
      if (current.jsonPaused) return;
      const t0 = performance.now();
      const str = current.value.trim();
      try {
        current.jsonValue = str ? JSON.parse(str) : undefined;
      } catch (e) {
        current.error = e.message || e;
        current.jsonValue = undefined;
      }
      current.jsonPaused = performance.now() - t0 > MAX_JSON_DURATION;
    },
    onStorageChanged(changes) {
      const data = changes[scriptStorageKey]?.newValue;
      if (data) {
        this.setData(data);
        const { current } = this;
        if (current) {
          const newText = this.getValue(current.key);
          if (newText === current.value) {
            current.isNew = false;
          } else {
            // Keeping the same this.current to avoid triggering `watch` observer
            Object.keys(current)
            .filter(k => k !== 'key' && k !== 'value')
            .forEach(k => delete current[k]);
            current.value = newText;
          }
        }
      }
    },
  },
};
</script>

<style>
.edit-values {
  &-row {
    border: 1px solid var(--fill-2);
    color: unset;
    text-decoration: none;
    &:not(:first-child) {
      border-top: 0;
    }
    > * {
      font-size: 12px;
      padding: 4px 6px;
      &:first-child {
        position: relative;
        width: 30%;
        max-width: 240px;
      }
      &:not(:first-child) {
        border-left: 1px solid var(--fill-2);
      }
    }
    &:focus,
    &:hover {
      background-color: var(--fill-0-5);
    }
    &:focus {
      text-decoration: underline;
    }
    &:focus .edit-values-btn,
    &:hover .edit-values-btn {
      display: block;
    }
  }
  &-empty {
    color: var(--fill-7);
  }
  &-panel {
    position: absolute;
    top: 0;
    right: 0;
    width: 50%;
    height: 100%;
    padding: 8px;
    box-shadow: -5px 0 5px var(--fill-2);
    background: var(--bg);
    z-index: 10;
    @media (max-width: 767px) {
      width: 100%;
    }
    input {
      width: 100%;
    }
    textarea {
      width: 100%;
      word-break: break-all;
      resize: none;
    }
  }
  &-btn {
    position: absolute;
    top: 0;
    right: 0;
    padding: 4px;
    background: inherit;
    box-shadow: 0 0 5px 5px var(--fill-0-5);
    display: none;
  }
}
</style>
