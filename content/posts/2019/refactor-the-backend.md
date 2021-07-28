---
title: "代码重构 - 后端部分代码"
date: 2019-02-28
categories: ["重构", "设计模式"]
tags: ["重构", "设计模式"]
---

[前一篇]({{< ref "./refactor-the-frontend.md" >}})主要写了一下前端部分的重构，这一篇则主要关注后端部分。

在前一篇后面说到了一个很实用的套路（模式），其类图如下图所示：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016203814.png)

在后端部分，我先将后端Java代码的类图画出来：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016203838.png)

可以发现，还是一样的套路。

## 代码实现
下面则具体说下实现，首先是说一下原来的系统的实现，然后是重构的实现。

### 原来的实现
在上一篇前端部分，拿了两个方法作为示例，在方法体的后面，则都是通过`POST`请求调用后端的Controller。
```js
postVue("${ctx}/BusinessOpportunity/openAcct",params,function (data) {......
postVue("${ctx}/BusinessOpportunity/openAmlInviteAcct",params,function (data) {......

```
这里是原有的`Controller`的实现：
![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016203911.png)

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016203937.png)

然后则是`Service`的实现：

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016204019.png)

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016204039.png)


然后就是`Service`调用不同的`MyBatis`的`dao`层实现。

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016204102.png)

从实现上来说，就是一一相对应，优点是代码过程清晰明了，基本上完全反映实现的意图。缺点自然也很明显，比如大量重复的代码，并且也没有代码复用等。

### 重构过程
这部分这样描述下重构的部分。上一篇中提到了我对于重构的两个原则：
1. 尽可能的代码复用  
2. 为使用方提供一致的调用外观

所以这一部分，还是会基于这两个主要原则，并且主要针对Controller层和Service层。

由于是前后端分离，所以这部分是Restful风格的，WEB API作为前后端的契约，一是URL尽量的符合REST语义，二是传输的数据(payload)。

这个是REST Controller的实现：
```
@PostMapping("/api/opportunities/batch")
public ResponseEntity<?> batchProcess(@RequestBody OpportunityBatchRequest request) {
	log.info("Process opportunity batch action - type: {}", request.getType());
	List<String> emailSendFailedList;
	try {
		businessOpportunityProcessorService.process(request);
		emailSendFailedList = businessOpportunityProcessorService.getAllEmailSendFailed();
	} catch (Exception e) {
		log.error("Process opportunity batch action failed, type: {}, root cause: {}", request.getType(), e);
		return WebUtil.error(e.getMessage());
	}

	String message = "保存成功,以下企业发送通知失败，请检查：" + String.join(", ", emailSendFailedList);

	return new ResponseEntity<>(RestResponse.builder().status(HttpStatus.OK.value()).body(Boolean.TRUE).message(message).build(), HttpStatus.OK);
}
```

`/api/opportunities` 这部分其实是放`RestController`类注解上的，这里只是为了说明。
另外，按照REST语义，针对不同的资源的操作，是区分GET、POST、DELETE等动词的，但是在真实的项目环境下，其实很难严格按照REST语义的，所以这里针对批处理的方式，统一使用了POST。


这个是`RequestBody`的部分，作为数据(schema)，`type`表述执行什么操作，`data`则是本次操作的数据部分。
```
@Data
@ToString
public class OpportunityBatchRequest {
    private String type;
    private List<RequestData> data;

    @Data
    public static class RequestData {
        private String companyName;
        private String contactNumber;
        private Integer customerId;
        private String email;
        private String opportunityId;
    }
}
```
这部分就是针对批处理操作的Controller层，下面则是Service层，由于在上面已经给出了UML类图，所以这里主要是给出一些具体的实现：

首先是定义接口，用以规范行为，这边的泛型其实并没有实际的意义，直接就是`OpportunityBatchRequest `这个类
```
public interface OpportunityProcessor<T> {
    boolean supports(String type);
    void process(T request) throws ProcessException;
}
```
主要是两个方法：
- supports：用以在具体的实现类中表达支持哪种`type`的处理
- process：  业务处理方法

具体的子类的实现示例如下：
```
/**
 * 开通反洗钱
 */
@Service
@Slf4j
public class OpenAmlAccountProcessor extends AbstractOpportunityProcessor<OpportunityBatchRequest> {
    private static final String SUPPORTS_TYPE = "OpenAmlAccount";

    @Override
    public boolean supports(String type) {
        return SUPPORTS_TYPE.equalsIgnoreCase(type);
    }

    @Override
    public void process(OpportunityBatchRequest request) throws ProcessException {
        log.info("Process OpenAmlAccount action");

    }
}
```

```
/**
 * 开通反洗钱邀请
 */
@Service
@Slf4j
public class OpenAmlInviteAccountProcessor extends AbstractOpportunityProcessor<OpportunityBatchRequest> {
    private static final String SUPPORTS_TYPE = "OpenAmlInviteAccount";

    @Override
    public boolean supports(String type) {
        return SUPPORTS_TYPE.equalsIgnoreCase(type);
    }

    @Override
    public void process(OpportunityBatchRequest request) throws ProcessException {
        log.info("Process OpenAmlInviteAccount action");

    }
}
```

这里的抽象父类`AbstractOpportunityProcessor`，用以处理一些较为公共的逻辑等，比如注入公共组件，抽些公共的方法等。引入这个抽象类的性价比是很高的。

### 重构后的代码
这个例子中，有几个操作都是开通账号、邀请相关的，所以，涉及开通账号方法，就可以放在这个抽象父类中，而相应的子类，则只需要说明账户类型，就可以了：

```
/**
 * 开通反洗钱
 */
@Service
@Slf4j
public class OpenAmlAccountProcessor extends AbstractOpportunityProcessor<OpportunityBatchRequest> {
    private static final String SUPPORTS_TYPE = "OpenAmlAccount";
    private static final String ACCOUNT_OPEN_MODE = Constants.AcctOpenMode.AML;

    @Override
    public boolean supports(String type) {
        return SUPPORTS_TYPE.equalsIgnoreCase(type);
    }

    @Override
    public void process(OpportunityBatchRequest request) throws ProcessException {
        log.info("Process OpenAmlAccount action");

        for (OpportunityBatchRequest.RequestData requestData : request.getData()) {
            openAccount(requestData, ACCOUNT_OPEN_MODE);
        }
    }
}

/**
 * 开通反洗钱邀请
 */
@Service
@Slf4j
public class OpenAmlInviteAccountProcessor extends AbstractOpportunityProcessor<OpportunityBatchRequest> {
    private static final String SUPPORTS_TYPE = "OpenAmlInviteAccount";
    private static final String ACCOUNT_OPEN_MODE = Constants.AcctOpenMode.INVITE_AML;

    @Override
    public boolean supports(String type) {
        return SUPPORTS_TYPE.equalsIgnoreCase(type);
    }

    @Override
    public void process(OpportunityBatchRequest request) throws ProcessException {
        log.info("Process OpenAmlInviteAccount action");

        for (OpportunityBatchRequest.RequestData requestData : request.getData()) {
            openAccount(requestData, ACCOUNT_OPEN_MODE);
        }
    }
}
```



![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20201016204126.png)

而对于其他的操作，则只需要在具体自己的子类中实现，如删除:
```
/**
 * 批量删除
 */
@Service
@Slf4j
public class DeleteOpportunitiesProcessor extends AbstractOpportunityProcessor<OpportunityBatchRequest> {
    private static final String SUPPORTS_TYPE = "DeleteOpportunities";

    @Override
    public boolean supports(String type) {
        return SUPPORTS_TYPE.equalsIgnoreCase(type);
    }

    @Override
    public void process(OpportunityBatchRequest request) throws ProcessException {
        log.info("Process DeleteOpportunities action");

        List<String> opportunityIdList = request.getData()
                .stream()
                .map(OpportunityBatchRequest.RequestData::getOpportunityId)
                .collect(Collectors.toList());
        businessOpportunityMapper.deleteOpportunitiesInBatch(opportunityIdList);
    }
}
```

## 后记
这是本次重构的后端部分，相应的套路（模式）也更多一些，自己也相对的更加熟悉一些。这两篇博客提到的模式，在很多场景下都可以使用。

谢谢阅读。

**Works，then better.** 

*(2019-02-28 发表于简书)*