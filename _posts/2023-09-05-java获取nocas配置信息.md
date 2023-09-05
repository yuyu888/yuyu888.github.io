---
layout: mypost
title: java 获取nocas配置信息
categories: [JAVA]
---

直接上代码

````java

import com.alibaba.cloud.nacos.NacosConfigManager;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.nacos.api.config.ConfigService;
import com.xxxxx.infocenter.configservice.finportal.logic.FinportalNocasConfigLogic;
import com.xxxxx.infocenter.infrastructure.common.exception.RRException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Slf4j
@Service("finportalNocasConfigLogic")
public class FinportalNocasConfigLogicImpl implements FinportalNocasConfigLogic {

    private String FRONTEND_FINACE_GROUP = "FRONTEND_FINANCE_GROUP";

    @Autowired
    private NacosConfigManager nacosConfigManager;

    @Override
    public JSONObject frontJsonConfig(String fileName) {
        ConfigService nacosService = nacosConfigManager.getConfigService();
        try{
            String configString = nacosService.getConfig(fileName,FRONTEND_FINACE_GROUP,1000);
            return JSONObject.parseObject(configString);
        }catch (Exception e){
            log.info("=================config error=================");
            log.info(e.getMessage());
            throw new RRException("配置读取错误！");
        }
    }
}
````