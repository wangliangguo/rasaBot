上文构建的基本款rasa上下文机器人服务不够鲁棒，出现解析错误的问题会报错．这部分通过三个方面对其进行改进提升．

- 处理意外输入

- 失败处理

- 更加复杂的上下文对话


## 1. 处理意外输入

如在表单处理过程中，用户没有按照表单的输入要求输入正确的槽值，或者rasa意图识别错误时会出现如下错误：
![](/home/wang/work/myproject/rasaBot/doc/pic/4.1.png) 

分成两种情况，第一种是处理表单过程中，用户发出了已定义的其他意图消息．第二种是用户发出了未定义意图的消息．

### 第一种情形

在表单处理过程中，用户发出了其他的意图消息．譬如该例中打招呼greet或者FAQ意图．这两种分别有个两种方案：

第一种方案，对某个意图不管上下文如何都是固定输出，使用Mapping Policy进行处理．对某个意图，它总是预测相同的action.并且返回从对话历史中移除掉以外输入交互的事件．之后继续表单的处理．

具体做法：

1. 在domain文件中actions下添加一个aciton_greet的action.将intents下的
greet修改为greet: {triggers: action_greet}．

2. 删除故事中的greet相关故事．

3. 在actions.py中增加action_greet的定义．

		from rasa_sdk import Action
		from rasa_sdk.events import UserUtteranceReverted

		class ActionGreetUser(Action):
		"""Revertible mapped action for utter_greet"""

		def name(self):
		    return "action_greet"

		def run(self, dispatcher, tracker, domain):
		    dispatcher.utter_template("utter_greet", tracker)
		    return [UserUtteranceReverted()]

之后重新训练与预测，不再报错，相当于rasa在表单处理过程中临时对用户的非表单意图进行了处理，并不做记录．
![](/home/wang/work/myproject/rasaBot/doc/pic/4.2.png) 



对FAQ的处理：

对于FAQ而言，处理方法是在故事中添加一个如下的故事：

	## just sales, continue
	* contact_sales
	    - sales_form
	    - form{"name": "sales_form"}
	* faq
	    - respond_faq
	    - sales_form
	    - form{"name": null}
	    
该故事即是表单处理过程中出现FAQ意图的处理方式：对FAQ进行回应，然后继续表单．

### 第二种情形

用户除了发出已定义的意图以外，还有可能在槽值填充中发出其他未定义的意图消息，譬如用户询问系统请求某个槽的原因．

这种情况需要提前进行设想，将用户的可能在表单处理过程中提出的问题提前进行定义并设置回复．

1. 修改domain全局文件．

添加explain意图，对用户针对某个槽的可能提问进行处理．

	intents:
	- greet: {triggers: action_greet_user}
	- bye
	- thank
	- faq
	- explain

槽的类型更改为categorical

	slots:
	  requested_slot:
	    type: categorical
	    values:
	      - business_email
	      - company
	      - person_name
	      - use_case
	      - budget
	      - job_function

添加解释的回复文本：

	responses:
	  utter_explain_why_budget:
	  - text: We need to know your budget to recommend a subscription
	  utter_explain_why_email:
	  - text: We need your email so we can contact you
	  
2. 添加训练数据
对于增加的explain意图，添加意图识别训练数据如下：

		## intent:explain
		- why
		- why is that
		- why do you need it
		- why do you need to know that?
		- could you explain why you need it?
		
在故事中，添加对话管理训练数据：


		## explain email
		* contact_sales
		    - sales_form
		    - form{"name": "sales_form"}
		    - slot{"requested_slot": "business_email"}
		* explain
		    - utter_explain_why_email
		    - sales_form
		    - form{"name": null}

		## explain budget
		* contact_sales
		    - sales_form
		    - form{"name": "sales_form"}
		    - slot{"requested_slot": "budget"}
		* explain
		    - utter_explain_why_budget
		    - sales_form
		    - form{"name": null}


##2. 失败处理

对了对话中出现的可能错误，主要有两种方式，第一种是意图识别中，所有定义的意图预测的概率都较低，这时候就算选择最高的意图可能也是错误的．譬如用户的输入本身就不在预定义意图之中．通过使用TwoStageFallbackPolicy,在configure文件中添加TwoStageFallbackPolicy．

		policies:
			- name: TwoStageFallbackPolicy
			   nlu_threshold:0.8
			   
同时在domain文件中添加out_of_scope意图．

		intents:
		 	- out_of_scope
	
上述处理之后，当用户的消息通过意图识别是，最高意图概率低于80%的时候会执行内置的out_of_scope意图操作．

也可以自定义out_of_scope对内置的处理进行替换．

在故事中添加：

	* out_of_scope
	  utter_out_of_scope

在nlu训练数据中添加

	## intent:out_of_scope
	- I want to order food
	- What is 2 + 2?
	- Who’s the US President?
	- I need a job

在domain文件中添加out_of_scope的文本：

	responses:
	  utter_out_of_scope:
	  - text: Sorry, I can’t handle that request.




## 3.更加复杂的上下文对话




		 	
		 	









