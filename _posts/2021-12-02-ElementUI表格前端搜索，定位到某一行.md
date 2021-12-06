---
layout: mypost
title: ElementUI表格前端搜索，定位到某一行
categories: [VUE, ElementUI]
---

## 需求描述
在已知表格数据中，查找到指定用户，并删除该用户

## 核心实现

### 查找到指定用户，并标示出来
这里用到的是table 的 :row-style="TableRowStyle" 属性  
具体实现  
````
        TableRowStyle ({ row, rowIndex }) {
          // 注意，这里返回的是一个对象
          let rowBackground = {}
          if (this.beSearchName !== '' && row.name.includes(this.beSearchName)) {
            rowBackground.background = '#F3E7A5'
            this.tableScrollMove('cityList', rowIndex)
            return rowBackground
          }
        }
````

### 将查找到的行滚动居中

具体实现
````
        tableScrollMove (refName, index = 0) {
          if (!refName || !this.$refs[refName]) return// 不存在表格的ref vm 则返回
          let vmEl = this.$refs[refName].$el
          if (!vmEl) return
          console.log(vmEl.getElementsByTagName('tr')[index + 1])
          vmEl.getElementsByTagName('tr')[index + 1].scrollIntoView({block: 'center'})
        },
````

之前尝试过以下方式， 实际效果不理想
````
const targetTop = vmEl.querySelectorAll('.el-table__body tr')[index].getBoundingClientRect().top
const containerTop = vmEl.querySelector('.el-table__body').getBoundingClientRect().top
const scrollParent = vmEl.querySelector('.el-table__body-wrapper')
scrollParent.scrollTop = targetTop - containerTop

````

实际使用简洁明了
````
   vmEl.getElementsByTagName('tr')[index + 1].scrollIntoView({block: 'center'})
````

## 完整代码
````
<template>
<div>
  <el-input v-model="beSearchName" style="margin-bottom: 10px;" placeholder="输入城市名"></el-input>
  <el-table
    ref="cityList"
    :row-style="TableRowStyle"
    height="500"
    :data="cityList">
    <el-table-column
      prop="code"
      label="代码">
    </el-table-column>
    <el-table-column
      prop="name"
      label="城市名">
    </el-table-column>
    <el-table-column
      prop="operate"
      label="操作">
      <template slot-scope="scope">
        <el-button type="primary" size="mini" @click="deleteUser(scope.row.code)">删除</el-button>
      </template>
    </el-table-column>
  </el-table>
</div>
</template>

<script>
    export default {
      name: 'table-search',
      data () {
        return {
          cityList: [],
          beSearchName: ''
        }
      },
      created () {
        this.beSearchName = ''
        this.cityList = [
          { 'code': '510104', 'name': '锦江区' },
          { 'code': '510105', 'name': '青羊区' },
          { 'code': '510106', 'name': '金牛区' },
          { 'code': '510107', 'name': '武侯区' },
          { 'code': '510108', 'name': '成华区' },
          { 'code': '510112', 'name': '龙泉驿区' },
          { 'code': '510113', 'name': '青白江区' },
          { 'code': '510114', 'name': '新都区' },
          { 'code': '510115', 'name': '温江区' },
          { 'code': '510116', 'name': '双流区' },
          { 'code': '510117', 'name': '郫都区' },
          { 'code': '510121', 'name': '金堂县' },
          { 'code': '510129', 'name': '大邑县' },
          { 'code': '510131', 'name': '蒲江县' },
          { 'code': '510132', 'name': '新津县' },
          { 'code': '510181', 'name': '都江堰市' },
          { 'code': '510182', 'name': '彭州市' },
          { 'code': '510183', 'name': '邛崃市' },
          { 'code': '510184', 'name': '崇州市' },
          { 'code': '510185', 'name': '简阳市' }
        ]
      },
      methods: {
        deleteUser (uniqueId) {
          this.removeUserListByValue(uniqueId)
          // this.$emit('listenUserListChange', this.userList)
        },
        removeUserListByValue (val) {
          for (let i = 0; i < this.cityList.length; i++) {
            if (this.cityList[i].code === val) {
              this.cityList.splice(i, 1)
              break
            }
          }
        },
        tableScrollMove (refName, index = 0) {
          if (!refName || !this.$refs[refName]) return// 不存在表格的ref vm 则返回
          let vmEl = this.$refs[refName].$el
          if (!vmEl) return
          console.log(vmEl.getElementsByTagName('tr')[index + 1])
          vmEl.getElementsByTagName('tr')[index + 1].scrollIntoView({block: 'center'})
        },
        TableRowStyle ({ row, rowIndex }) {
          // 注意，这里返回的是一个对象
          let rowBackground = {}
          if (this.beSearchName !== '' && row.name.includes(this.beSearchName)) {
            rowBackground.background = '#F3E7A5'
            this.tableScrollMove('cityList', rowIndex)
            return rowBackground
          }
        }
        // MergeArray (arr1, arr2) {
        //   let i
        //   const _arr = []
        //   for (i = 0; i < arr1.length; i++) {
        //     _arr.push(arr1[i])
        //   }
        //   for (i = 0; i < arr2.length; i++) {
        //     let flag = true
        //     let uid = arr2[i].trim()
        //     for (let j = 0; j < arr1.length; j++) {
        //       if (uid === arr1[j]) {
        //         flag = false
        //         break
        //       }
        //     }
        //     if (flag) {
        //       _arr.push(uid)
        //     }
        //   }
        //   return _arr
        // }
      }
    }
</script>

<style scoped>

</style>


````

