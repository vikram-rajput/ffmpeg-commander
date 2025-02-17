<template>
  <div class="editor">

    <!-- User can load pre-defined presets and can also access their
    custom saved presets here. -->
    <Presets v-model="preset" />

    <!-- Input and Output protocol and filenames. -->
    <b-form-row>
      <b-col>
        <b-form-group label="Input:" label-for="input">
          <b-input-group>
            <b-form-select
              v-if="!$store.state.ffmpegdEnabled || !$store.state.wsConnected"
              class="protocol"
              v-model="protocolInput"
              @change="setProtocol('input', $event)"
            >
              <option v-for="o in protocols" :key="o.id" :value="o.value">{{o.name}}</option>
            </b-form-select>

            <b-form-input
              v-if="!$store.state.ffmpegdEnabled || !$store.state.wsConnected"
              v-model="form.input"
              :state="Boolean(form.input)"
              placeholder="Example: output.mp4"
            ></b-form-input>

            <b-form-input
              v-else
              v-model="form.input"
              placeholder=""
              @focus="onFileFocus"
            ></b-form-input>

            <div v-if="showFileBrowser">
              <FileBrowser v-on:file="onFileSelect" v-on:close="onClose" />
            </div>

            <!-- <b-form-file
              v-else
              v-model="form.inputFile"
              :state="Boolean(form.inputFile)"
              placeholder="Choose a file or drop it here..."
              drop-placeholder="Drop file here..."
            ></b-form-file> -->
          </b-input-group>
        </b-form-group>
      </b-col>

      <b-col>
        <b-form-group label="Output:" label-for="output">
          <b-input-group>
            <b-form-select
              v-if="!$store.state.ffmpegdEnabled"
              class="protocol"
              v-model="protocolOutput"
              @change="setProtocol('output', $event)"
            >
              <option v-for="o in protocols" :key="o.id" :value="o.value">{{o.name}}</option>
            </b-form-select>

            <b-form-input
              v-model="form.output"
              :state="Boolean(form.output)"
              placeholder="Example: output.mp4"
            ></b-form-input>
          </b-input-group>
        </b-form-group>
      </b-col>
    </b-form-row>

    <!-- Tabs for each command building component.
    Format, Video, Audio, Filters and Options -->
    <b-tabs class="mt-4">
      <b-tab title="Format" class="mt-2">
        <Format v-model="form.format" />
      </b-tab>

      <b-tab title="Video" class="mt-2">
        <Video :container="form.format.container" v-model="form.video" />
      </b-tab>

      <b-tab title="Audio" class="mt-2">
        <Audio :container="form.format.container" v-model="form.audio" />
      </b-tab>

      <b-tab title="Filters" class="mt-2">
        <Filters v-model="form.filters" />
      </b-tab>

      <b-tab title="Options" class="mt-2">
        <Options v-model="form.options" />
      </b-tab>
    </b-tabs>
    <hr />

    <!-- FFmpeg generated command output with tooltips. -->
    <Command :cmd="cmd" />
    <p class="disclaimer">
      *Generated options may vary based on your FFmpeg version and build configuration.
      Tested on version 4.3.1.</p>

    <!-- Hidden textarea so we can use the browser copy function. -->
    <div class="hidden-cmd">
      <b-form-textarea
        ref="code"
        v-model="cmd"
        rows="0"
        max-rows="0"
        plaintext
      ></b-form-textarea>
    </div>

    <!-- Toolbar is the set of controls for copying the command output,
    saving and deleting presets. -->
    <Toolbar
      :preset="preset"
      v-model="controls"
      v-on:reset="reset"
      v-on:save="save"
      v-on:encode="encode"
    />

    <!-- JSON formatted viewer so the user can view or copy the configured
    JSON output for their generated commands. -->
    <JsonViewer :show="controls.showJSON" :form="form" />
  </div>
</template>

<script>
import path from 'path';
import merge from 'lodash.merge';
import clone from 'lodash.clonedeep';
import form from '@/form';
import presets from '@/presets';
import ffmpeg from '@/ffmpeg';
import util from '@/util';
import storage from '@/storage';

import Presets from './Presets.vue';
import Format from './Format.vue';
import Video from './Video.vue';
import Audio from './Audio.vue';
import Filters from './Filters.vue';
import Options from './Options.vue';
import Command from './Command.vue';
import Toolbar from './Toolbar.vue';
import JsonViewer from './JsonViewer.vue';
import FileBrowser from './FileBrowser.vue';

const {
  protocols,
  containers,
  codecs,
} = form;

export default {
  name: 'Editor',
  components: {
    Presets,
    Format,
    Video,
    Audio,
    Filters,
    Options,
    Command,
    Toolbar,
    JsonViewer,
    FileBrowser,
  },
  props: {},
  data() {
    return {
      default: {},
      preset: {
        id: 'custom',
        name: null,
      },
      protocolInput: 'movie.mp4',
      protocolOutput: 'movie.mp4',
      form: {
        input: 'input.mp4',
        inputFile: null,
        output: 'output.mp4',
        format: {
          container: 'mp4',
          clip: false,
          startTime: null,
          stopTime: null,
        },
        video: {
          codec: 'x264',
          preset: 'none',
          pass: '1',
          crf: 23,
          bitrate: null,
          minrate: null,
          maxrate: null,
          bufsize: null,
          gopsize: null,
          pixel_format: 'auto',
          frame_rate: 'auto',
          speed: 'auto',
          tune: 'none',
          profile: 'none',
          level: 'none',
          faststart: false,
          size: 'source',
          width: '1080',
          height: '1920',
          format: 'widescreen',
          aspect: 'auto',
          scaling: 'auto',
          codec_options: '',
        },
        audio: {
          codec: 'copy',
          channel: 'source',
          quality: 'auto',
          sampleRate: 'auto',
          volume: 100,
        },
        filters: {
          deband: false,
          deshake: false,
          deflicker: false,
          dejudder: false,
          denoise: 'none',
          deinterlace: 'none',
          brightness: 0,
          contrast: 0,
          saturation: 0,
          gamma: 0,

          acontrast: 33,
        },
        options: {
          extra: [],
          loglevel: 'none',
        },
      },
      protocols,
      containers,
      codecs,
      cmd: null,
      controls: {
        showJSON: false,
      },
      showFileBrowser: false,
    };
  },
  created() {
    this.generateCommand();
    this.default = clone(this.form); // Make copy of initial form as defaults.

    this.setDataFromQueryParams();
  },
  watch: {
    form: {
      handler() {
        // Update the output filename.
        this.updateOutput();

        // Generates the FFmpeg command.
        this.generateCommand();

        // Update all non-default form options to query parameters.
        this.updateQueryParams();
      },
      deep: true,
    },
    preset: {
      handler() {
        if (this.preset.id.startsWith('preset-')) {
          const preset = presets.getPresetFromLocalStorage(this.preset.id);
          this.form = merge(this.form, preset.data);
          this.preset.name = preset.name;
        } else if (this.preset.id !== 'custom' && !this.preset.id.startsWith('preset-')) {
          this.setPreset(this.preset.id);
          this.preset.name = null;
        }
      },
    },
  },
  methods: {
    setProtocol(type, value) {
      if (type === 'input') {
        this.form.input = value;
      } else if (type === 'output') {
        this.form.output = value;
      }
    },
    setPreset(value) {
      this.reset();
      const preset = presets.getPreset(value);
      this.form = merge(this.form, preset);
      this.preset.id = value;
    },
    generateCommand() {
      const opt = util.transform(this.form);
      this.cmd = ffmpeg.build(opt);
    },
    updateOutput() {
      if (this.form.output) {
        const { format, output } = this.form;
        const ext = path.extname(output);
        if (ext) {
          this.form.output = `${output.replace(ext, `.${format.container}`)}`;
        }
      }
    },
    setDataFromQueryParams() {
      const { query } = this.$route;
      util.transformFromQueryParams(this.form, query);
    },
    updateQueryParams() {
      const params = util.transformToQueryParams(this.form);
      this.$router.push({ query: params }).catch(() => {});
    },
    update(key, value) {
      this.$emit('input', { ...this.value, [key]: value });
    },
    reset() {
      // Restore form from default copy.
      this.form = merge(this.form, this.default);
      this.preset.id = 'custom';
      this.preset.name = null;
    },
    save(saveNew = false) {
      if (saveNew) {
        this.preset.name = null;
      }

      // Save the preset name and reload the presets list.
      const presetName = presets.savePresetToLocalStorage(
        this.preset.id, this.preset.name, this.form,
      );
      this.presets = presets.getPresetOptions();
      this.preset.id = presetName;
      this.preset.name = this.preset.name || this.preset.id;
    },
    encode() {
      const { input, inputFile, output } = this.form;
      const json = util.transformToJSON(this.form);
      storage.add('queue', {
        id: Date.now(),
        type: 'encode',
        payload: json,
        status: 'queued',
        input: inputFile ? inputFile.name : input,
        output,
        _showDetails: false,
      });
      this.$emit('onEncode');
    },
    onFileSelect(file) {
      this.form.input = file;
      this.showFileBrowser = false;
    },
    onFileFocus() {
      this.showFileBrowser = true;
    },
    onClose() {
      this.showFileBrowser = false;
    },
  },
};
</script>

<style scoped>
.disclaimer {
  font-style: italic;
  font-size: 0.8em;
  padding-top: 6px;
}
.protocol {
  flex: 0 0 20% !important;
}
.hidden-cmd {
  opacity: 0;
  height: 0;
}
.hidden-cmd textarea {
  height: 0;
}
</style>
