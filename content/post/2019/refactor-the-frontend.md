---
title: "代码重构 - 前端部分代码"
date: 2019-02-27
categories: ["重构","设计模式"]
tags: ["重构", "设计模式"]
---

## 缘起
由于工作的变动，转岗到了公司的另外一个项目里，目前的主要工作在编码方面，负责将一个原来标准的J2EE（Spring， SpringMVC，MyBatis)项目，重构成基于Restful的前后端分离的项目，后端采用Spring Boot，前端部分则采用Vue。

这里计划用两篇博客记录一下重构中的一些点，一篇为前端部分，一篇为后端部分，这篇为前端部分。
由于处在不同的职位上，所关心的内容是不同的，比如产品经理更加关心产品的功能、业务的完成度、项目经理更加关注项目的进度等，另外一方面，从代码层面来说，还是比较主观的，不同的开发人员，写出来的代码也会差别很大，所以这里仅是个人的重构记录。下面就闲话少说，“talk is cheap, show me the code”。

## 业务背景
这里首先交代下重构部分的业务，如下图所以，这是个比较典型的数据表格，展示了业务数据，以及相关的操作，查看详情，编辑，删除，并且可以多选，并且进行对多条数据进行批量的操作。

![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016202825.png)

下图是完成了的数据表格部分，隐藏了其中涉及到的业务数据

![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016202916.png)

其主要的功能为：
1. 表格右侧的“操作”部分，主要是针对单条记录的操作，如查看，修改，删除
2. 表格下方的批量操作部分，当选中多条记录之后，执行不同的批量操作，如批量发送邀请，批量创建账号，批量删除等。针对不同的操作，在执行相应的操作时候，需要进行不同的前端验证，如开通邀请，则需要验证所选择的记录的电话号码不能为空，邮箱不能为空，销帮帮ID是否存在，等。

## 代码实现
简单介绍完需求之后，逻辑并不复杂，从代码实现角度，主要就是如下几步：
1、响应按钮点击 
2、在响应的方法中遍历选中的数据
3、对数据进行校验，如何校验失败，则进行相应的提示
4、将数据提交到后端

实现起来也是相当的明了。

### 原来的实现
这部分主要描述下上述需求的原来的实现部分，这里不会贴出全部的代码，主要还是将意图表达出来。另外，这个项目的前端展示部分是基于JSP+jQuery+Vue的，所以原来页面部分逻辑较为复杂，一部分数据是传统的基于表单的，一部分数据是jQuery Ajax方式的，一部分数据则是Vue（axios）方式。

下面的三张截图，第一张是按钮部分，针对不同的功能，对应不同的响应方法:
![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016203036.png)

下面两张是其中两个方法的具体实现:

![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016203139.png)



![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016203208.png)

由于逻辑不复杂，所以实现上也是比较的明了。基本上完成过程式的代码实现，遍历选择的数据，对数据进行校验，然后调用后台对应的接口。

过程式的代码，优点是代码清晰明了，基本上完全反映实现的意图。缺点也比较的明显，大量的重复代码，拿贴出来的两段代码来看，基本上都是相同的，复制粘贴一个方法，然后稍微的改一改。这里不去写这些的弊端，重点还是放在重构部分的内容上。

### 重构过程
首先上面的代码，从功能角度来说，是可以工作的代码，但是，很多的重复代码。实际上很容易就可以发现其中可以重用的部分，如公司名称的校验，销帮帮ID的校验，邮箱的校验等。
那么，最简单的实现重用的方式，可以将其中的检验部分抽出来，形成一个一个单独的验证方法，如：
```
validateEmailXXX()
validateCompanyNameXXX()
```
这样在不同的方法中，就可以重用这些验证的逻辑，原来代码中的`isEmailAvailable()`方法就是工具方法层面的复用。比如说常见的代码中的很多的工具类，如`StringUtils`，`DateUtils`，就是这样的一个思路，实现了一些工具方法层面的重用。(对于Ruby，Kotlin，可以非常优雅的扩展父类方法)。

不过这部分重构，我并非仅仅是抽取了几个验证的方法。下面会描述，这里先说一下我考虑的两个原则：
1. 为使用方提供一致的调用外观
2. 尽可能的代码复用

#### 提供较为一致的调用外观
这部分主要是真的按钮的响应，所以这里统一了方法`handleOperation`的调用，然后通过`type`来区分。这部分看个人的习惯了，比如原来的方式，也是不错的，方法名就反映出来该方法的意图。
```
<el-button type="primary" :disabled="totalSelectedRows === 0" @click="handleOperation('openAmlAccount')"><span>开通反洗钱</span></el-button>
<el-button type="primary" :disabled="totalSelectedRows === 0" @click="handleOperation('openAmlInviteAccount')"><span>开通反洗钱邀请账号</span></el-button>
<el-button type="danger"  :disabled="totalSelectedRows === 0" @click="handleOperation('deleteOpportunities')"><span>批量删除</span></el-button>
<el-button type="primary" :disabled="totalSelectedRows === 0" @click="handleOperation('openNormalAccount')"><span>开通账号</span></el-button>
<el-button type="primary" :disabled="totalSelectedRows === 0" @click="handleOperation('openInviteAccount')"><span>开通邀请账号</span></el-button>
<el-button type="danger" :disabled="totalSelectedRows === 0" @click="handleOperation('physicallyDelete')"><span>删除账号</span></el-button>
<el-button type="danger" :disabled="totalSelectedRows === 0" @click="handleOperation('completelyPhysicalDelete')"><span>彻底删除</span></el-button>
```
`handleOperation`则根据不同的`type`，分别调用相对于的处理方法。
```
handleOperation(type) {
    console.log(type);
}
```

#### 尽可能的代码复用
针对代码复用的部分，这边则主要的就是那些验证的部分，验证公司名称不能为空，验证销帮帮ID不能为空等。我这里并没有采用工具方法，而且采用了更加面向对象的方式，比如验证公司名称，创建了相应的验证类`CompanyNameValidator.js`，实现如下：（我这里的名称其实并不是太好，因为没有表达出该验证类的意图 - 公司名称不能为空，`CompanyNameCannotBeNullValidator`会更加的合适一点）
```
/**
*  CompanyNameValidator.js
 * 公司名称不能为空
 */
var CompanyNameValidator = {
    validate: function (index, item) {
        if (!item.companyName) {
            return {
                msg: "第" + (index + 1) + "行数据没有公司名称",
                error: true,
            }
        } else {
            return {
                error: null
            }
        }
    }
};

export default CompanyNameValidator;
```
另外一个验证邮箱的实现，
```
/**
*  EmailValidator .js
 * 验证Email的格式
 */
var EmailValidator = {
    validate: function (index, item) {
        if (!isEmailValid(item.email)) {
            return {
                msg: "第" + (index + 1) + "行数据的【邮箱格式】有误，公司名称为：" + item.companyName,
                error: true
            }
        } else {
            return {
                error: null
            }
        }
    }
};

export default EmailValidator;
```

然后我可以统一导出这些验证器：
```
export {default as CompanyNameValidator} from './CompanyNameValidator'
export {default as XBBIdValidator} from './XBBIdValidator'
export {default as EmailValidator} from './EmailValidator'
export {default as PhoneNumberValidator} from './PhoneNumberValidator'
export {default as NormalOpportunityValidator} from './NormalOpportunityValidator'
```
这样，这些验证器就可以复用了:
```
import {CompanyNameValidator, XBBIdValidator, EmailValidator, PhoneNumberValidator, NormalOpportunityValidator} from './validators';
```
![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016203239.png)

如果需要增加新的功能，需要引入新的验证器，同样的增加一个，然后export导出就可以了。
验证器的使用：

```
handleOperation(type) {
	console.log(type);
	let selectedRows = this.selectedRows;
	let validators = [CompanyNameValidator];

	let errors = [];
	selectedRows.forEach((row, index) => {
		validators.forEach(validator => {
			let result = validator.validate(index, row);
			if (result.error != null) {
				errors.push(result);
			}
		});
	});

	if (errors.length > 0) {
		errors.forEach(error => {
			this.$message({
				type: 'info',
				message: error.msg
			});
		});
	} else {
		console.log('--------------------------------------')
	}
}
```
然后进一步的将验证器使用部分封装起来，
```
var ValidatorUtil = {
    validate: function(selectedRows, validators) {
        let errors = [];
        selectedRows.forEach((row, index) => {
            validators.forEach(validator => {
                let result = validator.validate(index, row);
                if (result.error != null) {
                    errors.push(result);
                }
            });
        });

        return errors;
    }
};

export default ValidatorUtil;
```
使用部分，只需要调用调用该工具：
```
import ValidatorUtil from './validators/ValidatorUtil';
....
ValidatorUtil.validate(selectedRows, validators);
```
所以使用部分，则进一步的简化了:
```
handleOperation(type) {
	......
	let selectedRows = this.selectedRows;
	let validators = [CompanyNameValidator, XBBIdValidator, EmailValidator];

	let errors = ValidatorUtil.validate(selectedRows, validators);

	if (errors.length > 0) {
		errors.forEach(error => {
			this.$message({
				type: 'info',
				message: error.msg
			});
		});
	} else {
		console.log('--------------------------------------')
	}
}
```
接下来就是不同的`type`的不同实现了，由于验证器部分已经可以达成复用，上述`handleOperation`已然是一个模板了，不同的方法，只要传入不同的`validators `数组就行了。

### 重构后的代码
根据上面的重构思路和描述的过程，贴出一部分重构后的代码：
```
handleOperation(type) {
	let selectedRows = this.selectedRows;

	switch (type) {
		case 'openAmlAccount':
			this.processOpenAmlAccount(selectedRows);
			break;
		case 'openAmlInviteAccount':
			this.processOpenAmlInviteAccount(selectedRows);
			break;
		case 'deleteOpportunities':
			this.processDeleteOpportunities(selectedRows);
			break;
		case 'openNormalAccount':
			this.processOpenNormalAccount(selectedRows);
			break;
		case 'openInviteAccount':
			this.processOpenInviteAccount(selectedRows);
			break;
		case 'physicallyDelete':
			this.processPhysicallyDelete(selectedRows);
			break;
		case 'completelyPhysicalDelete':
			this.processCompletelyPhysicalDelete(selectedRows);
			break;
	}
}
```

```
/**
 * 开通账号
 * @param selectedRows
 */
processOpenNormalAccount(selectedRows) {
	let validators = [CompanyNameValidator, XBBIdValidator, EmailValidator, PhoneNumberValidator];
	this.process('OpenNormalAccount', '开通账号', selectedRows, validators);
},

/**
 * 开通邀请账号
 * @param selectedRows
 */
processOpenInviteAccount(selectedRows) {
	let validators = [CompanyNameValidator, XBBIdValidator, PhoneNumberValidator];
	this.process('OpenInviteAccount', '开通邀请账号', selectedRows, validators);
},

/**
 * 删除账号
 * @param selectedRows
 */
processPhysicallyDelete(selectedRows) {
	let validators = [NormalOpportunityValidator];
	this.process('PhysicallyDelete', '删除账号', selectedRows, validators);
},


/**
 * 批量处理公共执行方法
 *
 * @param type                   批量操作类型
 * @param processMsg     操作说明，操作成功后提示信息
 * @param selectedRows  选中的表格行数据
 * @param validators        该操作涉及的验证器集合
 */
process(type, processMsg, selectedRows, validators) {
	let errors = ValidatorUtil.validate(selectedRows, validators);
	if (errors.length > 0) {
		errors.forEach(error => {
			this.$message({
				type: 'error',
				message: error.msg
			});
		});
	} else {
		let datas = [];
		selectedRows.forEach(row => {
			datas.push(
				{
					opportunityId: row.id,
					companyName: row.companyName,
					customerId: row.customerId,
					contactNumber: row.contactNumber,
					email: row.email
				}
			)
		});

		this.$axios.post('/api/opportunities/batch', {
			type: type,
			data: datas
		}, {}).then(res => {
			if (res.status === 200) {
				this.doSearch();
				this.$message({
					type: 'success',
					message: processMsg + '操作成功!'
				});
			}
		}).catch(error => {
			this.$message({
				type: 'error',
				message: processMsg + '操作失败! ' + error.msg
			});
		})
	}
}
```
## 后记
计划这一篇是前端部分，不过自己接触前端(Javascript, Vue)并没有多久，主要其实还是写后端代码，所以主要还是希望把意图表达清楚。下面则将视角稍微往上一点，首先从功能实现角度，都是工作的，所以这里也还是已技术为主要视角。

主要的套路(模式)见下面的UML类图，这是个很实用的套路，很多场景下都可以使用，下一篇将写一下后端代码(Java)部分的重构，其实也是基于这个套路。
![](https://raw.githubusercontent.com/cloudhuang/knowledge-base/master/pictures/20201016203304.png)

谢谢阅读。

**Works，then better.** 



*2019-02-27 发布于简*