<template>
  <!-- Two-way Data-Binding -->
  <section class="editor">
    <quill-editor
      v-if="form.format === 'richtext'"
      v-model="form.body"
      ref="quill"
      :options="options"
      :disabled="disabled"
      placeholder="Content here"
      @change="onEditorChange($event)"
    />

    <b-input v-if="form.format === 'html'"
      @input="onEditorChange"
      v-model="form.body" type="textarea" />

    <br />
    <div>
      <b-field label="Format">
        <div>
          <b-radio v-model="form.radioFormat"
            @input="onChangeFormat" :disabled="disabled" name="format"
            native-value="richtext">Rich text</b-radio>
          <b-radio v-model="form.radioFormat"
            @input="onChangeFormat" :disabled="disabled" name="format"
            native-value="html">Raw HTML</b-radio>
        </div>
      </b-field>
    </div>
  </section>
</template>

<script>
import 'quill/dist/quill.snow.css';
import 'quill/dist/quill.core.css';
import { quillEditor } from 'vue-quill-editor';

export default {
  components: {
    quillEditor,
  },

  props: {
    body: String,
    contentType: String,
    disabled: Boolean,
  },

  data() {
    return {
      form: {
        body: '',
        format: 'richtext',

        // Model bound to the checkboxes. This changes on click of the radio,
        // but is reverted by the change handler if the user cancels the
        // conversion warning. This is used to set the value of form.format
        // that the editor uses to render content.
        radioFormat: 'richtext',
      },

      // Quill editor options.
      options: {
        placeholder: 'Content here',
        modules: {
          toolbar: {
            container: [
              [{ header: [1, 2, 3, false] }],
              ['bold', 'italic', 'underline', 'strike', 'blockquote', 'code'],
              [{ color: [] }, { background: [] }, { size: [] }],
              [
                { list: 'ordered' },
                { list: 'bullet' },
                { indent: '-1' },
                { indent: '+1' },
              ],
              [
                { align: '' },
                { align: 'center' },
                { align: 'right' },
                { align: 'justify' },
              ],
              ['link', 'image'],
              ['clean', 'font'],
            ],
          },
        },
      },
    };
  },

  methods: {
    onChangeFormat(format) {
      this.$buefy.dialog.confirm({
        scroll: 'keep',
        message: 'The content may lose some formatting. Are you sure?',
        onConfirm: () => {
          this.form.format = format;
          this.onEditorChange();
        },
        onCancel: () => {
          // On cancel, undo the radio selection.
          this.form.radioFormat = format === 'richtext' ? 'html' : 'richtext';
        },
      });
    },

    onEditorChange() {
      // The parent's v-model gets { contentType, body }.
      this.$emit('input', { contentType: this.form.format, body: this.form.body });
    },
  },

  watch: {
    // Capture contentType and body passed from the parent as props.
    contentType(f) {
      this.form.format = f;
      this.form.radioFormat = f;
    },

    body(b) {
      this.form.body = b;
    },
  },
};
</script>
