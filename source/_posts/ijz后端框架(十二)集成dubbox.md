---
title: ijz后端框架(十二)集成dubbox
date: 2019-02-18 22:05:51
categories: ijz后端框架
tags:
	- dubbox
---
dubbo官方已经提供了[dubbo-spring-boot-starter](https://github.com/alibaba/dubbo-spring-boot-starter)，只需引入该依赖，我们便能快速地在项目中使用dubbo。但是我们系统现在用的是dubbox，这个没有提供starter，（论技术选型的重要性，为什么要选择已经停止维护了的第三方库？）那就只好自己实现个了>_<**[dubbox-spring-boot-starter](https://git.yonyouccs.com/ijz-modules-jars/dubbox-spring-boot-starter)**。通过对dubbo-spring-boot-starter简单的封装，我们也能享受到starter带来的便利了！

没使用starter之前，我们通常要配置繁琐的dubbo.xml，以资金为例：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.2.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.2.xsd
		http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
	<description>Dubbo服务配置</description>

    <dubbo:application name="ijz-finance" owner="yonyou"/>
    <dubbo:protocol name="dubbo" port="${dubbo.protocol.port}" />
	<dubbo:provider threads="${dubbo.provider.threads}" retries="0" timeout="${dubbo.provider.timeout}" filter="logcontext" />
    <dubbo:consumer retries="0" timeout="${dubbo.provider.timeout}" filter="logcontext" check="false"/>
    <dubbo:registry protocol="zookeeper" address="${zookeeper.addr}" />
    <dubbo:monitor protocol="registry" />
    
    <!-- ======== Dobbo Consumer Begin ======== -->
    <!-- 附件上传 -->
    <dubbo:reference id="fsAttachService"
        interface="com.yyjz.icop.file.service.FsAttachService" url="${dubbo.url.file}" check="false"/>
    
    <!-- 单据编码 -->
    <dubbo:reference id="billCodeService"
        interface="com.yyjz.icop.support.api.service.IBillCodeService" url="${dubbo.url.support}" check="false"/>
        
    <!-- 获取参数dubbo接口 -->
    <dubbo:reference id="regConfigService"
        interface="com.yyjz.icop.support.api.service.IRegConfigAPIService"
        url="${dubbo.url.support}" check="false" timeout="10000" />
    
    <!--项目档案 -->
    <dubbo:reference id="projectAPIService"
        interface="com.yyjz.icop.share.api.service.ProjectAPIService" url="${dubbo.url.share}" check="false"/>
    
    <!--设备档案 -->
    <dubbo:reference id="deviceAPIService"
        interface="com.yyjz.icop.share.api.service.DeviceAPIService" url="${dubbo.url.share}" check="false"/>
        
    <!--物资档案 -->
    <dubbo:reference id="materialAPIService"
        interface="com.yyjz.icop.share.api.service.MaterialAPIService" url="${dubbo.url.share}" check="false"/>
    
    <!--供应商档案 -->
    <dubbo:reference id="supplierAPIService"
        interface="com.yyjz.icop.share.api.service.SupplierAPIService" url="${dubbo.url.share}" check="false"/>
    
    <!--客户档案 -->
    <dubbo:reference id="customApiService"
        interface="com.yyjz.icop.share.api.service.CustomApiService" url="${dubbo.url.share}" check="false"/>
    
    <!--自定义档案-->
    <dubbo:reference id="defdocAPIService"
        interface="com.yyjz.icop.share.api.service.DefdocAPIService" url="${dubbo.url.share}" check="false"/>
        
    <!--计量单位-->
    <dubbo:reference id="UnitApiService"
        interface="com.yyjz.icop.share.api.service.UnitApiService" url="${dubbo.url.share}" check="false"/>
    
    <dubbo:reference id="accountPeriodService"
        interface="com.yyjz.icop.share.api.service.AccountPeriodAPIService" url="${dubbo.url.share}" check="false"/>
        
    <dubbo:reference id="userService"
        interface="com.yyjz.icop.usercenter.service.IUserService" url="${dubbo.url.usercenter}" check="false" />
        
    <!--公司信息-->
    <dubbo:reference id="companyService"
        interface="com.yyjz.icop.orgcenter.company.service.ICompanyService" url="${dubbo.url.orgcenter}" />
        
    <!-- 组织中心 -->
    <dubbo:reference id="orgCenterService"
        interface="com.yyjz.icop.orgcenter.orgcenter.service.IOrgCenterService" url="${dubbo.url.orgcenter}" />
    
    <!-- 消息 -->
    <dubbo:reference id="pushMsgService"
        interface="com.yonyou.message.center.service.PushMsgService" url="${dubbo.url.message}" check="false"/>
    <dubbo:reference id="autoSendMsgService"
        interface="com.yonyou.message.center.service.AutoSendMsgService" url="${dubbo.url.message}" check="false"/>
        
    <!-- 预警service -->
    <dubbo:reference id="ewTaskParamApiService"
        interface="com.yyjz.icop.earlywarn.api.ewtask.service.IEwTaskParamApiService"
        url="${dubbo.url.icop-earlywarn}" timeout="10000" check="false"/>
    
    <!-- ijz-contract 公共合同 -->
    <dubbo:reference id="commonContractApiService"
        interface="com.yyjz.ijz.contract.api.cont.contractsimple.service.ICommonContractApiService"
        url="${dubbo.url.ijz-contract}" timeout="10000" check="false"/>

    <!-- ijz-contract 虚拟合同API服务 -->
    <dubbo:reference id="virtualContractApiService"
        interface="com.yyjz.ijz.contract.api.cont.contractsimple.service.IVirtualContractApiService"
        url="${dubbo.url.ijz-contract}" timeout="10000" check="false"/>
    
    <!-- ijz-contract 公共结算API服务 -->
    <dubbo:reference id="commonSettleApiService"
        interface="com.yyjz.ijz.contract.api.cont.contractsimple.service.ICommonSettleApiService"
        url="${dubbo.url.ijz-contract}" timeout="10000" check="false"/>
    
    <!-- ijz-tax [作废]发票开具 -->
    <dubbo:reference id="invoiceIssueAPIService"
        interface="com.yyjz.ijz.tax.api.taxsimple.saleinvoice.invoiceissue.service.IInvoiceIssueAPIService"
        url="${dubbo.url.ijz-tax}" timeout="10000" check="false"/>
    
    <!-- ijz-tax [作废]收票登记 -->
    <dubbo:reference id="inInvoiceBusiAPIService"
        interface="com.yyjz.ijz.tax.api.taxsimple.receiptinvoice.invoicecheckin.service.IInInvoiceBusiAPIService"
        url="${dubbo.url.ijz-tax}" timeout="10000" check="false"/>
    
    <!-- ijz-tax 发票开具/开票登记 -->
    <dubbo:reference id="issueInvoiceAPIService"
        interface="com.yyjz.ijz.tax.api.taxsimple.saleinvoice.issueinvoice.service.IIssueInvoiceAPIService"
        url="${dubbo.url.ijz-tax}" timeout="10000" check="false"/>
    
    <!-- ijz-tax 收票登记 -->
    <dubbo:reference id="receiptInvoiceAPIService"
        interface="com.yyjz.ijz.tax.api.taxsimple.receiptinvoice.receiptinvoice.service.IReceiptInvoiceAPIService"
        url="${dubbo.url.ijz-tax}" timeout="10000" check="false"/>
    
    <!-- icop-myproject 我的项目 -->
    <dubbo:reference id="myProjectQueryService"
        interface="com.yyjz.icop.myproject.api.service.IMyProjectQueryService"
        url="${dubbo.url.icop-myproject}" timeout="10000" check="false"/>
    
    <!-- ijz-budget 总预算 -->
    <dubbo:reference id="totalBudgetAPIService"
        interface="com.yyjz.ijz.budget.api.budgetsimple.totalbudget.service.ITotalBudgetAPIService"
        url="${dubbo.url.ijz-budget}" timeout="10000" check="false"/>
    <!-- ======== Dobbo Consumer _End_ ======== -->
    
    <!-- ======== Dobbo Provider Begin ======== -->
    <!-- 债权台账（ijz-contract 对外报量） -->
    <dubbo:service ref="finAcctInService"
        interface="com.yyjz.ijz.finance.api.finance.acct.service.IFinAcctInServiceAPI"/>

    <!-- 债务台账（分包结算） -->
    <dubbo:service ref="finAcctOutService"
        interface="com.yyjz.ijz.finance.api.finance.acct.service.IFinAcctOutServiceAPI"/>
    
    <!-- 资金公共API服务 -->
    <dubbo:service ref="financeCommonAPIServiceImpl"
        interface="com.yyjz.ijz.finance.api.common.service.IFinanceCommonAPIService"/>
    <!-- ======== Dobbo Provider _End_ ======== -->
</beans>
```
使用starter之后，如需要对外暴露服务，只需在service上添加注解@Service，类似这样：
```java
import com.alibaba.dubbo.config.annotation.Service;

@Service(interfaceClass = ActualCostApiService.class)
public class ActualCostServiceApiServiceImpl implements ActualCostApiService {
    ...
}
```
如需要引入外部服务，也只需在属性上添加注解@Reference，类似这样：
```java
import com.alibaba.dubbo.config.annotation.Service;

@Service(interfaceClass = ActualCostApiService.class)
public class ActualCostServiceApiServiceImpl implements ActualCostApiService {
    @Reference(url = "${dubbo.url.orgCenter}")
    private ICompanyService companyService;
    ...
}
```
再也不用配置xml了，就是这么简单！
