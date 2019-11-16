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
      type: Number,
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
      raw: '',
      pretty: '',
      error: false,
      message: ''
    };
  },
  created () {
    this.evaluate(this.value.toString());
    this.current = this.pretty;
  },
  methods: {
    round (number) {
      const factor = Math.pow(10, this.precision);
      return Math.round(number * factor) / factor;
    },
    focus () {
      this.current = this.raw;
    },
    blur () {
      if (this.error === false) {
        this.current = this.pretty;
      }
    },
    enter () {
      this.$refs.vtf.validate();
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
    }
  }
};
</script>
