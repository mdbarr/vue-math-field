<template>
<v-text-field
  :class="textColor"
  :error-messages="message"
  :error="error"
  :success="!error && mode === 'display'"
  @blur="blur"
  @click:append-outer="$emit('click:append-outer', $event)"
  @click:append="$emit('click:append', $event)"
  @click:clear="clear"
  @click:prepend-inner="$emit('click:prepend-inner', $event)"
  @click:prepend="$emit('click:prepend', $event)"
  @click="click"
  @focus="focus"
  @keydown="keydown"
  ref="vtf"
  v-bind="attributes"
  v-model="current"
  >
  <slot v-for="(_, name) in $slots" :name="name" :slot="name" />
  <template v-for="(_, name) in $scopedSlots" :slot="name" slot-scope="slotData">
    <slot :name="name" v-bind="slotData" />
  </template>
</v-text-field>
</template>

<script>
const math = require('mathjs');

math.createUnit('voxel', {
  definition: '7.5 cm', aliases: [ 'voxel', 'voxels' ]
});

export default {
  name: 'v-math-field',
  props: {
    addUnits: {
      type: Boolean,
      default: true
    },
    autoSelect: {
      type: Boolean,
      default: true
    },
    displayPrecision: {
      type: Number,
      default: 3
    },
    enter: {
      type: String,
      default: 'render'
    },
    notation: {
      type: String,
      default: 'fixed'
    },
    numeric: {
      type: Boolean,
      default: true
    },
    precision: {
      type: Number,
      default: 8
    },
    rules: {
      type: Array,
      default: () => { return []; }
    },
    type: {
      type: String,
      default: 'number'
    },
    units: {
      type: String,
      default: 'm'
    },
    uuid: { type: String },
    value: {
      type: [ Number, String ],
      default: 0
    }
  },
  data: () => {
    return {
      current: '',
      mode: 'display',
      raw: '',
      pretty: '',
      error: false,
      errored: '',
      message: ''
    };
  },
  created () {
    this.mode = 'display';
    if (this.uuid && localStorage.getItem(this.uuid)) {
      this.raw = localStorage.getItem(this.uuid);
    } else {
      this.raw = this.value.toString();
    }

    if (this.addUnits && Number(this.raw).toString() === this.raw) {
      this.raw = `${ this.raw } ${ this.units }`;
    }

    this.attributes = {};
    for (const attr in this.$attrs) {
      if (attr !== 'type' && attr !== 'rules') {
        this.attributes[attr] = this.$attrs[attr];
      }
    }

    this.evaluate(true);
    this.current = this.pretty;
  },
  computed: { textColor () {
    if (this.$attrs.class) {
      return this.$attrs.class;
    }
    return this.mode === 'display' ? 'blue--text' : 'black--text';
  } },
  methods: {
    round (number) {
      const factor = Math.pow(10, this.precision);
      return Math.round(number * factor) / factor;
    },
    save (value) {
      this.raw = value;
      if (this.uuid) {
        localStorage.setItem(this.uuid, this.raw.toString());
      }
      return this.raw;
    },
    select () {
      this.$nextTick(() => {
        this.$refs.vtf.$refs.input.select();
      });
    },
    focus (...args) {
      this.mode = 'edit';
      this.current = this.raw;
      this.$emit('focus', ...args);
      if (this.autoSelect) {
        this.select();
      }
    },
    click () {
      if (this.mode === 'display') {
        this.mode = 'edit';
        this.current = this.raw;
        this.update();
      }
    },
    blur (...args) {
      if (this.error === false || (this.errored !== this.current && this.mode === 'edit')) {
        if (this.mode === 'edit') {
          this.save(this.current);

          this.evaluate();
          if (!this.error) {
            this.mode = 'display';
            this.current = this.pretty;
          }
        }
        this.$emit('blur', ...args);
      }
    },
    keydown (event) {
      if (this.mode === 'display' && this.enter === 'render' &&
          (event.keyCode !== 9 && event.keyCode !== 13)) {
        event.preventDefault();
        event.stopPropagation();
        this.mode = 'edit';
        this.update();
        if (this.autoSelect) {
          this.select();
        }
      } else if (event.keyCode === 13) {
        if (this.mode === 'display') {
          event.preventDefault();
          event.stopPropagation();
        } else {
          this.save(this.current);
          this.evaluate();

          if (this.enter === 'blur' && !this.error) {
            this.$refs.vtf.blur();
          } else if (this.enter === 'render' && !this.error) {
            this.mode = 'display';
            this.update();

            event.preventDefault();
            event.stopPropagation();
          } else {
            this.update();
          }
        }
      } else if (event.keyCode === 27) {
        if (this.mode === 'edit') {
          this.current = this.raw;
        } else {
          this.current = this.pretty;
        }
      }
      this.$emit('keydown', event);
    },
    clear (event) {
      this.current = '';
      this.save(this.current);
      this.pretty = '';
      if (this.numeric || this.type === 'number') {
        this.$emit('input', 0);
        this.$emit('change', 0);
      } else {
        this.$emit('input', '');
        this.$emit('change', '');
      }
      this.$emit('click:clear', event);
    },
    evaluate (force = false) {
      let value = this.raw;

      if (value.length) {
        if (value === this.pretty && !force) {
          return;
        }

        try {
          value = math.evaluate(value);
          if (typeof value === 'number') {
            value = math.unit(value, this.units);
          } else {
            value = value.to(this.units);
          }

          this.pretty = math.format(value, {
            notation: this.notation,
            precision: this.displayPrecision
          });

          this.error = false;
          this.errored = '';
          this.message = '';

          const number = this.round(value.toNumber(this.units));
          const returnValue = (this.numeric || this.type === 'number') ? number : this.pretty;

          for (const rule of this.rules) {
            const valid = rule(returnValue);
            if (valid !== true) {
              this.error = true;
              this.errored = this.raw;
              if (typeof valid === 'string') {
                this.message = valid;
              }
              break;
            }
          }

          if (this.error) {
            this.$emit('update:error', this.error);
          } else {
            this.$emit('input', returnValue);
            this.$emit('change', returnValue);
          }
        } catch (error) {
          this.error = true;
          this.errored = this.raw;
          this.message = error.message;
          this.$emit('update:error', this.error);
        }
      }
    },
    update (recompute = false) {
      if (recompute) {
        this.evaluate(true);
      }

      if (this.mode === 'display') {
        this.current = this.pretty;
      } else {
        this.current = this.raw;
      }
    }
  },
  watch: {
    value (newValue) {
      this.raw = newValue.toString();
      this.update(true);
    },
    precision () {
      this.update(true);
    },
    displayPrecision () {
      this.update(true);
    },
    numeric () {
      this.update(true);
    },
    units () {
      this.update(true);
    }
  }
};
</script>
