---
layout: mypost
title: MybatisPlus通过@Select注解配合Wrapper类实现联表查询
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

注意 queryWrapper里如果使用了entity实例，entity的设置不起作用， 比如下面的例子中的        entity.setStatus(1); 是不会代入where 条件中的

````java
        HecFbiExpenseExpenseDetailEntity entity = new HecFbiExpenseExpenseDetailEntity();
        entity.setStatus(1);
        QueryWrapper<HecFbiExpenseExpenseDetailEntity> queryWrapper = new QueryWrapper(entity);
````

原因参看：com.baomidou.mybatisplus.core.conditions.Wrapper#getCustomSqlSegment
````java
    /**
     * 获取自定义SQL 简化自定义XML复杂情况
     * <p>使用方法</p>
     * <p>`自定义sql` + ${ew.customSqlSegment}</p>
     * <p>1.逻辑删除需要自己拼接条件 (之前自定义也同样)</p>
     * <p>2.不支持wrapper中附带实体的情况 (wrapper自带实体会更麻烦)</p>
     * <p>3.用法 ${ew.customSqlSegment} (不需要where标签包裹,切记!)</p>
     * <p>4.ew是wrapper定义别名,可自行替换</p>
     */
    public String getCustomSqlSegment() {
        MergeSegments expression = getExpression();
        if (Objects.nonNull(expression)) {
            NormalSegmentList normal = expression.getNormal();
            String sqlSegment = getSqlSegment();
            if (StringUtils.isNotEmpty(sqlSegment)) {
                if (normal.isEmpty()) {
                    return sqlSegment;
                } else {
                    return Constants.WHERE + StringPool.SPACE + sqlSegment;
                }
            }
        }
        return StringPool.EMPTY;
    }


````
