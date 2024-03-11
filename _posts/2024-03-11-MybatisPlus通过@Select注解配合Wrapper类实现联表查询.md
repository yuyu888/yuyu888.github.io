---
layout: mypost
title: MybatisPlus通过@Select注解配合Wrapper类实现查询
categories: [JAVA]
---

在使用mybatisPlus时， 有时候我们需要一些复杂查询，比如联表查询，但是只使用QueryWrapper无法实现，但是又不想写在xml里面， 只想用sql去直观实现，这个时候我们可以使用@Select注解实现查询功能。

比如：
````java
    @Select("SELECT ed.* FROM expense_detail AS ed  LEFT JOIN expense AS e ON e.id=ed.expense_id  ${ew.customSqlSegment} and e.status not in (-4,-1,0)")
    List<HecFbiExpenseExpenseDetailEntity> getExpenseDetailIdsByExpenditureItemIds(@Param(Constants.WRAPPER) QueryWrapper<HecFbiExpenseExpenseDetailEntity> queryWrapper);
````

调用代码:
```` java
    @Override
    public List<Integer> getExpenseDetailIdsByExpenditureItemIds(HecFbiExpenseExpenseDetailPageForm pageForm) {
        if (CollectionUtils.isEmpty(pageForm.getExpenseIds()) && Objects.isNull(pageForm.getExpenditureItemIds())) {
            return Collections.emptyList();
        }
        QueryWrapper<HecFbiExpenseExpenseDetailEntity> queryWrapper = new QueryWrapper<HecFbiExpenseExpenseDetailEntity>();
        if(CollectionUtils.isNotEmpty(pageForm.getExpenseIds()) ){
            queryWrapper.in("ed.expense_id", pageForm.getExpenseIds());

        }
        if(CollectionUtils.isNotEmpty(pageForm.getExpenditureItemIds()) ){
            queryWrapper.in("ed.expenditure_item_id", pageForm.getExpenditureItemIds());
        }
        queryWrapper.gt("ed.invoice_count",0);
        List<HecFbiExpenseExpenseDetailEntity> list = baseMapper.getExpenseDetailIdsByExpenditureItemIds(queryWrapper);
        return CollectionUtils.isEmpty(list) ? Collections.emptyList() : CollUtil.map(list,HecFbiExpenseExpenseDetailEntity::getId,true);
    }
````

上述示例中会将代码${ew.customSqlSegment}替换成queryWrapper拼接的sql语句--- WHERE (ed.expenditure_item_id IN (?,?,?,?) AND ed.invoice_count > ?) 

其中  Constants.WRAPPER = "ew"  
所以 @Param(Constants.WRAPPER)  写成 @Param("ew") 也是可以的