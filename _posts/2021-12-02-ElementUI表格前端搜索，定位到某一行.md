---
layout: mypost
title: ElementUI表格前端搜索
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
          if (this.beSearchName !== '' && row.cusername.includes(this.beSearchName)) {
            rowBackground.background = '#F3E7A5'
            this.tableScrollMove('userList', rowIndex)
            return rowBackground
          }
        },
````

### 将查找到的行滚动居中
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
  <el-input v-model="beSearchName" style="margin-bottom: 10px;" placeholder="输入用户名"></el-input>
  <el-table
    ref="userList"
    :row-style="TableRowStyle"
    height="500"
    :data="userList">
    <el-table-column
      prop="cusername"
      label="姓名">
    </el-table-column>
    <el-table-column
      prop="username"
      label="Username">
    </el-table-column>
    <el-table-column
      prop="department_name"
      label="部门">
    </el-table-column>
    <el-table-column
      prop="operate"
      label="操作">
      <template slot-scope="scope">
        <el-button type="primary" size="mini" @click="deleteUser(scope.row.unique_id)">删除</el-button>
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
          userList: [],
          beSearchName: ''
        }
      },
      created () {
        this.beSearchName = ''
        this.userList = [{'department_name': 'Fishbowl研发组', 'cusername': '张洪波', 'unique_id': 'C100036', 'username': 'jacob.zhang'}, {'department_name': 'Fishbowl后端研发小组', 'cusername': '陈奎', 'unique_id': 'C100132', 'username': 'kui.chen'}, {'department_name': 'Fishbowl项目组', 'cusername': '陈俊洋', 'unique_id': 'C110186', 'username': 'jackie.chen'}, {'department_name': 'Fishbowl美术组', 'cusername': '张旖旎', 'unique_id': 'C120419', 'username': 'yini.zhang'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '张冰炎', 'unique_id': 'C120635', 'username': 'bingyan.zhang'}, {'department_name': 'Fishbowl美术组', 'cusername': '李昉', 'unique_id': 'C120647', 'username': 'fang.li'}, {'department_name': 'Ocean', 'cusername': '邹先童', 'unique_id': 'C130810', 'username': 'xiantong.zou'}, {'department_name': 'Fishbowl美术组', 'cusername': '董炽', 'unique_id': 'C130870', 'username': 'chi.dong'}, {'department_name': 'Fishbowl美术组', 'cusername': '樊柯', 'unique_id': 'C130933', 'username': 'ke.fan'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '刘亚男', 'unique_id': 'C130959', 'username': 'yanan.liu'}, {'department_name': 'Fishbowl测试小组', 'cusername': '王桃', 'unique_id': 'C130987', 'username': 'tao.wang'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '李博', 'unique_id': 'C130994', 'username': 'bo.li'}, {'department_name': 'Fishbowl美术组', 'cusername': '高若琪', 'unique_id': 'C141090', 'username': 'ruoqi.gao'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '曹凯', 'unique_id': 'C141106', 'username': 'kai.cao'}, {'department_name': 'Fishbowl美术组', 'cusername': '王佳南', 'unique_id': 'C141156', 'username': 'jianan.wang'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '李志昭', 'unique_id': 'C151379', 'username': 'zhizhao.li'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '金永来', 'unique_id': 'C151451', 'username': 'yonglai.jin'}, {'department_name': 'Fishbowl美术组', 'cusername': '马若梅', 'unique_id': 'C151463', 'username': 'ruomei.ma'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '杨蕾', 'unique_id': 'C151543', 'username': 'leilei.yang'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '陆冬', 'unique_id': 'C171827', 'username': 'dong.lu'}, {'department_name': 'Fishbowl测试小组', 'cusername': '陈华川', 'unique_id': 'C171853', 'username': 'huachuan.chen'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '刘世朋', 'unique_id': 'C181909', 'username': 'shipeng.liu'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '胡志武', 'unique_id': 'C181913', 'username': 'zhiwu.hu'}, {'department_name': 'Fishbowl测试小组', 'cusername': '杨天一', 'unique_id': 'C181967', 'username': 'persona.yang'}, {'department_name': 'Fishbowl项目组', 'cusername': '美花', 'unique_id': 'C182018', 'username': 'hua.mei'}, {'department_name': 'Fishbowl后端研发小组', 'cusername': '罗瑞森', 'unique_id': 'C192135', 'username': 'ruisen.luo'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '贺逸', 'unique_id': 'C192144', 'username': 'yi.he'}, {'department_name': 'Fishbowl后端研发小组', 'cusername': '封立伟', 'unique_id': 'C192232', 'username': 'liwei.feng'}, {'department_name': 'Fishbowl后端研发小组', 'cusername': '刘方东', 'unique_id': 'C192337', 'username': 'fangdong.liu'}, {'department_name': 'Fishbowl美术组', 'cusername': '李则岳', 'unique_id': 'C192368', 'username': 'zeyue.li'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '王若琬', 'unique_id': 'C202396', 'username': 'ruowan.wang'}, {'department_name': 'Fishbowl前端研发小组', 'cusername': '李栋凌', 'unique_id': 'C212927', 'username': 'dongling.li'}, {'department_name': 'Fishbowl测试小组', 'cusername': '郭轩', 'unique_id': 'C213006', 'username': 'xuan01.guo'}, {'department_name': 'Fishbowl游戏设计组', 'cusername': '吴征', 'unique_id': 'C213043', 'username': 'zheng.wu'}, {'department_name': 'Fishbowl测试小组', 'cusername': '杨昊', 'unique_id': 'C213074', 'username': 'hao.yang'}]
      },
      methods: {
        deleteUser (uniqueId) {
          this.removeUserListByValue(uniqueId)
          // this.$emit('listenUserListChange', this.userList)
        },
        removeUserListByValue (val) {
          for (let i = 0; i < this.userList.length; i++) {
            if (this.userList[i].unique_id === val) {
              this.userList.splice(i, 1)
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
          if (this.beSearchName !== '' && row.cusername.includes(this.beSearchName)) {
            rowBackground.background = '#F3E7A5'
            this.tableScrollMove('userList', rowIndex)
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

