<template>
  <div class="m__e-fixed-table">
    <table class="m__e-table">
      <tr class="m__e-table-row">
        <!-- <MTableColumn tag="th" className="col-fixed-left">
          <MCheckBox id="check-all" @checkboxSelected="checkboxSelected" :checked="checkedAll" />
        </MTableColumn> -->
        <MTableColumn
          tag="th"
          textAlign="center"
          width="40px"
          padding="unset"
          className="col-fixed-left"
        >
          STT
        </MTableColumn>
        <MTableColumn
          tag="th"
          :width="column.width"
          :textAlign="column.textAlign"
          v-for="(column, index) in columns"
          :key="index"
          >{{ column.title }}</MTableColumn
        >
        <!-- <MTableColumn tag="th" className="col-fixed-right" v-if="isShowColumnAction"
          >Chức năng</MTableColumn
        > -->
      </tr>
      <tr
        class="m__e-table-row"
        v-for="(row, index) in rows"
        :key="index"
        :class="{ active: rowsSelected.includes(row) }"
      >
        <!-- <MTableColumn className="col-fixed-left">
          <MCheckBox
            :id="row[`${tableName}Id`]"
            :checked="rowsSelected.includes(row[`${tableName}Id`])"
            @checkboxSelected="checkboxSelected"
            :disabled="!row.IsActive"
          />
        </MTableColumn> -->
        <MTableColumn tag="td" textAlign="center" className="col-fixed-left">
          {{ index + 1 }}
        </MTableColumn>
        <MTableColumn
          v-for="(column, index) in columns"
          :key="index"
          :textAlign="column.textAlign"
          @dblclick="row.IsActive && updateRows(row[`${tableName}Id`])"
          :className="row.IsActive == false ? 'locked-row' : ''"
        >
          <status-order
            v-model="row[column.name]"
            :isPaid="row.IsPaid"
            :orderId="row.OrderId"
            :dataList="statusReviewList"
            propValue="Status"
            propName="Title"
            :rowData="row"
            v-if="column.type == 'statusBook'"
            @selectedItem="(value, data) => $emit('selectedItem', value, data, column.type)"
          />
          <status-order
            v-model="row[column.name]"
            :isPaid="row.IsPaid"
            :orderId="row.OrderId"
            :dataList="statusReviewList"
            propValue="Status"
            propName="Title"
            :rowData="row"
            v-else-if="column.type == 'statusReview'"
            @selectedItem="(value, data) => $emit('selectedItem', value, data, column.type)"
          />
          <div v-else-if="column.type == 'image'">
            <img class="img-column_table" :src="row[column.name]" alt="" />
          </div>
          <div v-else-if="column.type == 'review-start'">
            <m-start-rate :startRate="row[column.name]" justifyContent="center" />
          </div>
          <div v-else-if="column.type == 'description'">
            <div v-html="row[column.name]"></div>
          </div>
          <div :class="addClass(column.type, row[column.name])" v-else>
            {{ formatColumn(column, row[column.name]) }}
          </div>
        </MTableColumn>
        <!-- <MTableColumn className="col-fixed-right col-center" v-if="isShowColumnAction">
          <div class="m__e-table-col-function-btn" ref="btnFunctionMenu">
            <span @click.prevent="actionRow(row)">{{ getActionTile }}</span>
            <div
              v-if="$state.nameTable != 'Order'"
              class="m__e-table-col-icon"
              v-click-outside="clickOutSideFunction"
              ref="btnFunction"
              :class="id === row[`${tableName}Id`] && isShowfunction ? 'active' : ''"
              @click="row.IsActive && showFunctionId($event, row[`${tableName}Id`])"
            >
              <div
                class="icon-drop-page-blue"
                v-if="this.$state.mode !== enumD.enumMode.view"
              ></div>
            </div>
          </div>
        </MTableColumn> -->
      </tr>
    </table>
    <div class="m__e-list-empty" v-if="rows.length == 0">
      src="@/assets/img/bg_report_nodata.76e50bd8.svg" alt="Không có dữ liệu" />
      <span>Không có dữ liệu</span>
    </div>
    <div
      class="m__e-table-col-function"
      :style="{
        top: `${functionTop}px `,
        left: `${functionLeft}px `
      }"
      v-show="isShowfunction"
      ref="functionContextMenu"
    >
      <li @click="confirmDeleteRow">Xóa</li>
      <li data-tip="Tính năng chưa phát triển">Ngừng sử dụng</li>
    </div>
  </div>
  <!-- Pop-up thông báo khi xóa -->
  <MPopUpWarn :isShow="isPopUpDelete" @close-pop-up="isPopUpDelete = false" width="450px">
    {{ `Bạn có thực sự muốn xóa không ?` }}
    <template #footer>
      <div class="warning__delete-btn">
        <div @click="isPopUpDelete = false">
          <MButton typeBtn="close">Không</MButton>
        </div>
        <div @click="deleteRow()">
          <MButton typeBtn="delete">Có</MButton>
        </div>
      </div>
    </template>
  </MPopUpWarn>
</template>
<script>
import common from '@/common/common'
import MButton from '../button/MButton.vue'
import MPopUpWarn from '../pop-up/MPopUpWarn.vue'
import MTableColumn from '../table-column/MTableColumn.vue'
import baseApi from '@/api/baseApi'
import enumD from '@/common/enum'
import StatusOrder from '../order/StatusOrder.vue'
import MStartRate from '../start/MStartRate.vue'
export default {
  name: 'MTable',
  emits: ['update:modelValue', 'delete'],
  components: {
    MPopUpWarn,
    MButton,
    MTableColumn,
    StatusOrder,
    MStartRate
  },
  props: {
    columns: Object,
    rows: Object,
    tableName: String,
    modelValue: Array,
    isLoadding: Boolean
  },
  created() {},
  data() {
    return {
      rowsSelected: [],
      id: undefined,
      isShowfunction: false,
      heightFunction: 0, // Chiều cao của list chức năng
      functionTop: 0, // Tọa dộ top của list chức năng
      functionLeft: 0, // Tọa dộ left của list chức năng
      isPopUpDelete: false,
      checkedAll: false,
      enumD: enumD,
      statusList: [
        {
          Status: 1,
          Title: 'Muốn đọc'
        },
        {
          Status: 2,
          Title: 'Đang đọc'
        },
        {
          Status: 3,
          Title: 'Đã đọc'
        }
      ],
      statusReviewList: [
        {
          Status: 1,
          Title: 'Chờ duyệt'
        },
        {
          Status: 2,
          Title: 'Duyệt'
        },
        {
          Status: 3,
          Title: 'Từ chối'
        }
      ]
    }
  },
  computed: {
    getActionTile() {
      if (this.$state.nameTable == 'Product') {
        return 'Thêm'
      } else if (this.$state.nameTable == 'Order') {
        return 'Xem chi tiết'
      } else {
        return 'Sửa'
      }
    },
    isShowColumnAction() {
      // if (
      //   this.$state.mode !== enumD.enumMode.view &&
      //   this.$state.tableName != "Order"
      // ) {
      //   return false;
      // }
      return this.$state.mode !== enumD.enumMode.view
    }
  },
  methods: {
    addClass(type, value) {
      if (type == 'paid-type') {
        return value != 0 ? 'paid' : 'unpaid'
      }
      return ''
    },
    /**
     * Hiển thị list chức năng của 1 dòng
     
     */
    showFunctionId(event, id) {
      if (this.id == id) {
        this.isShowfunction = false
        this.id = null
      } else {
        this.id = id
        this.isShowfunction = true
        if (window.innerHeight - this.heightFunction * 1.5 < event.clientY) {
          this.functionTop = event.clientY - 15 - this.heightFunction
          this.functionLeft = event.clientX - 100
        } else {
          this.functionTop = event.clientY + 15
          this.functionLeft = event.clientX - 100
        }
      }
    },
    /**
     * Xử lý click out side cho list action
     */
    clickOutSideFunction(e) {
      const check = this.$refs.btnFunction.filter((x) => x.contains(e.target))
      if (
        !this.$refs.functionContextMenu.contains(e.target) &&
        check.length <= 0 &&
        this.isShowfunction
      ) {
        this.isShowfunction = false
        this.itemId = null
      }
    },
    /**
     * Hàm xử lý checkbox
     
     */
    checkboxSelected(isChecked, id) {
      try {
        // Check all
        if (id === 'check-all') {
          if (!isChecked) {
            // Kiểm tra khi bỏ check all
            this.rows
              .map((x) => x[`${this.tableName}Id`])
              .forEach((e) => {
                this.rowsSelected = this.rowsSelected.filter((x) => x != e)
              })
            this.checkedAll = false
          } else {
            // Kiểm tra khi check all
            this.rowsSelected = this.rowsSelected.concat(
              this.rows
                .map((x) => x[`${this.tableName}Id`])
                .filter((x) => !this.rowsSelected.includes(x))
            )
            this.checkedAll = true
          }
        } else {
          // Kiểm tra các item check
          if (!isChecked) {
            this.rowsSelected = this.rowsSelected.filter((x) => x !== id)
          } else {
            this.rowsSelected.push(id)
          }
          this.testCheckAll()
        }
        this.$emit('update:modelValue', this.rowsSelected)
      } catch (error) {
        console.log(error)
      }
    },
    /**
     * Hàm check tổng số phần tử trùng nhau của 2 mang
     
     */
    testCheckAll() {
      // Kiểm tra coi có check all hay không
      var count = 0
      // Đếm số phần tử check của page
      this.rows
        .map((x) => x[`${this.tableName}Id`])
        .forEach((e) => {
          if (this.rowsSelected.filter((x) => x == e).length > 0) count++
        })

      // Kiểm tra để check
      if (this.rows.length > 0 && count == this.rows.length) {
        this.checkedAll = true
      } else if (count < this.rows.length) {
        this.checkedAll = false
      }
    },
    /**
     * Hiển thị pop-up xác nhận xóa
     
     */
    confirmDeleteRow() {
      this.isPopUpDelete = true
      this.isShowfunction = false
    },
    /**
     * Hiển thị pop-up xác nhận xóa
     
     */
    deleteRow() {
      this.isPopUpDelete = false
      this.$emit('delete', [this.id])
    },
    updateRows(id) {
      this.$state.idModel = id
      this.$state.isShowForm = true
    },
    actionRow(row) {
      if (this.$state.form == enumD.formName.product) {
        localStorage.setItem('parentName', row?.ProductName)
        this.$router.push('/products/' + row.ProductId)
      } else if (this.$state.form == enumD.formName.order) {
        this.$router.push('/orderdetails/' + row?.OrderId)
      } else {
        this.updateRows(row[`${this.$state.tableName}Id`])
      }
    },
    formatColumn(column, value) {
      if (column.type == 'date') {
        return common.formatDate(value)
      } else if (column.type == 'Price') {
        return this.$state.formatPrice(value)
      } else if (column.type == 'StatusOrder') {
        return this.getTitleStatusOrder(value)
      } else if (column.type == 'paid-type') {
        return this.getTitlePaid(value)
      }
      return value
    },
    async lockUpRow(row) {
      try {
        this.$state.isMask()
        const api = new baseApi(this.$state.tableName)
        await api.lockup([row[`${this.$state.tableName}Id`]])
        this.$state.unMask()
        this.$emit('refresh')
      } catch (error) {
        this.$state.unMask()
      }
    },
    getTitleStatusOrder(status) {
      return common.getTitleStatusOrder(status)
    },
    getTitlePaid(paid) {
      if (paid) {
        return 'Đã thanh toán'
      } else {
        return 'Chưa thanh toán'
      }
    }
  },
  watch: {
    modelValue: function () {
      if (this.modelValue.length == 0) {
        this.rowsSelected = []
        this.checkedAll = false
      }
    }
  }
}
</script>

<style scoped>
.locked-row {
  cursor: not-allowed !important ;
}
.cuser-pointer {
  cursor: pointer;
}
.img-column_table {
  height: 100px;
}
</style>
