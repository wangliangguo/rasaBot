FAQ机器人是最简单的机器人，不具备多轮对话能力.
这部分以简单的表格处理为例，介绍rasa中基于槽位填充的多轮对话机器人．

假设这样一个场景：我们需要构建一个收集用户如下信息的机器人：

- 工作
- 姓名
- 邮箱
- 薪水
- 工作单位

这些信息都是在对话中填充的槽．

在上篇内容的基础上进行修改

- 修改domain文件全局变量
- action文件修改
- 修改训练数据
-  修改语言理解和对话管理的方法策略
- 训练与预测


## 1.修改domain文件全局变量

对于表格填充这个任务，需要增加两种意图：第一种是启动表格填充的意图，表示用户当前的消息触发了表格填充这个任务．第二个是槽信息获取的意图，表示用户当前的消息中包含了需要填充槽的信息．这两种意图按官方教程命名为contact_sales和inform.

这个例子中，还需要增加表格forms，实体entites,以及槽slots的定义．其中表格中定义了表格action中的name属性值．实体和槽分别定义了命名实体识别需要识别的实体名称和槽填充中需要填充的槽名．这两个部分通常有很大的重合．
在responses部分，还需要添加当启动表格填充后，询问槽值的方式．


修改后完整的domain文件如下：

	intents:
	  - greet
	  - goodbye
	  - faq
	  - contact_sales
	  - inform


	actions:
	   - respond_faq

	forms:
	  - sales_form

	entities:
	  - company
	  - job_function
	  - person_name
	  - budget
	  - business_email
	  - use_case

	slots:
	  company:
	    type: unfeaturized
	  job_function:
	    type: unfeaturized
	  person_name:
	    type: unfeaturized
	  budget:
	    type: unfeaturized
	  business_email:
	    type: unfeaturized
	  use_case:
	    type: unfeaturized


	responses:
		  utter_greet:
		  - text: "Hey! How are you?"
		  utter_goodbye:
		  - text: "Bye"
		  utter_ask_business_email:
		  - text: "What's your business email?"
		  utter_ask_company:
		  - text: "What company do you work for?"
		  utter_ask_budget:
		  - text: "What's your annual budget for conversational AI? 💸"
		  utter_ask_job_function:
		  - text: "What's your job? 🕴"
		  utter_ask_person_name:
		  - text: "What's your name?"
		  utter_ask_use_case:
		  - text: "What's your use case"

	session_config:
	  session_expiration_time: 60
	  carry_over_slots_to_new_session: true

## 2.action文件修改
对表格填充action需要实例化一个具体的类对其进行处理．这个类包括三个方法：
name方法，返回表单名称，该名称在domain全局变量中进行声明．
required_slots方法，该方法指定表单需要完成的填充槽．
submit方法，该方法定义当required_slots全部填充完之后执行的操作．

	class SalesForm(FormAction):
	    """Collects sales information and adds it to the spreadsheet"""

	    def name(self):
		return "sales_form"

	    @staticmethod
	    def required_slots(tracker):
		return [
		    "job_function",
		    "use_case",
		    "budget",
		    "person_name",
		    "company",
		    "business_email",
		]

	    def submit(
		    self,
		    dispatcher: CollectingDispatcher,
		    tracker: Tracker,
		    domain: Dict[Text, Any],
	    ) -> List[Dict]:
		dispatcher.utter_message("Thanks for getting in touch, we’ll contact you soon")
		return []

这个例子里面最后填充完表单后只是显示了信息，实际上可以在此定义任何操作，如数据库连接，调用其他方法等

## 3.修改训练数据

训练数据包括nlu和story两个部分．

对于nlu部分，增加 contact_sales和inform两种意图的训练数据．由于inform意图里面通常包含了用户提供的槽值信息，还需要使用命名实体识别方法提取出其中的实体，即作为槽值对定义的槽进行填充．在提供的nlu文件中对实体进行标注，使得模型能够同时训练一个命名实体识别模型．

	## intent:contact_sales
	- I wanna talk to your sales people.
	- I want to talk to your sales people
	- I want to speak with sales
	- Sales
	- Please schedule a sales call
	- Please connect me to someone from sales
	- I want to get in touch with your sales guys
	- I would like to talk to someone from your sales team
	- sales please
		
	## intent:inform
	- [100k](budget)
	- [100k](budget)
	- [240k/year](budget)
	- [150,000 USD](budget)
	- I work for [Rasa](company)
	- The name of the company is [ACME](company)
	- company: [Rasa Technologies](company)
	- it's a small company from the US, the name is [Hooli](company)
	- it's a tech company, [Rasa](company)
	- [ACME](company)
	- [Rasa Technologies](company)
	- [maxmeier@firma.de](business_email)
	- [bot-fan@bots.com](business_email)
	- [maxmeier@firma.de](business_email)
	- [bot-fan@bots.com](business_email)
	- [my email is email@rasa.com](business_email)
	- [engineer](job_function)
	- [brand manager](job_function)
	- [marketing](job_function)
	- [sales manager](job_function)
	- [growth manager](job_function)
	- [CTO](job_function)
	- [CEO](job_function)
	- [COO](job_function)
	- [John Doe](person_name)
	- [Jane Doe](person_name)
	- [Max Mustermann](person_name)
	- [Max Meier](person_name)
	- We plan to build a [sales bot](use_case) to increase our sales by 500%.
	- we plan to build a [sales bot](use_case) to increase our revenue by 100%.
	- a [insurance tool](use_case) that consults potential customers on the best life insurance to choose.
	- we're building a [conversational assistant](use_case) for our employees to book meeting rooms.
	
	
对于story的部分，只需要添加form启动的故事即可：

	## sales form
	* contact_sales
	    - sales_form                   <!--Run the sales_form action-->
	    - form{"name": "sales_form"}   <!--Activate the form-->
	    - form{"name": null}           <!--Deactivate the form-->

这个故事表明，当用户的输入被识别为contact_sales意图时，就启动sales_form　action,form{"name": "sales_form"} 表明启动的form名称为sales_form．最后当表格action完成以后，使用null关闭表格action．



##4.修改语言理解和对话管理的方法策略

最后对config文件中的policies部分进行修改，添加FormPolicy.

	policies:
	  - name: MemoizationPolicy
	  - name: KerasPolicy
	  - name: MappingPolicy
	  - name: FormPolicy
	  
修改endpoints.yml文件，注释action_endpoint启动action的服务：

	action_endpoint:
	 url: "http://localhost:5055/webhook"

## 5.　训练与预测

训练：

	rasa train
	
训练完以后，需要启动一个action服务：

　  rasa run action

然后进行预测：
　　
　 rasa shell

![](/home/wang/桌面/picTmp/2.png) 

可以看到，目前的bot虽然能完成任务，但是鲁棒性不强，容易出错，下一篇介绍一些提升鲁棒性的方法．
 
	  
	  