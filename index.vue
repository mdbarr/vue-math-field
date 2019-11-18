<template>
<v-text-field
  v-bind="$attrs"
  v-model="current"
  @change="evaluate"
  @focus="focus"
  @blur="blur"
  :error="error"
  :error-messages="message"
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
      default: 'voxels'
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
    this.evaluate(this.value.toString());
    this.current = this.pretty;
  },
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
        this.mode = 'display';
        this.current = this.pretty;
        this.$emit('blur', ...args);
      }
    },
    evaluate (value) {
      if (value.length) {
        this.raw = value;
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
    update () {
      if (this.raw) {
        this.evaluate(this.raw);
        if (this.mode === 'display') {
          this.current = this.pretty;
        } else {
          this.current = this.raw;
        }
      }
    }
  },
  watch: {
    precision () {
      this.update();
    },
    displayPrecision () {
      this.update();
    },
    numeric () {
      this.update();
    }
  }
};
</script>
