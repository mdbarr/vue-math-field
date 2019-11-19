<template>
<v-text-field
  v-bind="$attrs"
  v-model="current"
  @keydown="keydown"
  @focus="focus"
  @blur="blur"
  @click:clear="clear"
  :error="error"
  :error-messages="message"
  :class="textColor"
  :success="!error && mode === 'display'"
  ref="vtf"
  />
</template>

<script>
const math = require('mathjs');

math.createUnit('voxel', {
  definition: '7.5 cm', aliases: [ 'voxel', 'voxels' ]
});

export default {
  name: 'v-math-field',
  props: {
    value: {
      type: [ Number, String ],
      default: 0
    },
    units: {
      type: String,
      default: 'meters'
    },
    displayPrecision: {
      type: Number,
      default: 2
    },
    precision: {
      type: Number,
      default: 4
    },
    notation: {
      type: String,
      default: 'fixed'
    },
    numeric: {
      type: Boolean,
      default: true
    },
    enter: {
      type: String,
      default: 'render'
    }
  },
  data: () => {
    return {
      current: '',
      mode: 'display',
      raw: '',
      pretty: '',
      error: false,
      message: ''
    };
  },
  created () {
    this.mode = 'display';
    this.raw = this.value.toString();
    this.evaluate(true);
    this.current = this.pretty;
  },
  computed: { textColor () {
    return this.mode === 'display' ? 'blue--text' : 'black--text';
  } },
  methods: {
    round (number) {
      const factor = Math.pow(10, this.precision);
      return Math.round(number * factor) / factor;
    },
    focus (...args) {
      this.mode = 'edit';
      this.current = this.raw;
      this.$emit('focus', ...args);
    },
    blur (...args) {
      if (this.error === false) {
        if (this.mode === 'edit') {
          this.raw = this.current;
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
      } else if (event.keyCode === 13) {
        if (this.mode === 'display') {
          event.preventDefault();
          event.stopPropagation();
        } else {
          this.raw = this.current;
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
    },
    clear () {
      this.current = '';
      this.raw = '';
      this.pretty = '';
      if (this.numeric) {
        this.$emit('input', 0);
      } else {
        this.$emit('input', '');
      }
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
          this.message = '';

          if (this.numeric) {
            const number = this.round(value.toNumber(this.units));
            this.$emit('input', number);
          } else {
            this.$emit('input', this.pretty);
          }
        } catch (error) {
          this.error = true;
          this.message = error.message;
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
