<script lang="ts">
  import { defineComponent, inject } from 'vue'
  import prefixCls from '../prefix'
  export default defineComponent({
    name: 'TabPane',
    props: {
      className: String,
      label: String,
      name: String,
      disabled: Boolean
    },
    setup(props, { slots }) {
      const childrenList: any = inject(`${prefixCls}ChildrenList`)
      // childrenList.push(Object.assign({}, props, { slots: slots }))
      childrenList && childrenList(Object.assign({}, props, { slots: slots }))
    },
    render() {
      return null
    }
  })
</script>
