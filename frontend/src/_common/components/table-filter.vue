<template>
  <div>
    <b-row no-gutters class="justify-content-center justify-content-md-between align-items-center">
      <slot name="actions"></slot>

      <slot name="search">
        <b-input-group v-if="filters.query !== undefined" class="mt-2 mt-md-0 ml-md-auto col-md-4">
          <b-input-group-prepend v-if="moreFilters">
            <b-button :variant="showFilters ? 'success' : 'secondary'" @click.prevent="showFilters = !showFilters">
              <fa :icon="['fas', 'filter']" />
            </b-button>
          </b-input-group-prepend>
          <b-form-input v-model="filters.query" placeholder="Поиск..." @keydown="changeQuery()" />
          <b-input-group-append>
            <b-button :disabled="!filters.query" @click.prevent="filters.query = ''">
              <fa :icon="['fas', 'times']" />
            </b-button>
          </b-input-group-append>
        </b-input-group>
      </slot>
    </b-row>

    <b-alert v-if="moreFilters" v-model="showFilters" class="mt-2" @dismissed="showFilters = false">
      <b-row class="text-center justify-content-around justify-content-md-between">
        <div v-if="filters.date !== undefined">
          <b-form-group label="Дата">
            <input-date v-model="filters.date" :range="true" :hide-buttons="true" />
          </b-form-group>
        </div>

        <div v-if="filters.role_id !== undefined">
          <b-form-group label="Группа юзеров">
            <b-form-select v-model="filters.role_id" :options="roles">
              <template slot="first">
                <option :value="null">Все</option>
              </template>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.course_id !== undefined">
          <b-form-group label="Курс">
            <b-form-select v-model="filters.course_id" :options="courses">
              <template slot="first">
                <option :value="null">Все</option>
              </template>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.service !== undefined">
          <b-form-group label="Оплата">
            <b-form-select v-model="filters.service">
              <option :value="null">Все</option>
              <option value="robokassa">Робокасса</option>
              <option value="paypal">PayPal</option>
              <option value="internal">Бесплатно</option>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.status !== undefined">
          <b-form-group label="Статус заказа">
            <b-form-select v-model="filters.status">
              <option :value="null">Все</option>
              <option :value="1">Новый</option>
              <option :value="2">Оплачен</option>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.confirmed !== undefined">
          <b-form-group label="Email проверен">
            <b-form-select v-model="filters.confirmed">
              <option :value="null">Все</option>
              <option :value="0">Не проверен</option>
              <option :value="1">Проверен</option>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.work_type !== undefined">
          <b-form-group label="Тип работы">
            <b-form-select v-model="filters.work_type">
              <option :value="null">Все</option>
              <option value="home">Домашняя работа</option>
              <option value="work">Обычный урок</option>
            </b-form-select>
          </b-form-group>
        </div>

        <div v-if="filters.active !== undefined">
          <b-form-group label="Статус пользователя">
            <b-form-select v-model="filters.active">
              <option :value="null">Все</option>
              <option :value="0">Неактивные</option>
              <option :value="1">Активные</option>
            </b-form-select>
          </b-form-group>
        </div>
      </b-row>
      <b-row v-if="changedFilters">
        <a href="#" class="ml-auto text-danger" @click.prevent="clearFilters">
          <fa :icon="['fas', 'backspace']" />
          Очистить
        </a>
      </b-row>
    </b-alert>
  </div>
</template>

<script>
export default {
  name: 'TableFilter',
  props: {
    filters: {
      type: Object,
      required: false,
      default() {
        return {
          query: '',
        }
      },
    },
    table: {
      type: String,
      required: true,
    },
    statuses: {
      type: Array,
      required: false,
      default() {
        return []
      },
    },
    visible: {
      type: Boolean,
      required: false,
      default: false,
    },
  },
  data() {
    const formatDate = {
      value2date: (value) => {
        return value ? this.$moment(new Date(value), 'DD.MM.YYYY').toDate() : null
      },
      date2value: (date) => {
        return date ? this.$moment(date).format('YYYY-MM-DD') : null
      },
    }
    return {
      roles: [],
      courses: [],
      showFilters: this.visible === true,
      formatDate,
    }
  },
  computed: {
    moreFilters() {
      const keys = Object.keys(this.filters).filter((item) => {
        return !['query', 'course_id', 'section', 'slider_id'].includes(item)
      })

      return keys.length > 0
    },
    changedFilters() {
      let changed = false
      for (const i in this.filters) {
        if (Object.prototype.hasOwnProperty.call(this.filters, i) && i !== 'query') {
          if (this.filters[i] !== null) {
            changed = true
            break
          }
        }
      }

      return changed
    },
  },
  watch: {
    filters: {
      deep: true,
      handler() {
        this.$root.$emit('app::' + this.table + '::update', this.filters)
      },
    },
  },
  async created() {
    for (const i in this.filters) {
      if (!Object.prototype.hasOwnProperty.call(this.filters, i)) {
        continue
      }

      let action, storage
      const params = {
        limit: 0,
      }

      if (i === 'role_id') {
        action = 'admin/user-roles'
        storage = 'roles'
      } else if (i === 'course_id') {
        action = 'admin/courses'
        storage = 'courses'
      }

      if (action && storage) {
        const {data: res} = await this.$axios.get(action, {params})
        const rows = []
        res.rows.forEach((v) => {
          rows.push({text: v.title, value: v.id})
        })
        this[storage] = rows
      }
    }
  },
  methods: {
    clearFilters() {
      for (const i in this.filters) {
        if (Object.prototype.hasOwnProperty.call(this.filters, i) && i !== 'query') {
          this.filters[i] = null
        }
      }
      this.showFilters = false
    },
    changeQuery() {
      this.$root.$emit('app::' + this.table + '::query', this.filters.query)
    },
    onDateClear() {
      this.filters.date = null
    },
    onDateChange() {
      this.$refs.datepicker.closePopup()
    },
  },
}
</script>

<style lang="scss">
.mx-range-wrapper {
  width: auto;
  display: flex;
  flex-wrap: wrap;
}

.mx-input-wrapper {
  .mx-input-append {
    top: 2px;
    cursor: pointer;
  }

  .mx-clear-wrapper {
    display: block;

    & + .mx-input-append {
      display: none;
    }
  }
}

.checkbox-group {
  legend {
    text-indent: -999999px;
  }

  .form-control {
    background: transparent;
    border-color: transparent;
  }
}
</style>
