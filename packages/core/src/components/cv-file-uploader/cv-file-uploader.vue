<template>
  <cv-form-item class="cv-file-uploader">
    <p :class="`${carbonPrefix}--file--label`">{{ label }}</p>
    <p :class="`${carbonPrefix}--label-description`">{{ helperText }}</p>
    <div :class="`${carbonPrefix}--file`" data-file>
      <label
        :for="uid"
        :class="[
          {
            [`${carbonPrefix}--file-browse-btn`]: kind !== 'button',
            [`${carbonPrefix}--btn`]: kind === 'button',
            [`${carbonPrefix}--btn--primary`]: kind === 'button',
          },
        ]"
        role="button"
        tabindex="0"
        ref="focusTarget"
        @keydown.enter.prevent="onShow()"
        @keydown.space.prevent
        @keyup.space.prevent="onShow()"
      >
        <cv-wrapper
          :tag-type="kind !== 'button' ? 'div' : ''"
          data-file-drop-container
          :class="`${carbonPrefix}--file__drop-container${allowDropClass}`"
          @dragover="onDragEvent"
          @dragleave="onDragEvent"
          @drop="onDragEvent"
        >
          <slot name="drop-target">{{ internalDropTargetLabel }}</slot>
          <input
            v-if="kind !== 'button'"
            v-bind="$attrs"
            type="file"
            :accept="accept"
            :class="`${carbonPrefix}--file-input`"
            :id="uid"
            data-file-uploader
            data-target="[data-file-container]"
            v-on="inputListeners"
            ref="file-input"
          />
        </cv-wrapper>
      </label>
      <input
        v-if="kind === 'button'"
        v-bind="$attrs"
        type="file"
        :accept="accept"
        :class="`${carbonPrefix}--file-input`"
        :id="uid"
        data-file-uploader
        data-target="[data-file-container]"
        v-on="inputListeners"
        ref="file-input"
      />

      <div data-file-container :class="`${carbonPrefix}--file-container`">
        <div
          v-for="(file, index) in internalFiles"
          :key="index"
          :class="
            isInvalid(index)
              ? `${carbonPrefix}--file__selected-file--invalid__wrapper`
              : `${carbonPrefix}--file__selected-file`
          "
        >
          <cv-wrapper
            :tag-type="isInvalid(index) ? 'div' : ''"
            :class="`${carbonPrefix}--file__selected-file ${carbonPrefix}--file__selected-file--invalid`"
          >
            <p :class="`${carbonPrefix}--file-filename`" :title="file.file.name">{{ file.file.name }}</p>

            <span
              :data-for="uid"
              :class="`${carbonPrefix}--file__state-container`"
              :data-test="file.state"
              :style="stateStyleOverides"
            >
              <div v-if="file.state === 'uploading'" :class="`${carbonPrefix}--inline-loading__animation`">
                <div data-inline-loading-spinner :class="`${carbonPrefix}--loading ${carbonPrefix}--loading--small`">
                  <svg :class="`${carbonPrefix}--loading__svg`" viewBox="-75 -75 150 150">
                    <circle :class="`${carbonPrefix}--loading__background`" cx="0" cy="0" r="37.5" />
                    <circle :class="`${carbonPrefix}--loading__stroke`" cx="0" cy="0" r="37.5" />
                  </svg>
                </div>
              </div>
              <CheckmarkFilled16 v-if="file.state === 'complete'" :class="`${carbonPrefix}--file-complete`" />
              <WarningFilled16 v-if="isInvalid(index)" :class="`${carbonPrefix}--file--invalid`" />
              <button
                type="button"
                :class="`${carbonPrefix}--file-close`"
                v-if="removable"
                :alt="removeAriaLabel"
                :arial-label="removeAriaLabel"
                @click="remove(index)"
              >
                <Close16 />
              </button>
            </span>
            <div v-if="isInvalid(index)" :class="`${carbonPrefix}--form-requirement`">
              <div :class="`${carbonPrefix}--form-requirement__title`">
                {{ file.invalidMessageTitle || 'Invalid file' }}
              </div>
              <p :class="`${carbonPrefix}--form-requirement__supplement`">{{ file.invalidMessage }}</p>
            </div>
          </cv-wrapper>
        </div>
      </div>
    </div>
  </cv-form-item>
</template>

<script>
import { uidMixin, carbonPrefixMixin, methodsMixin } from '../../mixins';
import CvFormItem from '../cv-form/cv-form-item';
import CheckmarkFilled16 from '@carbon/icons-vue/es/checkmark--filled/16';
import WarningFilled16 from '@carbon/icons-vue/es/warning--filled/16';
import Close16 from '@carbon/icons-vue/es/close/16';
import CvWrapper from '../cv-wrapper/_cv-wrapper';
import { STATES, KINDS } from './consts.js';

export default {
  name: 'CvFileUploader',
  components: { CvFormItem, CheckmarkFilled16, WarningFilled16, Close16, CvWrapper },
  mixins: [uidMixin, carbonPrefixMixin, methodsMixin({ focusTarget: ['blur', 'focus'] })],
  inheritAttrs: false,
  props: {
    clearOnReselect: Boolean,
    files: Array,
    kind: {
      type: String,
      default: 'drag-target',
      validator: val => {
        const validValues = Object.values(KINDS);

        if (!validValues.includes(val)) {
          console.warn(`CvFileUploader: valid values for 'kind' are ${validValues}`);
        }
        return true;
      },
    },
    label: String,
    helperText: String,
    accept: String,
    initialStateUploading: Boolean,
    removable: Boolean,
    buttonLabel: {
      type: String,
      default: undefined,
      validator: val => {
        if (val !== undefined && process.env.NODE_ENV === 'development') {
          console.warn('CvFileUploader: button-label prop deprecated in favour of drop-target-label');
        }
        return true;
      },
    },
    dropTargetLabel: { type: String, default: undefined },
    removeAriaLabel: { type: String, default: 'Remove selected file' },
  },
  model: {
    prop: 'files',
    event: 'change',
  },
  created() {
    this.STATES = Object.freeze(STATES);
  },
  data() {
    return {
      internalFiles: [],
      allowDrop: false,
    };
  },
  mounted() {
    this.internalFiles = this.files ? this.files : [];
  },
  watch: {
    files() {
      this.internalFiles = this.files ? this.files : [];
    },
  },
  computed: {
    // Bind listeners at the component level to the embedded input element and
    // add our own input listener to service the v-model. See:
    // https://vuejs.org/v2/guide/components-custom-events.html#Customizing-Component-v-model
    inputListeners() {
      return {
        ...this.$listeners,
        change: event => this.onChange(event),
      };
    },
    isInvalid() {
      return index => {
        const result = this.internalFiles[index].invalidMessage && this.internalFiles[index].invalidMessage.length;
        return result;
      };
    },
    internalDropTargetLabel() {
      return this.dropTargetLabel || this.buttonLabel || 'Drag and drop files here or upload';
    },
    stateStyleOverides() {
      // <style carbon tweaks - DO NOT USE STYLE TAG as it causes SSR issues
      return { display: 'inline-flex', alignItems: 'center' };
    },
    allowDropClass() {
      return this.allowDrop ? ` ${this.carbonPrefix}--file__drop-container--drag-over` : '';
    },
  },
  methods: {
    remove(index) {
      this.internalFiles.splice(index, 1);
      this.$emit('change', this.internalFiles);
    },
    addFiles(files) {
      for (const file of files) {
        const internalFile = {
          state: this.initialStateUploading ? STATES.UPLOADING : STATES.NONE,
          file,
          invalidMessageTitle: '',
          invalidMessage: '',
        };

        if (file.messageBundle) {
          internalFile.invalidMessageTitle = file.messageBundle.invalidMessageTitle;
          internalFile.invalidMessage = file.messageBundle.invalidMessage;
          delete file.messageBundle;
        }
        this.internalFiles.push(internalFile);
      }
      this.$emit('change', this.internalFiles);
    },
    onChange(ev) {
      if (ev.target.files.length !== 0 && this.clearOnReselect) {
        this.internalFiles = [];
      }
      this.addFiles(ev.target.files);
    },
    onShow() {
      this.$refs['file-input'].click();
    },
    setState(index, state) {
      if (['uploading', 'complete', ''].includes(state)) {
        this.internalFiles[index].state = state;
      }
    },
    clear() {
      this.internalFiles = [];
      this.$emit('change', this.internalFiles);
    },
    setInvalidMessage(index, message) {
      this.internalFiles[index].invalidMessage = message;
    },
    onDragEvent(evt) {
      if (Array.prototype.indexOf.call(evt.dataTransfer.types, 'Files') === -1) {
        return;
      }

      if (evt.type === 'dragleave') {
        this.allowDrop = false;
        return;
      }

      if (evt.type === 'dragover') {
        evt.preventDefault();
        const dropEffect = 'copy';
        if (Array.isArray(evt.dataTransfer.types)) {
          try {
            // IE11 throws a "permission denied" error accessing `.effectAllowed`
            evt.dataTransfer.effectAllowed = dropEffect;
          } catch (e) {
            // ignore
          }
        }
        evt.dataTransfer.dropEffect = dropEffect;
        this.allowDrop = true;
      }

      if (evt.type === 'drop') {
        evt.preventDefault();
        if (this.accept) {
          this.markInvalidFileTypes(evt.dataTransfer.files);
        }
        this.addFiles(evt.dataTransfer.files);
        this.allowDrop = false;
      }
    },
    markInvalidFileTypes(files) {
      const isFileAllowed = file => {
        const extension = '.' + file.name.split('.').pop();
        const allowed = this.accept.split(',').map(x => x.trim());
        if (allowed.includes('.jpg')) {
          allowed.push('.jpeg');
        }
        return allowed.includes(file.type) || allowed.includes(extension);
      };

      for (const file of files) {
        if (!isFileAllowed(file)) {
          file.messageBundle = {
            invalidMessageTitle: 'Invalid file type',
            invalidMessage: `"${file.name}" does not have a valid file type.`,
          };
        }
      }
    },
  },
};
</script>
