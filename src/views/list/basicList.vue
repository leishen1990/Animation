<template>
  <Page>
    <Header>
      <Navbar>
        <Title>BasicList</Title>
      </Navbar>
    </Header>
    <Content>
      <List>
        <Item button v-for="item in list">
          {{item}}
        </Item>
      </List>
    </Content>
  </Page>
</template>
<style scoped lang="scss">

</style>
<script type="text/ecmascript-6">
  export default{
    data(){
      return {
        list: [
          'Item 1',
          'Item 2',
          'Item 3',
          'Item 4',
          'Item 5',
          'Item 6',
          'Item 7',
          'Item 8',
          'Item 9',
          'Item 10',
          'Item 11',
          'Item 12',
          'Item 13',
          'Item 14',
        ]
      }
    },
    watch: {},
    computed: {},
    methods: {},
    created() {},
    mounted () {},
    activated () {},
    components: {}
  }
</script>
